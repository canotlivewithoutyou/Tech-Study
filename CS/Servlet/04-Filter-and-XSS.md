# Servlet Filter · Filter Chain · XSS

> 학습일: 2026-10-27    
> 범위: Filter 역할, Filter Chain, Mapping·Dispatcher Type, 등록·생명주기, XSSFilter·RequestWrapper

## 1. Filter란?

**Filter**는 Servlet/JSP 등 웹 컴포넌트가 실행되기 **전후에 Request와 Response를 가로채 공통·부가 기능을 수행하는 웹 컴포넌트**이다.

각 Controller마다 구현하기 쉬운 인증, 로깅, 보안 등의 기능을 별도로 분리할 수 있다.

| 주요 용도            | 설명                            |
| -------------------- | ------------------------------- |
| 인증(Authentication) | 로그인 상태 확인                |
| 인가(Authorization)  | 특정 기능 접근 권한 확인        |
| 로깅(Logging)        | 요청·응답 정보 기록             |
| 보안(Security)       | 악의적인 입력·공격 검사 및 차단 |
| 요청·응답 처리       | 공통 요청·응답 데이터 처리      |

**활용 예:** 미로그인·권한 부족 사용자 접근 제한, 필요 시 로그인 페이지로 Redirect, 오류 분석·행동 분석·요청 추적을 위한 로깅, XSS 방지.

## 2. Filter Chain

**Filter Chain**은 여러 Filter를 연결해 순서대로 실행하도록 하는 구조이다.

```text
Client
  │
  ▼
Filter 1 (요청 검사)
  ▼
Filter 2 (로깅)
  ▼
Filter 3 (인증 확인)
  ▼
Servlet / JSP
  │
  ▼
Filter 3 (응답 처리)
  ▼
Filter 2 (응답 처리)
  ▼
Filter 1 (응답 처리)
  ▼
Client
```

- **요청 단계:** Servlet 실행 전에 요청 검사·수정
- **응답 단계:** Servlet 처리 이후 응답 검사·수정
- **특징:** Filter는 **요청과 응답에 모두 관여**할 수 있다.

### `chain.doFilter()` 중심으로 이해하기

```java
public void doFilter(
    HttpServletRequest request,
    HttpServletResponse response,
    FilterChain chain
) throws IOException, ServletException {
    // 1. 요청 전처리
    System.out.println("filter 진입");

    // 2. 다음 Filter 또는 Servlet 호출
    chain.doFilter(request, response);

    // 3. 응답 후처리
    System.out.println("응답 수신");
}
```

1. `chain.doFilter()` 호출 전: **요청 전처리**
2. `chain.doFilter()`: 다음 Filter가 있으면 해당 Filter, 없으면 Servlet 호출
3. 다음 단계 완료 후: 현재 Filter로 돌아와 **응답 후처리**

> **중요:** `chain.doFilter()`를 호출하지 않으면 뒤의 Filter나 Servlet으로 요청이 진행되지 않는다. 인증 실패 등에서 요청을 차단할 수 있다. 응답이 이미 커밋된 이후에는 `forward()`나 `redirect()`가 제한될 수 있다.

## 3. Filter Mapping과 Dispatcher Type

### 적용 대상 설정

- **Servlet 기준:** 특정 Servlet에 적용
- **URL Pattern 기준:** 해당 URL에 접근하는 요청에 적용

| URL Pattern | 대상               |
| ----------- | ------------------ |
| `/*`        | 전체 경로          |
| `/main/*`   | `/main/` 하위 경로 |
| `*.do`      | `.do` 확장자 요청  |

### Dispatcher Type

Filter가 적용될 **요청 처리 유형**을 지정한다.

| 유형      | 적용 상황                           |
| --------- | ----------------------------------- |
| `REQUEST` | 클라이언트의 직접 요청 (기본값)     |
| `FORWARD` | `forward()`로 다른 자원에 이동      |
| `INCLUDE` | `include()`로 다른 자원의 결과 포함 |
| `ERROR`   | 에러 페이지로 전달                  |

**주의:** `REQUEST`만 설정하면 `forward()`로 이동한 요청에는 해당 Filter가 적용되지 않는다.

## 4. Filter 등록 방법

Filter는 `HttpFilter`를 상속해 구현할 수 있다.

### ① `@WebFilter` 애너테이션

```java
@WebFilter(
    dispatcherTypes = {DispatcherType.REQUEST},
    urlPatterns = {"/*"}
)
public class PerformanceFilter extends HttpFilter {
    public void doFilter(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain chain
    ) throws IOException, ServletException {
        // Filter 처리
    }
}
```

