---
layout: post
title: Threadsafe - 1
excerpt: "Thread Safe 란 무엇인지, 어떻게 적용할 수 있는지에 대한 방법을 알아보았다."
categories: [TIL - 기타]
comments: true
---

ThreadSafe 1
======

## Thread Safe
### 사전적 정의
thread-safe에 대해 wikipedia에서는 다음과 같이 정의하고 있다.

> **스레드 안전**(thread 安全, 영어: thread safety)은 멀티 스레드 프로그래밍에서 일반적으로 어떤 함수나 변수, 혹은 객체가 여러 스레드로부터 동시에 접근이 이루어져도 프로그램의 실행에 문제가 없음을 뜻한다. 보다 엄밀하게는 하나의 함수가 한 스레드로부터 호출되어 실행 중일 때, 다른 스레드가 그 함수를 호출하여 동시에 함께 실행되더라도 각 스레드에서의 함수의 수행 결과가 올바로 나오는 것으로 정의한다.
> 
> **Thread safety** is a computer programming concept applicable to multi-threaded code. Thread-safe code only manipulates shared data structures in a manner that ensures that all threads behave properly and fulfill their design specifications without unintended interaction. There are various strategies for making thread-safe data structures.

그래서 만약, "그래서 이 코드는 thread safe한거야?" 라고 질문하는 것은 "그래서 이 코드는 특정 상황에서 문제없이 동작하는거야?" 라는 질문이 되는 것이다. (특정 상황이란 동일 자원에 여러 개의 스레드가 접근하는 상황)
그래서, 이 코드가 문제없다는 것을 어떻게 결정할까? 이것에 대한 직접적인 해답은 지금까지 본 정의에서는 나와있지 않다.

### How to achieve thread-safety
내 주력 언어가 Java이니 Java에서 일반적으로 thread-safe한 코드를 작성할 지 간단하게 정리해보았다. 

#### 1. Stateless Implementation

multithreaded application에서 일어나는 에러들 중 대부분은 thread 간에 자원 공유 상태를 올바르게 다루지 않아 발생하게 된다.
그러므로, stateless한 구현을 우선적으로 고려할 필요가 있다.

예를 들어, 주어진 숫자의 차수를 갖는 factorial을 계산하는 정적메소드가 있다고 하자.
```java
public class MathUtils {
    
public static BigInteger factorial(int number) {
        BigInteger f = new BigInteger("1");
        for (int i = 2; i <= number; i++) {
            f = f.multiply(BigInteger.valueOf(i));
        }
        return f;
    }
}
```

factorial 메소드는 상태가 없고 결정론적인 함수이다.(참고 : 결정론적 알고리즘 - 예측한 그대로 동작하는 알고리즘) 특정 input에 대해 항상 동일한 결과를 출력한다.

이 메소드는 외부상태에 의존하지도 않고, 상태를 유지하지도 않는다. 따라서 이 메소드는 thread-safe이고, 여러 개의 thread에서 동시에 사용이 가능하다.    

#### 2. Immutable Implementations
만약 스레드 간 상태를 공유해야 한다고 하면, immtable한 클래스를 만들어 thread-safe를 만족시킬 수 있다.

Immutability(불변성)은 강력하고, 언어에 구애받지 않는 개념이며 Java에서 꽤 쉽게 구현이 가능하다.

간단하게 생각하자면, 인스턴스는 생성 후에 그것의 내부 멤버가 변경이 불가능하다면 immutable하다고 할 수 있다.

```java
public class MessageService{
    private final String message;

    public MessageService(String message){
        this.message = message;
    }

    // getter, 메소드 등.
}
```

위와 같이 setter를 제공하지 않는 클래스를 만들면 `MessageService`객체는 불변성을 갖게되고 thread-safe하게 사용이 가능하다.

#### 3. Thread-Local Fields
OOP에서 객체들은 사실 멤버변수를 통해 상태를 유지할 필요가 있으며, 하나 이상의 메소드를 통해 동작한다.

만약 상태를 진짜로 유지할 필요가 있다면, **thread-local** 필드를 만들어 스레드 간 상태를 공유하지 않는 thread-safe한 클래스를 만들 수 있다.

Thread 클래스를 상속받아 private 필드를 선언하는 것 만으로도 thread-local 필드를 만들 수 있다.

```java
public class ThreadA extends Thread {
    // thread-local field
    private final List<Integer> numbers = Arrays.asList(1,2,3,4,5,6);
    
    @Override
    public void run() {
        numbers.forEach(System.out::println);
    }
}

public class ThreadB extends Thread {
    // thread-local field
    private final List<Integer> letters = Arrays.asList("a", "b", "c", "d", "e", "f");
    
    @Override
    public void run() {
        letters.forEach(System.out::println);
    }
}
```

위 두 개의 클래스는 각자 '상태'를 가지고 있지만, 다른 스레드와 상태를 공유하지 않는다. 그러므로, thread-safe하다고 할 수 있다.

비슷하게 ThreadLocal 객체를 필드에 넣어 thread-local 필드를 만들 수 있다.

```java
public class StateHolder {
    private final String state;

    public StateHolder(String state){
        this.state = state;
    }

    // getter, 메소드들
}

public class ThreadState {
    public static final ThreadLocal<StateHolder> statePerThread = new ThreadLocal<StateHolder>() {
         
        @Override
        protected StateHolder initialValue() {
            return new StateHolder("active");  
        }
    };

    // ThreadLocal 클래스의 사용법은 찾아볼 것.
    public static StateHolder getState(){
        return statePerThread.get();
    }
}
```

Thread-local 필드는 getter와 setter를 통해 각 스레드가 독립적으로 초기화된 필드의 복사본을 얻어 자체적으로 상태를 갖는다는 점을 제외하면 일반 클래스의 필드와 비슷하다.

### 참고
- <https://www.baeldung.com/java-thread-safety>