# Front Controller Pattern

> 학습일: 2026-10-27   
> 범위: 단일 진입점, 요청 분배, action 파라미터, URL Mapping, ControllerHelper, 구구단 실습

## 1. 기존 Servlet 방식의 문제

요청마다 Servlet이 1:1로 매칭되면 애플리케이션이 커질수록 다음 문제가 발생한다.

| 문제            | 설명                                       |
| --------------- | ------------------------------------------ |
| Servlet 증가    | 새 요청마다 새로운 Servlet 생성 필요       |
| 코드 중복       | 로깅·예외 처리·응답 설정 등 공통 기능 반복 |
| 복잡도 증가     | 요청 처리 과정이 개별 Servlet로 흩어짐     |
| 유지보수 어려움 | 공통 기능 수정 시 여러 Servlet 변경 필요   |

## 2. Front Controller란?

**모든 요청을 하나의 진입점에서 받아 공통 작업을 수행한 뒤, 요청별 기능으로 분배하는 디자인 패턴**이다. 요청의 모든 비즈니스 로직을 직접 처리하는 것이 아니라 **공통 처리와 요청 분배**를 담당한다.

```text
Client (요청)
   ▼
Front Controller
  ├── 로깅·파라미터 확인·응답 설정
  └── 요청 종류에 따른 분기
       ├── 구구단 처리
       ├── 덧셈 처리
       └── 로그인 처리
```

| 장점        | 설명                                             |
| ----------- | ------------------------------------------------ |
| 단일 진입점 | 요청을 한곳에서 접수                             |
| 공통 처리   | 인증·권한 검사·로깅 등의 전후처리 일관화         |
| 확장성      | 기존 구조를 크게 바꾸지 않고 요청 처리 기능 추가 |
| 코드 간결성 | Servlet 중복 코드 감소                           |
| 유지보수성  | 요청 처리 구조가 일관되어 관리 용이              |

> **핵심:** Front Controller는 공통 처리·분배, 개별 메서드나 Controller는 실제 비즈니스 로직 처리.

## 3. 요청을 구분하는 방법

### 방법 ① `action` 파라미터

```text
/main?action=gugu&dan=3
/main?action=hello
/main?action=add&num1=10&num2=20
```

| 파라미터       | 역할             |
| -------------- | ---------------- |
| `action`       | 실행할 작업 구분 |
| `dan`          | 구구단의 단수    |
| `num1`, `num2` | 덧셈할 숫자      |

```java
String action = request.getParameter("action");

switch (action) {
    case "gugu" -> gugu(request, response);
    case "hello" -> hello(request, response);
    case "add" -> add(request, response);
}
```

### 방법 ② 와일드카드 URL Mapping

| 매핑       | 의미                              | 예시                           |
| ---------- | --------------------------------- | ------------------------------ |
| `/main/*`  | `/main/` 하위 경로 처리           | `/main/hello`                  |
| `/board/*` | `/board/` 하위 경로 처리          | `/board/regist`, `/board/list` |
| `*.do`     | `.do` 확장자 요청 처리            | `/hello.do`, `/gugu.do`        |
| `/`        | 별도 매핑이 없는 요청의 기본 매핑 | 기본 Servlet 매핑              |

- `getPathInfo()`: `/main/*` 같은 경로 기반 매핑에서 **추가 경로** 확인
- `getServletPath()`: Servlet 매핑에 해당하는 **요청 경로** 확인

**주의:** `/main/*` 매핑은 정적 리소스 요청까지 해당 Servlet으로 들어올 수 있고, `*.do` 매핑은 확장자를 기준으로 하므로 경로 구분에 제약이 있다.

## 4. Front Controller 구현

강의자료에서는 `MainFrontController`(`/main`)와 `MemberFrontController`(`/member`)를 예로 들고 공통 기능을 `ControllerHelper` 인터페이스로 분리한다.

### ① `ControllerHelper`의 `default` 메서드

```java
public interface ControllerHelper {
    default String getActionParameter(
        HttpServletRequest req,
        HttpServletResponse resp
    ) {
        String action = req.getParameter("action");
        if (action == null || action.isBlank()) {
            action = "index";
        }
        return action;
    }

    default void responseHtml(
        String title,
        String content,
        HttpServletResponse resp
    ) throws IOException {
        String html = "<html><body><h1>%s</h1>%s</body></html>"
                .formatted(title, content);

        resp.setContentType("text/html;charset=UTF-8");
        resp.getWriter().write(html);
    }
}
```

- `getActionParameter()`: `action` 값 추출, 없거나 비어 있으면 `index`로 처리
- `responseHtml()`: HTML 문자열 생성 및 응답의 Content-Type·인코딩 설정 후 출력
- 강의자료에는 `getActionParameter()`에서 **요청 URI, HTTP Method, 파라미터 목록을 출력하는 로깅**도 포함되어 있다.

### ② `MainFrontController` 예시

```java
@WebServlet("/main")
public class MainFrontController
        extends HttpServlet
        implements ControllerHelper {

    @Override
    protected void doGet(
        HttpServletRequest req,
        HttpServletResponse resp
    ) throws ServletException, IOException {
        String action = getActionParameter(req, resp);
        switch (action) {
            case "gugu" -> gugu(req, resp);
        }
    }

    @Override
    protected void doPost(
        HttpServletRequest req,
        HttpServletResponse resp
    ) throws ServletException, IOException {
        String action = getActionParameter(req, resp);
        switch (action) {
            case "login" -> login(req, resp);
        }
    }

    protected void gugu(
        HttpServletRequest req,
        HttpServletResponse resp
    ) throws ServletException, IOException {
        // 구구단 처리
    }

    protected void login(
        HttpServletRequest req,
        HttpServletResponse resp
    ) throws ServletException, IOException {
        // 로그인 처리
    }
}
```

| 요소                          | 역할                      |
| ----------------------------- | ------------------------- |
| `@WebServlet("/main")`        | `/main` 요청 Servlet 등록 |
| `extends HttpServlet`         | HTTP 요청 처리            |
| `implements ControllerHelper` | 공통 메서드 활용          |
| `doGet()` / `doPost()`        | HTTP Method별 요청 처리   |
| `switch(action)`              | 작업별 처리 메서드 호출   |

## 5. 실습 — 구구단 출력

**요청:** `/BE_02/main?action=gugu&dan=7`

```java
protected void gugu(
    HttpServletRequest request,
    HttpServletResponse response
) throws ServletException, IOException {
    int dan = Integer.parseInt(request.getParameter("dan"));
    String content = "";

    for (int i = 1; i < 10; i++) {
        content += String.format("%d * %d = %d<br>", dan, i, dan * i);
    }

    responseHtml("구구단(" + dan + ")", content, response);
}
```

**실행 흐름**

1. `/main?action=gugu&dan=7` 요청
2. `MainFrontController.doGet()` 실행
3. `getActionParameter()`로 `gugu` 추출
4. `switch`에서 `gugu()` 호출
5. `getParameter("dan")`으로 `7` 추출 후 정수 변환
6. 반복문으로 7단 생성
7. `responseHtml()`로 HTML 응답

## 핵심 정리

같은 `/main` URL을 사용해도 `action`에 따라 실행 기능이 달라질 수 있다. Front Controller는 **하나의 진입점에서 공통 처리하고 요청별 실제 처리 기능으로 분배**한다.
