# Servlet 기초 · URL Mapping · 생명주기

> 학습일: 2026-10-27   
> 범위: Servlet 정의와 요청 처리, 상속 구조, URL Mapping, 생명주기, 기본 실습

## 1. Servlet이란?

**Servlet**은 WAS의 **Servlet Container에서 실행되는 Java Web Component**로, 클라이언트의 **HTTP 요청을 처리하고 응답을 생성**한다.

```text
Client
  │ GET /BE_01/hello
  ▼
WAS (Tomcat)
  │ 1. Context Path 확인: /BE_01
  │ 2. URL Mapping 확인: /hello
  │ 3. Servlet 상태 확인 (필요 시 생성 및 초기화)
  │ 4. 요청 처리 스레드에서 실행
  ▼
HelloServlet
  │ service() → doGet()
  │ 응답 데이터 생성
  ▼
HTTP Response → Client
```

### 장점과 단점

| 장점               | 설명                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------- |
| 객체지향 기반      | Java OOP를 이용해 작성, 유지보수·재활용에 유리하고 플랫폼 독립적                      |
| 효율적인 요청 처리 | 요청마다 객체를 새로 만들지 않고 하나의 Servlet 객체가 여러 스레드의 요청을 처리 가능 |
| 확장성             | Filter, Listener, 다양한 프레임워크와 통합 가능                                       |

Servlet 코드에 **비즈니스 로직과 HTML 생성 로직이 함께 작성되면** 복잡해지기 쉽다. 강의자료에서는 이를 개선하는 방식으로 **Servlet + JSP를 분리하는 Model 2 방식**을 언급한다.

## 2. Servlet 주요 API와 상속 구조

```text
Servlet (interface)
      ▲
GenericServlet
      ▲
 HttpServlet
      ▲
 MyServlet
```

| 구성             | 역할                       |
| ---------------- | -------------------------- |
| `Servlet`        | Servlet의 기본 규약 정의   |
| `GenericServlet` | 일반적인 Servlet 기능 구현 |
| `HttpServlet`    | HTTP 요청 처리 기능 제공   |
| `MyServlet`      | 개발자가 작성하는 Servlet  |

일반적으로 `HttpServlet`을 상속해 작성한다.

```java
public class HelloServlet extends HttpServlet {
}
```

대표 패키지:

```java
import jakarta.servlet.*;
import jakarta.servlet.http.*;
import jakarta.servlet.annotation.*;
```

## 3. Servlet URL Mapping

URL Mapping은 **클라이언트의 요청 URL과 해당 요청을 처리할 Servlet을 연결**하는 설정이다.

```text
/hello → HelloServlet
/hi    → HiServlet
/add   → AddServlet
/login → LoginServlet
```

### ① `web.xml`을 이용한 등록

`web.xml`은 **Deployment Descriptor(배포 서술자)**로, 웹 애플리케이션의 설정을 작성하는 XML 파일이다.

```xml
<servlet>
    <servlet-name>hiServlet</servlet-name>
    <servlet-class>com.ssafy.live.controller.HelloServlet</servlet-class>
</servlet>
<servlet-mapping>
    <servlet-name>hiServlet</servlet-name>
    <url-pattern>/hi</url-pattern>
</servlet-mapping>
```

| 구성                | 의미                |
| ------------------- | ------------------- |
| `<servlet>`         | Servlet 등록        |
| `<servlet-name>`    | Servlet 이름        |
| `<servlet-class>`   | 실제 Servlet 클래스 |
| `<servlet-mapping>` | Servlet과 URL 연결  |
| `<url-pattern>`     | 요청 URL 패턴       |

### ② `@WebServlet` 애너테이션

```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
}
```

또는 다음과 같이 작성할 수 있다.

```java
@WebServlet(urlPatterns = {"/hello"})
public class HelloServlet extends HttpServlet {
}
```

| 속성            | 기능                     |
| --------------- | ------------------------ |
| `name`          | Servlet 이름 지정        |
| `value`         | URL 패턴 지정            |
| `urlPatterns`   | URL 패턴 지정            |
| `loadOnStartup` | Servlet 초기화 순서 지정 |
| `initParams`    | 초기화 파라미터 설정     |

`@WebServlet`은 클래스에 적용하며, 런타임에 Servlet Container가 해당 설정을 활용한다.

**URL 패턴:** 경로 기반(`/hello`, `/hello/hi`) 또는 확장자 기반(`*.do` → `/hello.do`, `/hi.do`) 매핑을 사용할 수 있다.

### Context Root와 URL Mapping

- **상대 경로:** `/`로 시작하지 않고 기준 URL에 따라 해석
- **절대 경로:** `/`로 시작하는 경로. 웹 애플리케이션에서는 기준 루트 구분 필요

```text
Container Root: http://localhost:8080
Context Root:   http://localhost:8080/BE_01
Context Path:   /BE_01
Servlet Mapping: /hello
최종 URL:       http://localhost:8080/BE_01/hello
```

> **핵심:** `Context Path + Servlet URL Mapping`을 조합해 최종 접근 URL이 결정된다.

## 4. HelloServlet 기본 코드

