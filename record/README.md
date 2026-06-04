<img width="300" height="300" alt="매트 얼빡" src="https://github.com/user-attachments/assets/baff53fd-cdab-4643-8a91-d5a09b250710" />

*일까???????*

<br><br><br><br><br><br><br><br><br><br>

# Java Record란 무엇일까?

Java에서 DTO 같은 데이터 객체를 만들다 보면 반복되는 코드가 꽤 많다.

필드를 선언하고, 생성자를 만들고, getter를 만들고, `equals()`, `hashCode()`, `toString()`까지 만든다.

물론 IntelliJ 같은 IDE가 요즘은 잘 되어 있어 자동으로 만들어주기도 하고, Lombok 라이브러리를 사용하면 이런 반복 코드를 많이 줄일 수 있다.

그러나 이런 환경에서도 개발하다 보면 반복되는 코드 때문에, 이 클래스가 어떤 의도로 만들어졌는지 코드만 보고 명확하게 드러나지 않는다고 느낄 때가 있다.

아래 예시를 함께 살펴보자.

```java
public class UserResponse {

    private final Long id;
    private final String name;
    private final String email;

    public UserResponse(Long id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getEmail() {
        return email;
    }
}
```

위 클래스는 단순히 데이터를 전달하기 위한 객체처럼 보인다.

하지만 일반 클래스는 상태와 동작을 모두 가질 수 있고, 상속될 수도 있으며, 내부 구현을 숨기고 외부 API만 노출할 수도 있다.

즉, 일반 클래스만 보고는 이 객체가 **“단순 데이터 전달 객체”**인지, **“도메인 로직을 가진 객체”**인지, **“변경 가능한 상태를 가진 객체”**인지 명확하게 알기 어렵다.

이런 배경에서 Java는 Record를 도입했다.

---

## Record란 무엇일까?

Record는 Java 14에서 preview 기능으로 등장했고, Java 16에서 정식 기능으로 추가된 클래스 유형이다.

Record는 주로 DTO, API 응답 객체, 조회 결과 객체처럼 데이터를 담고 전달하는 객체를 간결하게 정의하기 위해 사용된다.

예를 들어 위의 `UserResponse` 클래스를 Record로 작성하면 다음과 같다.

```java
public record UserResponse(
    Long id,
    String name,
    String email
) {
}
```

이 코드만으로 Java는 다음과 같은 것들을 자동으로 만들어준다.

```java
public UserResponse(Long id, String name, String email)

public Long id()
public String name()
public String email()

public boolean equals(Object o)
public int hashCode()
public String toString()
```

여기서 특징적인 점은 Record의 accessor가 JavaBean 스타일의 `getId()`가 아니라는 것이다.

```java
UserResponse user = new UserResponse(1L, "현준", "test@example.com");

user.id();
user.name();
user.email();
```

Record는 `getId()` 같은 getter를 자동으로 만들어주지는 않는다.  
대신 Record component의 이름과 같은 메서드를 만들어준다.

이 부분에서 Record가 일반적인 JavaBean 객체와는 조금 다른 철학을 가진다는 것을 알 수 있다.

---

## Record는 왜 등장했을까?

내가 이해한 Record의 핵심은 단순히 보일러플레이트 코드를 줄이는 것에만 국한되지는 않는다.

물론 코드는 확실하게 줄어든다.  
생성자, getter, `equals()`, `hashCode()`, `toString()`을 직접 작성하지 않아도 된다.

하지만 Record의 진짜 의미는 **“이 클래스는 데이터를 담기 위한 클래스”**라는 의도를 Java 언어 차원에서 표현할 수 있다는 데 있다고 생각한다.

기존에는 단순한 데이터 객체를 만들 때도 일반 클래스를 사용해야 했다.

```java
public class Point {

    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int x() {
        return x;
    }

    public int y() {
        return y;
    }
}
```

이 클래스가 정말 단순히 좌표 데이터를 담기 위한 객체인지, 앞으로 여러 동작을 포함할 도메인 객체인지, 상속을 고려한 클래스인지 명확하게 드러나지 않는다.

반면 Record는 선언 자체가 의도를 드러낸다.

```java
public record Point(int x, int y) {
}
```

이 코드를 보는 순간 우리는 이 객체가 `x`와 `y`라는 데이터를 담는 객체라는 것을 어느 정도 알 수 있다.

