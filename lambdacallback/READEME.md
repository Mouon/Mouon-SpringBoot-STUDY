# 전자위임장 대량 출력 결과 수신 방식 변경: API Callback과 토큰 인증

지난 글에서는 SQS 기반 큐를 이용해 비동기 작업 결과를 처리하는 구조를 소개했었다.

SQS를 이용한 구조는 아래와 같았다.

<img width="1238" height="336" alt="image" src="https://github.com/user-attachments/assets/bae9d7d5-c0cf-485c-916a-a74196878cd4" />

SQS를 사용했을때의 장점은 대략 다음과 같다.

큐에 메세지를 일정시간 보관하기 때문에, Spring 서버가 잠시 내려가 있어도 처리 실패 시 재시도 모델도 쉽게 구현할 수 있다.

또한 결과 전달을 HTTP callback에 직접 묶지 않아도 된다.

그런데 팀원들과 논의하면서 다른 관점이 나왔다.

전자위임장 출력이라는 기능은 1년 내내 일정하게 호출되는 기능이 아니라, 특정 시기에만 많이 사용되는 기능에 가깝다는 것이다.

사실, 전자 위임장 출력이라는 것은 정기주주총회가 많은 3월이 아니면 쓰이지않는 기능이다.

때문에 이 기능 하나를 위해 Spring 서버가 1년 내내 SQS를 polling하는 구조가 현재 스펙에 비해 과하다는 판단이었다.

그래서 다시 검토한 대안이 API callback 방식이었다.

## 1\. Lambda : InvocationType.EVENT

지난글에서도 말했던 것처럼, PDF 출력작업은 람다에서 실행되더라도 오래걸리는 작업이다.  
때문에 이를 동기방식으로 호출하면 스레드를 점유하고, 트랜잭션 또한 점유하는 리소스 낭비가 일어나게 된다.

때문에 Lambda 호출 자체는 여전히 비동기 호출로 유지한다.

```
InvokeRequest request = InvokeRequest.builder()
        .functionName(functionName)
        .invocationType(InvocationType.EVENT)
        .payload(...)
        .build();

lambdaClient.invoke(request);
```

즉 무거운 작업은 Lambda에서 수행하고, Spring 서버는 그 시간 동안 요청 처리 스레드와 트랜잭션을 붙잡고 있지 않는다.

여기까지는 SQS 구조와 동일하다.

달라진 것은 Lambda 결과를 다시 Spring 서버로 전달하는 방식이다.

## 2\. SQS 대신 API Callback

SQS를 제거하면 결과 전달 방식은 전보다 단순해진다.

Lambda가 작업을 끝낸 뒤 Spring 서버의 내부 API를 호출하면 된다.

<img width="1155" height="336" alt="image" src="https://github.com/user-attachments/assets/7c3c021b-4be0-4562-88d8-e3b288a2e958" />

Spring 서버 입장에서는 일반적인 Controller 방식으로 결과를 받을 수 있다.

```
@RestController
@RequestMapping("/system/jobs")
class SystemJobCallbackController {

    private final JobResultUseCase useCase;

    @CheckLambdaCallbackToken
    @PostMapping("/result")
    public NoContentResponse handleResult(@RequestBody JobResultRequest request) {
        useCase.handleResult(request.toCommand());
        return NoContentResponse.ok();
    }
}
```

결과 DTO도 필요한 정보만 담는다.

```
public record JobResultRequest(
        Long requestId,
        String fileName,
        Long fileSize,
        String objectKey,
        List<Long> successItemIds,
        List<FailedItem> failedItems,
        String errorMessage
) {
    public Command toCommand() {
        ...
    }
}
```

이제 별도의 SQS consumer는 필요하지 않다.

Spring 서버는 callback API를 통해 결과를 받고, 기존 use case 흐름으로 후속 처리를 수행하면된다.

## 3\. Lambda에는 Spring Security Context가 없다

callback 방식으로 바꾸면서 걸리는 문제가 있다. Lambda는 Spring 서버 내부 스레드가 아니다.

Spring 내부에서 `@Async`로 비동기 처리를 한다면 보안 컨텍스트 전달을 할 수 있지만, Lambda는 별도의 AWS 실행 환경에서 돈다.

때문에, Spring 서버가 Lambda를 `InvocationType.EVENT`로 호출하면 사실상 완전한 **fire-and-forget** 구조가 된다.

<img width="1280" height="390" alt="image" src="https://github.com/user-attachments/assets/8aaec510-e796-4dba-b4a9-65a8e0446ee9" />

그렇다고 callback API를 아무 인증 없이 열어둘 수는 없었다.

내부 비즈니스적인 검증이 복잡하긴하지만, 결과를 받아 상태를 변경하고, 필요한 후속 처리를 수행하는 command API기 때문이다.

## 4\. 커스텀 헤더 기반 인증

이번에는 Lambda와 Spring 서버가 서로 알고 있는 shared secret을 사용하기로 했다.

대략적인 플로우는 아래와 같다.

<img width="1280" height="263" alt="image" src="https://github.com/user-attachments/assets/3d5cecc5-40e0-4a50-9d4c-f157fb87b52b" />

