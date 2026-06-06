# 들어가며

트랜잭션이라는 말을 백엔드 개발을 하다보면 정말 자주 듣게된다.

회원가입을 한다거나, 주문을 한다거나, 포인트를 차감한다거나

이런 작업들은 대부분 중간에 애매하게 끝나면 안된다.

예를 들어 주문은 실패했는데 포인트만 차감된다거나

계좌 이체는 실패했는데 돈만 빠져나간다거나

이러면 이제 서비스가 아니라 금융 치료가 시작된다.

그래서 데이터베이스에서는 트랜잭션이라는 개념을 제공한다.

트랜잭션은 하나의 작업 단위라고 볼 수 있다.

그리고 트랜잭션의 특성은 흔히들 ACID로 정리한다.

## ACID

### Atomicity

원자성이다.

트랜잭션의 연산은 모두 수행되거나, 모두 수행되지 않아야 한다.

쉽게 말해서 중간만 성공하면 안된다는 뜻이다.

주문 생성은 됐는데 결제 기록이 안 남거나

결제 기록은 남았는데 주문 생성이 안되면 안된다.

이런 상황을 막기 위해 트랜잭션은 전부 성공하거나 전부 실패해야한다.

### Consistency

일관성이다.

트랜잭션 수행 전과 수행 후에도 데이터베이스는 일관된 상태를 유지해야 한다.

예를 들어 잔액이 음수가 될 수 없다는 규칙이 있다면

트랜잭션이 끝난 뒤에도 그 규칙은 지켜져야 한다.

데이터베이스가 갑자기 자기 마음대로 이상한 상태가 되면 안된다.

DB도 사회성이 필요하다.

### Isolation

고립성이다.

하나의 트랜잭션이 실행 중일 때

다른 트랜잭션이 중간 결과에 접근할 수 없어야 한다.

여기까지 들으면 그냥

“아 서로 방해 안하게 하면 되는구나”

정도로 이해할 수 있다.

**근데 사실 오늘 글에서 중요한 부분은 이 고립성이다.** 집중 집중!

고립성을 어느 정도까지 보장할 것인가에 따라

성능과 정합성이 달라지기 때문이다.

### Durability

영속성이다.

성공적으로 완료된 트랜잭션의 결과는

시스템 장애가 발생해도 영구적으로 유지되어야 한다.

커밋까지 했는데 서버가 재시작됐다고 데이터가 사라지면 안된다.

그건 영속성이 아니라 일회성이다.

## 여기까지는 기본이다

ACID는 트랜잭션을 공부하면 거의 제일 먼저 만나는 개념이다.

원자성, 일관성, 고립성, 영속성

이 네 가지는 정말 기본이다.

백엔드 개발자를 준비한다면

이 정도는 툭 치면 나와야 한다.

이랬던 것 같다.

하지만 막상 서버 개발을 하다보면

ACID만 알고 끝낼 수 없다. 왜냐하면 실무에서는 트랜잭션 하나만 고고하게 실행되는 것이 아니라 여러 트랜잭션이 동시에 실행되기 때문이다.

동시에 읽고, 동시에 수정하고, 동시에 추가한다.

그리고 ... 여기서부터 데이터가 꼬이기 시작한다.

그래서 오늘은 트랜잭션의 고립성 
그 중에서도 트랜잭션 격리 수준에 대해 알아보자.

## 트랜잭션 격리 수준

트랜잭션 격리 수준은 동시에 실행되는 여러 트랜잭션을

서로 얼마나 강하게 격리할 것인지 정하는 기준이다.

대표적으로 아래 네 가지가 있다.

