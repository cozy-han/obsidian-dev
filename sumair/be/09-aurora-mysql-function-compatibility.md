# Aurora MySQL 마이그레이션 - Mapper XML 종합 분석

> MySQL → Aurora MySQL 전환 시 mapper XML 쿼리 패턴 및 함수 호환성 종합 점검
> 분석일: 2026-02-09 (재검증)
> 참고: `mysql2aurora-claude.md`, `mysql2aurora-gemini.md` 보고서 기반 실제 코드 재확인

---

## 결론 요약

| 심각도 | 건수 | 설명 |
|--------|------|------|
| **CRITICAL** | 5건 | 즉시 수정 필요 (데이터 정합성/쿼리 실패) |
| **HIGH** | 4건 | 마이그레이션 전 수정 권장 |
| **MEDIUM** | 3건 | Aurora 파라미터 설정으로 대응 |
| **SAFE** | 다수 | 함수 호환성 문제 없음 |

---

## CRITICAL - 즉시 수정 필요 (5건)

### C1. `@ROW_NUMBER` 사용자 정의 변수

**파일:** `sumair-be-home/.../checkin/showseatmap/mapper/AirlineCheckinSeatMapServiceMapper.xml` (line 18, 32)

```sql
SELECT (@ROW_NUMBER := @ROW_NUMBER + 1) AS 'NUMBER_IN_PARTY', ...
CROSS JOIN (SELECT @ROW_NUMBER := 0) AS DUMMY
```

- `:=` 대입 구문은 MySQL 8.0에서 **deprecated**
- Aurora MySQL 옵티마이저가 평가 순서를 다르게 처리 → 행 번호 **비정상 출력** 가능
- **수정 방안:** `ROW_NUMBER() OVER (ORDER BY ...)` 윈도우 함수로 교체
  - 단, MySQL 5.7 Aurora에서는 윈도우 함수 미지원 → Aurora 버전 확인 필요
  - Aurora MySQL 2.x = MySQL 5.7 호환, Aurora MySQL 3.x = MySQL 8.0 호환

---

### C2. SQL 버그 — `=` 연산자 누락

**파일:** `sumair-be-pay/.../point/mapper/PointMapper.xml` (line 378)

```sql
WHERE A.BOOK_NO=B.BOOK_NO AND A.BOOK_SNO AND B.BOOK_SNO
```

- `A.BOOK_SNO = B.BOOK_SNO`가 되어야 하는 곳에 **`=` 누락**
- MySQL에서 `A.BOOK_SNO AND B.BOOK_SNO`는 두 값을 boolean truthy로 평가 → 0이 아닌 한 항상 TRUE
- Aurora 마이그레이션과 무관한 **기존 버그** — JOIN 조건이 사실상 빠져있음
- **수정:** `AND A.BOOK_SNO = B.BOOK_SNO`

---

### C3. `ONLY_FULL_GROUP_BY` 위반 (4개 파일)

Aurora MySQL 3.x 기본 `sql_mode`에 `ONLY_FULL_GROUP_BY` 포함 → **쿼리 즉시 실패**

| 파일 | 위반 내용 |
|------|----------|
| `sumair-be-ibe/.../airport/mapper/ComnAirport.xml` (line 27) | `GROUP BY PACB.AREA_CD` — `SORT_ORDR` 누락 |
| `sumair-be-message/.../ires/mapper/IresMapper.xml` (line 23) | `GROUP BY pacb.ARPRT_CD` — `pacd.ARPRT_NM` 누락 |
| `sumair-be-batch/.../point/mapper/PointMapper.xml` (line 27) | `GROUP BY ARLINE_CD, FLIGHT_NO, DEP_TM` — BOOK_NO 등 다수 누락 |
| `sumair-be-pay/.../point/mapper/PointMapper.xml` (line 141) | `GROUP BY BOOK_NO, BOOK_SNO` — DEP_DATE 등 누락 |

- **수정 방안 A:** GROUP BY에 누락 컬럼 추가 또는 적절한 집계 함수(MAX, MIN 등) 사용
- **수정 방안 B:** Aurora 파라미터 그룹에서 `sql_mode`를 소스 MySQL과 동일하게 설정 (임시 우회)