Lambda는 callback 요청을 보낼 때 헤더에 토큰을 담는다.

```
Lambda-Callback-Token: <shared-secret>
```

헤더 이름은 기존 팀에서 커스텀 헤더를 사용하는 패턴을 이용했다.

```
public final class CustomHttpHeaders {

    public static final String TOKEN = "Token";

    private CustomHttpHeaders() {
    }
}
```

토큰 값은 코드에 하드코딩하지 않고 환경변수로 관리하도록했다.

```
cloud:
  aws:
    lambda:
      token: ${TOKEN:}
```

Lambda 역시 같은 값을 환경변수로 가진다.

```
TOKEN=...
```

그리고 callback 요청 시 이 값을 헤더에 넣어 보낸다.

```
request = urllib.request.Request(
    callback_url,
    data=body,
    method="POST",
    headers={
        "Content-Type": "application/json",
        "Token": token,
    },
)
```

Spring Security에서는 통과시키고, 그 다음 단계에서 callback 전용 token을 검증하는 방식으로 문제를 해결한 것이다.

## 5\. Lambda 재시도 정책 : 멱등성 보장하기

callback 방식으로 바꾸고 나서 또 하나 신경 써야 할 점이 있었다. 바로 람다의 재시도 정책이다.

Lambda 비동기 호출은 실패 시 재시도 정책을 가진다.

[AWS Lambda 비동기 invoke 재시도 문서](https://docs.aws.amazon.com/lambda/latest/dg/invocation-retries.html)를 보면, 비동기 호출에서는 함수 오류나 **throttling** 등에 따라 재시도가 발생할 수 있다.

즉 같은 `requestId`에 대한 callback이 한 번만 들어온다고 가정할 수 없었다.

정상적인 상황에서는 한 번만 들어올 수 있다.

하지만 운영 환경에서는 네트워크 문제, Lambda 재시도, 수동 재처리, 중복 요청 같은 일이 생길 수 있다.

그래서 callback 처리에는 멱등성을 고려해야 한다.

처음에는 단순하게 아래처럼 이미 `SUCCESS` 또는 `FAILED` 상태인 요청에 callback이 다시 들어오면 예외를 던지도록 했다.

```
if (!request.isProcessing()) throw new Exeption();
```

그러나 이방식은 실패 처리 로직에 의해 이미 성공한 요청을 실패 상태로 바꾸는 식의 로직상 결함이 있었다.

내부적으로 예외시 **catch**문을 통해 해당요청을 실패로 되돌리는 부분이 있었기 때문이다.

그래서 방향을 예외를 던지지않고 흘리는 방식으로 바꿨다.

```
if (!request.isProcessing()) {
    notifyDuplicatedCallback(command, request);
    return;
}
```

아렇게되면 이미 처리된 요청이라면 정상 처리 흐름으로 진입하지 않게된다.

대신 로그를 남기고, 운영자가 확인할 수 있도록 알림만 보낸다.

```
private void notifyDuplicatedCallback(Command command, Request request) {
    log.warn("Request is not processing. requestId={}", command.requestId());
    alert(...);
}
```

## 6\. 마무리하며

<img width="645" height="321" alt="image" src="https://github.com/user-attachments/assets/9d0753b7-2e37-4e07-9a62-c980f4467400" />

이번 기능 리팩토링은 우여곡절이 많았다. (정말 많은 버전의 기능이 있는 기능이었다.. 커밋이 50개가 넘어가는..)

처음에는 Spring 서버 내부에서 문제를 해결해보려 했다. CPU 사용량을 줄이기 위해 CPU사용 작업에 세마포어를 적용하고,

메모리 부담을 낮추기 위해 청크 단위 처리도 시도했다.

하지만 전자위임장 출력 기능은 피크 시점에 결국 많은 메모리를 사용했고, 심한 경우 서버가 OOM으로 내려가는 문제가 있었다.

이 장애가 같은 서버의 다른 기능에까지 전파되었었기 때문에, 렌더링 작업을 Spring 서버 밖으로 분리하기로 했다.

그 과정에서 Lambda를 도입했고, 이후 동기 호출로 인한 트랜잭션 점유 문제를 발견해 비동기 invoke 방식으로 전환했다.

비동기 결과를 받는 방식으로는 SQS와 API callback을 검토했고, 기능이 특정 시기에 집중적으로 사용되는 피크성 기능이라는 점을 고려해 최종적으로 API callback 방식을 선택했다.

물론 이 모든 것을 처음부터 완벽하게 고려하고 개발한 것은 아니었다.

구현 과정에서 문제를 발견하고, 팀 내부 논의를 통해 하나씩 방향을 조정하며 단계적으로 개선했다.

이번 작업을 하면서 기술적으로 더 강한 구조가 항상 현재 문제에 가장 적절한 구조는 아니라는 점을 다시 느꼈다.

기능의 성격과 운영 맥락을 보고 지금 필요한 만큼의 스펙을 선택하는 것도 중요한 개발자의 능력이라는 것을 배우게된 과정이었다.
