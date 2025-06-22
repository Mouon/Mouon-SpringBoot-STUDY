# 프로젝트 Caffeine Cache를 도입기

최근 프로젝트를 진행하면서 동시성 처리가 필요한 상황이 발생했다. 사실 졸업 프로젝트 당시에도 분산락이나 `ReentrantLock`을 이용한 동시성 처리를 직접 구현해본 경험이 있었기 때문에 이번에도 동일한 방향으로 접근해보기로 했다.

이번 프로젝트는 **순수 Spring 기반**에 **단일 서버**로 운영되는 구조였기 때문에, 별도의 외부 분산락 시스템(Redis 기반 Redisson 등)을 도입하기보다는 코드 수준에서 해결하는 방향을 선택했다. 그래서 최초 구현 시 `ReentrantLock`을 사용하여 동시성 처리를 적용했다.

---

### 문제 발견!!!!!!!!!!!!!!!!! -> ConcurrentHashMap을 통한 lock 관리

그런데 구현 후 코드를 다시 살펴보던 중 중요한 결함을 발견했다.

기존 구조는 아래와 같은 형태였다.

```java
private final Map<Long, ReentrantLock> reservationLocks = new ConcurrentHashMap<>();

public ReservationCancelResponse updateReservationState(Long reservationId) {
    ReentrantLock lock = reservationLocks.computeIfAbsent(reservationId, key -> new ReentrantLock());
    try {
        lock.lock();
            // 기존 비즈니스 로직
            // 취소 처리
    } finally {
        lock.unlock();
    }
}

```

**문제는 lock 해제 후에도 `ConcurrentHashMap`에 해당 키가 남아 있다는 점**이었다.

즉, 처음 예약 ID 별로 lock 을 만들고 나면, 해당 lock 은 메모리에 계속 쌓이게 되고, 일정 시간이 지나면 잠재적인 **Memory Leak** 문제로 이어질 수 있었다.

초기 해결 방안으로는 `finally` 블록에서 `map.remove(reservationId)`를 호출하는 방식도 고려했지만, lock 해제 시점에서 어떤 다른 쓰레드가 해당 lock 을 쓰고 있는 경우 문제가 생길 수 있기 때문에 안전한 방식이 아니었다.

---

### Caffeine Cache 도입 검토

이후 관련 자료를 찾아보던 중, **Caffeine Cache** 또는 **Guava Cache**를 활용해 이런 문제를 해결하는 방법이 널리 쓰이고 있다는 것을 알게 되었다.

특히 Caffeine Cache는 Guava Cache의 후속으로 개발된 고성능 캐시 라이브러리로, 다음과 같은 장점이 있었다:

- TTL 지원 (expireAfterAccess, expireAfterWrite)
- 고성능 eviction 알고리즘 (TinyLFU)
- Thread-safe
- GC-friendly (GC 부하 최소화)
- 외부 infra 불필요 (JVM 내 캐시)

단일 서버 Spring 기반 프로젝트 특성상 복잡한 infra 없이 **가볍게 적용** 가능하다는 점이 매우 매력적이었다.

---

## 적용 방법

### 1. 의존성 추가

먼저 `build.gradle` 또는 `pom.xml` 에 Caffeine Cache 의존성을 추가했다.

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
    <version>3.1.8</version>
</dependency>

```

---

### 2. CacheConfig로 분리

Lock 관리를 위한 Cache 객체는 재사용 가능하고, 여러 Service 계층에서 공통으로 사용할 수 있으므로 별도의 `@Configuration` 클래스로 분리하여 Bean 으로 등록했다.

```java
@Configuration
public class CacheConfig {

    @Bean
    public Cache<Long, ReentrantLock> reservationLocks() {
        return Caffeine.newBuilder()
                .expireAfterAccess(10, TimeUnit.MINUTES) // 10분 동안 접근 없으면 자동 제거
                .build();
    }
}

```

이렇게 설정하면 일정 시간 동안 사용되지 않은 lock 은 자동으로 캐시에서 제거된다.

별도로 remove 처리를 하지 않아도 Memory Leak 을 걱정할 필요가 없어진다.

---

### 3. ReservationService 에 적용

기존에 직접 `ConcurrentHashMap`을 관리하던 코드를 아래와 같이 개선했다.

(필요한 경우 DI로 받아서 사용)

```java
@RequiredArgsConstructor
@Service
public class ReservationServiceImpl implements ReservationService {

    private final Cache<Long, ReentrantLock> reservationLocks;

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public ReservationCancelResponse updateReservationState(Long reservationId) {
        ReentrantLock lock = reservationLocks.get(reservationId, key -> new ReentrantLock());
        try {
            lock.lock();

            // 기존 비즈니스 로직
            // 취소 처리

        } finally {
            lock.unlock();
        }
    }
}
```

기존 `ConcurrentHashMap` 사용 시 발생했던 key 관리 이슈가 사라졌고 코드 자체도 훨씬 단순해졌다.

또한 `expireAfterAccess` 설정 덕분에 사용되지 않는 lock 은 자동으로 제거되므로 Memory Leak 걱정도 사라졌다.

---

## 개선 효과

이번 개선 작업을 통해 얻은 효과는 다음과 같다:

- **Memory Leak 문제 해결**
- **코드 복잡도 감소 (맵 키 삭제 등 별도 처리 불필요)**
- **lock 관리 자동화**
- **GC 부하 최소화 (Caffeine Cache 특성)**
- **유지보수성 향상**