```java
@WebServlet(
    description = "Hello 라고 인사하는 Servlet",
    urlPatterns = {"/hello"}
)
public class HelloServlet extends HttpServlet {

    protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws ServletException, IOException {
        response.getWriter()
                .append("Served at: ")
                .append(request.getContextPath());
    }

    protected void doPost(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws ServletException, IOException {
        doGet(request, response);
    }
}
```

| 코드                       | 역할                     |
| -------------------------- | ------------------------ |
| `@WebServlet`              | 요청 URL 매핑            |
| `extends HttpServlet`      | Servlet 기능 상속        |
| `doGet()`                  | GET 요청 처리            |
| `doPost()`                 | POST 요청 처리           |
| `request.getContextPath()` | 현재 Context Path 반환   |
| `response.getWriter()`     | 문자 응답 출력 객체 반환 |

강의 예제의 `doPost()`는 `doGet()`으로 위임한다. 강의자료는 실제 개발에서는 GET과 POST의 목적에 따라 구분해 처리해야 한다고 설명한다.

## 5. Servlet Life Cycle

**Servlet 객체의 생성부터 소멸까지 Servlet Container가 관리**한다. 개발자는 일반적으로 Servlet 객체를 직접 생성해 실행하지 않는다.

```text
Servlet 요청
  ▼
객체 존재 여부 확인
  ├── 없음 → 객체 생성 → init()
  └── 있음 ───────────┘
                         ▼
                      service()
                      ├── GET  → doGet()
                      └── POST → doPost()

WAS 종료 또는 애플리케이션 재배포 → destroy()
```

| 메서드      | 호출 시점                                       | 역할                                                    |
| ----------- | ----------------------------------------------- | ------------------------------------------------------- |
| `init()`    | Servlet 초기화 시, 일반적으로 생명주기 중 한 번 | 필요한 자원 준비                                        |
| `service()` | 요청이 발생할 때마다                            | 실제 HTTP 요청 처리, `doGet()` / `doPost()` 등으로 분기 |
| `destroy()` | Servlet 종료 전                                 | 사용한 자원 정리                                        |

> **주의:** 하나의 Servlet 객체가 여러 요청을 처리하므로 매 요청마다 `init()`이 실행되는 것은 아니다.

## 6. Servlet 기본 실습

### AddServlet — GET 요청으로 덧셈

```html
<form action="/[CONTEXT_NAME]/add">
    <input type="number" name="num1" /> +
    <input type="number" name="num2" /> = ?
    <button type="submit">더하기</button>
</form>
```

| 항목        | 값                       |
| ----------- | ------------------------ |
| URL Mapping | `/add`                   |
| HTTP Method | GET (`method` 생략 시)   |
| 파라미터    | `num1`, `num2`           |
| 주요 API    | `request.getParameter()` |
| 변환        | `Integer.parseInt()`     |

```java
@WebServlet("/add")
public class AddServlet extends HttpServlet {
    @Override
    protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws ServletException, IOException {
        String n1 = request.getParameter("num1");
        String n2 = request.getParameter("num2");

        int num1 = Integer.parseInt(n1);
        int num2 = Integer.parseInt(n2);
        int result = num1 + num2;

        response.setContentType("text/html;charset=UTF-8");
        response.getWriter().print("<h1>결과: " + result + "</h1>");
    }
}
```

**흐름:** 사용자 입력 → Form Submit(GET) → `/add` → `doGet()` → 파라미터 조회 → 정수 변환 → 덧셈 → HTML 응답

※ 이 예제는 정상적인 숫자 입력을 가정한다. 강의자료는 실제 서비스에서 누락 및 변환 오류 검사가 필요하다고 설명한다.

### LoginServlet — POST 요청으로 로그인 처리

```html
<form action="/[CONTEXT_NAME]/login" method="post">
    <input type="text" placeholder="아이디를 입력하세요" name="id" />
    <input type="password" placeholder="비밀번호를 입력하세요" name="pass" />
    <button>로그인</button>
</form>
```

| 항목        | 값           |
| ----------- | ------------ |
| URL Mapping | `/login`     |
| HTTP Method | POST         |
| 파라미터    | `id`, `pass` |
| 처리 메서드 | `doPost()`   |

```java
@WebServlet("/login")
public class LoginServlet extends HttpServlet {
    @Override
    protected void doPost(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws ServletException, IOException {
        String id = request.getParameter("id");
        String pass = request.getParameter("pass");

        // 입력값 검증
        // 비즈니스 로직을 통한 로그인 처리
        // 필요한 경우 DB 연동
        // 결과에 따른 응답 생성
    }
}
```

이 예제는 **파라미터와 Servlet의 역할**을 확인하는 학습용 코드이다. 실제 인증 로직은 구현되어 있지 않다.

### 두 실습의 공통 처리 순서

1. `getParameter()`로 요청 값 조회 (필요 시 로깅·검증)
2. 비즈니스 로직 처리
3. 필요한 경우 Persistence Logic을 통한 DB 연동
4. Content-Type 및 Character Encoding 설정
5. 응답 생성

## 핵심 정리

**Servlet은 HTTP 요청을 처리하고 응답을 생성하는 Java 컴포넌트이며, Servlet Container가 객체의 실행 및 생명주기를 관리한다.**
