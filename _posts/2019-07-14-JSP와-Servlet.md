---
layout: post
title: JSP와 Servlet
excerpt: "JSP와 Servlet에 대해 간단하게 정리해보았습니다"
categories: [TIL]
comments: true
---

### WebServer의 역할

<center><img src="/img/webservice.png"></center>


### Servlet & JSP

JSP를 실행하게 되면 JSP => Servlet => HTML의 변환과정을 거쳐 Client에게 전달됨.

JSP 예제
```java
    // HelloServlet
    public class HelloServlet extends HttpServlet{
        ...
        protected void doGet(HttpServletRequest request, HttpServletResponse response)...{
            String id = request.getParamter("id");

            response.setContentType("type/html;charset=UTF-8");
            response.setCharacterEncoding("UTF-8");

            // redirect 방식
            response.sendRedirect("ok.jsp?id=" + id);

            // forward 방식
            RequestDispatcher rd = request.getRequestDispatcher("ok.jps");
            rd.forward(request, response);
        }
    }

    // ok.jsp
    <%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8" %>
    <!DOCTYPE html>
    // 헤드생략
    <p> ${id} </p>
    // ${}는 Expression Language(el)이라 부르며
    // request의 attribute를 가져온다.
```

위 예제의 sendRedirect와 forward는 rails, django에서 사용하던 redirect, render의 차이이다.

> 그러나 forward가 java servlet이 내가 사용해본 프레임워크의 render와 다른 것은, request 객체를 공유하지만 parameter는 사라짐.


**문제점**
- JSP 내에 JavaCode를 많이 사용하게되면 View와 Logic의 분리가 되지 않으므로, 유지보수에 좋지 않다.
=> 그래서 사용하는 것이, **JSTL**과 **EL**



**Attribute란?**
- attribute는 Web application의 구성 컴포넌트(JSP, Servlet, Listenr) 내에서 메소드로 저장되고 관리되는 객체.)
- 각 컴포넌트가 attribute를 공유하는 공간
    - page : pageContext 객체를 이용
    하나의 jsp 페이지 내에서 공유
    - request : HttpServletRequest 객체를 이용
    요청 ~ 응답 범위까지 공유
    - session : HttpSession 객체를 이용
    클라이언트의 세션 종료까지 공유
    - application : ServletContext 객체를 이용
    Application 시작 ~ 종료까지 공유


#### EL(Expression Language) 이란?
데이터를 표현하기 위한 언어.
- 원래는 JSTL의 한 부분이였음.
- request 객체의 attribute 혹은 parameter를 jsp상에서 쉽게 사용 가능.
    ```java
    // jsp 표현식
    <%= request.getAttribute("id") %>
    <%= request.getParamter("id") %>

    // EL
    ${id}
    ${param.id}
    ```
- EL 태그? 내에 쓰인 것은 Attribute의 이름으로 해석됨
- EL이 Attribute를 탐색할 때 순서(작은 Scope -> 큰 Scope)
    ```
    page -> request -> session -> application
    ```

#### JSTL(JSP Standard Tag Library) 이란?
JSP 표준 태그 라이브러리
- 코어
    - 일반 프로그램이 언어에서 제공하는 것과 유사한 변수 선언, 실행 흐름의 제어기능을 제공하고, 다른 JSP페이지로 제어를 이동하는 기능도 제공
    - uri : http://java.sun.com/jsp/jstl/core
    - prefix : c
- 포매팅
    - 숫자, 날짜, 시간을 포매팅하는 기능과 국제화, 다국어 지원 기능을 제공
    - uri : http://java.sun.com/jsp/jstl/fmt
    - prefix : fmt
- 데이터베이스
    - 데이터베이스의 데이터를 입력/수정/삭제/조회하는 기능을 제공
    - uri : http://java.sun.com/jsp/jstl/sql
    - prefix : sql
- XML처리
    - XML문서를 처리할 때 필요한 기능을 제공
    - uri : http://java.sun.com/jsp/jstl/xml
    - prefix : x
- 함수
    - 문자열을 처리하는 함수를 제공
    - uri : http://java.sun.com/jsp/jstl/fuctions
    - prefix : Fn

