# 별거 없습니다. 프록시가 내 메서드를 무시하는 이유에 대해서.

## 들어가며
@Transactional 을 붙였는데 트랜잭션이 안 먹는다.
로그도 없고, 예외도 안 나고,
알고 보니 메서드에 public을 안 붙였다.

"CGLIB이면 protected도 프록시 되지 않나?"
"JDK 프록시는 인터페이스니까 그렇다 쳐도, CGLIB은 왜?"

스프링은 왜 굳이 모든 프록시 방식에서 public만 프록시 대상으로 제한했을까?
이번 글에서는 이 프록시의 비밀을 알아보려한다.


## private 메서드는 @Transactional 을 붙일 수 없다.

<img width="1280" height="307" alt="image" src="https://github.com/user-attachments/assets/6ace248d-02c0-4be7-8d75-fe2e3e0ae26f" />


인텔리제이에서 private 메서드에  @Transactional 을 선언하면 위와 같이 컴파일 에러를 띄워준다. protected 로 선언을 하라는건데..
우선 이 이유를 알아보자면 스프링 aop에서 프록시는 JDK Dynamic proxy또는 CGLIB으로 작동한다.

여기서 CGLIB은 동적으로 상속을 통해 프록시를 생성합니다. 따라서 private 메서드는 상속이 불가능하기 때문에 프록시로 만들어지지 않는것이다!


### protected는 됨?
그러면 저기 인텔리제이가 제안한대로 protected 로 작성하면 정상적으로 트랜잭션이 동작할까?
결론을 말하자면 protected 일때 또한 정상 작동하지 않는다!

"상속인데 protected가 왜안되지?????"

이유는 JDK Dynamic proxy 이녀석 때문입니다. JDK Dynamic proxy는 인터페이스를 기반으로 동작합니다. 그렇기 때문에 protected 메서드에서는 프록시가 동작할 수 없습니다.

"로그보니까 CGLIB 인거 같던데?"

맞습니다. 그런데 문제가 있습니다. CGLIB, JDK Dynamic proxy 방식의 차이때문에 AOP가 일관적이지 못하게 작동할 수 있습니다. 그렇기 때문에 스프링은 일관된 AOP적용을 위해서 protected로 선언된 메서드 또한 트랜잭션이 걸리지 않도록 하였습니다.
즉 프록시 설정에 따라서 트랜잭션이 적용됬다가 안됬다가~ 하는걸 방지하고자 하는것이죠
그래서 스프링은 AOP 대상을 public 으로 제한해두었습니다.



그런데 public 이라고 언제나 트랜잭션이 작동하는 것은 아닙니다.

<img width="210" height="210" alt="image" src="https://github.com/user-attachments/assets/81d6d1b1-1ba0-4e11-bc90-5bd0fb75ddcd" />

## public 메서드인데 트랜잭션이 작동을 안한다..?

```java

public class Test {
    public void test() {
        this.tranTest();
    }

    @Transactional
    public void tranTest() {
    }
}

```

위와 같은 구조에서는 tranTest 가 정상적으로 작동하지 못합니다. 왜일까요?


이건 Spring AOP에서 프록시의 동작 과정을 살펴보아야 합니다.
Spring AOP는 프록시를 통해 들어오는 외부 메서드 호출을 인터셉트 하여 작동합니다.
그렇기 때문에 예제처럼 내부호출을 통해 @Transactional 이 붙은 메서드를 호출하게되면, self-invocation이 라고 불리는 현상이 발생합니다. 쉽게 말하면 프록시내부에서 호출이 발생했다는 것입니다.

내부에서 호출을 하게되면 프록시가 인터셉트하지 못해서 트랜잭션이 동작하지 않습니다.



사실은.. 조금 더 딥한 수준의 기술로 내부호출 시에도 AOP적용을 할 수 있습니다. 제 수준에서는 아직 딥해서 간단하게 알아보자면 @Resource, @Autowired, @Inject 와 같은 어노테이션으로 외부에서 호출하도록 하거나.. AopContext 를 이용하면 된다던데.. 좀 더 공부해보도록 하겠습니다.

## 마무리!
스프링 AOP에서 프록시는 2가지 방식으로 작동한다!

스프링 aop에서 CGLIB은 클래스 자체를 상속받아 프록시를 생성함→ private은 상속불가라 프록시 생성 못함
JDK Dynamic proxy 는 인터페이스를 기반으로 동작함
그러나! 이렇게 프록시 설정따라 달라지면 일관성 없으니까 스프링이 public만 허용하도록 상황 정리!
