---
title: "GROUP BY로 안 되는 쿼리, 윈도우 함수로 해결하기"
date: 2025-12-01
tags: ["SQL", "윈도우 함수", "Window Function", "ROW_NUMBER", "GROUP BY"]
categories: ["SQL"]
summary: "그룹별 최근 1건 조회처럼 GROUP BY로 풀기 어려운 문제를 윈도우 함수로 해결하는 방법을 정리한다."
---

"각 그룹에서 가장 최근 한 건만 뽑고 싶다." 데이터를 다루다 보면 자주 만나는 요구사항이다. GROUP BY로는 풀기 어렵다. 윈도우 함수를 사용하면 간단하게 해결할 수 있다.

---

## 예제 테이블

**lecture_contracts** - 강의별 계약 이력

| lecture_id | contract_id | contracted_at |
| --- | --- | --- |
| 101 | 888 | 2023-01-01 |
| 101 | 889 | 2023-03-05 |
| 101 | 890 | 2023-10-11 |
| 102 | 901 | 2023-04-10 |
| 102 | 902 | 2023-07-12 |

목표는 **각 강의별로 가장 최근 계약 한 건만 조회**하는 것이다.

---

## GROUP BY로 시도하면

### MAX()로 최근 날짜만 구하기

```sql
SELECT lecture_id, MAX(contracted_at)
FROM lecture_contracts
GROUP BY lecture_id;
```

최근 날짜는 구할 수 있다. 하지만 **그 날짜에 해당하는 contract_id는 함께 조회할 수 없다.**

```sql
-- 이 쿼리는 동작하지 않는다
SELECT lecture_id, contract_id, MAX(contracted_at)
FROM lecture_contracts
GROUP BY lecture_id;  -- ERROR
```

GROUP BY에 포함되지 않은 컬럼은 SELECT할 수 없기 때문이다. 날짜만 알고 다른 정보를 모르는 쿼리는 쓸모가 없다.

### DISTINCT ON으로 해결? (PostgreSQL 한정)

PostgreSQL은 `DISTINCT ON`으로 해결할 수 있다.

```sql
SELECT DISTINCT ON (lecture_id) *
FROM lecture_contracts
ORDER BY lecture_id, contracted_at DESC;
```

다만 **PostgreSQL에서만 동작한다.** MySQL, MariaDB, Oracle, SQL Server는 지원하지 않는다.

---

## 윈도우 함수

윈도우 함수는 GROUP BY와 다르게 **행을 줄이지 않는다.** 각 행을 유지하면서 그룹 내 계산 값(순위, 합계, 최댓값 등)을 붙여준다.

핵심은 "요약하되, 원본 행을 그대로 유지한다"는 점이다.

### 문법

모든 윈도우 함수는 `OVER` 절을 사용한다.

```sql
<윈도우_함수>(인자)
OVER (
    [PARTITION BY expr1, expr2, ...]
    [ORDER BY expr]
    [frame_spec]
)
```

각 절의 역할은 다음과 같다.

- **PARTITION BY** : 행을 그룹(파티션)으로 나눈다. GROUP BY와 비슷하지만 행을 줄이지 않는다.
- **ORDER BY** : 각 파티션 내에서 행의 순서를 정한다.
- **frame_spec** : 계산에 포함할 행의 범위를 지정한다. 예: `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`

PARTITION BY를 생략하면 전체 테이블이 하나의 파티션이 된다. frame_spec을 생략하면 ORDER BY 유무에 따라 기본값이 달라진다.

### 순위 함수

| 함수 | 설명 |
| --- | --- |
| `ROW_NUMBER()` | 파티션 내 행 번호를 매긴다. 동일 값이어도 고유 번호를 부여한다. |
| `RANK()` | 동일 값에 같은 순위를 부여한다. 다음 순위는 건너뛴다. (1, 2, 2, 4) |
| `DENSE_RANK()` | 동일 값에 같은 순위를 부여한다. 다음 순위를 건너뛰지 않는다. (1, 2, 2, 3) |
| `NTILE(n)` | 파티션을 n개의 버킷으로 나눈다. |

### 집계 함수

`SUM()`, `AVG()`, `MIN()`, `MAX()`, `COUNT()`를 OVER 절과 함께 사용할 수 있다. 누적 합계, 이동 평균 등을 계산할 때 유용하다.

### 오프셋/분석 함수

