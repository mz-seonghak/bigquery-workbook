---
title: 빅쿼리 수집시간 파티션 테이블
slug: partition-ingestion-time
abstract: 수집시간(Ingestion Time) 기반 파티셔닝 가이드
---

## 개요

BigQuery에서 일반 테이블을 파티션 테이블로 변환하는 방법을 다룹니다. `_PARTITIONTIME` 의사 컬럼(pseudo column)을 활용한 **수집시간 기반 파티셔닝**을 중심으로, **HOUR(시간별)**, **DAY(일별)**, **MONTH(월별)** 세 가지 단위의 파티션 생성부터 데이터 삽입, 조회, 삭제, 그리고 일반 테이블에서 파티션 테이블로의 데이터 복제까지 실무에서 자주 사용되는 패턴을 정리합니다.

## _PARTITIONTIME vs _PARTITIONDATE

| 항목 | `_PARTITIONTIME` | `_PARTITIONDATE` |
|---|---|---|
| 타입 | TIMESTAMP | DATE |
| TIMESTAMP_TRUNC 사용 | 가능 | **불가능 (에러 발생)** |
| 기본 일별 파티션 | 사용 가능 | 사용 가능 |
| TRUNC 기반 파티션 (HOUR, MONTH, YEAR 등) | **반드시 사용** | 사용 불가 |

> **핵심**: `TIMESTAMP_TRUNC`로 시간별/월별/연별 파티션을 구성할 때는 반드시 `_PARTITIONTIME`을 사용해야 합니다. `_PARTITIONDATE`를 사용하면 에러가 발생합니다.

## 파티션 단위별 비교

| 항목 | HOUR | DAY | MONTH |
|---|---|---|---|
| 파티션 정의 | `TIMESTAMP_TRUNC(_PARTITIONTIME, HOUR)` | `_PARTITIONDATE` 또는 `TIMESTAMP_TRUNC(_PARTITIONTIME, DAY)` | `TIMESTAMP_TRUNC(_PARTITIONTIME, MONTH)` |
| INSERT 시 `_PARTITIONTIME` 형식 | `"2025-03-15 09:00:00"` (정시) | `"2025-03-15"` (아무 날짜) | `"2025-03-01"` (월의 1일) |
| 파티션 수 (1년 기준) | ~8,760개 | ~365개 | 12개 |
| 적합한 데이터 | 실시간 로그, IoT 센서 | 일별 트랜잭션, 이벤트 로그 | 월별 집계, 장기 보관 데이터 |
| 파티션 한도 (10,000개) 주의 | **약 417일분** (초과 주의) | 약 27년분 | 약 833년분 |

## 1단계: 수집시간 기반 파티션 테이블 생성

### MONTH(월별) 파티션

```sql
CREATE TABLE `my-gcp-project.my_dataset.part_table_monthly`
(
  user_id STRING,
  event_name STRING,
  event_value FLOAT64,
  created_at TIMESTAMP
)
PARTITION BY TIMESTAMP_TRUNC(_PARTITIONTIME, MONTH)
OPTIONS (
  require_partition_filter = TRUE
)
;
```

### DAY(일별) 파티션

```sql
CREATE TABLE `my-gcp-project.my_dataset.part_table_daily`
(
  user_id STRING,
  event_name STRING,
  event_value FLOAT64,
  created_at TIMESTAMP
)
PARTITION BY TIMESTAMP_TRUNC(_PARTITIONTIME, DAY)
OPTIONS (
  require_partition_filter = TRUE
)
;
```

일별 파티션은 `_PARTITIONDATE`로도 생성할 수 있습니다. 이 경우 `TIMESTAMP_TRUNC`가 필요 없습니다.

```sql
CREATE TABLE `my-gcp-project.my_dataset.part_table_daily_v2`
(
  user_id STRING,
  event_name STRING,
  event_value FLOAT64,
  created_at TIMESTAMP
)
PARTITION BY _PARTITIONDATE
OPTIONS (
  require_partition_filter = TRUE
)
;
```

### HOUR(시간별) 파티션

