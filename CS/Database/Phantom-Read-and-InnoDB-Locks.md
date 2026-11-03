# Phantom Read와 InnoDB 잠금 메커니즘

## 1. Phantom Read란?

**Phantom Read**는 한 트랜잭션 안에서 같은 범위 조건으로 여러 번 조회했을 때, 처음에는 없던 행이 나타나거나 기존 행이 사라져 **결과 행 집합이 달라지는 현상**이다.

### 발생 예시

```text
T1: SELECT id FROM item WHERE id > 10;  -> [12, 15]
T2: INSERT INTO item(id) VALUES (13);
T2: COMMIT;
T1: SELECT id FROM item WHERE id > 10;  -> [12, 13, 15]
```

새로운 `id = 13`이 유령처럼 나타나므로 Phantom Read라고 부른다.

`Non-Repeatable Read`와의 차이는 다음과 같다.

| 구분 | 변화 대상 |
|---|---|
| Non-Repeatable Read | 이미 조회한 **행의 값**이 달라진다. |
| Phantom Read | 조회 조건을 만족하는 **행의 집합**이 달라진다. |

---

## 2. InnoDB의 두 가지 조회 방식

InnoDB에서 Phantom Read를 이해하려면 일관 읽기와 잠금 읽기를 구분해야 한다.

| 구분 | 대표 SQL | 읽는 데이터 | 잠금 |
|---|---|---|---|
| **일관 읽기** | 일반 `SELECT` | Read View에 맞는 스냅샷 버전 | 일반적으로 행 잠금을 획득하지 않음 |
| **잠금 읽기/현재 읽기** | `SELECT ... FOR SHARE`, `SELECT ... FOR UPDATE` | 최신 커밋 버전을 기준으로 읽음 | 조회한 레코드 또는 범위를 잠금 |

### 일관 읽기에서의 팬텀 방지

MySQL InnoDB의 `REPEATABLE READ`에서는 같은 Read View를 재사용하므로, 일반 `SELECT`를 반복해도 다른 트랜잭션이 뒤늦게 커밋한 행이 기존 스냅샷에 나타나지 않는다.

```sql
SELECT id, col
FROM t_table
WHERE id > 10;
```

이는 새 행의 삽입 자체를 막는 것이 아니라 **같은 스냅샷을 계속 보여 주는 방식**이다.

### 잠금 읽기에서의 팬텀 방지

최신 데이터를 읽고 이후 변경까지 안전하게 수행하려면 잠금 읽기를 사용한다.

```sql
SELECT id, col
FROM t_table
WHERE id > 10
FOR SHARE;

SELECT id, col
FROM t_table
WHERE id > 10
FOR UPDATE;
```

InnoDB는 `REPEATABLE READ`의 범위 잠금 읽기에서 Next-Key Lock을 사용해 조회된 레코드뿐 아니라 사이의 간격도 보호한다. 이 때문에 다른 트랜잭션이 범위 안에 새 행을 삽입하는 것을 막을 수 있다.

---

## 3. 공유 잠금과 배타 잠금

### 공유 잠금 (S, Shared Lock)

```sql
SELECT *
FROM account
WHERE account_id = 1
FOR SHARE;
```

- 다른 트랜잭션도 같은 데이터에 공유 잠금을 획득할 수 있다.
- 충돌하는 배타 잠금과 수정 작업은 기다려야 한다.
- 데이터를 읽은 뒤 그 상태를 전제로 다른 작업을 수행할 때 사용한다.

### 배타 잠금 (X, Exclusive Lock)

```sql
SELECT *
FROM account
WHERE account_id = 1
FOR UPDATE;
```

- 같은 대상에 다른 공유 잠금이나 배타 잠금이 들어오는 것을 막는다.
- 현재 트랜잭션이 곧 해당 데이터를 수정할 때 사용한다.
- 다른 트랜잭션의 일반 `SELECT`는 MVCC 과거 버전을 읽을 수 있어 반드시 차단되는 것은 아니다.

### 기본 호환 관계

| 현재 잠금 | 요청 S Lock | 요청 X Lock |
|---|:---:|:---:|
| S Lock | 허용 | 대기 |
| X Lock | 대기 | 대기 |

> 이 표는 레코드 잠금의 기본 관계다. Gap Lock은 주로 해당 간격으로의 삽입을 막기 위한 잠금이어서 호환 규칙을 동일하게 단순화하면 안 된다.

---

## 4. InnoDB의 주요 행 수준 잠금

### 4.1 Record Lock

인덱스 레코드 자체를 잠근다.

```text
인덱스: 10, 20, 30
Record Lock on 20: [20] 레코드를 보호
```

- 이미 존재하는 특정 인덱스 레코드를 보호한다.
- InnoDB의 행 잠금은 실제로 인덱스 레코드에 설정된다.

### 4.2 Gap Lock

인덱스 레코드 사이의 빈 구간을 잠근다.

```text
인덱스: 10, 20, 30
Gap Lock (20, 30): 20과 30 사이에 새 값 삽입을 방지
```

