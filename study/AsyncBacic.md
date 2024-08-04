# 스프링에서 비동기 처리해보기

프로젝트를 진행하며 파일 업로드 API를 구현할 일이 생겼다.  
파일 업로드는 I/O작업이기 때문에 스레드를 블로킹하여 서버 성능을 저하할 우려가있다고 판단하여  
비동기로 파일 업로드를 진행하게 하도록 하였다. ( 사실 개인적인 공부 욕심이 조금 더 컸다 ㅎㅎ )

스프링부트에선 @Async 애노테이션을통해 메서드를 비동기로 처리할 수 있게 해준다.  
해당 애노테이션을 사용하기 위해선 @EnableAsync을 활성화 해야한다.  
@EnableAsync 애노테이션이 활성화되어 있으면, 스프링은 @Async 애노테이션이 있는 메서드를 가진 빈에 대해 프록시를 생성하게된다.  

아래처럼 사용하면 된다.
```java
@Service
public class MouonService {
    @Async
    public CompletableFuture<String> asyncMethod() {
        return CompletableFuture.completedFuture("비동기 작업");
    }
}
```

조금 구체적으로 작동 매커니즘을 살펴본다면  
클라이언트가 @Async 메서드를 호출할 때, 실제로는 프록시 객체의 메서드를 호출하게 된다. 
이때 프록시 객체는 호출을 가로채서 비동기 실행을 처리한다.
이렇게되면 비동기 작업이 새로운 스레드에서 실행되며, 새로운 스레드에서 
실제 메서드 로직이 수행되게 되는 것이다.  
(참고 : 프록시 객체가 메서드를 가로채기 위해서는 @Async 애노테이션을 사용하는 메서드는 private일 수 없다. 
private 메서드는 상속되지 않고 외부에서 접근할 수 없기 때문에 프록시 객체가 이를 가로챌 수 없기때문이다.)

### 조금은 이해가 되지않는 점이 있다...

#### 클라이언트가 @Async 메서드를 호출할 때 프록시 객체가 호출을 다로챈다는게 정확이 어떤거고 무슨 이유에서일까?

우선 첫번째 이유는 프록시 객체가 메서드 호출을 가로채서, 
새로운 스레드에서 비동기적으로 메서드를 실행하도록 처리하기 위함이다.
따라서 호출한 스레드가 아닌 별도의 스레드에서 메서드를 실행함으로써 비동기 처리가 이루어지게 된다.

### 그렇다면 @Async 메서드를 호출한 스레드는 반환되는 것인가?  
  
때에 따라 다르다!

#### 주 스레드가 반환될때
CompletableFuture의 thenApply와 같은 콜백 메서드를 사용하면 결과 처리는 
비동기 작업을 수행한 스레드 또는 별도로 지정된 스레드에서 이루어진다. 
다시말해서, 
호출한 주 스레드는 즉시 다음 작업을 수행할 수 있도록 반환되고, 
프록시 객체는 TaskExecutor를 통해 새로운 스레드에서 실제 메서드를 실행한다!
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
#### 주 스레드에서 작업을 마무리할때

주 스레드가 get()이나 join() 등의 메서들통해 결과를 기다리게 되면,
결과가 준비될 때까지 블록킹된다. 
이 경우 결과 처리는 주 스레드에서 이루어진다.


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

### 주 스레드에서 작업을 기다리면 비동기의 이점을 못 누리는거 아닌가?

맞는말이다. 단순하게 join()이나 get()을 사용해서 
비동기 작업의 결과를 기다리면 
비동기 이점을 충분히 사용하지 못하는 셈이다.

하지만 다음과같이 여러 비동기 메서드를 호출할때는 비동기의 이점을 살리면서
코드도 직관적이게 작성할 수 있다.

```java

@Service
public class MouonService {
    @Async
    public CompletableFuture<String> asyncMethod1() {
        return CompletableFuture.completedFuture("비동기 작업 1");
    }

    @Async
    public CompletableFuture<String> asyncMethod2() {
        return CompletableFuture.completedFuture("비동기 작업 2");
    }
}

@RestController
public class MouonController {

    @Autowired
    private MouonService service;

    @GetMapping("/async")
    public String asyncCall() throws Exception {
        CompletableFuture<String> future1 = service.asyncMethod1();
        CompletableFuture<String> future2 = service.asyncMethod2();
        
        CompletableFuture.allOf(future1, future2).join();

        List<String> results = Arrays.asList(future1.get(), future2.get());

        return results; 
    }
}

```

주 스레드를 반환하는 콜백 방식과 주 스레드에서 작업의 마무리를 하는 방식은 각각의 장단점이 있으니 유연하게
구현하면 될 것같다.


지금까지 아주 간단하게 @Async에 대해 알아보았다!