---

### C4. 더블쿼트(`"`) 문자열 리터럴 (2개 파일)

Aurora `sql_mode`에 `ANSI_QUOTES` 포함 시 `"Y"`를 **컬럼명으로 인식** → 쿼리 실패

| 파일 | 라인 | 코드 |
|------|------|------|
| `ComnAirport.xml` | 28 | `CCB.GROUP_CD = "AREA_CD"` |
| `ComnAirport.xml` | 43, 69 | `IF(PCRR.ARPRT_CD IS NOT NULL, "Y", "N")` |
| `ComnAirport.xml` | 80 | `PCRR.DEP_YN = "N"` |
| `OrdBookBasMapper.xml` | 82, 94 | `LAST_TXN_YN = "Y"` |

- **수정:** 모든 문자열 리터럴을 싱글쿼트(`'`)로 변경

---

### C5. MAX() 기반 시퀀스/ID 생성 — Race Condition (3개 파일)

`SELECT ... FOR UPDATE` 없이 MAX 값으로 다음 ID 생성 → 동시 요청 시 **중복 ID 위험**

| 파일 | 코드 |
|------|------|
| `sumair-be-ibe/.../OrdBasMapper.xml` (line 6) | `SELECT IFNULL(MAX(SUBSTR(ORD_NO,2)),0) FROM ORD_BAS WHERE DATE(CREAT_DT) = DATE(NOW())` |
| `sumair-be-ibe/.../OrdBookBasMapper.xml` (line 6) | `SELECT IFNULL(MAX(BOOK_SNO),0) FROM ORD_BOOK_BAS` |
| `sumair-be-common/.../BaseComSeqMapper.xml` (line 5-19) | `selectKey` + IF/CYCL 복합 로직으로 시퀀스 생성 (원자적이지 않음) |

- 기존 MySQL 단일 인스턴스에서도 동시성 문제 존재하지만, Aurora Writer 전환 시 더 위험
- **수정 방안:** `SELECT ... FOR UPDATE` 추가 또는 DB auto-increment/시퀀스 테이블 사용

---

## HIGH - 마이그레이션 전 수정 권장 (4건)

### H1. deprecated `VALUES()` 함수 in `ON DUPLICATE KEY UPDATE`

**파일:** `sumair-be-batch/.../weather/mapper/WeatherMapper.xml` (line 57-66)

```sql
ON DUPLICATE KEY UPDATE
TMN = IFNULL(values(TMN), TMN),
TMX = IFNULL(values(TMX), TMX),
...
```

- `VALUES()` 함수는 MySQL 8.0.20에서 **deprecated**
- Aurora MySQL 3.x(8.0 호환)에서는 아직 동작하지만 향후 제거 예정
- **수정:** row alias 문법 `INSERT INTO ... AS new VALUES ... ON DUPLICATE KEY UPDATE col = new.col`

---

### H2. 세미콜론(`;`) separator 다중 UPDATE

**파일:** `sumair-be-pay/.../point/mapper/PointMapper.xml` (line 359-367)

```xml
<foreach collection="list" item="ppb" separator=";" close=";">
    UPDATE PYM_POINT_BAS SET ...
</foreach>
```

- JDBC URL에 **`allowMultiQueries=true` 필수** — 없으면 즉시 실패
- 현재 `application-databases.yml`의 JDBC URL에 `allowMultiQueries=true` 포함 여부 확인 필요
- **판정:** URL에 이미 포함되어 있으면 문제 없음 → 확인 완료 (local/dev/prod 모두 포함)

---

### H3. GROUP_CONCAT + INSERT 동일 테이블 서브쿼리

**파일:** `sumair-be-common/.../BasePrdRevenueReportTxnMapper.xml` (line 456-465)

```sql
INSERT INTO PRD_REVENUE_REPORT_TXN (...) VALUES
<foreach ...>
    (...,
      (SELECT GROUP_CONCAT(T.REVENUE_SNO)
       FROM PRD_REVENUE_REPORT_TXN T
       WHERE T.FLIGHT_DATE = #{item.flightDate} ...),
    ...)
</foreach>
```