| 함수 | 설명 |
| --- | --- |
| `LAG(expr, n)` | 현재 행에서 n행 이전 값을 가져온다. |
| `LEAD(expr, n)` | 현재 행에서 n행 이후 값을 가져온다. |
| `FIRST_VALUE(expr)` | 파티션 내 첫 번째 값을 가져온다. |
| `LAST_VALUE(expr)` | 파티션 내 마지막 값을 가져온다. |

### 분포 함수

| 함수 | 설명 |
| --- | --- |
| `CUME_DIST()` | 현재 행의 누적 분포 값을 반환한다. (0~1 사이) |
| `PERCENT_RANK()` | 현재 행의 상대적 순위를 백분율로 반환한다. (0~1 사이) |

---

## ROW_NUMBER()로 그룹별 순위 매기기

가장 대표적인 패턴은 `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)`이다.

```sql
ROW_NUMBER() OVER (PARTITION BY lecture_id ORDER BY contracted_at DESC)
```

각 키워드의 의미는 다음과 같다.

- **PARTITION BY lecture_id** : lecture_id별로 그룹을 나눈다.
- **ORDER BY contracted_at DESC** : 각 그룹 내에서 최근 날짜 순으로 정렬한다.
- **ROW_NUMBER()** : 정렬 순서대로 1, 2, 3... 순위를 매긴다.

---

## 강의별 가장 최근 계약 조회를 윈도우 함수로 해본다면

```sql
WITH ranked AS (
    SELECT
        lecture_id,
        contract_id,
        contracted_at,
        ROW_NUMBER() OVER (
            PARTITION BY lecture_id
            ORDER BY contracted_at DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS rn
    FROM lecture_contracts
)
SELECT *
FROM ranked
WHERE rn = 1;
```

결과:

| lecture_id | contract_id | contracted_at | rn |
| --- | --- | --- | --- |
| 101 | 890 | 2023-10-11 | 1 |
| 102 | 902 | 2023-07-12 | 1 |

이 방식의 장점은 다음과 같다.

- **DB 종류에 관계없이 동작한다.** PostgreSQL, MySQL 8+, MariaDB, SQL Server, Oracle 모두 지원한다.
- **다른 컬럼을 자유롭게 SELECT할 수 있다.** GROUP BY의 제약이 없다.
- **TOP N 조회로 확장할 수 있다.** `WHERE rn <= 3`으로 최근 3건을 뽑을 수 있다.
- **정렬 기준이 명시적이다.** 최근순, 오래된순, 특정 조건 우선순위 등 자유롭게 지정할 수 있다.

---

## GROUP BY vs 윈도우 함수 비교

| 문제 | GROUP BY / DISTINCT | 윈도우 함수 |
| --- | --- | --- |
| 그룹별 최댓값 + 나머지 컬럼 조회 | 불가능 (GROUP BY 제약) | 가능 |
| SQL 표준 여부 | 구현체마다 다름 (DISTINCT ON은 PostgreSQL 전용) | ANSI SQL 표준 |
| TOP N 확장성 | 매우 제한적 | `WHERE rn <= N`으로 간단히 확장 |
| 가독성 | 서브쿼리가 복잡해짐 | PARTITION BY + ORDER BY가 의도를 명확히 드러냄 |

---

## 예시) 누적 합계로 잔액 계산하기

윈도우 함수는 순위 매기기 외에도 활용 범위가 넓다. 포인트 적립/사용 내역에서 시점별 잔액을 계산하는 경우를 보자.

**point_history** 테이블:

| id | user_id | value | created_at |
| --- | --- | --- | --- |
| 1 | 101 | 100 | 2023-01-01 |
| 2 | 101 | -30 | 2023-01-05 |
| 3 | 101 | 50 | 2023-01-10 |

```sql
SELECT
    user_id,
    value,
    created_at,
    SUM(value) OVER (
        PARTITION BY user_id
        ORDER BY created_at ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS balance
FROM point_history
ORDER BY created_at;
```

`SUM(value) OVER (...)`는 user_id별로 생성시간 순서대로 누적 합계를 구한다.

결과:

| created_at | value | balance |
| --- | --- | --- |
| 2023-01-01 | +100 | 100 |
| 2023-01-05 | -30 | 70 |
| 2023-01-10 | +50 | 120 |

GROUP BY로 이 결과를 만들려면 셀프 조인이나 서브쿼리가 필요하다. 윈도우 함수는 한 줄로 해결한다.
