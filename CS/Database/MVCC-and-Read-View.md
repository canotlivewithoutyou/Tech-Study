# MVCC와 Read View

## 1. 동시성 제어의 딜레마

여러 사용자가 동시에 같은 데이터에 접근하면 다음과 같은 문제가 생길 수 있다.

- 갱신 손실(Lost Update)
- 더티 리드(Dirty Read)
- 반복 조회 결과의 불일치

모든 접근을 잠금으로 직렬화하면 데이터를 안전하게 보호할 수 있지만, 읽기와 쓰기가 서로를 오래 기다려 동시성이 크게 떨어진다.

**MVCC(Multi-Version Concurrency Control)**는 하나의 데이터에 여러 버전을 유지하고, 각 트랜잭션에 적절한 버전을 보여 주어 이 문제를 완화한다.

---

## 2. MVCC란?

MVCC는 여러 버전의 레코드를 관리하여 **읽기와 쓰기의 동시성을 높이는 기법**이다.

- 읽기는 현재 행이 갱신 중이어도 필요한 과거 버전을 볼 수 있다.
- 쓰기는 새 버전을 만들고 이전 버전을 Undo Log에 연결한다.
- 각 트랜잭션은 자신의 시점에 맞는 **일관된 데이터 뷰**를 얻는다.
- 일반 조회가 쓰기 작업을 기다리지 않고, 쓰기 작업도 일반 조회를 막지 않는 경우가 많다.

```text
현재 레코드 (TRX_ID = 15, value = 50,000)
        |
        | ROLL_PTR
        v
Undo 버전 (TRX_ID = 8, value = 20,000)
        |
        | ROLL_PTR
        v
Undo 버전 (TRX_ID = 7, value = 10,000)
```

> MVCC는 일반 조회의 잠금 대기를 줄이는 기법이지, 모든 잠금을 없애는 기법은 아니다. 동시에 같은 행을 수정하거나 잠금 읽기를 수행하면 여전히 잠금과 대기가 발생한다.

---

## 3. InnoDB MVCC의 핵심 구성 요소

### 3.1 숨은 시스템 컬럼

InnoDB 레코드에는 MVCC를 위한 메타데이터가 함께 저장된다.

| 컬럼 | 역할 |
|---|---|
| `DB_TRX_ID` | 해당 레코드를 마지막으로 삽입하거나 수정한 트랜잭션 ID |
| `DB_ROLL_PTR` | 이전 버전이 기록된 Undo Log 레코드를 가리키는 포인터 |
| `DB_ROW_ID` | 적절한 고유 인덱스가 없을 때 InnoDB가 만드는 내부 행 ID |

`DB_ROW_ID`는 모든 테이블에 항상 추가되는 값이 아니며, 기본 키나 적절한 고유 인덱스가 없는 경우에 사용된다.

### 3.2 Undo Log

Undo Log는 변경 전의 값을 보관한다.

- 트랜잭션 롤백에 사용한다.
- MVCC가 과거 레코드 버전을 복원할 때 사용한다.
- 오래 실행되는 트랜잭션이 있으면 필요한 과거 버전을 정리하지 못해 Undo 데이터가 누적될 수 있다.

### 3.3 Read View

Read View는 **어떤 트랜잭션의 변경이 현재 조회에 보이는지 판단하는 스냅샷 메타데이터**다. 행 전체를 복사해 두는 것이 아니라, 활성 트랜잭션 ID와 경계값을 저장해 가시성을 계산한다.

---

## 4. 데이터 수정 시 버전이 만들어지는 과정

트랜잭션이 레코드를 수정하면 개념적으로 다음 순서로 처리된다.

1. 기존 값을 Undo Log에 기록한다.
2. 현재 레코드에 새로운 값을 반영한다.
3. 현재 레코드의 `DB_TRX_ID`를 수정 트랜잭션 ID로 바꾼다.
4. `DB_ROLL_PTR`로 이전 버전을 연결한다.
5. 다른 트랜잭션은 자신의 Read View에 맞는 버전을 찾는다.