즉, Record는 코드를 짧게 만드는 문법이기도 하지만, 더 본질적으로는 데이터 모델링을 명확하게 만들어주는 문법이다.

---

## Project Amber와 Record

Record는 Project Amber의 결과물이다.

Java는 오랫동안 굉장히 명시적인 언어였다.

좋게 말하면 안정적이고 예측 가능하지만, 나쁘게 말하면 단순한 코드를 작성할 때도 많은 관례적인 작업을 해야 했다.

특히 데이터 객체를 만들 때 이런 문제가 자주 드러났다.

단순히 이름과 나이를 담는 객체를 만들고 싶은데, 일반 클래스로 작성하면 코드가 길어진다.

```java
public class User {

    private final String name;
    private final Long age;

    public User(String name, Long age) {
        this.name = name;
        this.age = age;
    }

    public String name() {
        return name;
    }

    public Long age() {
        return age;
    }

    @Override
    public String toString() {
        return "User[name=" + name + ", age=" + age + "]";
    }
}
```

하지만 Record를 사용하면 다음처럼 표현할 수 있다.

```java
public record User(
    String name,
    Long age
) {
}
```

이렇게 하면 해당 클래스가 데이터를 담기 위한 클래스라는 점을 명확하게 표현할 수 있다.

---

## Record와 불변성

Record 클래스의 컴포넌트는 내부적으로 `private final` 필드가 된다.

즉, 한 번 생성된 Record 인스턴스의 필드는 다시 할당할 수 없다.

```java
public record User(
    String name,
    Long age
) {
}
```

위 Record는 개념적으로 아래 코드처럼 동작한다.

```java
public final class User extends Record {

    private final String name;
    private final Long age;

    public User(String name, Long age) {
        this.name = name;
        this.age = age;
    }

    public String name() {
        return name;
    }

    public Long age() {
        return age;
    }

    ...
}
```

Record는 생성 이후 필드를 변경할 수 없다.

그래서 API 응답 객체나 이벤트 메시지처럼 값이 중간에 바뀌면 안 되는 객체에 잘 어울린다.

```java
User user = new User("문코딩", 30L);

// user.name = "오타니"; // 컴파일 에러
```

다만 여기서 주의할 점이 있다.

Record의 불변성은 **얕은 불변성**이다.

예를 들어 다음과 같은 Record가 있다고 해보자.

```java
import java.util.List;

public record Team(List<String> members) {
}
```

Record의 `members` 필드 자체는 다른 리스트로 바꿀 수 없다.  
하지만 `members`가 가리키는 리스트 객체가 mutable하다면 리스트 내부 값은 변경될 수 있다.

```java
import java.util.ArrayList;
import java.util.List;

public class RecordExample {

    public static void main(String[] args) {
        List<String> members = new ArrayList<>();
        members.add("A");

        Team team = new Team(members);

        members.add("B");

        System.out.println(team.members()); // [A, B]
    }
}
```

이 경우 Record라고 해서 완전한 불변 객체가 되는 것은 아니다.

따라서 참조 타입을 Record component로 사용할 때는 필요하다면 방어적 복사를 해야 한다.

```java
import java.util.List;

public record Team(
    List<String> members
) {

    public Team {
        members = List.copyOf(members);
    }
}
```

이렇게 하면 외부에서 전달된 mutable list의 영향을 줄일 수 있다.

Record는 불변 객체를 만들기 쉽게 도와주지만, 내부 객체까지 알아서 깊은 불변으로 만들어주지는 않는다.

이건 Record를 사용할 때 꼭 알고 있어야 하는 부분이다.

---

## Compact Constructor

Record를 사용하면 생성자를 직접 정의할 수도 있다.

특히 Record에는 `compact constructor`라는 문법이 있다.

```java
public record Range(
    int start,
    int end
) {

    public Range {
        if (start > end) {
            throw new IllegalArgumentException("start는 end보다 클 수 없습니다.");
        }
    }
}
```

여기서 특징은 `this.start = start` 같은 대입 코드가 없다는 것이다.

일반 클래스라면 아래 예시와 같이 생성자에서 필드에 값을 직접 할당해야 한다.