```
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

이름만 보면 뭔가 있어보인다.

실제로도 있다.

대충 보면 아래로 갈수록 느슨하고

위로 갈수록 엄격하다고 볼 수 있다.

하지만 오늘은 가장 엄격한 SERIALIZABLE부터 살펴보겠다.

## SERIALIZABLE

SERIALIZABLE은 가장 엄격한 격리 수준이다.

이름 그대로 트랜잭션을 직렬화해서 처리한다고 볼 수 있다.

직렬화라는 말이 조금 딱딱한데

쉽게 말하면 트랜잭션을 순서대로 실행한 것처럼 보장한다는 뜻이다.

여러 트랜잭션이 동시에 실행되더라도

결과만 보면 하나씩 차례대로 실행된 것처럼 보이게 만든다.

그러면 데이터 부정합 문제가 생길 가능성이 거의 없다.

안전하다.

진짜 안전하다.

문제는 너무 안전하다는 것이다.

SERIALIZABLE은 안전한 대신 성능이 떨어진다.

동시에 처리할 수 있는 작업도 순차적으로 처리해야 할 수 있기 때문이다.

MySQL InnoDB 기준으로 SERIALIZABLE 격리 수준에서는

일반적인 SELECT도 그냥 읽고 끝나는 것이 아니다.

SELECT가 대상 인덱스 레코드에 shared next-key lock을 걸 수 있다.

즉, SELECT FOR UPDATE 같은 문법을 명시하지 않아도

읽기 작업이 다른 트랜잭션의 INSERT, UPDATE, DELETE를 막을 수 있다.

안전하긴 한데

너무 안전해서 주변 사람들까지 못 움직이게 만드는 느낌이다.

약간 조별과제에서

“제가 다 할게요. 아무도 건들지 마세요.”

하는 사람 같다.

결과는 안전할 수 있는데

분위기는 조금 무거워진다.

그래서 실무에서 모든 트랜잭션을 SERIALIZABLE로 두는 경우는 많지 않다.

정합성이 정말 중요한 일부 상황이 아니라면

보통은 다른 격리 수준을 사용하고, 필요한 지점에 락을 거는 방식으로 해결한다.

## REPEATABLE READ

다음은 REPEATABLE READ다.

MySQL InnoDB의 기본 격리 수준이기도 하다.

REPEATABLE READ는 이름 그대로

같은 트랜잭션 안에서 같은 데이터를 반복해서 읽었을 때

같은 결과를 보장하는 격리 수준이다.

여기서 중요한 개념이 나온다.

바로 MVCC다.

MVCC는 Multi-Version Concurrency Control의 약자다.

다중 버전 동시성 제어라고 부른다.

멀티버스 아니다.

멀티 버전이다.

처음 들으면 약간 마블 세계관 같은데

DB 세계관이다.

관계형 데이터베이스는 보통 데이터를 변경할 때

변경 전 데이터를 undo log에 백업해둔다.

그러면 같은 레코드에 대해

변경 전 데이터와 변경 후 데이터가 동시에 존재할 수 있다.

예를 들어 사용자 A가 member 테이블의 name을 변경했다고 하자.

변경된 값은 현재 데이터에 반영되지만

변경 전 값은 undo log에 남아있다.

이 덕분에 다른 트랜잭션은 자신이 봐야하는 시점의 데이터를 읽을 수 있다.

즉, 트랜잭션마다 보는 데이터의 버전이 달라질 수 있다.

이게 MVCC다.

## REPEATABLE READ 예시

예를 들어 사용자 B가 트랜잭션을 시작하고

id가 50인 회원을 조회했다고 하자.

```
START TRANSACTION;

SELECT *
FROM member
WHERE id = 50;
```

이때 결과가 1건 조회되었다.

아직 사용자 B의 트랜잭션은 끝나지 않았다.

그런데 이때 사용자 A가 다른 트랜잭션에서

id가 50인 회원의 이름을 수정하고 커밋한다.

```
START TRANSACTION;

UPDATE member
SET name = 'updated'
WHERE id = 50;

COMMIT;
```

그럼 사용자 B가 다시 같은 SELECT를 실행하면 어떻게 될까?

```
SELECT *
FROM member
WHERE id = 50;
```

REPEATABLE READ에서는 사용자 B가 처음 조회했던 시점의 데이터를 다시 읽는다.

사용자 A가 중간에 값을 변경하고 커밋했더라도

사용자 B의 트랜잭션 안에서는 같은 결과가 보장된다.

이것이 REPEATABLE READ다.

같은 트랜잭션 안에서 반복 조회했는데

갑자기 값이 바뀌면 곤란하다.

분명 아까는 이름이 문코딩이었는데

다시 보니까 김코딩이면 당황스럽다.

물론 개명했을 수도 있다.

근데 트랜잭션 중간에 그러면 안된다.

## Phantom Read

그런데 여기서 하나 더 생각해볼 문제가 있다.

바로 Phantom Read다.

Phantom Read는 같은 조건으로 조회했는데

두 번째 조회에서 이전에는 없던 행이 갑자기 나타나는 현상이다.

예를 들어 사용자 B가 age가 20 이상인 회원을 조회했다고 하자.

```
START TRANSACTION;

SELECT *
FROM member
WHERE age >= 20;
```

이후 사용자 A가 age가 25인 회원을 새로 추가하고 커밋한다.

```
START TRANSACTION;

INSERT INTO member(id, name, age)
VALUES (51, 'new-member', 25);

