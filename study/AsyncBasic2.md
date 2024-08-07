# @Async 기본 2탄
## @Async를 통해 알아보는 Blocking ,Non-Blocking 과 Synchronous , Asynchronous


**@Async를 공부하며** Blocking 과 Non-Blocking 그리고 Synchronous 와 Asynchronous의 개념을 
더 공부해보고 싶어졌다.

서버 애플리케이션의 성능을 최적화하고 반응성을 높이기 위해 블로킹(Blocking), 
논블로킹(Non-Blocking), 동기(Synchronous), 비동기(Asynchronous) 개념을 이해하는 것은 매우 중요하다.

이 개념들은 단일 스레드에서도 중요하지만, 멀티스레드 환경에서는 특히 더 중요하다!

먼저 각 용어의 의미를 살펴보겠다.

- **Blocking**  
  어떤 작업이 완료될 때까지 현재 스레드가 대기하는 상태이다. 이 동안 스레드는 아무 일도 하지 않고 작업이 끝나기만을 기다린다.  
- **Non-Blocking**  
  작업이 완료되지 않더라도 현재 스레드는 계속해서 다른 작업을 수행할 수 있다. 작업 결과는 나중에 처리된다.
- **Synchronous**  
  작업을 요청한 후 결과를 받을 때까지 기다리며, 그 동안 다른 작업을 하지 않는다. 호출자는 작업이 완료된 후에야 다음 작업을 수행할 수 있다.
- **Asynchronous**  
  작업을 요청한 후 결과를 기다리지 않고 다른 작업을 계속 수행할 수 있다. 결과는 나중에 콜백(callback)이나 이벤트를 통해 전달된다.

## 이 개념이 왜 스프링부트로 개발하는데 필요할까?
스프링부트와 같은 프레임워크에서는 기본적으로 멀티스레드 환경을 사용한다. 
특히, 스프링부트의 서블릿 디스패처는 요청을 처리하기 위해 여러 스레드를 할당하여 병렬로 작업을 수행할 수 있다. 
이러한 환경에서 블로킹, 논블로킹, 동기 및 비동기 개념을 이해하고 적절히 활용하는 것은 
애플리케이션의 성능과 반응성을 최적화하는 데 매우 중요하다.

### 서블릿 디스패처와 멀티스레드 환경

스프링부트의 서블릿 디스패처는 각 HTTP 요청을 처리하기 위해 별도의 스레드를 할당한다.  
하지만, 멀티스레드 환경에서 블로킹 작업이 많아지면 문제가 발생할 수 있다.

### 블로킹 작업의 문제점
<img width="1021" alt="스크린샷 2024-08-07 오후 3 06 59" src="https://github.com/user-attachments/assets/27139975-1561-4c2e-b5b6-3f499105b37d">  

- **블록킹** 작업은 하나의 스레드를 오래 점유하게 되어 스레드 풀의 크기가 한정된 환경에서는 성능 저하를 초래할 수 있다.
특히 Spring MVC모델은 다소 복잡하고 특정한 상호작용에서 일반적으로 Blocking 방식으로 처리된다.  
에를 들어 데이터를 수정하거나 가져오는 데이터베이스 호출하는 경우 요청이 Blocking 방식으로 처리되는데 
때문에 다른 요청을 처리할 스레드가 부족해질 수 있다.

### 논블로킹 작업의 장점

- **논블로킹** 작업은 스레드가 작업을 기다리지 않고 다른 작업을 할 수 있게 하여, 
스레드 풀의 효율성을 높인다. 예를 들어, 네트워크 I/O를 논블로킹 방식으로 처리하면 스레드는 
작업을 즉시 반환하고 다른 요청을 처리할 수 있다. 
주로 작업 완료 후 콜백을 통해 결과를 처리하는 방식으로 구현된다.

### 동기와 비동기 방식의 선택

