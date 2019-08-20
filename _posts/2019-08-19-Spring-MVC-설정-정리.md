---
layout: post
title: Spring MVC 설정 정리
excerpt: "Spring MVC를 사용하기 위한 설정 정리"
categories: [TIL]
comments: true
---

Spring MVC Configuration
=========

> 날짜 : 19.07.30

모든 프레임워크가 그렇듯, Spring은 알아서 해주는 것들이 참 많다.

Spring이 알아서 해주는 것들(지금까지 아는것만)
- IoC, DI를 통해 Bean객체 생성, 관리 (Spring)
- AOP에 사용되는 Aspect 핸들링 (Spring AOP)
- MVC 패턴 개발에 사용되는 Controller 관리 등(Spring MVC / 다음 Spring MVC에 자세히 정리할 것)
- DB 연결, SQL 컴파일 등 (Spring JDBC)
- Transaction 관리 (Spring tx)

여기까지가 (내가 아는) 기본적으로 많이 사용되는 Spring이 알아서 해준는 것 + 모듈.

이렇게 많은 일을 알아서 처리해주는데, 이것들을 모두 개발자가 작성한 configuration을 기반으로 동작한다.

configuration을 작성하는 것도 XML, JavaConfig 등 여러가지 방법이 있겠지만, 가장 많이 사용되는 방법인 xml + annotation 기반으로 configuration 작성 방법을 알아보려고 한다.

configuration에 대해 작성하기 전에 전체적으로 Spring 내부에서 동작이 어떻게 흘러가는지, configuration을 통해 무엇을 관리하는지 알아보고 넘어가자.

![](/img/SpringMVC_Architecture.png)
출처 : https://www.java4coding.com/contents/spring/08springMVCArchitecture.html

위 그림은 java4coding 홈페이지의 그림을 나름대로 다시 그려본 것

둥근 모서리는 Spring에서 만들어주는 것들
각진 모서리는 개발자가 개발해야하는 것들

동작에 대해서는 SpringMVC에 자세하게 작성할 것.

여기서는 configuration 파일만 다루겠음

둥근 모서리들이 configuration 파일에서 다루어져야 하는 것들

물론 다른 것들도 XML configuration으로 전부 관리할 수도 있지만, annotation-driven이라 가정하에 작성함.

## Configurations

### 1. web.xml
Tomcat에 맞닿아 있는 configuration으로 Deployment Descriptor(배포기술자) 라고 한다.

웹 어플리케이션당 하나만 존재하며, WAS 구동 시 web.xml을 읽어 컨테이너의 설정을 구성한다.

```xml
<web-app>

    <display-name> 프로젝트명 </display-name>

    <filter>
        <filter-name> 필터 닉 네임 </filter-name>  <filter-name>
        <filter-class> 필터 클래스 풀 네임(패키지 명까지) </filter-name>
        <init-param>
            <param-name> 매개변수 명 </param-name>
            <param-value> 값 </param-value>
        </init-param>
    </filter> 

    <filter-mapping>
        <filter-name> 필터 닉 네임 </filter-name>
        <url-pattern> 필터 클래스가 실행될 위치 </url-pattern>
    </filter-mapping>

    <!-- root context -->
    <serlvet>
        <servlet-name> 서블릿 닉 네임 </servlet-name>
        <serlvet-class> 서블릿 클래스 풀네임(패키지 명까지) </servlet-class>
        <init-param>
            <param-name> 매개변수명 </param-name>
            <param-value> 값 </param-value>
        </init-param>
        <load-on-startup> 실행 순서 값(0값은 서버임의실행) </load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name> 서블릿 닉 네임 </servlet-name>
        <url-pattern> url 패턴</url-pattern>
    </servlet-mapping>

    <!-- spring context도 root context와 마찬가지로 작성
    보통 appName-context.xml 이런식으로 작성함. -->

    <welcome-file-list>
        <welcome-file> 기본 파일 </welcome-file>
    </welcom-file-list>


    <!-- 세션 기간 설정 -->
    <session-config>
      <session-timeout>
        30
      </session-timeout>
    </session-config>

    <!-- mime 매핑 -->
    <mime-mapping>
      <extension>txt</extension>
      <mime-type>text/plain</mime-type>
    </mime-mapping>

    <!-- 시작페이지 설정 -->
    <welcome-file-list>
      <welcome-file>index.jsp</welcome-file>
      <welcome-file>index.html</welcome-file>
    </welcome-file-list>

    <!-- 존재하지 않는 페이지, 404에러시 처리 페이지 설정 -->
    <error-page>
      <error-code>404</error-code>
      <location>/error.jsp</location>
    </error-page>

    <!-- 태그 라이브러리 설정 -->
    <taglib>
      <taglib-uri>taglibs</taglib-uri>
      <taglib-location>/WEB-INF/taglibs-cache.tld</taglib-location>
    </taglib>

    <!-- resource 설정 -->
    <resource-ref>
      <res-ref-name>jdbc/jack1972</res-ref-name>
      <res-type>javax.sql.DataSource</res-type>
      <res-auth>Container</res-auth>
    </resource-ref>
</wep-app>
```
예시 출처 : https://devbox.tistory.com/entry/Servlet-%EC%84%9C%EB%B8%94%EB%A6%BF%EC%97%90%EC%84%9C-webxml-%ED%8C%8C%EC%9D%BC%EC%9D%98-%EC%97%AD%ED%95%A0


### 2. pom.xml
Maven을 이용해 project의 의존성 라이브러리(dependency)들을 관리하는 configuration

나중에 Maven만 따로 정리해서 올릴 것

### 3. root-context.xml

- Spring 안의 Web Application 전체에서 사용되는 Bean을 관리하는 곳
(view와 관련되지 않은 객체들) ex. Service(Biz), Repository(DAO) 등
- root-context.xml 안에 정의된 객체들은 모든 app에서 사용 가능.

### 4. servlet-context.xml
- client의 요청을 받기 위한 entry point 로서의 servlet 설정
- 요청의 처리를 위한 controller의 매핑 설정(HandlerMapping), 요청 처리 후 결과 매핑(ViewResolver) 등을 이 곳에서 함.
- root-context를 참조하여 자원(bean) 사용 가능

#### Dispatcher Servlet과 root-context & servlet-context
![](https://docs.spring.io/spring/docs/current/spring-framework-reference/images/mvc-context-hierarchy.png)
출처 : https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc