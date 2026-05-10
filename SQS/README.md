# Lambda invoke를 기다리지 않기: 전자위임장 출력 비동기화 개선기

지난 글에서는 전자위임장 출력 기능을 Lambda로 분리한 과정을 정리했다.

기존에는 Spring 서버가 직접 PDF 템플릿을 읽고, 값을 채우고, 서명과 신분증 이미지를 삽입하고, 여러 PDF를 병합했다.

이 과정에서 CPU와 메모리 사용량이 크게 증가했고, 실제로 Datadog에서 CPU 알람이 발생하거나 메모리 부족으로 서버가 위험해지는 상황도 있었다.

그래서 무거운 PDF 렌더링 작업을 Lambda로 분리했다.

Spring 서버는 비즈니스 데이터를 정리하고, Lambda는 실제 파일 렌더링만 담당하도록 역할을 나누었다.

여기까지 보면 꽤 괜찮은 구조처럼 보인다.

하지만 막상 운영 관점에서 다시 바라보니, 아직 찝찝한 부분이 하나 남아 있었다.

Lambda로 무거운 작업을 옮기긴 했는데, Spring 서버는 여전히 Lambda가 끝날 때까지 기다리고 있었다.

무거운 작업을 밖으로 빼긴 했는데, 기다리는 건 그대로였던 것이다.

## 1. 첫 번째 구조의 한계

처음 Lambda를 붙였을 때는 가장 단순한 방식으로 구현했다.

Spring 서버가 출력 요청에 필요한 데이터를 정리한 뒤 `lambdaClient.invoke(...)`를 호출하고, Lambda가 작업을 끝내면 결과를 응답으로 받는 구조였다.

흐름은 대략 다음과 같았다.

<img width="1280" height="347" alt="image" src="https://github.com/user-attachments/assets/c9631b39-fe9e-4cab-8b00-87e445eb22bf" />


코드로 보면 이런 느낌이다.

```
PdfRenderLambdaResponse response = pdfTemplateEditorPort.render(request);

Long fileId = internalVoteFileServicePort.registerPrivateFile(
        response.fileName(),
        response.fileSize(),
        response.objectKey()
);

ProxyPrintRequest completed = printRequestPersistencePort.update(
        processing.updateComplete(fileId)
);
```

이 구조의 장점은 명확했다.

Lambda의 응답을 바로 받을 수 있으니 흐름이 단순했다.

성공하면 파일을 등록하고 완료 처리하면 되고, 실패하면 실패 상태로 바꾸면 된다.

처음 구현할 때는 이게 가장 자연스러워 보였다.

그런데 문제는 이 호출이 동기 방식이라는 점이었다.

Spring 서버는 Lambda가 PDF 렌더링을 끝낼 때까지 계속 기다린다.

즉 PDF 렌더링 자체는 Lambda에서 수행하더라도, Spring 서버의 스레드는 여전히 그 작업이 끝날 때까지 점유된다.

더 큰 문제는 이 코드가 트랜잭션 안에서 실행되고 있었다는 점이다.

```
@Transactional
public void process(Long requestId) {
    ...
    PdfRenderLambdaResponse response = pdfTemplateEditorPort.render(request);
    ...
}
```

이렇게 되면 Lambda가 실행되는 동안 Spring 서버는 단순히 스레드만 잡고 있는 것이 아니다.

트랜잭션도 같이 열린 상태로 유지된다.

DB 커넥션, 영속성 컨텍스트, 트랜잭션 경계가 Lambda 응답을 기다리는 시간만큼 같이 묶여 있게 된다.

생각해보면 꽤 이상한 구조다.

우리는 무거운 PDF 렌더링을 Spring 서버 밖으로 빼기 위해 Lambda를 도입했다.

그런데 정작 Spring 서버는 그 Lambda가 끝날 때까지 가만히 기다리고 있다.

일은 남에게 맡겼는데 퇴근은 못 하는 느낌이다.

