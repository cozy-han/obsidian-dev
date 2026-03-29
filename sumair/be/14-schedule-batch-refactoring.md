# 스케줄 배치 리팩토링 (과거 데이터 보존)

## 배경

배치에서 IBS API(RetrieveFlightSchedule, GetTimeTableInfo)를 호출하여 `PRD_SCHDUL_BAS` / `PRD_SCHDUL_DTL`에 스케줄 데이터를 동기화한다. 기존 로직은 전체 삭제 후 재삽입 방식이어서 과거 운항 이력이 소실되는 문제가 있었다.

---

## 기존 로직 (변경 전)

### `ScheduleServiceImpl.saveData()`

```java
public void saveData(ScheduleDto dto) {
    // 1. 마스터 조회/등록
    Long schdulSno = scheduleMapper.getSchdulSno(dto);
    if (schdulSno == null) {
        scheduleMapper.insertSchedule(dto);
    } else {
        dto.setSchdulSno(schdulSno);
        scheduleMapper.updateSchedule(dto);  // 날짜를 API 값으로 덮어쓰기
    }

    // 2. 상세 전체 삭제
    scheduleMapper.deleteScheduleDetail(dto);

    // 3. 상세 전체 재삽입
    scheduleMapper.insertScheduleDetailList(dto);
}
```

> `saveData()` 호출 자체가 주석 처리되어 있었음 (line 183)

### `updateSchedule` SQL

```sql
UPDATE PRD_SCHDUL_BAS
SET SCHDUL_START_DATE = #{schdulStartDate}    -- API 값으로 덮어쓰기
  , SCHDUL_END_DATE = #{schdulEndDate}        -- API 값으로 덮어쓰기
  , SCHDUL_EXPSR_START_DT = #{schdulExpsrStartDt}
  , SCHDUL_EXPSR_END_DT = #{schdulExpsrEndDt}
WHERE SCHDUL_SNO = #{schdulSno}
```

### `deleteScheduleDetail` SQL

```sql
DELETE FROM PRD_SCHDUL_DTL
WHERE CREAT_USER_ID = #{creatUserId}
AND SCHDUL_SNO = #{schdulSno}
```

### 문제점

| 문제 | 설명 |
|------|------|
| 과거 데이터 소실 | 전체 DELETE 후 API 데이터(오늘~12개월)만 INSERT → 과거 이력 삭제됨 |
| 마스터 날짜 축소 | API가 오늘 기준으로 조회하므로 START_DATE가 과거보다 최근 날짜로 덮어씌워짐 |
| 실행 불가 | 위 문제로 인해 `saveData()` 호출이 주석 처리되어 DB 갱신 자체가 안됨 |

---

## 변경 로직 (현재)

### `ScheduleServiceImpl.saveData()`

```java
public void saveData(ScheduleDto dto) {
    // STEP 1. 마스터 동기화 (LEAST/GREATEST로 과거 범위 보존)
    Long schdulSno = scheduleMapper.getSchdulSno(dto);
    if (schdulSno == null) {
        scheduleMapper.insertSchedule(dto);
    } else {
        dto.setSchdulSno(schdulSno);
        scheduleMapper.updateSchedule(dto);
    }

    // STEP 2. UPSERT: 신규 → INSERT, PK 중복 → UPDT_DT/USER만 갱신
    scheduleMapper.updateScheduleDetailList(dto);
}
```

> `saveData()` 주석 해제 완료

### `updateSchedule` SQL (변경)

```sql
UPDATE PRD_SCHDUL_BAS
SET SCHDUL_START_DATE = LEAST(SCHDUL_START_DATE, #{schdulStartDate})      -- 더 이른 날짜 유지
  , SCHDUL_END_DATE = GREATEST(SCHDUL_END_DATE, #{schdulEndDate})         -- 더 늦은 날짜로 확장
  , SCHDUL_EXPSR_START_DT = LEAST(SCHDUL_EXPSR_START_DT, #{schdulExpsrStartDt})
  , SCHDUL_EXPSR_END_DT = GREATEST(SCHDUL_EXPSR_END_DT, #{schdulExpsrEndDt})
  , UPDT_USER_ID = #{updtUserId}
  , UPDT_DT = NOW()
WHERE SCHDUL_SNO = #{schdulSno}
```

### `updateScheduleDetailList` SQL (신규 - UPSERT)

