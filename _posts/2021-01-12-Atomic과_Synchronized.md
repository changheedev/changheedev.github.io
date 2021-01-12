---
title: Atomic과 Synchronized
categories: [Programming, Java]
tags: [Java, Atomic, Synchronized, Volatile, Concurrency]
---

mutable한 객체를 공유하는 환경에서 객체에 대한 액세스가 제대로 관리되지 않으면 응용 프로그램은 감지하기 어려운 동시성 오류에 노출 될 수 있다.

```java
public class Counter {
    int counter;

    public void increment() {
        counter++;
    }
}
```

위의 코드는 싱글 스레드 환경에서는 문제없이 동작한다. 하지만 여러 스레드가 write 하는것을 허용하면 일관성이 없는 결과를 얻게 된다.

`counter++` 명령은 원자적인 명령어 처럼 보이지만 실제로는 3개의 명령으로 조합되어 있다.

- value를 가져온다.
- value를 증가시킨다.
- value를 write 한다.

만약 두 스레드가 같은 시간에 value를 업데이트 한다면 변경된 사항을 잃게 되는 결과가 발생할 수 있다.

## Synchronized

자바에서 객체에 lock을 거는 방법 중 하나는 `synchronized` 키워드를 사용하는 것이다. synchronized 키워드는 한번에 오직 하나의 스레드만 메소드를 실행할 수 있도록 보장한다.

```java
public class SafeCounterWithLock {
    private volatile int counter;

    public synchronized void increment() {
        counter++;
    }
}
```

lock을 사용하면 동시성 문제를 해결할 수 있지만 성능은 떨어지게 된다.

여러 스레드가 lock을 얻으려고 시도하면 그중 하나가 lock을 획득하고 나머지 스레드는 차단되거나 일시 중단된다. 스레드를 일시 중단했다가 다시 시작하는 것은 비용이 많이 들고 시스템의 전반적인 효율성에 영향을 준다. 위의 코드 처럼 소규모 프로그램에서는 컨텍스트 전환에 소요되는 시간이 실제 코드 실행보다 훨씬 길어 전체 효율성이 크게 감소 할 수 있다.

## Atomic

자바의 Atomic 타입은 내부적으로 **volatile** 키워드와 **CAS** 알고리즘이 사용된다.

### Volatile

일반적으로 변수에 값을 읽거나 쓸때 CPU의 캐시를 이용하게 된다. 멀티스레드 환경에서는 각 CPU의 캐시를 참조하기 때문에 **캐시 일관성** 문제가 발생하게 된다.

![java-volatile-1](https://user-images.githubusercontent.com/17294694/104310873-c86b0e00-5517-11eb-8231-2ef65508517b.png)

volatile은 CPU 캐시를 거치지 않고 메인 메모리에서 직접 값을 읽어오거나 값을 씀으로써 캐시 일관성 문제를 해결한다.

하지만 여전히 참조하는 메모리가 같기 때문에 동시성 문제는 가지고 있다. volatile은 메인 스레드에서만 쓰기를 허용하고 나머지 스레드는 읽기만 가능한 상황에서 유용하다. Atomic 타입은 이러한 부분을 CAS 알고리즘을 통해 보완하고 있다.

### CAS (Compare And Swap)

CAS 알고리즘은 이전에 메모리에서 읽어온 값과 현재 메모리에서 읽어온 값을 비교하여 일치하는 경우에만 새로운 값으로 교체하고 일치하지 않는다면 교체하지 않는다.

여러 스레드가 CAS를 통해 동일한 값을 업데이트하려고하면 그 중 하나가 승리하고 값을 업데이트하고 다른 스레드는 업데이트에 실패한다. **그러나 synchronized의 경우와 달리 다른 스레드는 일시 중단되지 않는다**. 대신, 값을 업데이트하지 않았다는 정보를 받는다.

이러한 방식은 컨텍스트 전환이 발생하지 않기 때문에 synchronized 보다 성능상 유리하다. 하지만 프로그램 로직이 더 복잡해진다. CAS 작업이 성공하지 못한 시나리오를 처리해야하기 때문인데, 성공할 때까지 반복해서 재시도하거나 설계에 따라 아무것도하지 않고 계속 진행할 수 있다.

### Atomic Variables

Atomic 타입은 자바의 `java.util.concurrent.atomic` 패키지에 포함되어 있다. [AtomicInteger](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicInteger.html), [AtomicLong](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html), [AtomicBoolean](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicBoolean.html), [AtomicReference](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html) 타입이 존재하며 이름에서도 알 수 있듯이 각각 int, long, boolean 형 데이터를 다룬다. AtomicReference는 자바의 참조 타입을 Atomic으로 사용하고자 할 때 사용한다.

Atomic 변수의 주요 메소드는 아래와 같다.

- get() : 메모리로부터 값을 읽어오며 다른 스레드에서 변경된 값을 볼 수 있다. (volatile)
- set() : 메모리에 값을 쓴다. 변경된 내용은 다른 스레드에도 보여진다. (volatile)

- compareAndSet() : CAS 알고리즘과 동일하게 동작한다.

**예제코드**

```java
public class SafeCounterWithoutLock {
    private final AtomicInteger counter = new AtomicInteger(0);

    public int getValue() {
        return counter.get();
    }
    public void increment() {
        while(true) {
            int existingValue = getValue();
            int newValue = existingValue + 1;
            if(counter.compareAndSet(existingValue, newValue)) {
                return;
            }
        }
    }
}
```

increment 메서드에 대한 호출이 항상 값을 1 씩 증가 시키는 것을 보장하기 위해 compareAndSet 을 시도하고 실패시 다시 시도한다 .

## 보완할 내용

volatile과 관련하여 명령어 재정렬 & happens-before 에 대해 확실히 이해하고 내용 보충하기

## 참고자료

https://www.baeldung.com/java-atomic-variables

http://tutorials.jenkov.com/java-concurrency/volatile.html
