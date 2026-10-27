# Servlet Multi-Thread와 공유 자원 동기화

> 학습일: 2026-10-27  
> 범위: Servlet 멀티스레드 구조, Race Condition, Mutex·Semaphore, `synchronized`, 상태 관리

## 1. Servlet의 Multi-Thread 구조

Servlet은 일반적으로 **하나의 Servlet 인스턴스를 여러 Thread가 공유**하는 방식으로 동작한다. 요청마다 새로운 Servlet 객체가 생성되는 것이 아니라 여러 Thread가 같은 객체의 메서드를 실행한다.

```text
Client A 요청 ── Thread A ──┐
                           ▼
                        MyServlet
                           ▲
Client B 요청 ── Thread B ──┘

            하나의 Servlet 객체 공유
```

장점은 여러 요청을 동시에 처리할 수 있다는 것이다. 하지만 **동일한 데이터를 여러 Thread가 수정하면 문제가 발생할 수 있다.**

## 2. Race Condition(경쟁 상태)

**Race Condition:** 여러 Thread의 실행 순서에 따라 결과가 달라질 수 있는 상황.

### 예시 — 재고가 1개일 때

```java
private long stock = 1;
```

강의자료의 구매 처리 예시(핵심 부분):

```java
protected void buy(
    HttpServletRequest request,
    HttpServletResponse response
) throws ServletException, IOException {
    if (stock > 0) {
        try {
            Thread.sleep(1000 * 10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        stock--;
        responseHtml("판매 성공", "이제 남은 재고는 " + stock, response);
    } else {
        responseHtml("판매 부족", "다 팔렸어요.", response);
    }
}
```

| 순서 | Thread A                  | Thread B                  |
| ---- | ------------------------- | ------------------------- |
| 1    | `stock > 0` 검사 → `true` |                           |
| 2    | 판매 처리 시작            | `stock > 0` 검사 → `true` |
| 3    | 처리 중                   | 판매 처리 시작            |
| 4    | `stock--` → `0`           |                           |
| 5    |                           | `stock--` → `-1`          |

**결과:** 두 Thread가 모두 재고가 있다고 판단해 재고가 `-1`이 될 수 있다.

**원인:** 재고를 확인하는 `if (stock > 0)`과 재고를 줄이는 `stock--`가 **하나의 원자적 작업**으로 처리되지 않기 때문이다.

- 여러 Thread는 하나의 Servlet 객체를 공유한다.
- Servlet **멤버 변수도 공유**한다.
- 공유 변수를 동시에 읽고 수정하면 데이터 불일치가 발생할 수 있다.

## 3. Mutex와 Semaphore

공유 자원을 여러 Thread가 안전하게 사용하기 위한 동기화 방식으로 강의자료는 **Mutex**와 **Semaphore**를 다룬다.

### Mutex(뮤텍스)

한 번에 **하나의 Thread만 공유 자원에 접근**하도록 제한한다.

- 하나의 Thread만 공유 영역 사용
- 다른 Thread는 사용이 끝날 때까지 대기
- 비유: **열쇠가 하나뿐인 화장실**
- Java의 `synchronized` 블록이 관련 예시

### Semaphore(세마포어)

공유 자원에 **동시에 접근 가능한 Thread의 개수를 제한**한다.

- 여러 Thread의 동시 접근을 허용하되 개수 제어
- 비유: **주차 공간이 여러 개인 유료 주차장**
- 강의자료의 관련 예시: **Connection Pool**

| 비교         | Mutex              | Semaphore                    |
| ------------ | ------------------ | ---------------------------- |
| 접근 가능 수 | 한 번에 하나       | 설정된 개수만큼              |
| 동작 방식    | 상호 배제          | 허용 개수 제어               |
| 비유         | 열쇠 하나인 화장실 | 여러 칸이 있는 주차장        |
| 관련 예시    | `synchronized`     | Connection Pool 접근 수 제한 |

## 4. `synchronized` 동기화

Java에서 `synchronized`로 특정 코드 영역의 동시 접근을 제어할 수 있다.

**재고 확인과 감소를 하나의 임계 구역으로 묶는 예시:**

```java
private long stock = 1;

public void buy() {
    synchronized (this) {
        if (stock > 0) {
            stock--;
            System.out.println("구매 성공");
        } else {
            System.out.println("재고 부족");
        }
    }
}
```

**동작 과정**

1. Thread A가 `synchronized` 영역에 진입
2. Thread B는 동일 객체의 잠금이 해제될 때까지 대기
3. Thread A가 재고 확인·감소 수행
4. Thread A가 영역을 벗어나 잠금 해제
5. Thread B가 진입해 변경된 재고 확인

### `synchronized`의 단점

동기화 영역에 한 번에 하나의 Thread만 들어갈 수 있으므로 성능이 떨어질 수 있다. 특히 **오래 걸리는 작업**을 동기화 영역에 포함하면 다른 Thread의 대기 시간이 길어진다.

> **핵심:** 충돌을 막되, 동기화가 필요한 영역을 적절히 설정한다.

## 5. Servlet 상태 관리 주의사항

### 멤버 변수 vs 지역 변수

**멤버 변수 — Servlet 객체에 속하며 Thread 간 공유**

```java
public class MyServlet extends HttpServlet {
    private int count = 0;
}
```

**지역 변수 — 각 메서드 호출에 속함**

```java
protected void doGet(
    HttpServletRequest request,
    HttpServletResponse response
) {
    int count = 0;
}
```

- 멤버 변수는 하나의 Servlet 객체를 공유하는 Thread들이 함께 접근한다.
- 메서드 내부 지역 변수는 서로 다른 요청 처리에서 별도로 사용된다.
- **단, 지역 변수가 공유 객체를 참조한다면 그 객체 내부 상태는 여전히 공유될 수 있다.**

### 상태 관리 원칙

1. 가능한 한 Servlet 멤버 변수로 **요청별 상태를 관리하지 않는다.**
2. 요청 처리에 필요한 값은 **지역 변수**로 관리한다.
3. 공유 자원을 변경할 때는 **동시성 문제**를 고려한다.
4. 필요하면 `synchronized` 등의 동기화 방법을 사용한다.
5. 불필요하게 넓은 동기화 영역은 성능을 낮출 수 있다.

## 핵심 정리

| 키워드         | 의미                                         |
| -------------- | -------------------------------------------- |
| Multi-Thread   | 여러 Thread가 동시에 요청 처리               |
| Race Condition | Thread 실행 순서에 따라 결과가 달라지는 문제 |
| Mutex          | 하나의 Thread만 공유 자원에 접근하도록 제한  |
| Semaphore      | 일정 개수의 Thread 접근 허용                 |
| `synchronized` | Java의 동기화 키워드                         |
| Thread Safe    | 멀티스레드 환경에서 안전하게 동작하는 특성   |

**가장 중요한 원칙: 가급적 Servlet의 멤버 변수로 변경 가능한 상태를 관리하지 않는다.**
