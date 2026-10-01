# Fetch API

> SSAFY Frontend 수업자료의 개념·요청·응답·GET/POST 예제를 정리했다.
> 오류 처리와 `async/await` 예제는 이해를 돕기 위한 보충 내용이다.

## 1. Fetch API란?

JavaScript에서 HTTP 요청을 보내고 응답을 처리하는 API이다. 기존 `XMLHttpRequest`보다 간결하게 요청을 작성할 수 있으며, **Promise를 기반으로 비동기 처리**한다.

- `fetch()`의 반환값은 `Promise<Response>`이다.
- 요청을 보내고 기다리는 동안 다른 JavaScript 코드를 실행할 수 있다.
- 응답을 받은 뒤 필요한 화면 부분만 갱신할 수 있다.

### 사용 예: 아이디 중복 확인

1. 사용자가 아이디를 입력한다.
2. 서버에 중복 확인 요청을 보낸다.
3. 서버 응답을 기다리는 동안 사용자는 다른 입력을 계속한다.
4. 응답 결과를 이용해 아이디 입력창 아래에 안내 문구를 표시한다.

> 비동기는 요청이 끝날 때까지 전체 작업을 멈추지 않고, 완료된 결과를 나중에 처리하는 방식이다. Fetch가 화면을 자동으로 변경하는 것은 아니며, 응답을 받은 코드에서 DOM을 수정해야 한다.

## 2. 요청 작성

```javascript
fetch(input, init);
```

| 인자    | 필수 여부 | 의미                                          |
| ------- | --------- | --------------------------------------------- |
| `input` | 필수      | 요청할 URL 또는 `Request` 객체                |
| `init`  | 선택      | 요청 방식, 헤더, 본문 등을 지정하는 옵션 객체 |

### 자주 사용하는 옵션

| 옵션      | 의미                             | 예시                                     |
| --------- | -------------------------------- | ---------------------------------------- |
| `method`  | HTTP 요청 메서드. 기본값은 `GET` | `"POST"`                                 |
| `headers` | 요청에 대한 부가 정보            | `{ "Content-Type": "application/json" }` |
| `body`    | 서버에 전달할 요청 본문          | `JSON.stringify(data)`                   |

```javascript
fetch("/api/posts", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ title: "첫 게시글" }),
});
```

`Content-Type`은 **지금 보내는 본문의 형식**을 서버에 알려준다. 객체를 JSON으로 보내려면 `JSON.stringify()`로 JSON 문자열을 만든다.

### 본문 형식과 Content-Type

| 본문 형식         | `body` 예시                        | Content-Type                        |
| ----------------- | ---------------------------------- | ----------------------------------- |
| JSON 문자열       | `JSON.stringify({ name: "John" })` | `application/json`                  |
| URL 인코딩 문자열 | `"name=John&age=10"`               | `application/x-www-form-urlencoded` |
| FormData          | `formData` 객체                    | 브라우저가 자동 설정                |

```javascript
const formData = new FormData();
formData.append("name", "John");

fetch("/api/members", {
  method: "POST",
  body: formData,
});
```

> 브라우저에서 FormData를 보낼 때는 `Content-Type`을 직접 지정하지 않는다. 브라우저가 데이터 구분에 필요한 boundary를 포함해 설정한다.

**보충:** Fetch의 `GET` 요청에는 `body`를 넣을 수 없다. 조회 조건은 URL의 쿼리 문자열로 전달한다.

```javascript
const params = new URLSearchParams({ keyword: "Java", page: "1" });
fetch(`/api/posts?${params}`);
```

## 3. 응답 처리는 두 단계

### 단계 1: Response 객체 받기

`fetch()`의 Promise가 이행되면 `Response` 객체를 받는다. 여기에서 상태 코드와 헤더 등을 확인할 수 있다.

| 속성               | 의미                                    |
| ------------------ | --------------------------------------- |
| `response.status`  | HTTP 상태 코드. 예: `200`, `404`, `500` |
| `response.ok`      | 상태 코드가 `200~299`이면 `true`        |
| `response.headers` | 응답 헤더                               |

**Response 객체 자체가 서버의 JSON 데이터인 것은 아니다.** 응답 본문을 따로 읽어야 한다.

### 단계 2: 응답 본문 읽기

| 메서드            | 읽어낸 결과                               |
| ----------------- | ----------------------------------------- |
| `response.json()` | JSON을 해석한 JavaScript 값. 객체·배열 등 |
| `response.text()` | 문자열                                    |
| `response.blob()` | 이미지·파일 등의 데이터를 담는 Blob       |

