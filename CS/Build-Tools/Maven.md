# Maven과 Java 웹 프로젝트 관리

> 학습일: 2026-10-27  
> 범위: Maven의 역할, `pom.xml`, 디렉터리 구조, 의존성 관리

## 1. Maven이란?

**Maven**은 Java 기반 프로젝트의 **빌드 자동화와 의존성 관리**에 사용하는 도구이다. 프로젝트마다 필요한 라이브러리와 버전을 `pom.xml`에 설정하여 관리한다.

| 기능                 | 설명                                    |
| -------------------- | --------------------------------------- |
| 프로젝트 구조 표준화 | 일관된 디렉터리 구조 제공               |
| 의존성 관리          | 필요한 라이브러리와 버전 관리           |
| 빌드 자동화          | 컴파일, 테스트, 패키징 등의 과정 자동화 |

## 2. `pom.xml`

**POM = Project Object Model**. 프로젝트 정보와 라이브러리 등을 XML로 정의하는 Maven의 핵심 설정 파일이다.

| 요소           | 의미                              |
| -------------- | --------------------------------- |
| `groupId`      | 프로젝트가 속한 그룹의 식별자     |
| `artifactId`   | 프로젝트 또는 산출물의 식별자     |
| `version`      | 프로젝트 버전                     |
| `packaging`    | 패키징 방식                       |
| `properties`   | 공통 설정값 및 버전 변수          |
| `dependencies` | 프로젝트에 필요한 라이브러리 목록 |

```xml
<groupId>com.ssafy</groupId>
<artifactId>BE_01_maven</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>war</packaging>
```

- `groupId`와 `artifactId`를 조합해 프로젝트를 식별할 수 있다.
- `war`는 웹 애플리케이션을 배포하기 위한 패키징 방식이다.

## 3. Maven 기본 디렉터리 구조

```text
project/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/          # Java 소스 코드
│   │   ├── resources/     # 설정 및 리소스
│   │   └── webapp/        # 웹 리소스
│   │       ├── WEB-INF/
│   │       │   ├── lib/
│   │       │   └── web.xml
│   │       └── index.html
│   └── test/
│       ├── java/
│       └── resources/
└── target/                # 빌드 결과물
```

| 경로              | 역할                                      |
| ----------------- | ----------------------------------------- |
| `src/main/java`   | Servlet 등의 Java 코드 작성               |
| `src/main/webapp` | HTML, CSS, JavaScript, JSP 등의 웹 리소스 |
| `WEB-INF/web.xml` | 웹 애플리케이션 배포 설정                 |
| `pom.xml`         | 의존성 및 빌드 관리                       |
| `target`          | Maven 빌드 결과물                         |

## 4. 의존성(Dependency) 관리

필요한 라이브러리를 `pom.xml`에 선언하면 Maven이 관리한다.

**Servlet API 의존성 예시(강의자료):**

```xml
<dependencies>
    <dependency>
        <groupId>jakarta.servlet</groupId>
        <artifactId>jakarta.servlet-api</artifactId>
        <version>6.1.0</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

| 요소         | 역할                   |
| ------------ | ---------------------- |
| `groupId`    | 라이브러리 제공 그룹   |
| `artifactId` | 라이브러리 이름        |
| `version`    | 라이브러리 버전        |
| `scope`      | 의존성이 적용되는 범위 |

### `provided` Scope

**컴파일 시에는 필요하지만 실행 환경에서 제공될 것으로 가정하는 의존성**에 사용한다. Servlet API는 Tomcat과 같은 Servlet Container에서 제공하므로 `provided`로 지정할 수 있다.

### `properties`로 버전 관리

```xml
<properties>
    <jdk.version>21</jdk.version>
    <maven.compiler.release>${jdk.version}</maven.compiler.release>
    <jakarta.servlet.version>6.1.0</jakarta.servlet.version>
</properties>
```

```xml
<version>${jakarta.servlet.version}</version>
```

- `${변수명}` 형태로 설정값을 재사용한다.
- 버전을 한 곳에서 관리하면 설정 변경이 편리하다.
- 강의자료에는 Servlet API인 `jakarta.servlet-api`와 JSP API인 `jakarta.servlet.jsp-api`가 등장한다.
- 라이브러리 검색 사이트: [Maven Repository](https://mvnrepository.com/)