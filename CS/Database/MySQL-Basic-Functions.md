# MySQL 기본 함수

## 날짜 추출과 출력 형식

`DATETIME` 값이 `2018-01-22 14:32:00`일 때

| 식 | 결과 | 반환값 |
| --- | --- | --- |
| `DATE(DATETIME)` | `2018-01-22` | 날짜 |
| `DATE_FORMAT(DATETIME, '%Y-%m-%d')` | `2018-01-22` | 문자열 |
| `DATE_FORMAT(DATETIME, '%Y-%m')` | `2018-01` | 문자열 |
| `DATE_FORMAT(DATETIME, '%H:%i:%s')` | `14:32:00` | 문자열 |

- **월** `%m` → `01`, **분** `%i` → `32`
- 시간대별 건수 집계 → `GROUP BY HOUR(DATETIME)`과 `COUNT(*)`
- 연월별 집계 → `GROUP BY YEAR(DATETIME), MONTH(DATETIME)`

[프로그래머스 59414 · DATETIME에서 DATE로 형 변환](https://school.programmers.co.kr/learn/courses/30/lessons/59414)

```sql
SELECT ANIMAL_ID, NAME, DATE(DATETIME) AS 날짜
FROM ANIMAL_INS
ORDER BY ANIMAL_ID;
```

## 개수와 중복 집계

| 식 | 집계 대상 | 이름이 `초코`, `초코`, `NULL`일 때 |
| --- | --- | --- |
| `COUNT(*)` | 전체 행 | 3 |
| `COUNT(NAME)` | 이름이 있는 행 | 2 |
| `COUNT(DISTINCT NAME)` | 서로 다른 이름 (`NULL` 제외) | 1 |

**두 번 이상 등장한 이름**

```sql
SELECT NAME, COUNT(*) AS 이름_등장_횟수
FROM ANIMAL_INS
WHERE NAME IS NOT NULL
GROUP BY NAME
HAVING COUNT(*) >= 2;
```

- `WHERE` → 그룹화 전 개별 행 필터링
- `HAVING` → 그룹화 후 집계 결과 필터링

## 합계·평균·최솟값·최댓값

| 목적 | 식 |
| --- | --- |
| 총판매량 | `SUM(QUANTITY)` |
| 평균 가격 | `AVG(PRICE)` |
| 평균 계산 후 소수 첫째 자리까지 반올림 | `ROUND(AVG(PRICE), 1)` |
| 최초·최근 입소 시각 | `MIN(DATETIME)` · `MAX(DATETIME)` |

**최초 입소 동물 조회** · 같은 시각에 입소한 동물 모두 포함

```sql
SELECT NAME, DATETIME
FROM ANIMAL_INS
WHERE DATETIME = (SELECT MIN(DATETIME) FROM ANIMAL_INS);
```

## NULL 처리

| 목적 | 식 |
| --- | --- |
| 이름이 없는 행 조회 | `WHERE NAME IS NULL` |
| 이름이 없으면 대체 문구 표시 | `IFNULL(NAME, '이름 없음')` |
| 왼쪽부터 첫 번째 NULL이 아닌 값 선택 | `COALESCE(별명, 이름, '미등록')` |

## 날짜 차이

| 계산 기준 | 식 | 3월 1일 ~ 3월 3일 |
| --- | --- | --- |
| 두 날짜의 차이 | `DATEDIFF(종료일, 시작일)` | 2일 |
| 시작일과 종료일 모두 포함 | `DATEDIFF(종료일, 시작일) + 1` | 3일 |

## 문자열 함수

| 목적 | 예시 | 결과 |
| --- | --- | --- |
| 앞 두 글자 추출 | `LEFT('AB1234', 2)` | `AB` |
| 세 번째 글자부터 두 글자 추출 | `SUBSTRING('AB1234', 3, 2)` | `12` |
| 문자열 연결 | `CONCAT('서울', '-', 'AB12')` | `서울-AB12` |

## 참고 문서

- [MySQL 날짜·시간 함수](https://dev.mysql.com/doc/refman/8.4/en/date-and-time-functions.html)
- [MySQL 집계 함수](https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions.html)