```sql
CREATE TABLE `my-gcp-project.my_dataset.part_table_hourly`
(
  user_id STRING,
  event_name STRING,
  event_value FLOAT64,
  created_at TIMESTAMP
)
PARTITION BY TIMESTAMP_TRUNC(_PARTITIONTIME, HOUR)
OPTIONS (
  require_partition_filter = TRUE
)
;
```

> **주의**: HOUR 파티션은 파티션 수가 매우 빠르게 증가합니다. BigQuery의 테이블당 파티션 한도는 10,000개이므로, 약 417일(약 14개월) 분량의 데이터만 유지할 수 있습니다. 또한 한 번의 작업(job)당 수정 가능한 파티션 수는 4,000개로 제한됩니다.

### 주요 포인트

- `require_partition_filter = TRUE`: 쿼리 시 파티션 필터를 강제하여 전체 테이블 스캔을 방지합니다.
- `_PARTITIONTIME`은 테이블 스키마에 정의하지 않는 **의사 컬럼(pseudo column)** 입니다.
- `TIMESTAMP_TRUNC`를 사용하는 경우(HOUR, MONTH 등) `_PARTITIONDATE`가 아닌 **`_PARTITIONTIME`을 반드시 사용**해야 합니다.
- DAY 파티션만 `_PARTITIONDATE`로 직접 생성할 수 있습니다.

## 2단계: 데이터 삽입

### MONTH 파티션에 삽입

```sql
INSERT INTO `my-gcp-project.my_dataset.part_table_monthly`
  (_PARTITIONTIME, user_id, event_name, event_value, created_at)
VALUES
  ("2025-03-01", "u001", "purchase", 29900.0, CURRENT_TIMESTAMP()),
  ("2025-05-01", "u002", "login", 0.0, CURRENT_TIMESTAMP()),
  ("2025-04-01", "u003", "purchase", 15000.0, CURRENT_TIMESTAMP()),
  ("2025-06-01", "u004", "signup", 0.0, CURRENT_TIMESTAMP());
```

`_PARTITIONTIME` 값은 **해당 월의 1일**이어야 합니다. `"2025-01-31"` 같은 값을 넣으면 에러가 발생합니다.

| 입력값 | 결과 |
|---|---|
| `"2025-03-01"` | 2025년 3월 파티션에 저장 |
| `"2025-05-15"` | **에러 발생** (월의 1일이 아님) |
| `"2025-04-01"` | 2025년 4월 파티션에 저장 |

### DAY 파티션에 삽입

```sql
INSERT INTO `my-gcp-project.my_dataset.part_table_daily`
  (_PARTITIONTIME, user_id, event_name, event_value, created_at)
VALUES
  ("2025-03-15", "u001", "purchase", 29900.0, CURRENT_TIMESTAMP()),
  ("2025-03-16", "u002", "login", 0.0, CURRENT_TIMESTAMP()),
  ("2025-04-01", "u003", "purchase", 15000.0, CURRENT_TIMESTAMP()),
  ("2025-04-02", "u004", "signup", 0.0, CURRENT_TIMESTAMP());
```

일별 파티션은 아무 날짜나 사용할 수 있습니다. 단, 시간 부분은 `00:00:00`이어야 합니다.

| 입력값 | 결과 |
|---|---|
| `"2025-03-15"` | 2025년 3월 15일 파티션에 저장 |
| `"2025-03-15 12:00:00"` | **에러 발생** (시간이 00:00:00이 아님) |
| `"2025-04-02"` | 2025년 4월 2일 파티션에 저장 |

### HOUR 파티션에 삽입

```sql
INSERT INTO `my-gcp-project.my_dataset.part_table_hourly`
  (_PARTITIONTIME, user_id, event_name, event_value, created_at)
VALUES
  ("2025-03-15 09:00:00", "u001", "purchase", 29900.0, CURRENT_TIMESTAMP()),
  ("2025-03-15 10:00:00", "u002", "login", 0.0, CURRENT_TIMESTAMP()),
  ("2025-03-15 11:00:00", "u003", "purchase", 15000.0, CURRENT_TIMESTAMP()),
  ("2025-03-15 14:00:00", "u004", "signup", 0.0, CURRENT_TIMESTAMP());
```

