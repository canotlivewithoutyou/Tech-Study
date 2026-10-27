# HTTP Request · Response와 Servlet API

> 학습일: 2026-10-27   
> 범위: HTTP 메시지, Method, 안전성·멱등성, Request/Response API, 상태 코드, Content-Type·Encoding

## 1. HTTP 메시지 구조

HTTP 통신은 **Request(요청)**와 **Response(응답)**로 이루어진다. 강의자료는 메시지 구조를 네 부분으로 설명한다.

1. **Start Line:** 요청 또는 응답의 기본 정보
2. **Headers:** 메시지 관련 부가 정보
3. **Empty Line:** 헤더와 본문을 구분하는 빈 줄
4. **Body:** 전달할 실제 데이터(없는 경우도 있음)

### HTTP Request 예시

```http
POST /api/data HTTP/1.1
Host: developer.mozilla.org
User-Agent: curl/8.6.0
Accept: */*
Content-Type: application/x-www-form-urlencoded

data=ABC123
```

| 구성                      | 설명                                  |
| ------------------------- | ------------------------------------- |
| `POST /api/data HTTP/1.1` | Method, 요청 경로, HTTP 버전          |
| `Host`                    | 요청 대상 서버                        |
| `User-Agent`              | 요청한 클라이언트 정보                |
| `Accept`                  | 클라이언트가 받을 수 있는 데이터 형식 |
| `Content-Type`            | 요청 본문의 데이터 형식               |
| `data=ABC123`             | 요청 본문 데이터                      |

### HTTP Response 예시

```http
HTTP/1.1 200 OK
Server: Apache
Content-Type: text/html
Cache-Control: no-store

<html>...</html>
```

| 구성               | 설명                  |
| ------------------ | --------------------- |
| `HTTP/1.1 200 OK`  | HTTP 버전과 상태 코드 |
| `Server`           | 서버 관련 정보        |
| `Content-Type`     | 응답 데이터 형식      |
| `Cache-Control`    | 캐시 관련 정책        |
| `<html>...</html>` | 응답 본문             |

## 2. HTTP Method

HTTP Method는 클라이언트가 서버에 어떤 작업을 요청하는지 나타낸다.

| Method  | 역할                       | 멱등성(강의자료 기준) |
| ------- | -------------------------- | --------------------- |
| GET     | 리소스 조회                | O                     |
| POST    | 데이터 제출 및 처리 요청   | X                     |
| PUT     | 리소스 생성 또는 전체 교체 | O                     |
| DELETE  | 리소스 삭제                | O                     |
| PATCH   | 리소스 일부 수정           | 일반적으로 X          |
| HEAD    | 응답 본문 없이 헤더 조회   | O                     |
| OPTIONS | 지원하는 통신 옵션 조회    | O                     |
| TRACE   | 요청 메시지 진단           | O                     |
| CONNECT | 대상 서버와 통신 터널 설정 | X                     |

### GET vs POST

| GET                                   | POST                                           |
| ------------------------------------- | ---------------------------------------------- |
| 서버 데이터 조회 중심                 | 데이터 제출·처리 요청 중심                     |
| Query String으로 파라미터 전달 가능   | HTML Form 데이터는 일반적으로 요청 본문에 포함 |
| HTML Form에서 `method` 생략 시 기본값 | 로그인, 데이터 등록 등에 활용                  |

### 안전성(Safe)과 멱등성(Idempotent)

- **안전한 메서드:** 요청의 본래 의미가 서버 상태를 변경하지 않는 메서드. 대표적으로 GET, HEAD, OPTIONS.
- **멱등성:** 같은 요청을 한 번 수행하든 여러 번 수행하든, **의도된 서버 상태의 최종 결과가 같은 성질**.

예를 들어 `DELETE /users/1`을 여러 번 보내도 최종적으로 해당 리소스가 삭제된 상태인 것은 같다. 반면 POST로 주문을 생성하면 여러 번 요청했을 때 여러 주문이 만들어질 수 있다.

> **주의:** 멱등성은 **매번 HTTP 응답이 같다는 뜻이 아니라 최종 서버 상태에 대한 개념**이다.

## 3. `HttpServletRequest`

클라이언트가 보낸 HTTP 요청을 추상화한 인터페이스로, **헤더·파라미터·속성·본문 등의 정보**를 제공한다.

```java
protected void doGet(
    HttpServletRequest request,
    HttpServletResponse response
) {
    // 요청 처리
}
```

### 주요 Request API

| 메서드                      | 기능                                         |
| --------------------------- | -------------------------------------------- |
| `getMethod()`               | HTTP 요청 메서드 반환                        |
| `getParameter(name)`        | 지정한 요청 파라미터 값 반환                 |
| `getParameterValues(name)`  | 같은 이름을 가진 파라미터 값들을 배열로 반환 |
| `getParameterNames()`       | 요청 파라미터 이름들 반환                    |
| `getParameterMap()`         | 전체 파라미터를 Map으로 반환                 |
| `getHeader(name)`           | 지정한 요청 헤더값 반환                      |
| `getHeaderNames()`          | 요청 헤더 이름들 반환                        |
| `setAttribute(name, value)` | 요청에 속성 저장                             |
| `getAttribute(name)`        | 요청의 속성 조회                             |
| `getSession()`              | 요청과 연관된 세션 반환                      |

### Request Parameter

Form 또는 Query String 등을 통해 클라이언트가 전달한 값이다.

```text
/hello?name=ssafy&age=23
```

```java
String name = request.getParameter("name"); // "ssafy"
String age = request.getParameter("age");   // "23"
```