## 2. 진짜로 분리하려면 기다리지 않아야 했다

이 문제를 해결하기위해서 해당 기능의 비즈니스 요구사항을 다시한번 살펴볼 필요가 있었다.

이 작업은 사용자에게 즉시 PDF 파일을 내려주는 요청이 아니다.

전자위임장 대량 출력은 백그라운드 작업이고, 사용자는 요청을 생성한 뒤 나중에 완료된 결과를 확인하면 된다.

그렇다면 Spring 서버가 Lambda의 결과를 즉시 기다릴 필요가 없었다.

Spring 서버는 Lambda에게 작업을 맡기고, 요청 상태를 `PROCESSING`으로 바꾼 뒤, 일단 자신의 일을 끝내면 된다.

즉 구조를 `request-response`에서 `fire and forget` 형태로 바꾸는 것이 맞았다.

AWS Lambda invoke에서 이를 위해 `InvocationType.EVENT`를 사용하였다.

```
InvokeRequest invokeRequest = InvokeRequest.builder()
        .functionName(functionName)
        .invocationType(InvocationType.EVENT)
        .payload(SdkBytes.fromUtf8String(payload))
        .build();

lambdaClient.invoke(invokeRequest);
```

`InvocationType.EVENT`로 호출하면 Lambda는 비동기로 실행된다.

Spring 서버 입장에서는 Lambda가 요청을 정상적으로 받았는지만 확인하고 바로 흐름을 끝낼 수 있다.

성공시엔 HTTP **202 Accepted** 응답을 준다.

<img width="1280" height="193" alt="image" src="https://github.com/user-attachments/assets/47d085a3-c0d7-49ec-be87-1b444353dca3" />

이렇게 바꾸면 흐름은 다음처럼 달라진다.

<img width="1280" height="296" alt="image" src="https://github.com/user-attachments/assets/b8882f75-9e06-491e-ab24-260c003ff2df" />

이제 Spring 서버는 더 이상 PDF 렌더링 시간을 기다리지 않는다.

스레드도 오래 점유하지 않고, 트랜잭션도 렌더링 시간만큼 길게 유지되지 않는다.

이게 이번 개선의 가장 핵심이었다.

Lambda로 작업을 옮기는 것만으로는 부족했고, 호출 방식까지 비동기로 바꿔야 진짜 분리가 되었다.

## 3. 그런데 결과는 어떻게 받을 것인가

하지만 여기서 새로운 문제가 생긴다.

비동기 invoke를 사용하면 Spring 서버는 Lambda의 return 값을 받을 수 없다.

기존의 동기 호출에서는 Lambda가 이런 응답을 내려줄 수 있었다.

```
public record PdfRenderLambdaResponse(
        boolean success,
        String fileName,
        Long fileSize,
        String objectKey,
        String errorMessage
) {}
```

Spring 서버는 이 응답을 보고 파일을 등록하거나, 요청을 실패 처리하면 됐다.

하지만 `InvocationType.EVENT`로 바꾸면 이 응답 흐름이 사라진다. (payload가 null로 내려오기때문이다.)

Lambda는 혼자 실행되고, Spring 서버는 이미 자기 일을 끝낸 상태다.

그렇다면 Lambda가 렌더링을 끝냈을 때, 그 결과를 어딘가에 남겨야 한다.

그리고 Spring 서버는 그 결과를 다시 읽어서 후속 처리를 해야 한다.

여기서 선택지가 몇 가지 있었다.

첫 번째는 **Lambda가 Spring API를 직접 호출**하는 방식이다.

<img width="1280" height="223" alt="image" src="https://github.com/user-attachments/assets/0e6a61fb-ae23-4533-b742-302e91116756" />

겉으로 보면 간단해 보인다.

Lambda가 작업을 끝낸 뒤 `/proxy-print/render-result` 같은 API를 호출하면 된다.

하지만 이 방식은 조금 과했다.

우선 내부 작업 결과를 받기 위해 API를 새로 열어야 한다.

인증 토큰 문제도 생긴다.

Lambda가 Spring API를 호출하려면 토큰을 어떻게 발급하고, 어디에 저장하고, 어떻게 갱신할지 고민해야 한다.

무엇보다 이 기능은 외부 클라이언트가 호출하는 공개 API가 아니다.

완전히 **내부 백그라운드 작업**이다.

그런데 람다의 결과를 받는 용으로 API 를 만드는 것은 자연스러운 개발이 아니라고 생각했다, API 로 만들어지는 순간, 토큰 관리 등의 보안 문제를 신경써야한다.

이미 람다를 인증기반의 SDK로 호출하고 있던 터라, 같은 과정인데 다른 보안 요구사항이 생긴다는 것은 좋지않아 보였다.

두 번째 선택지는 **결과를 이벤트로 남기는 방식**이었다.

이미 우리 서버는 내부적으로 이벤트 기반 구조를 많이 사용하고 있었다. 심지어, 전자위임장 출력 작업자체도 그런 메커니즘으로 작동한다.

어떤 작업이 끝나면 이벤트를 발행하고, 그 이벤트를 소비하는 쪽에서 후속 처리를 수행하는 방식이다.

그렇다면 Lambda의 렌더링 결과도 하나의 이벤트로 바라볼 수 있었다.

“렌더링이 완료되었다.”

“렌더링에 실패했다.”

이 사실을 어딘가에 남겨두고, Spring 서버가 그것을 소비하면 된다.

그래서 **SQS**를 사용하기로 했다.

## 4. Lambda 결과를 SQS에 남기기

<img width="256" height="309" alt="image" src="https://github.com/user-attachments/assets/1438f7fd-0784-448b-8dd8-05bd24ac6390" />

변경된 구조는 다음과 같다.

<img width="1280" height="260" alt="image" src="https://github.com/user-attachments/assets/01d1e8df-a493-4b76-a5e4-6b7ea1ab17ac" />

이제 Lambda는 Spring 서버에 직접 API를 호출하지 않는다.

대신 렌더링 결과를 SQS 메시지로 남긴다.

Spring 서버는 해당 큐를 구독하고 있다가 메시지를 소비한다.

메시지에는 후속 처리에 필요한 정보만 담는다.

```
public record ProxyPrintResultSqsMessage(
        Long requestId,
        String fileName,
        Long fileSize,
        String objectKey,
        List<Long> successItemIds,
        List<FailedItem> failedItems,
        String errorMessag
) {}
```

Spring 서버는 이 메시지를 받아 기존 상태 관리 로직을 수행한다.

```
@SqsListener("${cloud.aws.sqs.proxy-print-result.queue-name}")
public void consume(ProxyPrintResultSqsMessage message) {
    log.info("Received proxy print result message. requestId={}", message.requestId());
    proxyPrintUseCase.handleRenderResult(message.toCommand());
}
```

이렇게 하니 책임이 더 명확해졌다.

Lambda는 파일을 만든다.

결과를 SQS에 남긴다.

Spring 서버는 결과 이벤트를 소비해서 비즈니스 상태를 변경한다.

즉 Lambda가 Spring 서버의 내부 API를 알 필요가 없다.

Spring 서버도 Lambda의 실행 시간을 기다릴 필요가 없다.

## 5. SQS가 또 병목이 되지는 않을까?

<img width="201" height="251" alt="image" src="https://github.com/user-attachments/assets/dc862b37-8aee-4f08-87f4-b7249d8e4b65" />

여기서 또 하나의 의문이 생겼다.

“_SQS는 큐 아닌가?_”

만약 Lambda 결과 메시지가 많이 쌓였는데 Spring 서버가 큐에서 pop하면서 SQS 메시지를 하나씩만 처리한다면,

이번에는 SQS가 새로운 병목이 될 수도 있다.

특히 **FIFO 큐**를 사용하고, 하나의 message group으로만 메시지를 넣는다면 병렬성이 크게 제한될 수 있다.

