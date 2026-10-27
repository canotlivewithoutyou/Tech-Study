# Servlet Listener와 ServletContextListener

> 학습일: 2026-10-27   
> 범위: Listener 개념·종류, ServletContextListener, 등록 방법, Servlet과의 차이

## 1. Listener란?

**Listener**는 웹 애플리케이션에서 발생하는 이벤트를 감지해 그에 맞는 작업을 수행하는 객체이다.

- Servlet: **HTTP 요청 처리**
- Filter: **요청과 응답을 가로채 전후 처리**
- Listener: **웹 애플리케이션 내부 이벤트 감지**

대표적인 이벤트:

- 웹 애플리케이션 시작 및 종료
- HTTP Session 생성 및 소멸
- Request 생성 및 소멸
- 각 영역의 Attribute 변경

## 2. Listener 종류

| 구분               | 주요 인터페이스                   | 역할                                |
| ------------------ | --------------------------------- | ----------------------------------- |
| Servlet Context    | `ServletContextListener`          | 웹 애플리케이션 생성·소멸 감지      |
| Context Attribute  | `ServletContextAttributeListener` | Context 속성 변경 감지              |
| HTTP Session       | `HttpSessionListener`             | Session 생성·소멸 감지              |
| Session Attribute  | `HttpSessionAttributeListener`    | Session 속성 변경 감지              |
| Session Activation | `HttpSessionActivationListener`   | Session 활성화·비활성화 이벤트 감지 |
| Session Binding    | `HttpSessionBindingListener`      | 객체의 Session 바인딩 이벤트 감지   |
| Servlet Request    | `ServletRequestListener`          | Request 생성·소멸 감지              |
| Request Attribute  | `ServletRequestAttributeListener` | Request 속성 변경 감지              |

## 3. `ServletContextListener`

웹 애플리케이션의 **생성부터 소멸까지의 주요 이벤트**를 감지하는 Listener이다. 강의자료는 여러 Listener 중 이를 중심으로 설명한다.

**활용 목적**

- 애플리케이션 초기 설정
- 공통 자원 초기화
- 애플리케이션 종료 시 자원 정리

개별 Servlet이 실행되기 전에 초기화해야 하는 공유 자원에 활용할 수 있다.

### 주요 메서드와 흐름

| 메서드                 | 호출 시점            | 역할        |
| ---------------------- | -------------------- | ----------- |
| `contextInitialized()` | 웹 애플리케이션 시작 | 초기화 작업 |
| `contextDestroyed()`   | 웹 애플리케이션 종료 | 정리 작업   |

```text
웹 애플리케이션 시작
       ▼
contextInitialized()
       ▼
웹 애플리케이션 동작
       ▼
contextDestroyed()
       ▼
웹 애플리케이션 종료
```

## 4. Listener 등록 및 구현

### ① `@WebListener` 이용

```java
@WebListener
public class MyContextListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println(
            "context 초기화 중입니다. 전체 애플리케이션 초기화와 관련된 작업 진행"
        );
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println(
            "context 종료합니다. 정리할 작업들 진행해주세요."
        );
    }
}
```

| 코드                                | 의미                                   |
| ----------------------------------- | -------------------------------------- |
| `@WebListener`                      | Listener 등록                          |
| `implements ServletContextListener` | 애플리케이션 생명주기 이벤트 감지      |
| `contextInitialized()`              | 시작 시 실행                           |
| `contextDestroyed()`                | 종료 시 실행                           |
| `ServletContextEvent`               | 발생한 Context 이벤트의 관련 정보 전달 |

### ② `web.xml` 이용

```xml
<listener>
    <listener-class>com.ssafy.live.listener.MyContextListener</listener-class>
</listener>
```

`@WebListener` 대신 배포 서술자인 `web.xml`로 등록할 수도 있다.

## 5. Servlet vs Listener

| 비교        | Servlet               | Listener                                     |
| ----------- | --------------------- | -------------------------------------------- |
| 주요 역할   | 클라이언트 요청 처리  | 이벤트 감지·대응                             |
| 실행 기준   | HTTP 요청             | 애플리케이션·세션·요청 등의 이벤트           |
| 대표 메서드 | `doGet()`, `doPost()` | `contextInitialized()`, `contextDestroyed()` |
| 사용 목적   | 비즈니스 로직 수행    | 초기화·종료 처리·상태 변화 감지              |

> **핵심:** Listener는 클라이언트가 직접 호출하는 것이 아니라 **특정 이벤트가 발생하면 Servlet Container에 의해 호출**된다.