```sql
INSERT INTO PRD_SCHDUL_DTL(
    SCHDUL_SNO, FLIGHT_NO, NVG_START_DATE, NVG_END_DATE,
    ARCRFT_MODEL_CD, DEP_TM, ARR_TM, NVG_DAY_VALUE,
    CREAT_USER_ID, CREAT_DT, UPDT_USER_ID, UPDT_DT,
    ADD_DAY_VALUE, FLIGHT_SUFFIX_VALUE
) VALUES (...)
ON DUPLICATE KEY UPDATE
    UPDT_USER_ID = VALUES(UPDT_USER_ID),
    UPDT_DT = NOW()
```

### 동작 원리

| 상황 | 처리 |
|------|------|
| PK 없음 (신규 레코드) | INSERT |
| PK 있음 (기존 레코드) | UPDT_DT/USER만 갱신 (데이터 변경 없음) |
| DB에만 있는 레코드 | 건드리지 않음 (과거 이력 자연 보존) |

---

---

## saveData() 설계 과정 (검토했던 방안들)

최종 UPSERT 방식에 도달하기까지 여러 방안을 검토하고 문제점을 발견하여 수정해나갔다.

### 방안 1: 미래만 삭제 + INSERT IGNORE

```
DELETE WHERE NVG_END_DATE >= CURDATE()   → 미래 레코드 삭제
INSERT IGNORE                            → 과거는 PK 충돌 시 SKIP
```

**문제**: `NVG_START_DATE`가 과거이고 `NVG_END_DATE`가 미래인 레코드(과거~미래 걸침)도 삭제됨. 예: `01-29~10-24` 레코드의 `NVG_END_DATE >= 오늘`이므로 삭제 → 과거 이력 소실.

### 방안 2: NVG_START_DATE 기준 삭제 + INSERT IGNORE

```
DELETE WHERE NVG_START_DATE >= CURDATE()  → 순수 미래만 삭제
INSERT IGNORE                             → 나머지는 PK 충돌 시 SKIP
```

**문제**: INSERT IGNORE로 과거 데이터는 보존되지만, 과거~미래 걸친 레코드의 운항요일(NVG_DAY_VALUE) 등이 API에서 변경되어도 기존 값이 유지됨. 변경 사항 반영 불가.

### 방안 3: 3그룹 분류 + 서비스 로직 비교

```
그룹A (순수 과거, END < 오늘)    → INSERT IGNORE
그룹B (순수 미래, START >= 오늘)  → DELETE 후 INSERT
그룹C (과거~미래 걸침)           → DB 조회 후 9필드 비교
  Case A: 전부 일치 → UPDT_DT만 갱신
  Case B: DB에만 있음 → NVG_END_DATE=어제로 마감
  Case C: API에만 있음 → INSERT
  Case D: PK 매칭+값 다름 → 기존 마감 + 신규 INSERT
```

**문제**:
- ON DUPLICATE KEY UPDATE 사용 시 과거 데이터의 NVG_DAY_VALUE 등이 덮어씌워짐 (예: 월요일만 → 월수금으로 변경되면 과거도 변경)
- NVG_END_DATE를 어제로 수정하면 PK(NVG_END_DATE 포함)가 변경되어 다음 배치에서 원본과 비교 불가능 → 무한 누적 문제
- 서비스 로직이 과도하게 복잡

### 방안 4: 전체 비교 방식 (서비스에서 비교) 

```
1. API 범위의 DB 전체 조회
2. 9필드 비교: 일치 → touch, 불일치 → INSERT, DB전용 → 그대로 둠
```

**문제**: 서비스에서 하는 비교 로직이 결국 SQL의 ON DUPLICATE KEY UPDATE와 동일한 효과. 불필요한 복잡성.

### 최종 방안: ON DUPLICATE KEY UPDATE (UPSERT)

```sql
INSERT INTO PRD_SCHDUL_DTL(...) VALUES (...)
ON DUPLICATE KEY UPDATE
    UPDT_USER_ID = VALUES(UPDT_USER_ID),
    UPDT_DT = NOW()
```

**핵심 깨달음**:
- PK에 `NVG_DAY_VALUE`를 추가하면 운항요일 변경 시 별도 행으로 INSERT됨 → PK 충돌 없음
- PK가 동일한 레코드는 데이터가 같으므로 UPDT_DT만 갱신하면 충분
- DB에만 있는 레코드(API에서 더 이상 반환하지 않는 과거 데이터)는 건드리지 않음 → 자연 보존
- DELETE 불필요, 서비스 비교 로직 불필요 → SQL 한 줄로 해결

---

## 변경 비교 요약