시간별 파티션은 **정시(분, 초가 00)**여야 합니다.

| 입력값 | 결과 |
|---|---|
| `"2025-03-15 09:00:00"` | 3월 15일 09시 파티션에 저장 |
| `"2025-03-15 09:30:00"` | **에러 발생** (정시가 아님) |
| `"2025-03-15 14:00:00"` | 3월 15일 14시 파티션에 저장 |

### _PARTITIONTIME 값 규칙 요약

각 파티션 단위마다 `_PARTITIONTIME`에 넣을 수 있는 값의 형식이 정해져 있습니다. 파티션 경계에 맞지 않는 값을 넣으면 에러가 발생합니다.

| 파티션 단위 | 허용 형식 | 예시 |
|---|---|---|
| HOUR | `YYYY-MM-DD HH:00:00` | `"2025-03-15 09:00:00"` |
| DAY | `YYYY-MM-DD` | `"2025-03-15"` |
| MONTH | `YYYY-MM-01` | `"2025-03-01"` |

## 3단계: 데이터 조회

### 전체 데이터 조회

`_PARTITIONTIME`은 의사 컬럼이므로 `SELECT *`에 자동으로 포함되지 않습니다. 명시적으로 지정하거나 `WHERE` 절에서 사용해야 합니다.

```sql
SELECT *
FROM `my-gcp-project.my_dataset.part_table_monthly`
WHERE _PARTITIONTIME IS NOT NULL
;
```

### 파티션 분포 확인

파티션별로 데이터가 올바르게 분할되었는지 확인합니다. 세 가지 파티션 단위 모두 동일한 쿼리 패턴을 사용합니다.

```sql
-- MONTH 파티션 분포
SELECT
  _PARTITIONTIME AS pt,
  COUNT(1) AS cnt
FROM `my-gcp-project.my_dataset.part_table_monthly`
WHERE _PARTITIONTIME IS NOT NULL
GROUP BY _PARTITIONTIME
;
```

MONTH 파티션 결과 예시:

| pt | cnt |
|---|---|
| 2025-03-01 00:00:00 UTC | 1 |
| 2025-04-01 00:00:00 UTC | 1 |
| 2025-05-01 00:00:00 UTC | 1 |
| 2025-06-01 00:00:00 UTC | 1 |

```sql
-- DAY 파티션 분포
SELECT
  _PARTITIONTIME AS pt,
  COUNT(1) AS cnt
FROM `my-gcp-project.my_dataset.part_table_daily`
WHERE _PARTITIONTIME IS NOT NULL
GROUP BY _PARTITIONTIME
;
```

DAY 파티션 결과 예시:

| pt | cnt |
|---|---|
| 2025-03-15 00:00:00 UTC | 1 |
| 2025-03-16 00:00:00 UTC | 1 |
| 2025-04-01 00:00:00 UTC | 1 |
| 2025-04-02 00:00:00 UTC | 1 |

```sql
-- HOUR 파티션 분포
SELECT
  _PARTITIONTIME AS pt,
  COUNT(1) AS cnt
FROM `my-gcp-project.my_dataset.part_table_hourly`
WHERE _PARTITIONTIME IS NOT NULL
GROUP BY _PARTITIONTIME
;
```

HOUR 파티션 결과 예시:

| pt | cnt |
|---|---|
| 2025-03-15 09:00:00 UTC | 1 |
| 2025-03-15 10:00:00 UTC | 1 |
| 2025-03-15 11:00:00 UTC | 1 |
| 2025-03-15 14:00:00 UTC | 1 |

## 4단계: 파티션 삭제

### SQL DELETE 방식

파티션 단위에 맞는 `_PARTITIONTIME` 값으로 필터링하여 삭제합니다.