- INSERT 대상 테이블을 서브쿼리에서 동시에 읽는 패턴
- 대량 배치 시 MVCC 스냅샷에 의해 비결정적 결과 가능
- `group_concat_max_len` 초과 시 데이터 무단 절삭
- **수정 방안:** `group_concat_max_len` Aurora 파라미터 그룹에서 명시적 설정

---

### H4. Multi-table UPDATE

**파일:** `sumair-be-home/.../customer/update/mapper/CustomerUpdateMapper.xml` (line 84-90)

```sql
UPDATE PTY_CSTMR_BAS PCB, PTY_CSTMR_DTL PCD
SET PCB.LOGIN_ID = #{loginId}, PCD.EMAIL = #{loginId}
WHERE PCB.CSTMR_NO = #{cstmrNo} AND PCD.CSTMR_NO = #{cstmrNo}
```

- Aurora 분산 스토리지에서 다중 테이블 UPDATE 시 부분 업데이트가 Reader에 노출 가능
- **수정 방안:** 별도 UPDATE 문으로 분리하고 트랜잭션으로 묶기 (또는 현행 유지 후 모니터링)

---

## MEDIUM - Aurora 파라미터 설정 필요 (3건)

### M1. `@@TIME_ZONE` 시스템 변수

**파일:** `sumair-be-batch/.../admin/mapper/AdminMapper.xml` (line 116)

```sql
INSERT INTO QRTZ_CRON_TRIGGERS (..., TIME_ZONE_ID) VALUES (..., @@TIME_ZONE)
```

- Aurora 파라미터 그룹에서 `time_zone = Asia/Seoul` 설정 필수 (기본값 UTC)
- **확인:** `SELECT @@TIME_ZONE;`

### M2. `GROUP_CONCAT` 최대 길이

- 8개+ 파일에서 사용 중 (AuthUserMapper, CustomerCommonMapper, AirlineScheduleMapper 등)
- 기본 `group_concat_max_len = 1024` — 잘림 시 LIKE/SUBSTRING_INDEX 결과 오류
- **설정:** Aurora 파라미터 그룹에서 `group_concat_max_len` 증가 (예: 10240)

### M3. `INSERT ... NOT EXISTS` 동일 테이블

**파일:** `sumair-be-batch/.../customer/mapper/CustomerMapper.xml` (line 81-102)

```sql
INSERT INTO PTY_CSTMR_STATUS_CNG_HIST (...)
SELECT ... FROM PTY_CSTMR_BAS
WHERE NOT EXISTS (SELECT 1 FROM PTY_CSTMR_STATUS_CNG_HIST WHERE ...)
```

- Aurora 분산 스토리지에서 locking 동작 차이 가능
- 배치 잡으로 실행되므로 동시성 위험은 낮음
- **판정:** 마이그레이션 후 동시성 테스트 권장

---

## SAFE - 함수 호환성 문제 없음

Aurora MySQL은 MySQL 5.7/8.0과 함수 레벨에서 **완전 호환**. 아래 함수들은 모두 정상 동작.

### 날짜/시간 함수

| 함수 | 사용 횟수 | 주요 사용 모듈 |
|------|----------|--------------|
| `NOW()` | 100+ | 전 모듈 (INSERT/UPDATE 시 CREAT_DT, UPDT_DT 등) |
| `DATE_FORMAT()` | 50+ | 전 모듈 |
| `DATE_ADD()` | 15+ | batch, home, pay, common |
| `DATE_SUB()` | 5+ | home, batch |
| `STR_TO_DATE()` | 5 | home, batch |
| `TIME_FORMAT()` | 30+ | home (MypageReservationMapper 집중) |
| `DATEDIFF()` | 3 | batch (CustomerMapper) |
| `CURDATE()` | 4 | home (CommonHolidayMapper) |
| `CURTIME()` | 1 | home (MainWeatherMapper) |
| `DATE()` | 3 | home, ibe |
| `YEAR()` | 2 | home, common |
| `MONTH()` | 1 | home |
| `DAYOFWEEK()` | 10+ | home (AirlineScheduleMapper) |
| `ADDDATE()` | 10+ | home (AirlineScheduleMapper) |
| `LAST_DAY()` | 1 | common (BaseComSeqMapper) |
| `MAKEDATE()` | 1 | common (BaseComSeqMapper) |