COMMIT;
```

그 다음 사용자 B가 다시 같은 조건으로 조회했을 때

새로운 회원이 보인다면 이것이 Phantom Read다.

말 그대로 유령처럼 갑자기 나타난 것이다.

물론 실제 서비스에서 유령이 나오면 장애다.

## MySQL의 REPEATABLE READ는 조금 다르다

보통 이론적으로는 REPEATABLE READ가 Phantom Read를 막지 못한다고 설명한다.

그런데 MySQL InnoDB 기준으로는 조금 다르게 봐야 한다.

MySQL InnoDB의 REPEATABLE READ에서는

일반 SELECT가 MVCC 기반의 consistent read로 동작한다.

그래서 같은 트랜잭션 안에서 일반 SELECT를 반복하면

처음 읽은 스냅샷을 기준으로 데이터를 조회한다.

즉, 다른 트랜잭션이 중간에 INSERT를 커밋해도

현재 트랜잭션의 일반 SELECT에서는 그 데이터가 보이지 않는다.

```
SELECT *
FROM member
WHERE age >= 20;

SELECT *
FROM member
WHERE age >= 20;
```

이런 일반 SELECT 반복 조회에서는

MVCC 덕분에 Phantom Read가 발생하지 않는다.

그럼 SELECT FOR UPDATE는 어떨까?

SELECT FOR UPDATE는 단순 조회가 아니라 잠금 읽기다.

```
SELECT *
FROM member
WHERE id >= 50
FOR UPDATE;
```

이 경우 MySQL InnoDB는 조건에 해당하는 레코드뿐만 아니라

그 주변 범위에도 락을 걸 수 있다.

여기서 나오는 개념이 next-key lock이다.

next-key lock은 record lock과 gap lock을 합친 개념이다.

```
record lock : 실제 존재하는 인덱스 레코드에 거는 락
gap lock : 인덱스 레코드 사이의 빈 공간에 거는 락
next-key lock : record lock + gap lock
```

예를 들어 사용자 B가 id가 50 이상인 데이터를 SELECT FOR UPDATE로 조회했다면

사용자 A가 그 범위 안에 새로운 데이터를 INSERT하려고 할 때 대기할 수 있다.

```
INSERT INTO member(id, name)
VALUES (51, 'new-member');
```

사용자 B의 트랜잭션이 끝날 때까지 기다리다가

너무 오래 기다리면 락 타임아웃이 발생한다.

이런 방식으로 MySQL InnoDB는 REPEATABLE READ에서도

많은 경우 Phantom Read를 방지한다.

정리하면 MySQL 기준으로는 아래처럼 볼 수 있다.

```
SELECT 이후 SELECT
-> MVCC 때문에 Phantom Read 발생 X

SELECT FOR UPDATE 이후 SELECT
-> gap lock 때문에 Phantom Read 발생 X

SELECT FOR UPDATE 이후 SELECT FOR UPDATE
-> gap lock 때문에 Phantom Read 발생 X
```

출처: MangKyu's Diary - MySQL 트랜잭션 격리 수준과 부정합 문제들

https://mangkyu.tistory.com/299

여기서 중요한 것은

“REPEATABLE READ는 무조건 Phantom Read가 발생한다” 라고 외우면 안된다는 것이다.

DBMS마다 구현이 다르다.

MySQL InnoDB는 MVCC와 next-key lock을 이용해서

REPEATABLE READ에서도 Phantom Read를 막는 경우가 많다.

## READ UNCOMMITTED

다음은 READ UNCOMMITTED다.

READ UNCOMMITTED는 커밋되지 않은 데이터도 읽을 수 있는 격리 수준이다.

다른 트랜잭션에서 아직 커밋하지 않은 변경 내용이

내 트랜잭션에서 바로 보일 수 있다.

예를 들어 사용자 A가 데이터를 수정했다.

```
START TRANSACTION;

