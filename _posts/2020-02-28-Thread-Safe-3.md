---
layout: post
title: Threadsafe - 3
excerpt: "Thread Safe 란 무엇인지, 어떻게 적용할 수 있는지에 대한 방법을 알아보았다."
categories: [TIL - 기타]
comments: true
---

ThreadSafe 3
======

Thread Safe 관련 글
- [첫번째 글](https://move02.github.io/articles/2020-02/Thread-Safe-1)
- [두번째 글](https://move02.github.io/articles/2020-02/Thread-Safe-2)

### How to achieve thread-safety

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


#### 8. Synchronized Statement
간단한 몇 줄의 코드를 Thread-safe하게 만들기 위해 하나의 메소드 전체에 동기화를 거는 것은 과도할 때가 있다.

이를 해결하기 위해 statement 단위로 동기화를 거는 것이 가능하다.

```java
public void incrementCounter(){
    // 동기화가 필요없는 statements
    // ..........
    synchronized(this){
        counter += 1
    }
}
```

`incrementCounter()`메소드에 부가적인 기능들이 추가되었다고 가정했을 때, 메소드 전체에 동기화를 거는 것 보다는 객체 내의 공유자원인 counter에 접근하는 부분에 대해서 만 동기화를 걸어 thread-safe하게 만드는 것이 효율적일 수 있다.

동기화의 비용은 만만치않기 때문에, synchronized statement를 적절히 활용하는 것이 중요하다.

#### 9. Volatile Fields
Syncrhnized 메소드와 블록은 스레드 간 변수의 가시성 문제를 해결하는데 편리하다. 그렇다해도, 정규 클래스의 멤버변수는 CPU에 캐시되어 있을 가능성이 높다. 따라서 특정 필드이 대한 후속 업데이트는 동기화된 경우에도 다른 스레드에서 보이지 않을 수 있다.

이러한 상황을 막기 위해 `volatile` 클래스의 필드를 사용할 수 있다.

```java
public class Counter {
    private volatile int counter;

    // 생성자, getter 등
}
```

`volatile` 키워드를 이용하여 원하는 변수를 메인 메모리에 저장하도록 JVM과 compiler에 명령을 할 수 있다. 이렇게하면 JVM이 예시의 counter 변수의 값을 읽을 때마다 CPU 캐시가 아닌 메인 메모리에서 읽어온다. 마찬가지로, JVM이 counter 변수의 값을 쓸 때마다 메인 메모리에 그 값이 기록된다.

게다가 volatile 변수의 사용은 지정된 스레드에 표시되는 모든 변수를 메인 메모리에서도 읽을 수 있다.

```java
public class User {
    private String name;
    private volatile int age;

    // 생성자, getter 등
}
```

이러한 경우에는, JVM은 age 변수를 메인 메모리에 쓸 때마다, volatile이 아닌이 메인 메모리에 name 변수도 메인 메모리에 쓰게된다. 이것은 두 변수의 가장 마지막 값이 메인 메모리에 저장되는 것을 보장하여 이후에 일어나는 값의 업데이트도 자동으로 다른 스레드에서 볼 수 있게된다.

volatile 변수가 제공하는 확장된 보증은 **full volatile visibility guarantee** 라고도 한다.

### 참고
- <https://www.baeldung.com/java-thread-safety>
