# PostgreSQL vs MySQL 스키마 비교

> 작성일: 2026-02-08
> 관련 문서: [[아키텍처 설계서]], [[요구사항 정의서]]

---

## 1. 자동 증가 (Auto Increment)

```sql
-- PostgreSQL
id BIGSERIAL PRIMARY KEY          -- SERIAL 타입이 시퀀스 자동 생성

-- MySQL
id BIGINT PRIMARY KEY AUTO_INCREMENT   -- AUTO_INCREMENT 키워드
```

---

## 2. 데이터 타입 비교

| 용도 | PostgreSQL | MySQL |
|------|-----------|-------|
| 자동증가 정수 | `BIGSERIAL` | `BIGINT AUTO_INCREMENT` |
| 가변 문자열 | `VARCHAR(100)` | `VARCHAR(100)` (동일) |
| 긴 텍스트 | `TEXT` | `TEXT` (동일하나 내부 동작 다름) |
| 불리언 | `BOOLEAN` (진짜 bool) | `TINYINT(1)` (0/1로 저장) |
| 날짜시간 | `TIMESTAMP` | `DATETIME` 또는 `TIMESTAMP` |
| UUID | `UUID` (네이티브 타입) | `CHAR(36)` 또는 `BINARY(16)` |
| JSON | `JSONB` (바이너리, 인덱싱 가능) | `JSON` (텍스트 저장, 인덱싱 제한적) |
| 정밀 숫자 | `NUMERIC(30,8)` | `DECIMAL(30,8)` |

---

## 3. JSON 처리

### PostgreSQL — JSONB

```sql
-- 바이너리 저장, 인덱스/검색 강력
parameters JSONB NOT NULL DEFAULT '{}'

-- GIN 인덱스로 JSON 내부 검색 가능
CREATE INDEX idx_params ON strategy_configs USING GIN (parameters);

-- 직접 조회
SELECT * FROM strategy_configs WHERE parameters->>'period' = '20';
```

### MySQL — JSON

```sql
-- 텍스트 저장, 기능 제한적
parameters JSON NOT NULL

-- Generated Column으로 인덱스 우회 필요
ALTER TABLE strategy_configs
  ADD period INT GENERATED ALWAYS AS (parameters->>'$.period');
CREATE INDEX idx_period ON strategy_configs(period);
```

---

## 4. 파티셔닝

### PostgreSQL — 선언적 파티셔닝 (유연)

```sql
CREATE TABLE candles (
    id BIGSERIAL,
    candle_date_time_utc TIMESTAMP NOT NULL,
    PRIMARY KEY (id, candle_date_time_utc)   -- 파티션 키 포함 필수
) PARTITION BY RANGE (candle_date_time_utc);

-- 파티션을 별도 테이블로 생성
CREATE TABLE candles_2026_02
  PARTITION OF candles
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
```

### MySQL — 파티셔닝 (제약 많음)

```sql
CREATE TABLE candles (
    id BIGINT AUTO_INCREMENT,
    candle_date_time_utc DATETIME NOT NULL,
    PRIMARY KEY (id, candle_date_time_utc)
) PARTITION BY RANGE (TO_DAYS(candle_date_time_utc)) (
    PARTITION p202602 VALUES LESS THAN (TO_DAYS('2026-03-01')),
    PARTITION p202603 VALUES LESS THAN (TO_DAYS('2026-04-01'))
);
-- 함수 사용에 제약, UNIQUE KEY에 파티션 키 필수 포함
```

---

## 5. 인덱스

### 부분 인덱스 (Partial Index)

```sql
-- PostgreSQL: 조건부 인덱스 지원
CREATE INDEX idx_event_publication_incomplete
    ON event_publication(completion_date)
    WHERE completion_date IS NULL;    -- NULL인 행만 인덱싱!

-- MySQL: 부분 인덱스 미지원
CREATE INDEX idx_event_publication_incomplete
    ON event_publication(completion_date);   -- 전체 행 대상만 가능
```

> PostgreSQL의 부분 인덱스는 인덱스 크기를 줄이고 성능을 높이는 데 효과적이다.

---

## 6. 프로시저 / 동적 SQL

### PostgreSQL — DO 블록

```sql
DO $$
DECLARE
    start_date DATE;
BEGIN
    EXECUTE format('CREATE TABLE %I PARTITION OF candles ...', partition_name);
END $$;
```

### MySQL — PREPARE/EXECUTE

```sql
DELIMITER //
CREATE PROCEDURE create_partition()
BEGIN
    SET @sql = CONCAT('ALTER TABLE candles ADD PARTITION ...');
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
END //
```

---

## 7. 이 프로젝트에서 PostgreSQL을 선택한 이유

| 기능 | PostgreSQL | MySQL |
|------|-----------|-------|
| `JSONB` + GIN 인덱싱 | ✅ 강력 | ⚠️ 제한적 |
| 부분 인덱스 (Partial Index) | ✅ | ❌ |
| 선언적 파티셔닝 | ✅ 유연 | ⚠️ 제약 많음 |
| `UUID` 네이티브 타입 | ✅ | ❌ |
| `BOOLEAN` 타입 | ✅ 진짜 bool | ❌ TINYINT |
| 동적 DDL (DO 블록) | ✅ | ⚠️ PROCEDURE 필요 |
| TimescaleDB 전환 가능 | ✅ 확장 | ❌ 불가 |

### 프로젝트 적용 포인트

- **전략 설정** → `JSONB`로 유연한 파라미터 저장 + 인덱싱
- **캔들 데이터** → 월별 파티셔닝으로 대량 시계열 데이터 관리
- **Spring Modulith 이벤트** → `UUID` 네이티브 타입 활용
- **미완료 이벤트 조회** → 부분 인덱스로 효율적 쿼리
- **추후 확장** → TimescaleDB (PostgreSQL 확장)로 무중단 전환 가능