- 빈 공간에 새로운 인덱스 값이 삽입되는 것을 막는다.
- 경계 레코드 자체를 잠그는 Record Lock과 역할이 다르다.
- 기존 레코드의 일반적인 갱신을 모두 차단하는 잠금은 아니다.

### 4.3 Next-Key Lock

**Record Lock + 그 레코드 앞의 Gap Lock**을 결합한 잠금이다.

```text
Next-Key Lock (10, 20] = Gap (10, 20) + Record 20
```

- 특정 레코드와 그 앞 간격을 함께 보호한다.
- 범위 검색 중 새로운 행이 들어와 결과 집합을 바꾸는 것을 막는다.
- InnoDB `REPEATABLE READ`에서 잠금 읽기 시 Phantom Read를 방지하는 핵심 장치다.

---

## 5. 조회 조건에 따른 잠금 범위

다음처럼 `id`가 기본 키이고 레코드가 존재한다고 가정하자.

| id (PK) | col |
|---:|---|
| 10 | A |
| 20 | B |
| 30 | C |
| 40 | D |

그리고 `REPEATABLE READ`에서 `SELECT ... FOR UPDATE`를 실행한다고 하자.

| 조회 조건 | 개념적인 주요 잠금 | 결과 |
|---|---|---|
| `id = 20` | Record Lock | 고유 인덱스로 기존 한 행을 정확히 찾으므로 20번 레코드를 잠근다. |
| `id <= 20` | 범위에 대한 Next-Key Lock | 조건에 포함되는 레코드와 관련 간격을 보호해 범위 안 삽입을 막는다. |
| `id >= 20` | Record/Next-Key Lock과 끝 구간 잠금 | 20 이상 레코드 및 이후 범위로 들어오는 삽입을 막는다. |
| `id = 25` | Gap Lock `(20, 30)` | 일치하는 행은 없지만 20과 30 사이에 25가 삽입되는 것을 막는다. |

### 중요한 전제

- 잠금 범위는 **사용된 인덱스와 실제 실행 계획**에 따라 달라진다.
- 적절한 인덱스가 없으면 더 많은 인덱스 레코드를 검사하고 넓게 잠글 수 있다.
- 고유 인덱스의 모든 컬럼을 사용해 기존 행 하나를 정확히 찾으면 보통 Gap Lock 없이 Record Lock만 사용한다.
- `READ COMMITTED`에서는 검색과 인덱스 스캔에 대한 Gap Lock 사용이 대부분 줄어들지만, 외래 키 검사와 중복 키 검사 등에는 사용될 수 있다.

---

## 6. 두 세션으로 확인하는 잠금 동작

### 세션 1: 범위를 잠근다

```sql
START TRANSACTION;

SELECT *
FROM item
WHERE id BETWEEN 20 AND 30
FOR UPDATE;
```

### 세션 2: 범위 안에 삽입을 시도한다

```sql
START TRANSACTION;

INSERT INTO item (id, col)
VALUES (25, 'NEW');
```

세션 1이 `REPEATABLE READ`에서 해당 인덱스 범위를 Next-Key Lock으로 보호하고 있다면, 세션 2의 삽입은 세션 1이 `COMMIT` 또는 `ROLLBACK`할 때까지 대기한다.

```sql
-- 세션 1
COMMIT;

-- 이후 세션 2의 INSERT가 진행 가능
COMMIT;
```

---

## 7. 잠금 범위를 줄이는 실무 원칙

1. **조건절에 맞는 인덱스를 만든다.**
   - 전체 스캔으로 불필요한 레코드를 잠그지 않게 한다.
2. **트랜잭션을 짧게 유지한다.**
   - 잠금을 잡은 상태에서 네트워크 호출이나 긴 계산을 하지 않는다.
3. **필요할 때만 잠금 읽기를 사용한다.**
   - 단순 조회라면 MVCC 일관 읽기의 장점을 활용한다.
4. **접근 순서를 통일한다.**
   - 여러 행을 수정할 때 모든 코드가 같은 순서로 잠그게 한다.
5. **실행 계획과 잠금 대기를 함께 관찰한다.**
   - SQL 모양만으로 잠금 범위를 단정하지 않는다.
6. **교착상태 재시도를 준비한다.**
   - 올바른 잠금 설계에서도 교착상태는 발생할 수 있다.

## 핵심 정리

- Phantom Read는 반복한 범위 조회에서 **행 집합이 달라지는 현상**이다.
- InnoDB `REPEATABLE READ`의 일반 `SELECT`는 같은 스냅샷을 보여 주는 방식으로 팬텀을 피한다.
- 잠금 읽기는 최신 데이터를 읽으며, 범위 보호에는 Next-Key Lock을 사용한다.
- Record Lock은 레코드, Gap Lock은 빈 구간, Next-Key Lock은 레코드와 앞 간격을 보호한다.
- 실제 잠금 범위는 **격리 수준, 인덱스, 조건식, 실행 계획**에 따라 결정된다.