이 메서드들도 **Promise를 반환**하므로 `.then()` 또는 `await`로 결과를 받아야 한다.

```javascript
fetch("https://jsonplaceholder.typicode.com/posts/1")
  .then((response) => response.json()) // Response → 본문 읽기
  .then((data) => console.log(data)); // 읽기가 끝난 데이터 사용
```

첫 번째 `.then()`에서 `response.json()`을 반환하면, 다음 `.then()`은 본문 읽기가 완료된 결과를 받는다.

```javascript
// 중괄호를 쓰면 return을 직접 작성한다.
.then((response) => {
  return response.json();
})
```

## 4. 오류 처리

**HTTP 오류와 Fetch의 Promise 거부를 구분해야 한다.**

| 상황                                   | `fetch()`의 Promise | 처리 방법                        |
| -------------------------------------- | ------------------- | -------------------------------- |
| `200` 등 응답 수신                     | fulfilled           | 본문 읽기                        |
| `404`, `500` 응답 수신                 | fulfilled           | `response.ok` 또는 `status` 확인 |
| 네트워크 장애, 브라우저의 CORS 차단 등 | rejected            | `.catch()` 또는 `try/catch`      |

서버가 `404`나 `500`으로 응답해도 응답을 받았으므로, Fetch가 자동으로 `.catch()`에 보내지 않는다. HTTP 오류도 공통 오류 처리로 보내려면 직접 `throw`한다.

```javascript
fetch("https://jsonplaceholder.typicode.com/posts/1")
  .then((response) => {
    if (!response.ok) {
      throw new Error(`HTTP 오류: ${response.status}`);
    }
    return response.json();
  })
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error("요청 또는 응답 처리 실패:", error);
  });
```

`response.json()`에서 JSON 해석에 실패하는 경우도 이 체인의 `.catch()`에서 처리할 수 있다.

## 5. POST 예제

```javascript
fetch("https://jsonplaceholder.typicode.com/posts", {
  method: "POST",
  headers: {
    "Content-Type": "application/json; charset=UTF-8",
  },
  body: JSON.stringify({
    title: "foo",
    body: "bar",
    userId: 1,
  }),
})
  .then((response) => {
    if (!response.ok) {
      throw new Error(`HTTP 오류: ${response.status}`);
    }
    return response.json();
  })
  .then((data) => console.log(data))
  .catch((error) => console.error(error));
```

수업자료에서 제시한 응답 형태:

```javascript
{
  id: 101,
  title: "foo",
  body: "bar",
  userId: 1
}
```

> JSONPlaceholder는 연습용 API다. POST 응답으로 생성된 것처럼 보여주지만 데이터가 실제로 영구 저장되는 것은 아니다.

## 6. 보충: async/await로 작성하기

`.then()`으로 처리한 내용을 `async/await`로도 작성할 수 있다.

```javascript
async function getPost() {
  try {
    const response = await fetch(
      "https://jsonplaceholder.typicode.com/posts/1",
    );

    if (!response.ok) {
      throw new Error(`HTTP 오류: ${response.status}`);
    }

    const data = await response.json();
    console.log(data);
  } catch (error) {
    console.error("게시글 조회 실패:", error);
  }
}

getPost();
```

- 첫 번째 `await`: Response 객체를 기다린다.
- 두 번째 `await`: 응답 본문을 읽고 해석한 결과를 기다린다.
- `await`는 해당 async 함수의 진행을 잠시 멈추지만, 브라우저 전체를 멈추지는 않는다.

## 7. 복습 질문

| 질문                                    | 핵심 답변                                              |
| --------------------------------------- | ------------------------------------------------------ |
| Fetch의 반환값은?                       | `Promise<Response>`                                    |
| 기본 요청 메서드는?                     | `GET`                                                  |
| `.then()`을 두 번 쓰는 이유는?          | 응답 객체 수신과 본문 읽기가 각각 비동기 작업이기 때문 |
| JSON 요청 본문을 만드는 방법은?         | `JSON.stringify()`                                     |
| JSON 응답 본문을 읽는 방법은?           | `response.json()`                                      |
| 404 응답은 자동으로 catch에 들어가는가? | 아니다. 상태를 확인하고 직접 오류를 발생시켜야 한다.   |
| FormData의 Content-Type은?              | 브라우저에 자동 설정을 맡긴다.                         |

## 관련 주제와 참고자료

- [SOP와 CORS](./CORS.md)
- 선행 개념: HTTP 요청·응답, JSON, Promise
- 수업자료: 제공된 `Fetch API` PDF 1~2페이지
- [MDN: Using the Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)
- [JSONPlaceholder](https://jsonplaceholder.typicode.com/)