### 문자열 함수

| 함수 | 사용 횟수 | 주요 사용 모듈 |
|------|----------|--------------|
| `CONCAT()` | 30+ | 전 모듈 |
| `SUBSTRING_INDEX()` | 3 | home (MypageReservationMapper) |
| `SUBSTR()` | 1 | batch (ReservationMapper) |
| `REPLACE()` | 1 | batch (ReservationMapper) |
| `UPPER()` | 5+ | home (CustomerCommonMapper, AuthUserMapper) |

### 조건/집계 함수

| 함수 | 사용 횟수 | 주요 사용 모듈 |
|------|----------|--------------|
| `IF()` | 40+ | home, pay, common |
| `IFNULL()` | 20+ | home, pay, batch, ibe |
| `COALESCE()` | 1 | home (AirlineBookingCancelMapper) |
| `CASE WHEN` | 30+ | 전 모듈 |
| `GROUP_CONCAT()` | 10+ | home, common |
| `UUID()` | 2 | batch (AdminMapper) — Writer 전용, 문제 없음 |

### DML 패턴

| 패턴 | 사용 횟수 | 비고 |
|------|----------|------|
| `useGeneratedKeys` | 10 | Failover 시 auto-increment 갭 가능 (기능 문제 없음) |
| `selectKey` | 4 | common, batch |
| `ON DUPLICATE KEY UPDATE` | 1 | batch (WeatherMapper) — VALUES() deprecated 주의 (H1 참조) |

---

## 미사용 패턴 (검사 완료)

| 함수/패턴 | Aurora 주의사항 |
|-----------|---------------|
| `GET_LOCK()` / `RELEASE_LOCK()` | 멀티노드에서 노드간 Lock 공유 안됨 |
| `LOAD DATA LOCAL INFILE` | Aurora에서 제한적 지원 |
| `SQL_CALC_FOUND_ROWS` | MySQL 8.0.17+ deprecated |
| `FORCE INDEX` / `USE INDEX` | 옵티마이저 차이 |
| `LOCK TABLES` / `UNLOCK TABLES` | Aurora에서 권장하지 않음 |
| `REPLACE INTO` | - |
| `STRAIGHT_JOIN` | - |

---

## 마이그레이션 전 필수 조치 체크리스트

| 우선순위   | 조치                                                         | 파일                                               | 상태       |
| ------ | ---------------------------------------------------------- | ------------------------------------------------ | -------- |
| **1**  | Aurora `sql_mode` 확인 (`ONLY_FULL_GROUP_BY`, `ANSI_QUOTES`) | 파라미터 그룹                                          | [ ]      |
| **2**  | C1: `@ROW_NUMBER` → `ROW_NUMBER() OVER()`                  | AirlineCheckinSeatMapServiceMapper.xml           | [x] 완료   |
| **3**  | C2: SQL 버그 `=` 누락 수정                                       | pay/PointMapper.xml:378                          | [x] 완료   |
| **4**  | C3: GROUP BY 위반 4건 수정                                      | ComnAirport, IresMapper, PointMapper(batch,pay)  | [x] 완료   |
| **5**  | C4: 더블쿼트 → 싱글쿼트                                            | ComnAirport, OrdBookBasMapper                    | [x] 완료   |
| **6**  | C5: MAX() 시퀀스 FOR UPDATE 추가                                | OrdBasMapper, OrdBookBasMapper, BaseComSeqMapper | [x] 완료   |
| **7**  | H1: VALUES() → row alias 문법                                | WeatherMapper.xml                                | [x] 완료   |
| **8**  | `allowMultiQueries=true` JDBC URL 확인                       | application-databases.yml                        | [x] 확인완료 |
| **9**  | Aurora 파라미터: `time_zone`, `group_concat_max_len`           | 파라미터 그룹                                          | [ ]      |
| **10** | 마이그레이션 후 성능 테스트 (복잡 서브쿼리)                                  | MypageReservationMapper, PointMapper             | [ ]      |