이 구조 덕분에 현재 값이 아직 커밋되지 않았거나 조회 시점보다 미래의 값이더라도, Undo Log를 따라가 과거의 가시 버전을 읽을 수 있다.

---

## 5. Read View의 주요 정보

강의 자료의 이름을 기준으로 보면 Read View는 다음 값을 이용한다.

| 구성 요소 | 의미 |
|---|---|
| `up_limit_id` | Read View 생성 시점에 활성 상태인 트랜잭션 중 가장 작은 ID |
| `low_limit_id` | 아직 할당되지 않은 다음 트랜잭션 ID, 즉 생성 시점의 상한 경계 |
| `active transaction list` | Read View 생성 시점에 아직 종료되지 않은 트랜잭션 ID 목록 |
| `creator_trx_id` | Read View를 만든 자기 자신의 트랜잭션 ID |

> 내부 필드의 실제 이름은 MySQL 버전과 구현 설명에 따라 다르게 표기될 수 있다. 중요한 것은 **최소 활성 ID, 다음 할당 ID, 활성 ID 목록**을 이용해 가시성을 판정한다는 점이다.

---

## 6. 레코드 버전의 가시성 판단

조회할 레코드 버전의 트랜잭션 ID를 `trx_id`라고 하자. 개념적인 판단 순서는 다음과 같다.

1. `trx_id`가 자기 트랜잭션 ID라면 자신의 변경이므로 볼 수 있다.
2. `trx_id < up_limit_id`라면 Read View 생성 전에 이미 완료된 트랜잭션의 버전이므로 볼 수 있다.
3. `trx_id >= low_limit_id`라면 Read View 생성 뒤 시작된 미래 트랜잭션의 버전이므로 볼 수 없다.
4. `up_limit_id <= trx_id < low_limit_id`라면 활성 목록을 확인한다.
   - 활성 목록에 있으면 생성 시점에 진행 중이었으므로 볼 수 없다.
   - 활성 목록에 없으면 생성 시점에 이미 완료된 것으로 판단해 볼 수 있다.
5. 볼 수 없다면 `DB_ROLL_PTR`를 따라 Undo Log의 이전 버전으로 이동하고 다시 판단한다.

```text
             trx_id < up_limit_id
                    |
                    v
                 보임

up_limit_id <= trx_id < low_limit_id
                    |
             활성 목록에 있는가?
               /           \
             예             아니오
          이전 버전 탐색       보임

             trx_id >= low_limit_id
                    |
                    v
              이전 버전 탐색
```

---

## 7. 가시성 판단 시나리오

다음 Read View를 기준으로 세 가지 경우를 살펴보자.

```text
up_limit_id = 8
low_limit_id = 12
active transaction list = [8, 10]
```

### 시나리오 1: 과거에 완료된 버전

현재 레코드의 `DB_TRX_ID`가 `5`라면 다음 조건을 만족한다.

```text
5 < up_limit_id(8)
```

Read View가 생성되기 전에 완료된 버전이므로 즉시 볼 수 있다.

### 시나리오 2: 생성 시점에 진행 중인 버전

현재 레코드의 `DB_TRX_ID`가 `9`이고 활성 목록이 `[8, 9]`라고 가정한다.

```text
up_limit_id(8) <= 9 < low_limit_id(12)
9는 활성 목록에 포함됨
```

현재 버전은 볼 수 없다. `DB_ROLL_PTR`를 따라 이전 버전 `TRX_ID = 4`를 찾고, `4 < 8`이므로 그 버전을 읽는다.

### 시나리오 3: Read View 생성 후의 미래 버전

현재 레코드의 버전 연결이 다음과 같다고 가정한다.

```text
TRX_ID 15: 50,000원
   -> TRX_ID 8: 20,000원
      -> TRX_ID 7: 10,000원
```