**`getParameter()`의 반환형은 `String`**이므로 숫자 연산 시 별도로 형변환한다.

```java
int age = Integer.parseInt(request.getParameter("age"));
```

같은 이름의 값이 여러 개일 때:

```java
String[] values = request.getParameterValues("hobby");
```

전체 파라미터 조회:

```java
Map<String, String[]> params = request.getParameterMap();

params.forEach((key, values) -> {
    System.out.println(key + " : " + Arrays.toString(values));
});
```

- `Key`: 파라미터 이름
- `Value`: 해당 이름에 전달된 값들의 배열

### Parameter vs Attribute

| 구분   | Parameter                | Attribute                        |
| ------ | ------------------------ | -------------------------------- |
| 의미   | 클라이언트가 전달한 값   | 서버에서 요청에 저장·조회하는 값 |
| 조회   | `getParameter()`         | `getAttribute()`                 |
| 저장   | 클라이언트 요청으로 전달 | `setAttribute()`                 |
| 반환형 | `String`                 | `Object`                         |

```java
String name = request.getParameter("name");
request.setAttribute("message", "Hello");
String message = (String) request.getAttribute("message");
```

## 4. `HttpServletResponse`

클라이언트에 보낼 HTTP 응답을 추상화한 인터페이스이다. 응답 상태 코드, 헤더, 본문 등을 설정한다.

| 메서드             | 역할                     |
| ------------------ | ------------------------ |
| `setStatus()`      | HTTP 응답 상태 코드 설정 |
| `setHeader()`      | 응답 헤더 설정           |
| `setContentType()` | 응답 Content-Type 설정   |
| `getWriter()`      | 문자 응답 스트림 반환    |
| `addCookie()`      | 응답에 쿠키 추가         |
| `sendRedirect()`   | 지정한 URL로 리다이렉트  |
| `sendError()`      | 오류 응답 전송           |

```java
response.setContentType("text/html;charset=UTF-8");
response.getWriter().append("<h1>Hello Servlet</h1>");
```

- `setContentType()`: 응답 형식과 인코딩 설정
- `getWriter()`: 문자 데이터를 클라이언트에 출력

## 5. HTTP Status Code

서버가 요청을 처리한 결과를 숫자 코드로 나타낸다.

| 분류              | 의미                      |
| ----------------- | ------------------------- |
| 1xx Informational | 요청 처리 중 정보 전달    |
| 2xx Success       | 요청 처리 성공            |
| 3xx Redirection   | 추가 이동 또는 처리 필요  |
| 4xx Client Error  | 클라이언트 요청 관련 오류 |
| 5xx Server Error  | 서버 처리 중 오류         |

| 코드 | 이름                  | 의미                                    |
| ---- | --------------------- | --------------------------------------- |
| 200  | OK                    | 요청 정상 처리                          |
| 201  | Created               | 새로운 리소스 생성 성공                 |
| 302  | Found                 | 요청한 리소스가 임시로 다른 위치에 있음 |
| 400  | Bad Request           | 잘못된 요청                             |
| 401  | Unauthorized          | 인증 필요 또는 인증 실패                |
| 403  | Forbidden             | 접근 권한 없음                          |
| 404  | Not Found             | 요청한 리소스를 찾을 수 없음            |
| 405  | Method Not Allowed    | 허용되지 않은 HTTP Method 사용          |
| 500  | Internal Server Error | 서버 내부 오류                          |

**구분해서 암기하기**

- `401` **인증** 문제 ↔ `403` **접근 권한** 문제
- `400` **잘못된 요청** ↔ `404` **리소스를 찾지 못함**
- `200` **일반 요청 성공** ↔ `201` **새 리소스 생성 성공**

## 6. Content-Type과 Character Encoding

### Content-Type

전달되는 데이터가 어떤 형식인지 나타내는 HTTP 헤더이다.

| Content-Type                        | 용도                            |
| ----------------------------------- | ------------------------------- |
| `text/html`                         | HTML 문서                       |
| `application/json`                  | JSON 데이터                     |
| `image/jpeg`                        | JPEG 이미지                     |
| `multipart/form-data`               | 파일 업로드 및 복합 Form 데이터 |
| `application/x-www-form-urlencoded` | 일반적인 Form 데이터 전송       |

### Character Encoding

문자를 컴퓨터가 처리·전송할 수 있도록 바이트로 변환하는 규칙이다. 강의자료는 기본 응답 인코딩으로 설명한 ISO-8859-1이 한글 표현에 적합하지 않으므로 **한글 응답에서는 UTF-8을 명시**해야 한다고 설명한다.

```java
response.setContentType("text/html;charset=UTF-8");
response.getWriter().append(
    "<html><body>한글도 문제 없지</body></html>"
);
```

**실행 순서:** 응답 본문을 출력하기 전에 Content-Type과 인코딩을 설정한다.

```java
// 1. 응답 형식·인코딩 지정
response.setContentType("text/html;charset=UTF-8");
// 2. 문자 출력
response.getWriter().print("안녕하세요");
```

출력이 시작된 뒤에는 헤더·인코딩 설정이 제대로 반영되지 않을 수 있다.

## 핵심 정리

- **Request:** 클라이언트 → 서버, `HttpServletRequest`로 정보 조회
- **Response:** 서버 → 클라이언트, `HttpServletResponse`로 응답 설정·출력
- **Parameter:** 클라이언트의 전달 값 / **Attribute:** 서버에서 저장하는 속성
- **Content-Type:** 데이터 형식 / **Encoding:** 문자 인코딩 규칙
