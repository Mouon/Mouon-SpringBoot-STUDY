## WebClient


최근 프로젝트에서 외부 API로 요청을 보내야 하는 작업을 진행하게 되었다.  
최초에는 RestTemplate을 통해 구현하였지만, 최근 스프링에서는 WebClient를 권장한다는 이야기를 듣고 RestTemplate을 WebClient로 교체하는 작업을 진행하였다.  

처음에는 단순하게 최신 기술을 도입한다는 생각으로 작업을 시작했지만, 적용해보니 여러 장점들을 발견할 수 있었다.  

내가 WebClient로의 전환을 하고 느낀 가장 큰 장점은 빌더 패턴을 활용할 수 있다는 점이었다.  
이 점이 가장 마음에 들었는데, 프로젝트에서 DTO 등에 빌더 패턴을 사용하고 있었고, 레포지토리도 복잡한 경우엔 QueryDSL을 활용하기 때문에 빌더 패턴을 사용 중이었다.  
WebClient를 이용함으로써 코드 스타일이 일관되게 정리된 것 같아 좋다고 느꼈다.  
물론 코드 패턴의 일관성만이 WebClient의 장점은 아니었다. RestTemplate을 사용할 때는 HTTP 요청을 구성하는 과정이 다소 번거롭고 구성 방식이 선언적이지 못해 구현에 어려움이 있었다(파라미터로 값을 받고, 예외 처리도 세부적으로 해줘야 했다).  

반면 WebClient는 빌더 패턴을 통해 요청을 구성할 수 있어 코드가 훨씬 간결해지고 가독성도 좋아졌다.  
개발 측면에서도 훨씬 안전한 개발을 할 수 있었고, 유지 보수성과 확장성도 좋아졌다.  

실제 코드를 비교해보면 그 차이가 확연히 드러난다. 먼저 RestTemplate을 사용할 때의 예이다.
  
```java
public void broadCastUploadDataResponse(long studyroomId, long memberId, UploadDataResponse response) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);

        Map<String, Object> body = new HashMap<>();
        body.put("studyroomId", studyroomId);
        body.put("memberId", memberId);
        body.put("event", "fileUploaded");
        body.put("data", response);

        HttpEntity<Map<String, Object>> request = new HttpEntity<>(body, headers);

        try {
            ResponseEntity<String> responseEntity = restTemplate.postForEntity(SOCKET_SERVER_URL, request, String.class);
            if (responseEntity.getStatusCode().is2xxSuccessful()) {
                log.info("Broadcast sent successfully");
            } else {
                log.error("Failed to send broadcast. Status code: " + responseEntity.getStatusCodeValue());
            }
        } catch (RestClientException e) {
            log.error("Error sending broadcast: " + e.getMessage());
        }
    }
```

같은 기능을 WebClient로 구현하면 다음과 같이 간결해진다. RestTemplate을 사용할 때와 Map으로 요청을 구성하는 부분은 같지만, 빌더 패턴으로 조립만 하면 되어서 유연한 구성이 가능해졌다.

또한 기본 URL, 기본 헤더 등을 쉽게 설정할 수 있어 재사용성이 높아졌다.  

```java
    public void broadCastUploadDataResponse(long studyroomId, long memberId, UploadDataResponse response) {
        WebClient webClient = WebClient.builder().baseUrl(SOCKET_SERVER_URL).build();

        Map<String, Object> body = new HashMap<>();
        body.put("studyroomId", studyroomId);
        body.put("memberId", memberId);
        body.put("event", "fileUploaded");
        body.put("data", response);

        webClient.post()
                .bodyValue(body)
                .retrieve()
                .bodyToMono(Void.class)
                .subscribe(
                        null,
                        e -> log.error("Error sending broadcast : {}", e.getMessage()),
                        () -> log.info("Broadcast sent successfully: studyroomId = {}", studyroomId)
                );
    }
```

### 더 나아가서..

WebClient에는 위에 언급한 점 이외에도 비동기 및 논블로킹 지원, Reactive 스택과의 통합, 대용량 데이터 스트리밍 지원 등의 더 큰 장점들이 있다고 한다.  
이러한 기능들은 현재 진행 중인 프로젝트에서는 필요하지 않았지만, 향후 프로젝트의 규모가 커지거나 요구사항이 변화할 때 유용할 것 같다.  