```sql
-- MONTH 파티션 삭제
DELETE FROM `my-gcp-project.my_dataset.part_table_monthly`
WHERE _PARTITIONTIME = "2025-03-01"
;

-- DAY 파티션 삭제
DELETE FROM `my-gcp-project.my_dataset.part_table_daily`
WHERE _PARTITIONTIME = "2025-03-15"
;

-- HOUR 파티션 삭제
DELETE FROM `my-gcp-project.my_dataset.part_table_hourly`
WHERE _PARTITIONTIME = "2025-03-15 09:00:00"
;
```

이 방법은 데이터가 적을 때는 편리하지만, 대량 데이터의 경우 성능이 떨어질 수 있습니다.

### bq CLI 방식 (대량 삭제 권장)

대량의 파티션 데이터를 삭제할 때는 `bq rm` 명령을 사용하는 것이 더 효율적입니다. 파티션 decorator(`$`) 뒤에 파티션 단위에 맞는 형식을 지정합니다.

```bash
# MONTH 파티션 삭제 (YYYYMMDD 형식, 월의 1일)
bq rm -t 'my-gcp-project:my_dataset.part_table_monthly$20250301'

# DAY 파티션 삭제 (YYYYMMDD 형식)
bq rm -t 'my-gcp-project:my_dataset.part_table_daily$20250315'

# HOUR 파티션 삭제 (YYYYMMDDHH 형식)
bq rm -t 'my-gcp-project:my_dataset.part_table_hourly$2025031509'
```

| 방식 | 장점 | 단점 |
|---|---|---|
| SQL DELETE | SQL 문법으로 간편 | 대량 데이터 시 느림, DML 할당량 소모 |
| bq rm | 빠르고 효율적, 메타데이터 수준 삭제 | CLI 환경 필요 |

## 5단계: 일반 테이블에서 파티션 테이블로 데이터 복제

기존에 파티션 없이 운영하던 테이블의 데이터를 파티션 테이블로 마이그레이션하는 방법입니다.

### 대상 파티션 테이블 생성

복제 대상이 되는 파티션 테이블을 원하는 단위로 생성합니다.

```sql
-- MONTH 파티션 테이블
CREATE TABLE `my-gcp-project.my_dataset.part_table_migrated_monthly`
(
  user_id STRING,
  event_name STRING,
  event_value FLOAT64,
  created_at TIMESTAMP
)
PARTITION BY TIMESTAMP_TRUNC(_PARTITIONTIME, MONTH)
OPTIONS (
  require_partition_filter = TRUE
)
;

-- DAY 파티션 테이블
CREATE TABLE `my-gcp-project.my_dataset.part_table_migrated_daily`
(
  user_id STRING,
  event_name STRING,
  event_value FLOAT64,
  created_at TIMESTAMP
)
PARTITION BY TIMESTAMP_TRUNC(_PARTITIONTIME, DAY)
OPTIONS (
  require_partition_filter = TRUE
)
;

-- HOUR 파티션 테이블
CREATE TABLE `my-gcp-project.my_dataset.part_table_migrated_hourly`
(
  user_id STRING,
  event_name STRING,
  event_value FLOAT64,
  created_at TIMESTAMP
)
PARTITION BY TIMESTAMP_TRUNC(_PARTITIONTIME, HOUR)
OPTIONS (
  require_partition_filter = TRUE
)
;
```

### INSERT ... SELECT로 데이터 복제

기존 테이블의 `created_at` 컬럼을 기준으로 `TIMESTAMP_TRUNC`의 단위를 파티션에 맞게 지정하여 삽입합니다.

