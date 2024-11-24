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

이번 구현을 통해 몇 가지 느낀점이 있다.

1. 프레임워크의 제약사항과 좋은 객체 설계 사이의 균형 잡기
2. 빌더 패턴의 실용적 활용
3. NoSQL 기반 시스템에서의 데이터 모델링

특히 도메인 객체의 불변성을 지키면서도 프레임워크의 요구사항을 만족시키는 과정이 흥미로웠다. (물론 setter가 열려있기때문에 어쩔수없이 불변성이 깨짐에 취약하기는 하다...)