1. `15 >= low_limit_id(12)`이므로 현재 버전은 볼 수 없다.
2. Undo Log에서 `TRX_ID = 8` 버전을 찾는다.
3. `8`은 활성 목록에 있으므로 이 버전도 볼 수 없다.
4. 다시 이전 버전인 `TRX_ID = 7`을 찾는다.
5. `7 < up_limit_id(8)`이므로 10,000원을 읽는다.

---

## 8. 일관 읽기와 현재 읽기

### 일관 읽기 (Consistent Read)

일반적인 `SELECT`는 Read View를 이용한 비잠금 조회다.

```sql
SELECT balance
FROM account
WHERE account_id = 1;
```

- 자신의 스냅샷에 맞는 버전을 읽는다.
- 다른 트랜잭션의 일반적인 행 잠금 때문에 대기하지 않는 경우가 많다.
- 과거 버전이 필요하면 Undo Log를 따라간다.

### 현재 읽기 (Current Read)

다음 작업은 최신 커밋 데이터와 잠금 상태를 기준으로 읽는다.

```sql
SELECT * FROM account WHERE account_id = 1 FOR SHARE;
SELECT * FROM account WHERE account_id = 1 FOR UPDATE;
UPDATE account SET balance = balance - 10000 WHERE account_id = 1;
DELETE FROM account WHERE account_id = 1;
```

- 필요하면 다른 트랜잭션의 잠금 해제를 기다린다.
- 읽은 레코드 또는 범위에 잠금을 획득한다.
- 데이터를 곧 수정할 예정이라면 일관 읽기 후 별도 수정하는 것보다 `FOR UPDATE` 같은 잠금 읽기가 적합할 수 있다.

---

## 9. 격리 수준에 따른 Read View 차이

| 격리 수준 | Read View 사용 방식 |
|---|---|
| `READ UNCOMMITTED` | 커밋되지 않은 최신 버전까지 읽을 수 있어 일관된 Read View 보장이 약하다. |
| `READ COMMITTED` | 일반적으로 각 일관 읽기 SQL 문마다 새로운 Read View를 만든다. |
| `REPEATABLE READ` | 트랜잭션의 첫 일관 읽기에서 만든 Read View를 이후 조회에서도 재사용한다. |
| `SERIALIZABLE` | 동시 실행을 더 엄격히 제한하며 일반 조회도 잠금 읽기처럼 동작할 수 있다. |

MySQL InnoDB의 `REPEATABLE READ`에서 `START TRANSACTION WITH CONSISTENT SNAPSHOT`을 사용하면 첫 일관 읽기까지 기다리지 않고 트랜잭션 시작 시점에 스냅샷을 확보할 수 있다. 다른 격리 수준에서는 같은 효과를 기대하면 안 된다.

---

## 10. MVCC 사용 시 주의할 점

- MVCC는 쓰기-쓰기 충돌을 제거하지 않는다.
- 오래 열린 트랜잭션은 Undo Log 정리를 지연시키고 저장 공간과 조회 비용을 늘릴 수 있다.
- 스냅샷 조회와 현재 읽기를 한 트랜잭션에서 섞으면 서로 다른 시점의 데이터가 보이는 것처럼 느낄 수 있다.
- Lost Update가 업무상 허용되지 않는다면 `SELECT ... FOR UPDATE`, 원자적 `UPDATE`, 낙관적 락 버전 컬럼 등을 검토한다.
- 트랜잭션은 필요한 작업만 포함해 짧게 유지한다.

## 핵심 정리

- MVCC는 여러 데이터 버전을 유지해 읽기와 쓰기의 동시성을 높인다.
- InnoDB는 `DB_TRX_ID`, `DB_ROLL_PTR`, Undo Log, Read View를 이용한다.
- Read View는 데이터 복사본이 아니라 **버전 가시성을 판단하기 위한 기준 정보**다.
- 보이지 않는 최신 버전은 Undo Log를 따라가며 과거의 가시 버전을 찾는다.
- 일반 `SELECT`는 일관 읽기, 잠금 SQL과 DML은 현재 읽기로 이해하면 핵심 차이가 보인다.

