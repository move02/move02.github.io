---
layout: post
title: Spring AOP 용어 정리
excerpt: "Spring AOP 및 AOP 관련 용어 정리"
categories: [TIL]
comments: true
---

Spring AOP
=========

> 날짜 : 19.07.03


## Spring AOP

**AOP(Aspect Oriented Programming)** : 관점 지향 프로그래밍
-> 관심사를 분리하여 모듈의 응집도를 높이는 것이 목표

나오게 된 이유 : 서비스의 비즈니스 로직 (핵심 관심 Core Concerns)을 부가적인 기능 (횡단관심 - Crosscutting Concerns)과 분리시켜 응집도를 높이고 유지보수를 편하게 하려하기 위함.

### AOP 용어 정리
- 타겟 (Target) 
    - 부가기능을 부여할 대상을 얘기합니다. 
    - 핵심기능을 담당하는 메소드

- 애스펙트 (Aspect) 
    - 핵심기능에 부가적인 기능을 추가하는 모듈
    - Aspect는 어드바이스와 포인트컷을 함께 가짐

> 참고로 AOP(Aspect Oriented Programming)라는 뜻 자체가 어플리케이션의 핵심적인 기능에서 부가적인 기능을 분리해서 Aspect 독특한 모듈로 만들어서 설계하고 개발하는 방법을 뜻함 (출처 : https://jojoldu.tistory.com/71)

- 어드바이스 (Advice) 
    - 실질적으로 부가기능을 담은 모듈의 구현체
    - Object에 종속되지 않기 때문에 부가기능에만 집중
    - Aspect가 '무엇'을 '언제' 할지를 정의
    - `Before` : JoinPoint의 실행 전
    - `Around` : JoinPoint의 앞 뒤
    > `Around` 메소드는 `ProceedingJoinPoint` 객체(PointCut)를 받아와 proceed() 메소드를 호출해야 한다. / 포인트컷의 앞, 뒤로 부가기능을 실행하기 때문에 
    - `After throwing` : JoinPoint 예외 발생 시
    - `After returning` : JoinPoint 정상적으로 종료 시
    - `Introduction` : 클래스에 인터페이스와 구현을 추가하는 특수한 Advice

- 조인포인트 (JoinPoint) 
    - 어드바이스가 적용될 수 있는 모든 위치
    - 다른 AOP 프레임워크와 달리 Spring에서는 메소드 조인포인트만 제공

- 포인트컷 (PointCut) 
    - 실제 Advice가 적용되는 위치
    - PointCut 표현식(정규표현식이나 AspectJ 문법)을 이용해 정의
    ```java
    // Annotation을 이용한 AOP 설정
        @Aspect // Aspect 역할을 하는 클래스인 것을 명시
        public class LogAdvisor {
            ...
            @Pointcut("execution(public * org.move02.sample.*Sample.*(..))") // 
            public void targetMethod() {
                // pointcut annotation 값을 참조하기 위한 dummy method
            }
            ...

            @Around("targetMethod()")
            // 혹은
            @Around("execution(public * org.move02.sample.*Sample.*(..))")
            public void logging(){

            }
        }
    ```
    - 위 예제처럼 PointCut을 정의하는 빈 메소드를 만들어서 호출하는 방법과 어드바이스 annotation에 직접 PointCut을 지정하는 방법이 있음
    - XML을 이용하여 설정할 수도 있음.
    ```xml
    <!-- XML을 이용한 AOP 설정 -->
    <bean id="logAdvisor" class="org.move02.sample.advisor.LogAdvisor">
    <aop:config>
        <aop:aspect ref="logAdvisor">
        <aop:pointcut id="loggingPointCut" expression="execution(public * org.move02.sample.*Sample.*(..))">
        <aop:around method="aroundAOP" pointcut-ref="loggingPointCut">
        <aop:before method="breforeAOP" pointcut-ref="loggingPointCut">
        <aop:after method="afterAOP" pointcut-ref="loggingPointCut">
        <aop:after-returning method="afterReturningAOP" pointcut-ref="loggingPointCut" returning="retValue">
        <aop:after-throwing method="afterThrowing" pointcut-ref="loggingPointCut" throwing="ex">
    </aop:config>
    ```

- 프록시 (Proxy) 
    - Target의 요청을 대신 받아주는 랩핑(Wrapping) 오브젝트
    - **Runtime에** 호출자 (클라이언트)에서 target을 호출하게 되면 타겟을 감싸고 있는 프록시가 호출되어, 타겟 메소드 실행전에 선처리, 타겟 메소드 실행 후, 후처리를 실행시키도록 구성되어있음.
    - JDK Dynamic Proxy(Interface의 구현체)와 CGLIB Proxy(Interface가 없는 경우)로 나뉨
    > 최근에는 CGLIB의 지원이 더 좋아 많이 사용한다고 함.
    
    - proxy 동작 과정 (그림출처 : https://ooz.co.kr/201)
    ![](/img/aop_proxy.png)

- 위빙 (Weaving)
    - PointCut에 Aspect를 적용하여 프록시 객체를 만드는 과정

