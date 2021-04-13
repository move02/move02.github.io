---
layout: post
title: Blocking/Non-blocking vs Sync/Async
categories: [TIL-기타]
comments: true
---

Reactor로 토이 프로젝트를 하고있는데, 전체 스택을 자연스럽게 비동기나 논블로킹한 스택으로 쓰다보니 개념을 확실히 정리하고 사용하는 것이 좋겠다고 생각이 들었다.

## Context에 따라 달라지는 의미
많은 글에서 이 헷갈리는 용어들의 비교를 대부분 '관심사항이 다르다.' 라고 표현하고 있다.
예를 들면 Blocking과 Non-Blocking은 제어권의 반환 시점에 관심을 두고있고, Synchronous와 Asynchronous는 작업 완료 여부를 어디서 신경쓰는지에 관심을 둔다고 한다.
틀린 말은 아닌 것 같은데 그렇다면 일반적인 API 호출에서는 Blocking 한 작업과 Synchronous 한 작업의 실행 결과는 같지 않나? 에 대한 의문이 들었고, 조금 더 찾아보다 보니 가려운 부분을 긁어주는 글을 보게되었다.

Index라는 용어가 바닥에 깔려있는 의미는 동일하지만 어떤 문맥에서 얘기를 하냐에 따라 정확한 의미가 달라지듯, Blocking/Non-Blocking - Synchronous/Asynchronous도 마찬가지라는 뜻이다.

### Method API 호출 또는 Thread 에서의 의미
아래에서 설명하겠지만 이 경우에는 System call이 일어나지 않는 단순 함수 API 호출이나 Thread 관점에서의 두 용어를 정리하는 부분이다.
이 관점에서는 synchronous와 blocking은 같은 개념으로 간주한다.
그렇기 때문에, synchronous(또는 blocking)는 api call을 보낸 후 응답을 받을 때까지 대기 상태에 있다 응답을 받은 후 종료되고 asynchronous는 api call을 보낸 후 실행 여부와 관계없이 바로 응답을 받는다.

### 어플리케이션에서 운영체제로의 System Call 이 발생할 때의 의미
대부분의 글이 전재로 하는 상황이 바로 이것이다. 대표적인 경우가 I/O가 발생하는 상황이라고 할 수 있다.

#### Blocking / Non-Blocking
어플리케이션의 실행이 운영체제의 대기 큐에 **들어가고** system call이 완료될 때까지 **기다리는 것이** Blocking
어플리케이션의 실행이 운영체제의 대기 큐에 **들어가지 않고** system call 완료 여부와 **관계 없이** 바로 응답을 보내는 것이 Non-Blocking

I/O 가 일어나는 상황을 예로 들어 조금 더 구체적으로 정리하자면.
- Blocking I/O는 자신이 해당 자원을 사용할 수 있을 때까지 기다렸다가 작업 수행을 수행한다. 이 경우, 기다리는 시간동안 어플리케이션은 다른 작업을 할 수 없다.
- Non-Blocking I/O는 I/O가 실제로 일어나는지 진행상황에 관계없이 결과를 바로 반환한다. 
    - 만약 사용할 수 없는 경우, `EWOULDBLOCK` 과 같은 결과를 리턴한다.
    - 만약 사용이 가능하다면, buffer에 있는 데이터를 복사하고 데이터의 길이를 리턴한다. 
    - 실제 I/O 작업 시간과는 상관없이 빠른 리턴이 일어나기 때문에, 유저 프로세스의 작업이 중지되어 있는 시간이 적다.

#### Synchronous / Asynchronous
system call이 완료되고 다음 코드를 실행하는 것이 Synchronous
systemc call이 완료되는 것과 관계없이 다음 코드를 실행하는 것이 Asynchronous

#### Blocking vs Synchronous
system call 반환을 기다리는 동안 대기 큐에 머무는 것이 필수가 아니면 synchronous
system call 반환을 기다리는 동안 대기 큐에 머무는 것이 필수이면 blocking

#### Non-Blocking vs Asynchronous
system call이 반환될 때 실행된 결과와 함께 반환될 경우는 Non-blocking
system call이 반환될 때 실행된 결과와 함께 반환되지 않는 경우 Asynchronous

둘의 핵심 차이점은 **누가 system call 완료에 신경을 쓰는지** 이다.
단순 Non-Blocking만 채택할 경우 코드의 실행 자체는 Synchronous 하게 되며 polling 방식처럼 데이터를 정상적으로 받을 떄까지 재요청을 보낸다.
반면, Asynchronous의 경우 system call 완료에 신경쓰지 않고 운영체제의 callback을 받은 시점에 완료 처리를 하게된다. (이 때, Async + Blocking으로 코드를 짠다면 callback을 운영체제에게 맡기긴 하지만 어플리케이션 자체는 운영체제 대기큐에 들어있으므로 어플리케이션이 block 상태가 된다. Async를 쓰는 의미가 없다는 뜻.)


> Java의 경우 7부터 나온 NIO2 패키지의 `AsynchronousChannel` 인터페이스를 통해 완전한 Non-blocking(+ Asynchronous) IO를 수행할 수 있다.
> 예시.
> ```java
> AsynchronousServerSocketChannel listener
>  = AsynchronousServerSocketChannel.open().bind(null);
>
>listener.accept(
>  attachment, new CompletionHandler<AsynchronousSocketChannel, Object>() {
>    public void completed(
>      AsynchronousSocketChannel client, Object attachment) {
>          // listener IO 완료 시
>      }
>    public void failed(Throwable exc, Object attachment) {
>          // listener IO 실패 시
>      }
>  });
>
> ```

## 결론
간단하게 정리할 수 있을 줄 알았는데 생각보다 오랜 시간이 걸렸다. 단순히 외우기 보다는 이해를 하고 정리하려다 보니 운영체제 지식을 꽤나 많이 요구하는 것 같다. 역시 개발자는 기본이 탄탄해야 새로운 것을 받아들이기 수월하다는 생각이 든다.
\+ 이 모든 것은 Reactive Programming에 대한 개념을 정리하기 위한 초석.


## 참고
- [slipp.net](https://www.slipp.net/questions/367)
- [StackOverflow](https://stackoverflow.com/questions/2625493/asynchronous-vs-non-blocking)