| 항목 | 기존 | 변경 |
|------|------|------|
| 상세 저장 방식 | DELETE ALL → INSERT ALL | ON DUPLICATE KEY UPDATE (UPSERT) |
| 마스터 날짜 갱신 | API 값으로 덮어쓰기 | LEAST/GREATEST (과거 범위 보존) |
| 과거 데이터 | 삭제됨 | 영구 보존 |
| DB에만 있는 레코드 | 삭제됨 | 그대로 유지 |
| `saveData()` 실행 | 주석 처리 (미실행) | 주석 해제 (실행) |

---

## DDL 변경 (필요)

`PRD_SCHDUL_DTL` PK에 `NVG_DAY_VALUE` 추가. 운항요일이 변경된 경우 별도 레코드로 INSERT 가능하도록 한다.

```sql
ALTER TABLE PRD_SCHDUL_DTL DROP PRIMARY KEY;
ALTER TABLE PRD_SCHDUL_DTL ADD PRIMARY KEY
    (SCHDUL_SNO, FLIGHT_NO, NVG_START_DATE, NVG_END_DATE, DEP_TM, ARR_TM, NVG_DAY_VALUE);
```

### PK 변경 이유

기존 PK `(SCHDUL_SNO, FLIGHT_NO, NVG_START_DATE, NVG_END_DATE, DEP_TM, ARR_TM)`에서는 동일 편명+날짜범위+시각에 운항요일만 변경된 경우 PK 충돌이 발생한다. `NVG_DAY_VALUE`를 PK에 포함시켜 운항요일 변경 시 별도 행으로 관리한다.

---

## 조회 쿼리 변경 (AirlineScheduleMapper.xml)

데이터 누적으로 같은 편명+출발시각에 여러 레코드(범위 레코드 + 일별 레코드)가 공존할 수 있어 중복 방지 처리를 추가했다.

### selectScheuleList (주간 스케줄)

- 기존: `GROUP BY DEP_ARPRT_CD, ARR_ARPRT_CD, FLIGHT_NO, DEP_TM, ARR_TM`
- 변경: GROUP BY에 `PSB.SCHDUL_SNO, PSB.SCHDUL_START_DATE, PSB.SCHDUL_END_DATE, PSB.SCHDUL_EXPSR_START_DT, PSB.SCHDUL_EXPSR_END_DT` 추가
- 이유: MySQL 8.x `only_full_group_by` 모드 호환

### selectDepArrScheduleList (출도착 조회)

- 변경: 서브쿼리 + `ROW_NUMBER()` + `WHERE RN = 1`
- 정렬 우선순위:
  1. PSCB(지연/변경 테이블) 매칭된 레코드 우선
  2. 날짜 범위 좁은 것 우선 (일별 > 범위)
  3. 최신 갱신 순 (UPDT_DT DESC)

```sql
ROW_NUMBER() OVER (
    PARTITION BY PSD.FLIGHT_NO, PSD.DEP_TM
    ORDER BY CASE WHEN PSCB.FLIGHT_NO IS NOT NULL THEN 0 ELSE 1 END,
             (PSD.NVG_END_DATE - PSD.NVG_START_DATE) ASC,
             PSD.UPDT_DT DESC
) AS RN
```

### selectDepArrScheduleConfirm (운항정보확인서)

- 변경: 동일한 `ROW_NUMBER()` + `WHERE RN = 1` 적용

### selectScheduleDepartingWithin3Hours (탑승 3시간 전 알림)

- 변경: 동일한 `ROW_NUMBER()` + `WHERE RN = 1` 적용
- 위치: `ScheduleMapper.xml` (batch 모듈)

---

## IBS API 데이터 특성

GetTimeTableInfo API는 조회 기간에 따라 다른 형태의 데이터를 반환한다.

### 과거 구간 → 일별 레코드

```
XU2591, 2026-01-29 ~ 2026-01-29, Operationfreq=4  (목요일)
XU2591, 2026-01-30 ~ 2026-01-30, Operationfreq=5  (금요일)
```

- `NVG_START_DATE == NVG_END_DATE` (1일 단위)
- 648건/노선/3개월

### 미래 구간 → 범위 레코드

```
XU2591, 2026-04-01 ~ 2026-10-24, Operationfreq=1234567  (매일)
```

- `NVG_START_DATE != NVG_END_DATE` (수개월 범위)
- 7건/노선/3개월

### 누적 결과

