# 스레드 세이프란?

**스레드 세이프(Thread-Safe)** 란 멀티스레드 환경에서 여러 스레드가 동시에 같은 함수, 변수, 또는 객체에 접근하더라도 정상적인 실행 결과를 보장하는 코드의 속성을 말한다.

쉽게 말해, 하나의 스레드가 어떤 함수를 실행하고 있을 때, 동시에 다른 스레드가 같은 함수를 호출하더라도 서로 영향을 주지 않고 각자의 실행 결과가 올바르게 유지되는 것을 의미한다.

### **스레드 세이프가 중요한 이유**

멀티스레드 환경에서는 여러 스레드가 동일한 자원(예: 변수, 객체 등)에 동시에 접근할 수 있기 때문에, 적절한 조치를 하지 않으면 Race Condition이나 Data Corruption 같은 문제가 발생할 수 있다.

이런 문제를 방지하기 위해 스레드 세이프를 보장할 필요가 있다.

### **스레드 세이프를 위한 대표적인 방법**

스레드 세이프를 구현하기 위한 주요 기법은 다음과 같다.

1. 상호 배제(Mutual Exclusion)
   하나의 스레드만 특정 코드 영역을 실행할 수 있도록 제한하는 방식.
2. 원자적 연산(Atomic Operation)
   중단되거나 간섭받지 않고 한 번에 수행되는 연산.
3. 스레드 로컬 스토리지(Thread-Local Storage)
   각 스레드가 독립적인 데이터를 갖도록 하여 공유 자체를 방지한다.
4. 재진입성(Reentrancy)
   한 스레드가 어떤 메서드를 실행 중일 때, 그 메서드를 재귀적으로 다시 호출하더라도 문제가 없는 특성.
5. 락 프로그래밍(Lock Programming)
   명시적인 락 객체(ReentrantLock 등)를 사용하여 동기화 제어를 세밀하게 하는 방식.

### **자바에서 스레드 세이프를 위한 도구들**

자바는 멀티스레드 환경에서 스레드 세이프를 구현하기 위해 다양한 기능을 제공한다.

- synchronized 키워드
  간단하고 직관적인 동기화 방식이다. 메서드나 특정 코드 블록에 락을 걸어 한 번에 하나의 스레드만 진입할 수 있도록 보장한다.
- Lock 인터페이스 및 ReentrantLock 클래스
  더 세밀한 동기화 제어가 필요할 때 사용한다. tryLock, lockInterruptibly, Condition 등 유연한 기능 제공한다.
- Atomic 클래스들 (AtomicInteger, AtomicReference 등)
  락 없이도 스레드 안전하게 연산을 수행할 수 있는 클래스들이다.

**예제: synchronized로 구현한 스레드 세이프한 카운터**

```java
package org.example;

public class Test {
    private int count = 1000;

    public synchronized void decrement() {
        count--;
    }

    public synchronized int getCount() {
        return count;
    }

    public static void main(String[] args) throws InterruptedException {
        Test ts = new Test();

        Thread[] threads = new Thread[1000];
        for (int i = 0; i < threads.length; i++) {
            threads[i] = new Thread(ts::decrement);
            threads[i].start();
        }

        for (Thread t : threads) {
            t.join();
        }

        System.out.println("Final count: " + ts.getCount()); // 출력 : 0
    }
}
```

위 예제에서 decrement()와 getCount() 메서드에 synchronized를 사용함으로써, 동시에 하나의 스레드만 해당 메서드를 실행할 수 있도록 보장한다.

만약 synchronized가 없었다면, 다수의 스레드가 동시에 count-- 를 수행하게 되며, 의도하지 않은 결과가 나올 수 있다.

### **마무리**

멀티스레드 환경에서 개발을 하다 보면 동시성 이슈는 피할 수 없는 주제다.

특히 자바와 같은 멀티스레드 기반 백엔드 시스템을 개발할 때는 스레드 세이프를 어떻게 구현할 것인지에 대한 고려가 필수다.

이번 글에서는 스레드 세이프의 기본 개념부터, 자바에서의 구현 방식과 대표 예제를 살펴보았다. 앞으로 개발할 때 이러한 개념을 잘 적용한다면, 더욱 신뢰성 높은 시스템을 만드는 데 큰 도움이 될 것같다!

### 사실은…

사실은 면접때 스레드 세이프에대한 질문을 받았는데 당시에 “스레드 세이프”라는 단어를 떠올리지 못한 안타까움에 이 글을 쓰게 되었다.. 레디스의 원자적 연산, 자바에서의 여러 스레드 세이프한 개발 등 .. 평소에 고민이 많았던 부분이고 자신있던 부분인데 면접장에서 말하지 못한게 너무 너무 너무 아쉽다..

[면접 기록 블로그 글](https://moon-kotlin.tistory.com/78)