```java
public class Range {

    private final int start;
    private final int end;

    public Range(int start, int end) {
        if (start > end) {
            throw new IllegalArgumentException("start는 end보다 클 수 없습니다.");
        }

        this.start = start;
        this.end = end;
    }

    public int start() {
        return start;
    }

    public int end() {
        return end;
    }
}
```

하지만 Record의 compact constructor에서는 검증 로직만 작성하면 된다.  
필드 대입은 Java가 알아서 처리한다.

즉, compact constructor는 Record의 값을 검증하거나 정규화할 때 사용하기 좋다.

```java
public record User(
    String name,
    Long age
) {

    public User {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("이름은 비어 있을 수 없습니다.");
        }

        if (age == null || age < 0) {
            throw new IllegalArgumentException("나이는 null이거나 음수일 수 없습니다.");
        }
    }
}
```

Record가 단순히 아무 로직도 넣을 수 없는 데이터 덩어리는 아니라는 점이 여기서 드러난다.

Record도 생성자 검증을 할 수 있고, 일반 메서드도 정의할 수 있다.

```java
public record User(
    String name,
    Long age
) {

    public String getFullInformation() {
        return "Name: " + name + ", Age: " + age;
    }
}
```

다만 Record는 어디까지나 데이터를 표현하기 위한 타입이다.

따라서 너무 많은 상태 변경 로직이나 복잡한 도메인 행위를 넣기 시작하면 Record를 사용하는 의미가 흐려진다.  
그럴 거라면 일반 클래스를 사용하는 것이 더 자연스럽다.

---

## Record와 명시적 타이핑

Record를 이해할 때 Java의 명시적 타이핑도 중요하다.

명시적 타이핑은 타입의 구조가 아니라 이름을 기준으로 타입을 구분하는 방식이다.

예를 들어 다음 두 Record가 있다고 해보자.

```java
public record UserId(Long value) {
}

public record ProductId(Long value) {
}
```

두 Record는 내부 구조가 완전히 같다.

둘 다 `Long value` 하나만 가지고 있다.

하지만 Java에서는 이 둘을 서로 다른 타입으로 본다.

```java
UserId userId = new UserId(1L);
ProductId productId = new ProductId(1L);

// userId = productId; // 컴파일 에러
```

이것이 명시적 타이핑이다.

구조가 같다고 해서 같은 타입이 되는 것이 아니라, 이름이 다르면 다른 타입이다.  
Java의 개발 철학에서 이름은 중요한 요소이다.

Java는 오래전부터 명시적 타이핑을 기반으로 설계된 언어다.

그래서 Java 입장에서 단순한 구조적 튜플을 도입하는 것은 언어 철학과 잘 맞지 않았을 수 있다.

예를 들어 이런 식의 튜플이 있다고 해보자.

```java
(String, Long)
```

이 구조만으로는 이 값이 사용자를 의미하는지, 상품을 의미하는지, 주문 정보를 의미하는지 알기 어렵다.

반면 Record는 이름을 가진 타입이다.

```java
public record User(String name, Long age) {
}

public record Product(String name, Long price) {
}
```

둘 다 `String`과 `Long`을 가지고 있어도 `User`와 `Product`는 명확히 다른 타입이다.

이 점에서 Java다움을 느낄 수 있다.

Java는 단순히 구조가 같은 데이터를 호환시키기보다는, 의미 있는 이름을 가진 타입을 정의하고 그 타입을 기준으로 안정성을 확보하는 방향을 선택했다고 볼 수 있다.

---

## Record는 일반 클래스와 무엇이 다를까?

Record는 클래스다.

하지만 일반 클래스와 완전히 같지는 않다.

Record는 암묵적으로 `final`이다.  
따라서 다른 클래스가 Record를 상속할 수 없다.

```java
public record User(String name) {
}

// 컴파일 에러
// class AdminUser extends User {
// }
```

또한 Record는 다른 클래스를 상속할 수도 없다.

Record는 암묵적으로 `java.lang.Record`를 상속한다.

그리고 Record 내부에는 새로운 인스턴스 필드를 추가할 수 없다.

```java
public record User(String name) {

    // 컴파일 에러
    // private int count;
}
```

---

## Record는 언제 사용하면 좋을까?

아마 앞의 개념적인 내용보다 실무에서는 이 질문에 대한 답이 가장 중요할 것이다.