배치가 매일 실행되면 과거 일별 레코드가 계속 누적되고, 미래 범위 레코드는 PK가 동일하면 UPDT_DT만 갱신된다. 시간이 지나면서 범위 레코드 구간이 일별 레코드로 대체되며 정확한 과거 이력이 구축된다.

---

## 연관 프로세스

### 배치 실행 흐름

```
SumairScheduler (Quartz)
  └→ ScheduleJob.execute()
       └→ ScheduleService.getFlightSchedule()
            ├→ 1. selectRouteBasList()          : PRD_ROUTE_BAS에서 USE_YN='Y' 노선 조회
            ├→ 2. getFlightScheduleRest()       : IBS RetrieveFlightSchedule API 호출 (FlightPort)
            │     → 노선별 스케줄 범위 목록 (ACTIVE 필터, 가장 이른/늦은 날짜 추출)
            ├→ 3. getTimeTableRest()            : IBS GetTimeTableInfo API 호출 (AvailabilityPort)
            │     → 3개월 단위 반복 호출 + 페이징 처리
            │     → 과거: 일별 레코드 / 미래: 범위 레코드
            ├→ 4. 중복 제거                      : flightNo+nvgStartDate+nvgEndDate+depTm+arrTm 기준
            └→ 5. saveData()                    : PRD_SCHDUL_BAS 마스터 동기화 + PRD_SCHDUL_DTL UPSERT
```

### 조회 프로세스 (home 모듈)

```
AirlineScheduleController
  ├→ getScheduleList()          : 주간 스케줄 조회 (selectScheuleList)
  │     → PRD_SCHDUL_BAS + PRD_SCHDUL_DTL JOIN
  │     → GROUP_CONCAT(NVG_DAY_VALUE)로 운항요일 합산
  │
  ├→ getDepArrScheduleList()    : 출도착 조회 (selectDepArrScheduleList)
  │     → PRD_SCHDUL_DTL + PRD_SCHDUL_CNG_BAS JOIN
  │     → 지연/결항 정보(PROC_CD, ACT_DEP_TM 등) 반영
  │     → ROW_NUMBER로 편명+시각당 1건 선택
  │
  └→ getDepArrScheduleConfirm() : 운항정보확인서 (selectDepArrScheduleConfirm)
       → 단건 조회 (편명+노선+날짜)
       → REASON(비정상 사유) 포함
```

### 탑승 알림 프로세스 (batch 모듈)

```
ScheduleJob 또는 ScheduleController
  └→ boardingReminder()
       ├→ 1. selectScheduleDepartingWithin3Hours()  : 3시간 내 출발 스케줄 조회
       │     → PRD_SCHDUL_DTL + PRD_SCHDUL_CNG_BAS JOIN
       │     → ROW_NUMBER로 중복 방지
       ├→ 2. getPassengerManifest()                 : IBS 탑승객 목록 조회
       ├→ 3. getSearchGuest()                       : IBS 체크인 상태 + 게이트 번호 조회
       └→ 4. sendApi() (Infobip)                    : 이메일/SMS 알림 발송
```

### 테이블 관계

```
PRD_ROUTE_BAS (노선기본)
  └→ PRD_SCHDUL_BAS (스케줄기본) : 노선별 1건, SCHDUL_SNO PK
       └→ PRD_SCHDUL_DTL (스케줄상세) : 편명별 다건, FK → SCHDUL_SNO
            └→ PRD_SCHDUL_CNG_BAS (스케줄변경기본) : 지연/결항 정보, FLIGHT_NO+DEP_DATE로 JOIN
```

---

## 수정 파일 목록

| 파일                                                   | 변경 내용                                                                                                                         |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `sumair-be-batch/...impl/ScheduleServiceImpl.java`   | `saveData()` UPSERT 방식으로 변경 + 주석 해제                                                                                           |
| `sumair-be-batch/...mapper/ScheduleMapper.java`      | `updateScheduleDetailList` 메서드 추가                                                                                             |
| `sumair-be-batch/...mapper/ScheduleMapper.xml`       | `updateSchedule` LEAST/GREATEST, `updateScheduleDetailList` UPSERT 추가, `selectScheduleDepartingWithin3Hours` ROW_NUMBER 중복 방지 |
| `sumair-be-home/...mapper/AirlineScheduleMapper.xml` | 3개 조회 쿼리 중복 방지 (ROW_NUMBER, GROUP BY 확장)                                                                                      |
| DDL                                                  | `PRD_SCHDUL_DTL` PK에 `NVG_DAY_VALUE` 추가                                                                                       |
