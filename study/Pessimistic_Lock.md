# 스터디룸 동시가입 상황 제어하기
### 동시성 제어 구현기

최근 프로젝트에서 동시성 제어 관련 이슈를 겪었다. 
문제 해결 과정에서 비관적 락과 synchronized의 차이점을 깊이 있게 이해하게 되어 이를 공유하고자 한다.

## 비관적 락(Pessimistic Lock)의 이해

비관적 락은 이름 그대로 '비관적'인 가정에서 출발한다. 데이터 수정 시 충돌이 발생할 것이라고 가정하고, 우선 락을 걸고 보는 방식이다.

### 구현 방식
데이터베이스 레벨에서 제공하는 락 매커니즘을 사용한다.

```sql
SELECT ... FOR UPDATE
```
이 쿼리는 다음과 같은 작업을 수행한다:
1. 해당 row(또는 rows)에 대한 배타적 락 획득
2. 다른 트랜잭션은 이 row에 대한 수정 불가
3. 락이 해제될 때까지 대기

JPA에서는 `@Lock` 어노테이션으로 이를 구현한다:
```java
@Query("SELECT sr From Studyroom sr " +
            "JOIN FETCH sr.memberStudyroomList msl " +
            "WHERE sr.studyroomId = :studyroomId " +
            "AND sr.status = 'ACTIVE'")
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Studyroom> findById(long studyroomId);
```

## 문제 상황: 스터디룸 동시 가입 

### 초기 상태
```sql
-- 스터디룸(id=1)에 이미 5명의 멤버가 존재
INSERT INTO member_studyroom 
(member_id, studyroom_id, role, status) VALUES 
(1, 1, 'CAPTAIN', 'ACTIVE'),
(2, 1, 'CAPTAIN', 'ACTIVE'),
(3, 1, 'CAPTAIN', 'ACTIVE'),
(4, 1, 'CAPTAIN', 'ACTIVE'),
(5, 1, 'CAPTAIN', 'ACTIVE');
```

스터디룸의 최대 인원은 6명. 현재 5명이 있는 상태에서 마지막 한 자리를 두고 두 명의 사용자가 동시에 가입을 시도하는 상황이다.

### 첫 번째 시도: 비관적 락만 사용
```java
@Transactional
public void joinStudyroom(JoinStudyroomRequest request){
    Studyroom studyroom = studyroomRepository.findById(request.getStudyroomId())
            .orElseThrow();
    
    long activeCount = studyroom.getMemberStudyroomList().stream()
            .filter(m -> m.getStatus().equals(BaseStatus.ACTIVE))
            .count();

    // 트랜잭션 종료 시점에 락 해제
    if(activeCount < 6){
        memberStudyroomRepository.save(new MemberStudyroom(...));
    }
}
```

테스트 결과:
```java
@Test
void 동시_가입_테스트() throws Exception {
    // given
    long initialCount = 5;  // 초기 멤버 수
    CountDownLatch latch = new CountDownLatch(2);
    
    // when
    for (int i = 0; i < 2; i++) {
        executorService.submit(() -> {
            try {
                studyroomService.joinStudyroom(joinStudyroomRequest);
            } finally {
                latch.countDown();
            }
        });
    }
    
    latch.await();
    
    // then
    long finalCount = memberstudyroomRepository
        .countByStudyroomAndStatus(studyroom, BaseStatus.ACTIVE);
    assertEquals(initialCount + 1, finalCount);  // 실패!
}
```

### 실패 원인 분석

비관적 락은 트랜잭션 범위 내에서만 유효하다. 문제는 다음과 같은 시나리오에서 발생한다:

1. Thread A: 트랜잭션 시작, 비관적 락 획득
2. Thread A: 현재 인원 확인 (5명)
3. Thread A: 트랜잭션 종료, 락 해제
4. Thread B: 트랜잭션 시작, 비관적 락 획득
5. Thread B: 현재 인원 확인 (여전히 5명)
6. Thread A: 새 트랜잭션에서 저장
7. Thread B: 새 트랜잭션에서 저장

이는 Check-Then-Act 문제의 전형적인 예시다. 상태 확인과 액션 사이에 원자성이 보장되지 않는다.

### 해결: synchronized 추가

```java
@Transactional
public void joinStudyroom(JoinStudyroomRequest request){
    synchronized(this) {
        Studyroom studyroom = studyroomRepository.findById(request.getStudyroomId())
                .orElseThrow();
        
        long activeCount = studyroom.getMemberStudyroomList().stream()
                .filter(m -> m.getStatus().equals(BaseStatus.ACTIVE))
                .count();

        if(activeCount >= 6){
            throw new StudyroomException(OVER_MEMBER_STUDYROOM);
        }

        memberStudyroomRepository.save(new MemberStudyroom(...));
    }
}
```

`synchronized` 블록은 Check-Then-Act 연산의 원자성을 보장한다. 인원 체크부터 저장까지의 모든 과정이 하나의 원자적 연산으로 실행된다.

결과:
```
[Thread-1] : joinStudyroom
[Thread-2] : joinStudyroom
[Thread-1] : Success save memberStudyroom
[Thread-2] : Exception: 스터디룸맴버 초과입니다
```

## 비관적 락 vs synchronized

### 비관적 락
- 데이터베이스 레벨의 동시성 제어
- 트랜잭션 범위에서만 유효
- 분산 환경에서도 유효
- Row 단위의 락

### synchronized
- JVM 레벨의 동시성 제어
- 명시적인 블록 범위에서 유효
- 단일 JVM 내에서만 유효
- 메서드 또는 블록 단위의 락

## 결론

동시성 제어는 단순히 락을 거는 것을 넘어서, 비즈니스 로직의 원자성을 어떻게 보장할 것인지에 대한 고민이 필요하다. 특히 트랜잭션 범위와 락의 범위가 일치하지 않는 경우, 예기치 않은 동시성 문제가 발생할 수 있다.

이번 케이스에서는 비관적 락과 synchronized를 함께 사용함으로써:
1. 데이터베이스 레벨의 동시성 제어
2. 비즈니스 로직의 원자성 보장

두 가지 목표를 모두 달성할 수 있었다.


### 나아갈 점

뭔가 최선의 방법이 아니라는 생각이 직관적으로 든다... 더 알아봐야겠다.  
