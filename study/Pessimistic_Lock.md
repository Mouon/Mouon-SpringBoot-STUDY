# 스터디룸 동시가입 상황 제어하기
<br>  

### 동시성 제어 구현기

최근 프로젝트를 리팩토링을 하던중 동시성 제어 관련 이슈를 겪게되었다.  
스터디룸 인원 제한이 6명인데 동시에 5명인 스터디룸에 동시에 여러 사용자가 동시 가입하는 상황을 방지하고자하는 목적에서 구현 + 학습이 시작되었다!  

### 비관적 락(Pessimistic Lock)
해결책으로 먼저 알게된것은 **Pessimistic Lock** 이었다.
비관적 락은 말 그대로 '비관적'인 가정에서 출발한다. 데이터 수정 시 충돌이 발생할 것이라고 가정하고, 우선 락을 걸고 보는 방식이다.
<br>  
### 데이터베이스에서의 구현
먼저 데이터베이스 레벨에서 비관적 락이 어떻게 동작하는지 살펴보자. 
MySQL을 예로 들면, **FOR UPDATE** 구문을 통해 락을 구현한다.  

```sql
SELECT * FROM studyroom WHERE id = 1 FOR UPDATE;
```

### MySQL에서의 비관적 락 동작 방식

MySQL(InnoDB)에서 FOR UPDATE 구문을 사용하면 다음과 같은 일이 발생한다.

```sql
-- Session A
START TRANSACTION;
SELECT * FROM studyroom WHERE id = 1 FOR UPDATE;
```

이 순간 MySQL
1. InnoDB 에서 해당 row에 대한 배타적 락(Exclusive Lock)을 설정한다.
2. 내부적으로 트랜잭션 시스템 테이블에 락 정보를 기록한다.
3. 해당 row의 인덱스 레코드에 락 플래그를 설정한다.

이제 다른 세션에서 같은 row에 접근하려고 하면

```sql
-- Session B
START TRANSACTION;
SELECT * FROM studyroom WHERE id = 1 FOR UPDATE;
```

Session B는 Session A의 트랜잭션이 끝날 때까지 대기하게 된다. 

JPA에서는 `@Lock` 어노테이션으로 이를 간단하게 구현한다.  

```java
@Query("SELECT sr From Studyroom sr " +
            "JOIN FETCH sr.memberStudyroomList msl " +
            "WHERE sr.studyroomId = :studyroomId " +
            "AND sr.status = 'ACTIVE'")
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Studyroom> findById(long studyroomId);
```

<br><br><br><br>
    
## 스터디룸 동시 가입 문제 해결하기
<br>  

### 초기 상태
```sql
-- 스터디룸(id=1)에 이미 5명의 멤버가 존재
INSERT INTO member_studyroom 
(member_id, studyroom_id, role, status) VALUES 
(1, 1, 'CAPTAIN', 'ACTIVE'),
(2, 1, 'CREW', 'ACTIVE'),
(3, 1, 'CREW', 'ACTIVE'),
(4, 1, 'CREW', 'ACTIVE'),
(5, 1, 'CREW', 'ACTIVE');
```

스터디룸의 최대 인원은 6명, 현재 5명이 있는 상태에서 마지막 한 자리를 두고 두 명의 사용자가 동시에 가입을 시도하는 상황이다.

### 첫 번째 시도: 비관적 락만 사용
```java
    @Transactional
    public void joinStudyroom(JoinStudyroomRequest request){
        log.info("[StudyroomService.joinStudyroom]");
        Member member = memberRepository.findById(request.getMemberId())
                .orElseThrow(()->new MemberException(NOT_FOUND_MEMBER));
            Studyroom studyroom = studyroomRepository.findById(request.getStudyroomId())
                    .orElseThrow(() -> new StudyroomException(NOT_FOUND_STUDYROOM));

            long activeCount = studyroom.getMemberStudyroomList().stream()
                    .filter(memberStudyroom -> memberStudyroom.getStatus().equals(BaseStatus.ACTIVE))
                    .count();

            if (activeCount > 5) {
                throw new StudyroomException(OVER_MEMBER_STUDYROOM);
            }

            MemberStudyroom memberStudyroom = new MemberStudyroom(
                    null,
                    BaseStatus.ACTIVE,
                    request.getMemberRole(),
                    member,
                    studyroom);
            memberstudyroomRepository.save(memberStudyroom);
            log.info("Success save memberStudyroom");
    }
```

테스트 결과
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

<img width="757" alt="스크린샷 2024-12-25 오전 3 45 10" src="https://github.com/user-attachments/assets/5fe0914b-aa09-42cd-8e64-a218699325ab" />  

멀티스레드 환경에서 두개의 스레드가 동시에 스터디룸 가입에 시도하였다.  
6명만 가입이 성공해야하는데 7명이 성공해버렸다... 


### 실패 원인 분석

비관적 락은 트랜잭션 범위 내에서만 유효하다. 문제는 다음과 같은 시나리오에서 발생한다.

1. Thread A: 트랜잭션 시작, 비관적 락 획득
2. Thread A: 현재 인원 확인 (5명)
3. Thread A: 트랜잭션 종료, 락 해제
4. Thread B: 트랜잭션 시작, 비관적 락 획득
5. Thread B: 현재 인원 확인 (여전히 5명)
6. Thread A: 새 트랜잭션에서 저장
7. Thread B: 새 트랜잭션에서 저장

여기서 핵심적인 부분은 @Transactional이 메서드 레밸에서 선언되어있더라도, 각각의 스레드는 독립적인 트랜잭션 환경에서 실행된다는 점이다. 
따라서 두 스레드가 동시에 **joinStudyroom** 메서드를 호출하면, 각자의 트랜잭션에서 인원을 확인하고 저장을 시도하게 되어 결과적으로 인원 제한을 초과하는 문제가 발생하게 된다.

이는 Check-Then-Act 문제의 전형적인 예시다. 상태 확인과 액션 사이에 원자성이 보장되지 않는다.  


### 해결: synchronized 추가

그러던 와중 **synchronized**가 이 문제를 해결해줄 수 있다는 것을 알게되었다.  
어떻게 synchronized가 이 문제를 해결해줄 수 있을까?  
synchronized는 한 번에 하나의 스레드만 코드 블록에 접근할 수 있도록 제어한다. 
이를 통해 인원 확인부터 멤버 추가까지의 전체 과정을 하나의 원자적 작업으로 만들 수 있다. 
즉, 첫 번째 스레드가 작업을 완전히 마칠 때까지 두 번째 스레드는 대기하게 되므로, 더 이상 동시성 문제가 발생하지 않는다. 


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

결과적으로 `synchronized` 블록은 Check-Then-Act 로직의 원자성을 보장한다. 인원 체크부터 저장까지의 모든 과정이 하나의 원자적 연산으로 실행된다.

결과

<img width="757" alt="스크린샷 2024-12-25 오전 3 46 06" src="https://github.com/user-attachments/assets/5750c336-26dc-4894-b568-9e39fef783b1" />  


## 결론

동시성 제어는 단순히 락을 거는 것을 넘어서 비즈니스 로직의 원자성을 어떻게 보장할 것인지에 대한 고민이 필요하다. 특히 트랜잭션 범위와 락의 범위가 일치하지 않는 경우, 예기치 않은 동시성 문제가 발생할 수 있다.

이번 케이스에서는 비관적 락과 synchronized를 함께 사용함으로써 "데이터베이스 레벨의 동시성 제어"와 "비즈니스 로직의 원자성 보장"이라는 두 가지 목표를 모두 달성할 수 있었다.

 