이러면 구조는 비동기지만, 결과 처리 단계에서 다시 대기를 하게 된다.

따라서 이 부분을 해결할 수 있는지 알아보았다.

SQS 자체는 여러 consumer가 동시에 메시지를 소비할 수 있다.

Standard Queue를 사용하면 메시지 순서를 강하게 보장하지 않는 대신, 여러 **consumer가 병렬로 메시지를 가져가 처리할 수 있다.**

즉 SQS를 쓴다고 해서 무조건 하나씩 처리되는 것은 아니다.

중요한 것은 Spring 쪽 listener container가 몇 개의 메시지를 동시에 처리할 수 있도록 설정되어야 한다는 것이다.

Spring Cloud AWS의 `@SqsListener`는 내부적으로 listener container를 사용한다.

이 container의 동시 처리 수를 조정하면, Spring 서버가 SQS 메시지를 병렬로 소비할 수 있게된다.

```
@Configuration
public class SqsListenerConfig {

    @Bean
    public SqsMessageListenerContainerFactory<Object> proxyPrintResultSqsListenerFactory(
            SqsAsyncClient sqsAsyncClient
    ) {
        SqsMessageListenerContainerFactory<Object> factory = new SqsMessageListenerContainerFactory<>();
        factory.setSqsAsyncClient(sqsAsyncClient);
        factory.configure(options -> options
                .maxConcurrentMessages(20)
                .maxMessagesPerPoll(10)
                .pollTimeout(Duration.ofSeconds(10)));
        return factory;
    }
}
```

그리고 해당 consumer에서 이 factory를 사용하도록 지정한다.

```
@SqsListener(
        value = "${cloud.aws.sqs.proxy-print-result.queue-name}",
        factory = "proxyPrintResultSqsListenerFactory"
)
public void consume(ProxyPrintResultSqsMessage message) {
    log.info("Received proxy print result message. requestId={}", message.requestId());
    proxyPrintUseCase.handleRenderResult(message.toCommand());
}
```

여기서 중요한 설정은 `maxConcurrentMessages`다.

이 값은 listener container가 동시에 처리할 수 있는 메시지 수를 의미한다.

`maxMessagesPerPoll`은 한 번의 polling에서 가져올 수 있는 메시지 수다.

즉 결과 메시지가 여러 개 쌓여 있을 때, Spring 서버가 메시지를 하나씩만 처리하지 않고 여러 메시지를 동시에 처리할 수 있게 된다.

물론 이 값을 무작정 크게 잡으면 안 된다.

메시지 하나를 처리할 때 DB 업데이트, 파일 등록, 이벤트 발행 같은 작업이 수행된다.

따라서 DB 커넥션 풀 크기, 트랜잭션 처리 시간, 외부 호출 여부를 함께 고려해야 한다.

이번 작업에서는 렌더링 자체는 이미 Lambda에서 끝난 뒤이고, Spring 서버는 결과 상태만 반영하면 되기 때문에 적당한 수준의 병렬 소비가 가능하다고 판단했다.

## 6. 상태 관리는 여전히 Spring 서버가 담당한다

SQS를 붙였다고 해서 상태 관리 책임까지 Lambda로 넘어가는 것은 아니다.

이 점은 이전 구조와 동일하게 유지했다.

Lambda는 렌더링 실행기다.

PDF를 만들고, S3에 업로드하고, 결과 메시지를 남긴다.

출력 요청이 성공인지 실패인지 확정하고, 파일을 등록하고, 후속 이벤트를 발행하는 주체는 여전히 Spring 서버다.

```
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void handleRenderResult(ProxyPrintRenderResultCommand command) {
    ProxyPrintRequest request = printRequestPersistencePort.findById(command.requestId())
            .orElseThrow(() -> new CustomException(ResponseCode.NOT_FOUND));

    if (request.getStatus() == ProxyPrintRequestStatus.SUCCESS
            || request.getStatus() == ProxyPrintRequestStatus.FAILED) {
        log.info("Proxy print result already handled. requestId={}, status={}",
                request.getId(), request.getStatus());
        return;
    }

    ...
}
```