```sql
-- MONTH 파티션으로 복제
INSERT INTO `my-gcp-project.my_dataset.part_table_migrated_monthly`
  (_PARTITIONTIME, user_id, event_name, event_value, created_at)
SELECT
  TIMESTAMP_TRUNC(created_at, MONTH),
  user_id, event_name, event_value, created_at
FROM `my-gcp-project.my_dataset.source_table`
WHERE created_at IS NOT NULL
;

-- DAY 파티션으로 복제
INSERT INTO `my-gcp-project.my_dataset.part_table_migrated_daily`
  (_PARTITIONTIME, user_id, event_name, event_value, created_at)
SELECT
  TIMESTAMP_TRUNC(created_at, DAY),
  user_id, event_name, event_value, created_at
FROM `my-gcp-project.my_dataset.source_table`
WHERE created_at IS NOT NULL
;

-- HOUR 파티션으로 복제
INSERT INTO `my-gcp-project.my_dataset.part_table_migrated_hourly`
  (_PARTITIONTIME, user_id, event_name, event_value, created_at)
SELECT
  TIMESTAMP_TRUNC(created_at, HOUR),
  user_id, event_name, event_value, created_at
FROM `my-gcp-project.my_dataset.source_table`
WHERE created_at IS NOT NULL
;
```

### 핵심 포인트

- `TIMESTAMP_TRUNC(created_at, MONTH|DAY|HOUR)`: 파티션 단위에 맞게 타임스탬프를 잘라 `_PARTITIONTIME` 값으로 사용합니다.
- `TIMESTAMP_TRUNC`의 두 번째 인자가 대상 파티션 테이블의 단위와 일치해야 합니다.
- 원본 테이블의 실제 데이터 시점에 맞는 파티션에 자동으로 배치됩니다.
- 원본이 파티션 테이블이라면 `WHERE _PARTITIONTIME IS NOT NULL` 필터가 필요하고, 일반 테이블이라면 `WHERE created_at IS NOT NULL` 등으로 NULL을 제외합니다.

### 결과 확인

```sql
SELECT _PARTITIONTIME, *
FROM `my-gcp-project.my_dataset.part_table_migrated_monthly`
WHERE _PARTITIONTIME IS NOT NULL
;
```

## 전체 흐름 요약

```
┌───────────────────────────────────────────────────────────────┐
│  1. 파티션 테이블 생성                                         │
│     TIMESTAMP_TRUNC(_PARTITIONTIME, HOUR | DAY | MONTH)       │
├───────────────────────────────────────────────────────────────┤
│  2. 데이터 삽입                                               │
│     HOUR → 정시 / DAY → 날짜 / MONTH → 월의 1일               │
├───────────────────────────────────────────────────────────────┤
│  3. 조회 및 확인                                              │
│     _PARTITIONTIME은 SELECT *에 자동 포함 안됨                  │
├───────────────────────────────────────────────────────────────┤
│  4. 파티션 삭제                                               │
│     소량 → SQL DELETE / 대량 → bq rm 권장                      │
├───────────────────────────────────────────────────────────────┤
│  5. 일반 테이블 → 파티션 테이블 마이그레이션                       │
│     INSERT ... SELECT + TIMESTAMP_TRUNC(col, HOUR|DAY|MONTH)  │
└───────────────────────────────────────────────────────────────┘
```

## 주의사항 정리

1. **TIMESTAMP_TRUNC 사용 시 반드시 `_PARTITIONTIME`**: `_PARTITIONDATE`를 사용하면 에러가 발생합니다. (DAY 파티션의 경우만 `_PARTITIONDATE` 단독 사용 가능)
2. **파티션 경계에 맞는 값만 INSERT 가능**: MONTH는 1일, DAY는 `00:00:00`, HOUR는 정시(`HH:00:00`)여야 합니다.
3. **`_PARTITIONTIME`은 의사 컬럼**: `SELECT *`에 포함되지 않으므로 명시적으로 조회해야 합니다.
4. **대량 파티션 삭제는 bq 명령 사용**: SQL DELETE보다 `bq rm`이 성능면에서 유리합니다.
5. **HOUR 파티션의 파티션 한도 주의**: 테이블당 최대 10,000개 파티션이므로 HOUR 파티션은 약 417일분만 유지할 수 있습니다. 한 번의 작업당 수정 가능한 파티션 수는 4,000개입니다.
6. **마이그레이션 시 TIMESTAMP_TRUNC 단위 일치**: `TIMESTAMP_TRUNC`의 단위(HOUR, DAY, MONTH)를 대상 파티션 테이블의 단위와 맞춰야 합니다.