Record는 다음과 같은 경우에 잘 어울린다.  
실제 실무에서 사용 중인 상황 위주로 정리해보았다.

### 1. API 응답 객체를 만들 때

```java
public record UserResponse(
    Long id,
    String name,
    String email
) {
}
```

### 2. API 요청 객체를 만들 때

```java
public record CreateUserRequest(
    String name,
    String email
) {
}
```

### 3. 조회 결과를 표현할 때

```java
public record UserSummary(
    Long id,
    String name
) {
}
```

### 4. 이벤트 메시지를 표현할 때

```java
public record UserCreatedEvent(
    Long userId,
    String email
) {
}
```

### 5. 값 객체를 간단하게 표현할 때

```java
public record Money(
    long amount,
    String currency
) {

    public Money {
        if (amount < 0) {
            throw new IllegalArgumentException("금액은 음수일 수 없습니다.");
        }

        if (currency == null || currency.isBlank()) {
            throw new IllegalArgumentException("통화는 비어 있을 수 없습니다.");
        }
    }
}
```

공통적으로 보면, Record는 **“값을 담고 전달하는 객체”**에 잘 맞는다.

그러나 JPA Entity를 Record로 만드는 것은 조심해야 한다.

JPA Entity는 보통 기본 생성자가 필요하고, 프록시 생성을 위해 상속 가능성이 필요하며, 변경 감지를 위해 상태 변경이 자연스럽게 발생한다.

이러한 JPA Entity를 불변 객체처럼 사용하면 JPA가 제공하는 더티 체킹이나 프록시 기반 지연 로딩 같은 장점을 충분히 활용하기 어렵게 된다.  
사실 JPA를 사용하는 큰 이유 중 하나를 사용하지 않게 되는 셈이다.

그래서 Record는 Entity보다는 DTO, Response, Request, Projection 같은 영역에서 먼저 고려하는 것이 자연스럽다.

---

## 마무리

Record는 단순히 DTO를 짧게 쓰기 위한 문법은 아니다.

물론 DTO를 만들 때 굉장히 유용하다.  
생성자, accessor, `equals()`, `hashCode()`, `toString()`을 자동으로 만들어주기 때문에 코드가 훨씬 간결해진다.

하지만 Record의 진짜 의미는 **“데이터를 데이터답게 표현하는 타입”**이라는 데 있다고 생각한다.

일반 클래스는 상태와 동작을 함께 캡슐화하고, 내부 구현을 숨기며, 다양한 방식으로 확장할 수 있는 객체지향의 기본 도구다.

반면 Record는 정해진 데이터를 투명하게 담고 전달하기 위한 도구다.

객체지향 프로그래밍에서 복잡한 구조도 여전히 중요하지만, 현대 애플리케이션에서는 외부 API와 데이터를 주고받고, 계층 간에 값을 전달하고, 이벤트를 발행하는 일이 점점 많아지고 있다.

이런 흐름 속에서 Record는 Java가 선택한 데이터 모델링 간소화 도구라고 볼 수 있다.

앞으로 클래스를 정의할 때, 이 객체가 **“행위를 가진 객체”**인지, 아니면 **“값을 전달하는 데이터 객체”**인지 먼저 생각해보자.

후자라면 Record는 좋은 선택지가 될 수 있다.

---

# 근황

요즘 회사 생활도 적응이 다 되어가고, 일도 어느 정도 손에 익어서 행복한 나날들을 보내고 있다.

복잡한 도메인을 코드로 풀어내고, 새로운 기술을 도입하고, 도입한 기술을 미팅에서 설명하면서 정말 여러 방면에서 성장하고 있음을 느끼는 하루하루다.

요즘 일을 하면서 느끼는 건, 성장에는 늘 약간의 불편함과 부담이 따라온다는 것이다. (하고싶은거 다하면서는 원하는 걸 못얻는다는 그런..)

준비가 다 되면 시작하겠다는 말은 듣기엔 그럴듯하지만, 때로는 가장 편한 핑계가 되기도 한다.

환경 탓을 하기 전에, 내가 오늘 얼마나 움직였는지를 먼저 돌아봐야 한다.

결국 달라지는 사람은 완벽해서 시작하는 사람이 아니라, 부족해도 끝까지 해보는 사람이다.

나는 말보다 행동으로, 핑계보다 결과로 증명하는 사람이 되고 싶다.