| 요소              | 역할             |
| ----------------- | ---------------- |
| `@WebFilter`      | Filter 등록      |
| `dispatcherTypes` | 적용할 요청 유형 |
| `urlPatterns`     | 적용할 URL       |
| `doFilter()`      | 실제 필터 기능   |

### ② `web.xml` 설정

```xml
<filter>
    <filter-name>PerformanceFilter</filter-name>
    <filter-class>com.ssafy.live.filter.PerformanceFilter</filter-class>
</filter>
<filter-mapping>
    <filter-name>PerformanceFilter</filter-name>
    <url-pattern>/*</url-pattern>
    <dispatcher>REQUEST</dispatcher>
</filter-mapping>
```

`@WebFilter`와 `web.xml` 모두 Filter를 등록하고 적용 범위를 설정하는 방법이다.

## 5. Filter 생명주기

```text
Filter 객체 생성
      ▼
    init()
      ▼
  doFilter()   ← 요청 처리 시 반복 호출
      ▼
  destroy()
```

| 메서드       | 역할                         |
| ------------ | ---------------------------- |
| 생성자       | Filter 객체 생성             |
| `init()`     | Filter 초기화                |
| `doFilter()` | 요청·응답에 대한 실제 필터링 |
| `destroy()`  | 종료 시 정리 작업            |

## 6. XSS와 XSSFilter

### XSS(Cross-Site Scripting)

사용자 입력 등에 악의적인 스크립트를 넣어 **웹페이지에서 실행되게 하는 공격**이다.

```html
<script>alert('계좌이체 감사')</script>
```

입력 문자열이 적절히 처리되지 않은 상태로 HTML에 출력되면 브라우저가 스크립트로 해석할 수 있다.

**강의자료의 문자 치환 예시**

| 입력 문자 | 치환 문자열 |
| --------- | ----------- |
| `<`       | `&lt;`      |
| `>`       | `&gt;`      |

이렇게 치환하면 `<script>`와 같은 문자열이 HTML 태그로 직접 해석되는 것을 막는 데 도움이 된다.

### ① Filter에서 RequestWrapper 적용

```java
@WebFilter("/*")
public class XssRequestFilter extends HttpFilter {
    public void doFilter(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain chain
    ) throws IOException, ServletException {
        chain.doFilter(new XssRequestWrapper(request), response);
    }
}
```

원본 Request가 아니라 `XssRequestWrapper`로 감싼 요청 객체를 다음 단계에 전달한다.

### ② `HttpServletRequestWrapper` 구현

```java
class XssRequestWrapper extends HttpServletRequestWrapper {
    public XssRequestWrapper(HttpServletRequest request) {
        super(request);
    }

    @Override
    public String getParameter(String name) {
        String value = super.getParameter(name);
        if (value != null) {
            value = value.replace("<", "&lt;")
                         .replace(">", "&gt;");
        }
        return value;
    }
}
```

**동작 흐름**

1. 입력값이 포함된 요청 전송
2. `XssRequestFilter` 실행
3. `XssRequestWrapper`로 Request 감싸기
4. Servlet에서 `getParameter()` 호출
5. Wrapper의 재정의된 메서드 실행
6. `<`, `>`가 치환된 문자열 반환

### Wrapper Pattern

**기존 객체를 감싸 특정 동작을 재정의하거나 추가하는 방식**이다. `HttpServletRequestWrapper`를 상속하면 기존 Request 기능을 유지하면서 `getParameter()` 등의 동작을 변경할 수 있다.

> **보안 주의:** 강의자료의 구현은 Wrapper 사용법을 설명하기 위한 **단순 예제**이다. `<`, `>`만 치환하거나 `getParameter()`만 재정의하는 것으로 모든 XSS를 막을 수 없다. 실제 서비스에서는 출력되는 HTML 문맥에 맞는 이스케이프 처리 등의 대책이 필요하다.

## 핵심 정리

- **Filter:** Servlet의 요청·응답 앞뒤에 공통 기능 적용
- **Filter Chain:** 여러 Filter 순차 실행, 응답 단계는 역순으로 돌아옴
- **`chain.doFilter()`:** 다음 Filter 또는 Servlet에 요청 전달
- **Dispatcher Type:** Filter가 적용되는 요청 처리 유형
- **Wrapper:** 기존 요청 객체를 감싸 메서드 동작 변경