결과 메시지 처리는 별도의 트랜잭션으로 수행한다.

기존 `process()` 트랜잭션 안에서 Lambda 결과를 기다리던 구조와 다르게, 이제는 결과 메시지를 소비하는 시점에 필요한 만큼만 트랜잭션을 연다.

이 차이는 꽤 중요하다.

이전 구조에서는 하나의 트래잭션으로 모든 작업이 묶여있었기 때문에, 렌더링이 끝날 때까지 트랜잭션이 길게 유지될 수 있었다.

그러나 변경 후에는 Lambda 호출 트랜잭션과 결과 처리 트랜잭션이 분리된다.

Spring 서버는 요청을 `PROCESSING(진행중중)`으로 바꾸고 Lambda를 비동기로 호출한 뒤 종료한다.

나중에 SQS 메시지가 도착하면 그때 `handleRenderResult()`에서 새로운 트랜잭션을 열고 상태를 갱신한다.

<img width="1280" height="304" alt="image" src="https://github.com/user-attachments/assets/8cab287d-462a-446e-b2cd-f155f66e9cb0" />

이렇게 나누면 트랜잭션의 생명주기가 훨씬 짧고 명확해진다.

무거운 렌더링 작업을 기다리기 위해 DB 리소스를 붙잡고 있을 필요가 없다.

## 7. 변경 후 구조

최종 구조는 다음과 같다.

<img width="1280" height="609" alt="image" src="https://github.com/user-attachments/assets/2ba44ba1-97d9-4f9d-817f-86dff868d625" />

이전 구조와 비교하면 가장 큰 차이는 동기식 대기가 사라졌다는 것이다.

기존에는 Spring 서버가 Lambda 호출 이후 결과를 기다렸다.

변경 후에는 Spring 서버가 Lambda 실행을 요청하고 바로 빠진다.

그리고 결과는 SQS 이벤트를 통해 나중에 처리한다.

## 8. 마무리하며

전자 위임장 출력 작업은 총 세 번의 개선 과정을 거쳤다.

-   docx 기반 렌더링에서 PDF 기반 렌더링 구조로 전환
-   무거운 렌더링 작업을 메인 서버에서 Lambda 서버로 분리
-   Lambda 서버의 응답을 SQS 기반 비동기 방식으로 구독하도록 개선

그 결과, 메인 서버의 부담은 크게 줄었고 전체 작업 흐름도 더 가볍고 안정적으로 개선할 수 있었다.

물론 이러한 구조를 처음부터 모두 예상하고 바로 적용했던 것은 아니다.  
여러 차례의 백엔드 미팅과 코드 리뷰를 통해 문제를 인지하고 원인을 파악했으며,

팀원들과 다양한 방향성을 논의하는 과정 속에서 조금씩 현재 구조를 만들어 나갔다.

이 과정에서 Lambda를 invoke 방식으로 호출한다는 것의 의미와 장점을 이해하게 되었고,  
나아가 Lambda를 비동기로 호출하는 방법, 그리고 그 비동기 응답을 메인 서버에서 안정적으로 수신하는 방식까지 자연스럽게 학습할 수 있었다.

특히 이번 작업에서는 기능의 빠른 적용과 개선을 위해 처음으로 Codex(GPT-5.5)를 적극적으로 활용해보았다.  
AI를 통해 전체적인 작업 방향과 구조를 빠르게 검토하고, 이후 내가 팀의 스타일과 실제 서비스 환경에 맞게 보완하고 조정하는 방식으로 업무를 진행했다.

채널톡의 Perrys님이 말씀하신 것처럼,

> “AI 시대의 개발자는 문제를 정의하고, AI의 응답이 적절한지 판단하는 역량이 중요하다.”

라는 말에 깊이 공감하게 된 작업이었다.
