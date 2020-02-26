---
layout: post
title: Threadsafe - 2
excerpt: "Thread Safe 란 무엇인지, 어떻게 적용할 수 있는지에 대한 방법을 알아보았다."
categories: [TIL - 기타]
comments: true
---

ThreadSafe 2
======

[지난 번 포스트](https://move02.github.io/articles/2020-02/Thread-Safe-1)에 이어 Thread Safe를 Java로 어떻게 구현하는지 알아보았다.

### How to achieve thread-safety

#### 4. Synchronized Collections
동기화 wrapper들을 이용하여 thread-safe한 콜렉션을 만들 수 있다.

```java
Collection<Integer> syncCollection = Collections.synchronizedCollection(new ArrayList<>());
Thread thread1 = new Thread(() -> syncCollection.addAll(Arrays.asList(1,2,3,4,5,6)));
Thread thread2 = new Thread(() -> syncCollection.addAll(Arrays.asList("a","b","c","d","e","f")));

thread1.start();
thread2.start();
```
synchronized 컬렉션은 각자 메소드에서 고유 로킹(intrinsic locking)을 한다는 것을 명심해야 한다.

이것은 메소드가 한 번에 하나의 스레드에서만 접근 가능하며, 다른 스레드는 unlock될 때까지 접근이 막힌다는 것을 의미한다.
따라서, 동기화는 synchhronized access의 기본적인 로직에 의해 성능상 불이익이 생긴다.

#### 5. Concurrent Collections
synchronized collection을 대체하기 위해 concurrent collection을 사용할 수 있다.
Java에서는 `ConcurrentHashMap`과 같은 동시성을 지닌 collection들을 포함하는 `java.util.concurrent` 패키지를 제공한다.

```java
Map<String,String> concurrentMap = new ConcurrentHashMap<>();
concurrentMap.put("1", "one");
concurrentMap.put("2", "two");
concurrentMap.put("3", "three");
```

synchronized와는 다르게 concurrent collection은 데이터를 분할하는 방식(segment(부분) 잠금 방식)을 이용하여 thread-safety를 구현한다. 예를 들어, `ConcurrentHashMap`에서는 여러 스레드가 서로 다른 Map segment에서 락을 걸 
수 있으므로, 여러 스레드가 하나의 Map에 동시에 접근이 가능하다.

> synchronized collection과 concurrent collection은 collection 자체만을 thread-safe하게 만들 뿐이지 그 내용(요소)까지 thread-safe하게 만들지는 못한다.

#### 6. Atomic Objects

Java에서 제공하는 **Atomic Class**를 이용하여 thread-safe한 코드를 구현할 수도 있다.
Atomic class들은 동기화 없이 thread-safe한 Atomic operation(*원자성* 연산)을 제공한다. Atomic operation은 단 하나의 machine level operation이다.

> atomic : DB에서 트랜잭션의 특성 중 하나인 atomicity(원자성)과 동일하다고 생각하면 된다.
> 하나의 원자 트랜잭션은 모두 성공하거나 모두 실패한다.
> => 하나의 연산은 모두 성공하거나 모두 실패한다.

atomic에 대해 다음의 예제를 보자.

```java
public class Counter {
    private int counter = 0;

    public void incrementCounter() {
        counter += 1;
    }

    public int getCounter() {
        return counter;
    }
}
```

두 개의 스레드가 동시에 `incrementCounter()` 메소드에 접근하여 race condition이 생겼다고 가정해보자.

이론적으로는 최종적으로 counter 필드의 값은 2가 되어야 한다. 그러나, 두 개의 스레드가 동시에 동일한 코드를 실행하고 증가연산(incrementation) 자체는 atomic하지 않기 때문이다.

```java
public class AtomicCounter {
    private final AtomicInteger counter = new AtomicInteger();

    public void incrementCounter() {
        counter.incrementAndGet();
    }

    public int getCounter() {
        return conter.get();
    }
}
```

위처럼 **AtomicInteger**클래스를 이용하여 증가연산(++) 자체는 하나 이상의 연산을 거치지만, `incrementAndGet`은 atomic하게되어 thread-safe하게 구현할 수 있다.

#### 7. Synchronized Methods
지금까지 봤던 접근방법들이 collection과 primitive에 좋은 방법이지만, 동시에 연산자체보다 많은 컨트롤을 요구한다.

thread-safety를 얻기 위한 또 다른 공통적인 접근법은 **synchronized method**를 구현하는 것이다.

간단하게 키워드를 쓰기만 하면, 단 하나의 thread만 synchronized method에 접근이 가능하고 동시에 다른 스레드의 접근은 막힌다. 먼저 들어간 thread가 작업을 끝내거나 예외가 발생하기 전까지 block은 유지된다.

```java
public synchronized void incrementCounter(){
    counter += 1;
}
```

Synchronized method는 "intrinsic locks" 또는 "monitor locks"이라고 불리우는 방법을 사용한다.(각각 고유 락, 모니터 락)
고유 락(Intrinsic lock)은 특정 클래스의 인스턴스와 관련된 암시적 내부 속성이다.(== 모든 객체가 고유 락을 갖고있다.)

스레드가 Synchronized 메소드를 호출하면 그 **스레드는** 고유 락을 얻게 된다. 스레드가 메소드 실행을 종료한 뒤에 락이 해제되므로 다른 스레드가 락을 얻어 메소드에 접근할 수 있다.

### 참고
- <https://www.baeldung.com/java-thread-safety>