UPDATE member
SET name = 'dirty'
WHERE id = 1;
```

아직 커밋하지 않았다.

그런데 사용자 B가 같은 데이터를 조회했을 때

수정된 값이 보일 수 있다.

```
SELECT *
FROM member
WHERE id = 1;
```

문제는 사용자 A가 이 트랜잭션을 롤백할 수도 있다는 것이다.

```
ROLLBACK;
```

그러면 사용자 B는 실제로는 확정되지 않은 데이터를 읽은 셈이다.

이런 현상을 **Dirty Read**라고 한다.

Dirty Read는 말 그대로 더러운 읽기다.

이름부터 쓰지 말라고 말해주는 것 같다.

커밋되지 않은 데이터는 아직 진짜 데이터가 아니다.

그런데 그걸 읽고 비즈니스 로직을 수행하면 정합성이 깨질 수 있다.

그래서 READ UNCOMMITTED는 일반적으로 잘 사용하지 않는다.

## READ COMMITTED

다음은 READ COMMITTED다.

READ COMMITTED는 커밋된 데이터만 읽는 격리 수준이다.

READ UNCOMMITTED와 다르게 다른 트랜잭션이 아직 커밋하지 않은 데이터는 읽지 않는다.

그래서 Dirty Read는 발생하지 않는다. 예를 들어 사용자 A가 데이터를 수정했지만 아직 커밋하지 않았다면

사용자 B는 그 변경 내용을 읽지 않는다. 대신 이전에 커밋된 데이터를 읽는다.

그런데 **READ COMMITTED**에서는

같은 트랜잭션 안에서 같은 SELECT를 두 번 실행했을 때

결과가 달라질 수 있다. 왜냐하면 READ COMMITTED는 매 SELECT마다

그 시점에 커밋된 데이터를 읽기 때문이다.

예를 들어 사용자 B가 트랜잭션을 시작하고

id가 1인 회원을 조회했다.

```
START TRANSACTION;

SELECT *
FROM member
WHERE id = 1;
```

이때 name이 moon이었다고 하자.

그런데 사용자 A가 같은 회원의 이름을 수정하고 커밋한다.

```
START TRANSACTION;

UPDATE member
SET name = 'updated'
WHERE id = 1;

COMMIT;
```

이후 사용자 B가 같은 트랜잭션 안에서 다시 조회한다.

```
SELECT *
FROM member
WHERE id = 1;
```

READ COMMITTED에서는 두 번째 SELECT에서 updated가 보일 수 있다.

즉, 같은 트랜잭션 안에서 같은 데이터를 반복 조회했는데

결과가 달라질 수 있다.

이것을 Non-Repeatable Read라고 한다.

READ COMMITTED는 Dirty Read는 막지만

Non-Repeatable Read는 막지 못한다.

REPEATABLE READ와의 차이는 여기서 나온다.

```
READ COMMITTED
-> 매 SELECT마다 그 시점에 커밋된 데이터를 읽는다.

REPEATABLE READ
-> 트랜잭션 안에서 처음 읽은 스냅샷을 기준으로 반복 조회 결과를 유지한다.
```

그래서 READ COMMITTED는 REPEATABLE READ보다 동시성 측면에서 유리할 수 있지만

반복 조회의 일관성은 약하다.

## 격리 수준별 정리

간단히 정리하면 아래와 같다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
| --- | --- | --- | --- |
| READ UNCOMMITTED | 발생 가능 | 발생 가능 | 발생 가능 |
| READ COMMITTED | 방지 | 발생 가능 | 발생 가능 |
| REPEATABLE READ | 방지 | 방지 | DBMS 구현에 따라 다름 |
| SERIALIZABLE | 방지 | 방지 | 방지 |

여기서 REPEATABLE READ의 Phantom Read는 꼭 DBMS 기준으로 봐야한다.

MySQL InnoDB 기준에서는 MVCC와 next-key lock을 통해

일반적인 Phantom Read를 방지하는 경우가 많다.

반대로 다른 DBMS에서는 동작 방식이 다를 수 있다.

그러니까 면접에서

“REPEATABLE READ는 Phantom Read 발생합니다.”

라고만 말하면 조금 위험할 수 있다.

면접관이

“MySQL에서도요?”

라고 물어보면 그때부터 눈동자가 undo log로 백업된다.

## 마무리

오늘은 트랜잭션의 기본 개념과 격리 수준에 대해 알아보았다.

ACID는 트랜잭션의 기본이다.

하지만 백엔드 개발을 하려면 ACID에서 끝나면 안된다.

실제로 중요한 것은

여러 트랜잭션이 동시에 실행될 때 어떤 문제가 발생하고

격리 수준에 따라 DB가 어디까지 막아주는지 이해하는 것이다.

정리하면 아래와 같다.

```
SERIALIZABLE
가장 엄격하다.
안전하지만 성능 비용이 크다.

REPEATABLE READ
MySQL InnoDB의 기본 격리 수준이다.
MVCC를 통해 반복 조회 일관성을 보장한다.

READ COMMITTED
커밋된 데이터만 읽는다.
하지만 같은 트랜잭션 안에서도 반복 조회 결과가 달라질 수 있다.

READ UNCOMMITTED
커밋되지 않은 데이터도 읽을 수 있다.
Dirty Read 문제가 발생할 수 있어 일반적으로 사용하지 않는다.
```
