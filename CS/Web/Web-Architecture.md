# 웹 프로그램과 3-Tier Architecture

> 학습일: 2026-10-27  
> 출처: `Backend - Servlet.pdf`, pp. 1–5  
> 범위: 웹 프로그램의 동작, Frontend/Backend, 3-Tier Architecture, Web Server·WAS·Container·Context

## 1. 웹 프로그램(Web Program)

웹 프로그램은 네트워크 너머의 서버에 존재하며, 웹을 통해 사용자에게 서비스를 제공하는 프로그램이다.

| 구분 | Frontend | Backend |
| --- | --- | --- |
| 역할 | 사용자 인터페이스 제공 | 비즈니스 로직 처리 |
| 주요 기술 | HTML, CSS, JavaScript | Java, Servlet |
| 주요 기능 | 화면 구성, 스타일링, 이벤트 처리 | 요청 처리, 데이터 가공, DB 연동 |
| 실행 환경 | 주로 웹 브라우저 | 서버 |

**Frontend의 구성 요소**

| 요소 | 역할 | 주요 기능 |
| --- | --- | --- |
| Content | 페이지 구조와 내용 | 텍스트, 이미지, 링크 |
| Presentation | 페이지 디자인 | 색상, 폰트, 위치, 배치 |
| Behavior | 사용자 상호작용 | 이벤트 처리, 유효성 검사 |

### 웹 프로그램의 기본 동작

```text
클라이언트(브라우저)
   │ HTTP Request
   ▼
서버
   │ 비즈니스 로직 처리 (필요 시 DB 접근)
   ▼
응답 생성
   │ HTTP Response
   ▼
클라이언트
```

핵심은 **클라이언트가 요청(Request)을 보내면 서버가 처리 결과를 응답(Response)**한다는 것이다.

## 2. 3-Tier Architecture

애플리케이션을 **3개의 논리적 계층**으로 분리해 개발·운영하는 구조이다. 계층을 분리하면 유지보수와 확장에 유리하고 필요에 따라 독립적으로 관리할 수 있다.

| 계층 | 역할 | 주요 구성 |
| --- | --- | --- |
| Presentation Tier | 사용자에게 화면과 결과 제공 | Web Server, HTML, CSS, JS |
| Application Tier | 동적 요청 및 비즈니스 로직 처리 | WAS, Servlet, Java |
| Data Tier | 데이터 저장 및 조회 | DB Server, MySQL |

```text
Client
  │ HTTP 요청
  ▼
Web Server
  ├── 정적 요청(HTML, CSS, JS) → 정적 리소스 응답
  │
  └── 동적 요청
        ▼
       WAS
        ▼
      Servlet
        ▼
   Business Logic
        ▼
  Persistence Logic
        │ JDBC
        ▼
     DB Server
```

### 정적 요청과 동적 요청

| 구분 | 정적 요청(Static Request) | 동적 요청(Dynamic Request) |
| --- | --- | --- |
| 개념 | 서버에 이미 존재하는 리소스 요청 | 사용자 요청에 따라 결과가 달라지는 요청 |
| 예시·특징 | HTML, CSS, JavaScript, 이미지 등 리소스 반환 | Servlet에서 처리하며 비즈니스 로직·DB 연동이 필요할 수 있음 |

> **핵심:** 강의에서 다룬 구조에서는 Web Server가 정적 리소스 제공을, WAS가 동적 요청 처리를 담당한다.

## 3. WAS · Servlet Container · Context

### WAS (Web Application Server)

웹 애플리케이션을 실행하고 HTTP 요청에 따라 **동적인 웹 서비스**를 제공하는 서버이다.

- 서버는 네트워크에서 접근할 수 있는 IP 주소를 가진다.
- HTTP 기반으로 클라이언트와 통신한다.
- HTTP는 일반적으로 80 포트를 사용하며 개발 환경에서는 8080 포트를 많이 사용한다.
- 하나의 WAS에서 여러 웹 애플리케이션을 실행할 수 있다.
- 대표 예: **Apache Tomcat**

### Servlet Container

Servlet의 **생성, 실행, 소멸을 관리하는 실행 환경**이다.

1. Servlet 객체의 생명주기 관리
2. HTTP 요청 수신 및 Servlet 연결
3. Request/Response 객체 제공
4. 다중 요청 처리를 위한 스레드 관리

강의에서는 Tomcat과 같은 WAS가 Servlet Container의 역할을 수행한다는 점을 강조한다.

### Context와 Context Path

**Context**는 각 웹 애플리케이션의 실행 환경과 관련 정보를 제공하는 단위이다. 하나의 WAS에서 여러 Context를 운영할 수 있다.

```text
Tomcat (localhost:8080)
 ├── /contextA
 │    ├── Servlet
 │    └── Web Resources
 └── /contextB
      ├── Servlet
      └── Web Resources
```

**Context Path**는 웹 애플리케이션을 구별하는 URL 경로이다.

```text
http://localhost:8080/contextA/hello
```

| 구성 | 의미 |
| --- | --- |
| `http` | 통신 프로토콜 |
| `localhost` | 서버 주소 |
| `8080` | 포트 번호 |
| `/contextA` | Context Path |
| `/hello` | Servlet URL Mapping |

| 개념 | 구분 |
| --- | --- |
| Container Root | WAS 서버의 기본 URL |
| Context Root | 특정 웹 애플리케이션의 시작 경로 |
| Servlet URL Mapping | 애플리케이션 내부에서 요청을 처리할 Servlet 경로 |

## 핵심 정리

- **Web Server vs WAS:** 정적 리소스 제공 중심 / 동적 웹 애플리케이션 처리
- **Servlet vs Servlet Container:** HTTP 요청을 처리하는 컴포넌트 / 컴포넌트의 실행과 생명주기를 관리하는 환경
- **Context Path vs URL Mapping:** 웹 애플리케이션 경로 / 그 내부 Servlet 경로
