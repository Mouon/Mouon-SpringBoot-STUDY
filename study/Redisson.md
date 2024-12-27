# 스터디룸 동시성 제어: synchronized에서 분산락까지

지난번 [스터디룸 동시성 제어](https://github.com/Mouon/Mouon-SpringBoot-STUDY/blob/master/study/Pessimistic_Lock.md)에 대한 블로그를 정리하면서… synchronized로 스레드를 직렬화시키는 것이 근본적인 해결책이 될 수 없을 것 같다는 생각이 들었다. 실제 서비스는 여러 대의 서버로 운영될 텐데 synchronized는 하나의 서버 안에서만 동작할 것 같았기 때문이다. (사실 직감적으로 한계가 있을 것만 같았다 🥲)

## Redis의 발견

여러 자료를 찾아보다가 Redis를 활용한 분산락이라는 개념을 알게 됐다. 여러 서버가 하나의 Redis를 바라보면서 동시성을 제어할 수 있다는 점이 내가 느꼈던 한계를 해결해줄 수 있을 것 같았다.

우선 Redisson이라는 Redis 클라이언트를 설정했다.

```java
@Configuration
public class RedissonConfig {
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://localhost:6379");
        return Redisson.create(config);
    }
}

```

그리고 실제 분산락을 적용해봤다.

분산락을 코드상에 적용하는 것은 어렵지않았다. ReentrantLock과 같은 기존에 자바에서 제공하는 락과 같이 사용하면 된다.

```java
public void joinStudyroom(long studyroomId long memberId MemberRole role) {
    RLock lock = redissonClient.getLock("studyroomLock:" + studyroomId);
    lock.lock();
    try {
        Studyroom studyroom = studyroomRepository.findById(studyroomId)
                .orElseThrow(() -> new StudyroomException(NOT_FOUND_STUDYROOM));
        validateHeadCount(studyroom);
        MemberStudyroom memberStudyroom = new MemberStudyroom(
                null BaseStatus.ACTIVE role member studyroom);
        memberstudyroomRepository.save(memberStudyroom);
    } finally {
        lock.unlock();
    }
}

```

studyroomId를 기반으로 락을 획득하므로 이제 각 스터디룸마다 독립적인 락이 생성된다. 서버가 여러 대여도 모두 같은 Redis를 보고 있으니 진정한 의미의 동시성 제어가 가능해졌다.

## 새로 알게 된 점

분산락을 공부하면서 재미있는 점들을 많이 발견했다. synchronized는 스레드를 막아버리는데 비해 Redis는 인메모리 데이터베이스라 속도도 빠르다. 또 finally로 락 해제를 보장하니 데드락 걱정도 줄었다.

처음에는 단순히 찜찜한 느낌에서 시작했지만 분산 환경에서의 동시성 제어라는 새로운 세계를 알게 됐다. 

이젠 새로운 문제에 대해 학습해보려한다. 바로 분산락을 이용했을때 터질수 있는 문제 방지이다. 예를 들면 레디스 장애 발생 등이 있겠다.