- **동기** 방식은 이해하기 쉽고 구현하기 간단하지만, 자원을 비효율적으로 사용할 수 있다. 
모든 작업이 순차적으로 이루어지므로, 하나의 작업이 완료될 때까지 다른 작업이 대기해야 한다.
- **비동기** 방식은 복잡성을 증가시킬 수 있지만, 자원을 효율적으로 사용할 수 있어 대규모 시스템에서 성능 향상을 기대할 수 있다. 
비동기 방식에서는 작업을 요청한 후 결과를 기다리지 않고 다른 작업을 수행할 수 있으며, 작업 완료 후 콜백이나 이벤트를 통해 결과를 처리한다.

### 스프링부트에서의 적용 예시

스프링부트에서 @Async 애노테이션을 사용하면 손쉽게 비동기 메소드를 구현할 수 있다. 
이를 통해 블로킹 작업을 비동기적으로 처리하여 애플리케이션의 반응성을 향상시킬 수 있다. 
그러나, 비동기 프로그래밍의 복잡성을 잘 이해하고 적절히 다루는 것이 중요하다. 

#### 이제 예제들을 살펴보며 각 개념들이 어떻게 적용되는지 살펴보자!

먼저 이전 글에서도 보았겠지만 아래는 @Async애노테이션을 통해 비동기적으로 실행되는 메소드를 정의한 것이다.

```java
@Service
public class MouonService {
    @Async
    public CompletableFuture<String> asyncMethod() {
        return CompletableFuture.completedFuture("비동기 작업");
    }
}
```

이제 컨트롤러 계층에서 이 메소드를 호출하는 각기 다른 방식을 살펴보겠다.

먼저 **Non-blocking** & **Asynchronous 조합이다.**

```java
@RestController
public class MouonController {

    @Autowired
    private MouonService service;

    @GetMapping("/async")
    public CompletableFuture<String> asyncCall() throws Exception {
        return service.asyncMethod()
                .thenApply(result -> {
                    return result;
                });
    }
}
```

이 컨트롤러 메소드는 service.asyncMethod()를 호출하여 비동기 작업을 수행한다. thenApply 콜백 메소드를 사용하여 비동기 작업이 완료된 후 결과를 처리한다.

이때 주 스레드는 asyncMethod() 호출 후 즉시 스레드 풀로 반환되어 다른 작업을 수행할 수 있으므로 Non-blocking & Asynchronous이다.

다음은 **Blocking** & **Asynchronous 조합이다.**

```java
@RestController
public class MouonController {

    @Autowired
    private MouonService service;

    @GetMapping("/async")
    public String asyncCall() throws Exception {
        CompletableFuture<String> future = service.asyncMethod();
        return future.get(); // 주 스레드가 결과를 기다린다.
    }
}
```

이 컨트롤러 메소드는 service.asyncMethod()를 호출한 후 future.get()을 사용하여 결과를 기다린다.

이 경우 주 스레드는 비동기 작업이 완료될 때까지 블록킹되므로 Blocking & Asynchronous 이다.

Blocking & Asynchronous는 비동기 작업을 수행하면서도 결과를 기다리기 때문에, 호출한 스레드는 블록킹된다.

다른 작업을 동시에 수행할 수 있는 비동기적 접근의 장점을 누리지 못하기 때문에, 이런 설계는 단순 비동기메서드 호출 작업에서는 되도록 
피하는 것이 좋다.

## 마무리  
비동기 프로그래밍은 서버 애플리케이션의 성능을 향상시킬 수 있는 강력한 도구이다. 
그러나 잘못된 설계는 오히려 복잡성을 증가시키고 문제를 일으킬 수 있다. 
또한 무분별한 비동기 프로그래밍은 오히려 스레드 풀 관리의 복잡성을 증가시켜 심각한 장애를 야기하며, 
유지 보수의 어려움 줄 수 있다!
따라서 각 개념을 잘 이해하고 적절하게 사용하는 것이 중요하다.
