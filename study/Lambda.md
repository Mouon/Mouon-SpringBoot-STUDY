# AWS Lambda와 DynamoDB

<img src="https://github.com/user-attachments/assets/ba13bf32-a91e-4485-948b-640fa2dcfef1" width="200" height="200">


이번에 AWS Lambda와 DynamoDB를 사용해 데이터 수집 시스템을 구현하게 되었다.  
개발을 진행하면서 Spring과는 꽤 다른 패턴들을 많이 마주쳤는데, 
특히 도메인 객체를 다루는 방식에서 차이점들을 발견했다. 그래서 스프링이 아님에도 간단하게 배운점 + 느낀점들을 정리하게 되었다.

### DynamoDB의 도메인 객체

가장 먼저 눈에 띈 것은 도메인 객체의 구조였다. Spring에서는 보통 아래와 같은 방식으로 작성했다.

```java
@Entity
public class Event {
    @Id
    private String id;
    private final String title;

    protected Event() {}
}

```

하지만 DynamoDB에서는 조금 달랐다.

```java
@DynamoDbBean
public class CultureEvent {
    private String event_id;
    private String title;

    public CultureEvent() {}  /** public 기본 생성자 필수 */

    /** public setter 필수 */
    public void setTitle(String title) {
        this.title = title;
    }
}

```

위처럼 setter가 필수적으로 필요했다. Spring에서는 당연하게 생각했던 객체의 불변성이 깨질 수 있는 상황이다.

Spring으로 개발하던 방식을 생각하고 최초에는 setter를 없애고 빌더패턴으로만 구성해뒀는데 객체 생성이 되지않고 오류가 생겼다. 그 이유를 알고 보니 DynamoDB의 Enhanced Client가 Jackson 라이브러리를 사용해서 객체 매핑을 하는데, 이 과정에서 기본 생성자와 setter가 필요했던 것이다. 이는 Spring MVC에서 @RequestBody로 JSON을 DTO로 변환할 때와 비슷한 패턴이다. ( @RequestBody에서도 Jackson 라이브러리를 사용)

### 불변성을 지켜보기 위해..

이 문제를 해결하기 위해 직접 빌더 패턴을 도입했다.. 롬복이 없어서 직접 만들었다..

```java
public class CultureEvent {
    private final String event_id;  // final로 선언

    private CultureEvent(Builder builder) {
        this.event_id = builder.event_id;
    }

    public static class Builder {
        private String event_id;

        public Builder event_id(String event_id) {
            this.event_id = event_id;
            return this;
        }
				...
        public CultureEvent build() {
            return new CultureEvent(this);
        }
    }
}

```

빌더 패턴이 뜻밖의 코드 스타일 일관성을 가져다 주었다..
이유는 AWS SDK V2 자체가 빌더 패턴을 적극적으로 사용하고 있기때문..

```java
DynamoDbClient ddb = DynamoDbClient.builder()
    .region(Region.AP_NORTHEAST_2)
    .build();

// Enhanced Client도 마찬가지..
DynamoDbEnhancedClient enhancedClient = DynamoDbEnhancedClient.builder()
    .dynamoDbClient(ddb)
    .build();

```

### Spring과는 다른 Enhanced Client

JPA의 EntityManager가 있다면, DynamoDB에는 Enhanced Client가 있다. 얘도 나름대로 깔끔하게 테이블과 객체를 매핑

```java
// 테이블 매핑
DynamoDbTable<CultureEvent> table = enhancedClient.table("events",
    TableSchema.fromBean(CultureEvent.class));

// 저장도 깔끔하게
table.putItem(event);

```

### 배운 점들 🚀

#### 이번 구현을 통해 몇 가지 느낀점들을 아래처럼 정리해볼 수 있을 것 같다.


### AWS Lambda와 Spring Boot 개발을 통해 배운 설계 철학의 차이

설계 원칙의 차이
- Spring Boot: 확장성과 유지보수성을 중시하는 "준비된 프레임워크"
    - 복잡한 비즈니스 로직을 체계적으로 관리
    - 하나의 서비스에서 여러 기능을 통합 관리 (수직적 확장)
    - 미래의 확장을 고려한 설계 중시
- AWS Lambda: 단순성과 효율성을 중시하는 "필요한 것만" 접근
    - YAGNI(You Aren't Gonna Need It) 원칙 적용
    - 각 기능을 독립적인 함수로 분리 (수평적 확장)
    - 현재 필요한 기능에만 집중


  
결론 : 프레임워크의 제약사항과 좋은 객체 설계 사이의 균형을 맞추는 것이 중요하며, 각 플랫폼의 특성에 맞는 적절한 설계 철학을 적용하는 것이 핵심인 것같다.
