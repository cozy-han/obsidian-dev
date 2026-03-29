# 08. 배치 & 스케줄링

> Quartz 스케줄러 구성, 배치 작업 목록, 동적 Job 관리, 장애 시 재실행 전략

---

## 목차

1. [Quartz 아키텍처](#1-quartz-아키텍처)
2. [배치 작업 목록](#2-배치-작업-목록)
3. [동적 Job/Trigger 관리](#3-동적-jobtrigger-관리)
4. [배치 서비스 상세](#4-배치-서비스-상세)
5. [외부 API 호출](#5-외부-api-호출)
6. [장애 처리 & 재실행](#6-장애-처리--재실행)
7. [관리 API](#7-관리-api)

---

## 1. Quartz 아키텍처

### 구성 개요

| 항목 | 값 |
|------|-----|
| **모듈** | `sumair-be-batch` (:8300) |
| **Quartz 버전** | Spring Boot 2.7.10 내장 |
| **Job Store** | DB 기반 (JDBC, StdJDBCDelegate) |
| **테이블 Prefix** | `QRTZ_` |
| **클러스터링** | 활성화 (isClustered: true) |
| **스레드 풀** | SimpleThreadPool, 20 threads |
| **체크인 주기** | 15초 |
| **Misfire 임계값** | 60,000ms (1분) |

### 설정 파일

**application-quartz.yml:**
```yaml
org:
  quartz:
    scheduler:
      instanceName: quartzTest
      instanceId: AUTO
      batchTriggerAcquisitionMaxCount: 20
      idleWaitTime: 1000
      skipUpdateCheck: true
    threadPool:
      class: org.quartz.simpl.SimpleThreadPool
      threadCount: 20
    jobStore:
      driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
      useProperties: true
      misfireThreshold: 60000
      tablePrefix: QRTZ_
      isClustered: true
      clusterCheckinInterval: 15000
      acquireTriggersWithinLock: true
```

### 핵심 설정 클래스

| 클래스 | 파일 경로 | 역할 |
|--------|---------|------|
| `QuartzConfiguration` | `batch/config/quartz/QuartzConfiguration.java` | SchedulerFactoryBean, 트랜잭션 매니저 |
| `QuartsDataSource` | `batch/config/quartz/QuartsDataSource.java` | Quartz용 DataSource (main 재사용) |
| `AutowiringSpringBeanJobFactory` | `batch/config/quartz/AutowiringSpringBeanJobFactory.java` | Job에 @Autowired 주입 |
| `QuartzJobListner` | `batch/config/quartz/QuartzJobListner.java` | Job 실행 전/후 로깅 |
| `SumairScheduler` | `batch/SumairScheduler.java` | DB 기반 동적 Job/Trigger 등록 |

---

## 2. 배치 작업 목록

### 전체 Job 클래스 (7개)

| Job                     | 클래스                                  | 용도          | 기본 스케줄       |
| ----------------------- | ------------------------------------ | ----------- | ------------ |
| **HolidayJob**          | `batch/job/HolidayJob.java`          | 공휴일 데이터 갱신  | 매월 1일 05:00  |
| **WeatherJob**          | `batch/job/WeatherJob.java`          | 날씨 예보 수집    | 주기적 (3시간 간격) |
| **ScheduleJob**         | `batch/job/ScheduleJob.java`         | 운항 스케줄 동기화  | 매일 05:00     |
| **PointSaveJob**        | `batch/job/PointSaveJob.java`        | 마일리지 적립 처리  | 매일           |
| **PointExtinctJob**     | `batch/job/PointExtinctJob.java`     | 만료 포인트 소멸   | 매일 05:00     |
| **BoardingReminderJob** | `batch/job/BoardingReminderJob.java` | 탑승 3시간 전 알림 | 주기적          |
| **RevenueReportJob**    | `batch/job/RevenueReportJob.java`    | 매출 보고서 처리   | 매일           |

### Job 어노테이션

| Job | @DisallowConcurrentExecution | @PersistJobDataAfterExecution |
|-----|:---------------------------:|:-----------------------------:|
| BoardingReminderJob | O | O |
| 나머지 6개 | X | X |

`BoardingReminderJob`은 이전 실행 시간(prevFireTime/currFireTime)을 JobDataMap에 저장하여 중복 알림 방지

---

## 3. 동적 Job/Trigger 관리

### ADM_BATCH_BAS 테이블

| 컬럼 | 설명 |
|------|------|
| JOB_NO | PK |
| JOB_CLASS_NAME | Job 클래스명 (예: "HolidayJob") |
| CRON_EXPRESSION | Cron 표현식 (예: "0 0 5 1 * ?") |

### SumairScheduler 동작 Flow (ApplicationReadyEvent)

```
1. 애플리케이션 시작
   │
2. BLOCKED 트리거 복구
   │  └─ 이전 인스턴스 비정상 종료 시 남은 BLOCKED 상태 해제
   │
3. ADM_BATCH_BAS 테이블 조회
   │  └─ adminMapper.selectBatchList()
   │
4. Job 클래스별 그룹핑
   │
5. 각 Job에 대해:
   │  ├─ Class.forName("com.sumair.batch.job." + jobClassName)
   │  ├─ JobDetail 생성/확인
   │  │
   │  ├─ [기존 트리거와 비교]
   │  │  ├─ Cron 일치 → 유지
   │  │  ├─ Cron 불일치 → rescheduleJob
   │  │  ├─ ERROR/EXPIRED → 복구 후 재등록
   │  │  └─ 신규 Cron → CronTrigger 생성
   │  │
   │  └─ DB에 없는 트리거 → 삭제
   │
6. DB에 없는 Job → 삭제
   │
7. QuartzJobListner 등록
```

### 핵심 특징

- **핫 리로드**: 애플리케이션 재시작 시 DB 설정 자동 반영
- **페일오버**: BLOCKED 트리거 자동 복구
- **Misfire 정책**: `withMisfireHandlingInstructionFireAndProceed()` → 누락된 실행 즉시 처리
- **리플렉션**: `Class.forName()`으로 Job 클래스 동적 로딩

---

## 4. 배치 서비스 상세

### 4.1 HolidayService - 공휴일

```
HolidayJob.execute()
  └─ holidayService.getHolidayApi()
       ├─ 당해년도 + 다음해 데이터 요청 (2년치)
       ├─ data.go.kr API 호출 (XML 응답)
       ├─ resultCode == "00" 검증
       ├─ deleteHoliday() → PRD_HOLIDY_BAS 기존 데이터 삭제
       └─ updateHoliday() → PRD_HOLIDY_BAS 배치 INSERT
```

### 4.2 WeatherService - 날씨

```
WeatherJob.execute()
  └─ weatherService.getWeather()
       ├─ COM_WEATHER_COORD_BAS에서 공항 좌표 조회
       ├─ 가장 가까운 3시간 간격 기준시간 계산
       ├─ 공항별 API 호출 (3초 딜레이 - rate limiting 방지)
       ├─ 카테고리별 파싱: TMN, TMX, TMP, POP, PTY, PCP, SKY, VEC, WSD
       └─ COM_WEATHER_BAS INSERT ON DUPLICATE KEY UPDATE
```

### 4.3 ScheduleService - 운항 스케줄

```
ScheduleJob.execute()
  └─ scheduleService.getFlightSchedule()
       ├─ PRD_ROUTE_BAS에서 활성 노선 조회
       ├─ 노선별 12개월 스케줄 조회 (IBE 모듈 호출)
       ├─ 3개월 단위 페이지네이션 (API 제한)
       ├─ 중복 제거 후 INSERT
       └─ PRD_SCHDUL_BAS + PRD_SCHDUL_DTL 저장

  └─ boardingReminder()
       ├─ 3시간 이내 출발 항공편 조회
       └─ Infobip 이메일 알림 발송 (Message 모듈)
```

### 4.4 PointService - 포인트

**적립 (PointSaveJob):**
```
1. 적립 대상 조회 (탑승 완료 + 3일 경과, 미적립)
2. 탑승객별 요금 조회 (ORD_PSNGR_CHRG_TXN)
3. Revenue 탑승 이력 매칭 (금액 기반)
4. 적립 기본금 계산: 매칭요금 - (사용포인트 × 매칭요금/총요금)
5. Pay 모듈 /point/multiAdd 호출
```

**소멸 (PointExtinctJob):**
```
Pay 모듈 /point/expiry 호출 → 만료 포인트 일괄 소멸 처리
```

### 4.5 ReservationService - 매출 보고서

```
RevenueReportJob.execute()
  └─ reservationService.getRevenueReport()
       ├─ SFTP에서 Excel 파일 읽기
       │   └─ DOC_LEVEL_REVENUE_REPORT_ID_ddMMMyyyy_Part_N.xls
       ├─ PRD_REVENUE_REPORT_TXN에서 마지막 처리 위치 확인
       ├─ XLS 파싱 → 행 데이터 추출
       └─ DB INSERT (POINT_APPLY_STATUS, BOARDING_APPLY_STATUS)
```

---

## 5. 외부 API 호출

### 모듈 간 내부 호출

| WebClient 서비스 | 대상 모듈 | 엔드포인트 | 용도 |
|-----------------|---------|-----------|------|
| `IbeWebClientService` | IBE (:8100) | `/schedule/flightSchedule` | 운항 스케줄 |
| | | `/retrieveTimeTable/timeTable` | 시간표 상세 |
| | | `/passengerManifest/getPassengerManifest` | 탑승객 목록 |
| | | `/checkin/searchGuest` | 탑승객 조회 |
| `MsgWebClientService` | Message (:8400) | `/infobip/send` | 알림 발송 |
| `PointWebClientService` | Pay (:8200) | `/point/expiry` | 포인트 소멸 |
| | | `/point/multiAdd` | 포인트 적립 |

### 외부 API 호출

| WebClient 서비스 | 대상 | 엔드포인트 |
|-----------------|------|-----------|
| `HolidayWebClientService` | data.go.kr | `/getRestDeInfo` |
| `WeatherWebClientService` | data.go.kr | `/getVilageFcst` |

- 날씨 API: 타임아웃 30초, 3회 재시도 (3초 간격)

---

## 6. 장애 처리 & 재실행

### Misfire 처리

- **정책**: `FireAndProceed` → 누락된 실행을 즉시 1회 실행 후 정상 스케줄 복귀
- **임계값**: 60초 → 60초 이상 지연 시 misfire로 판단

### BLOCKED 트리거 복구

```java
// SumairScheduler 시작 시
// 이전 인스턴스의 BLOCKED 트리거 → WAITING으로 복구
for (TriggerKey triggerKey : blockedTriggers) {
    scheduler.resetTriggerFromErrorState(triggerKey);
}
```

### 클러스터 페일오버

- 15초 간격 체크인 → 비응답 인스턴스의 Job 다른 노드가 인수
- `acquireTriggersWithinLock: true` → 트리거 획득 시 락 사용

### AdminService 트리거 복구 API

```
PUT /batch/admin/trigger/recover

→ ERROR 상태 트리거: 새 CronScheduleBuilder로 재등록
→ EXPIRED nextFireTime: 재스케줄링
→ 반환: 복구된 트리거 목록 (triggerName, jobName, reason, oldTime, newTime)
```

### Job 레벨 에러 처리

- 모든 Job.execute()에서 try-catch로 예외 감싸기
- 예외 로깅 후 무시 (Job 실패가 스케줄러 중단시키지 않음)
- **별도 알림 없음** → 로그 모니터링으로 감지 필요

---

## 7. 관리 API

**Controller:** `batch/admin/controller/AdminController.java`
**Base Path:** `/batch/admin`

| 메서드 | 경로 | 설명 |
|--------|------|------|
| GET | `/jobs` | 전체 Job 목록 + 트리거 |
| GET | `/triggers` | 전체 트리거 목록 (상태, nextFireTime) |
| POST | `/trigger` | 트리거 추가 |
| PUT | `/trigger/cron` | Cron 표현식 변경 |
| DELETE | `/trigger` | 트리거 삭제 (ADM_BATCH_BAS 동기화) |
| PUT | `/trigger/pause` | 트리거 일시 중지 |
| PUT | `/trigger/resume` | 트리거 재개 |
| PUT | `/trigger/recover` | ERROR/EXPIRED 트리거 복구 |

### 운영 시나리오

**스케줄 변경:**
1. `PUT /trigger/cron` → Cron 표현식 변경
2. 또는 `ADM_BATCH_BAS` 테이블 직접 수정 후 애플리케이션 재시작

**긴급 중지:**
1. `PUT /trigger/pause` → 특정 Job 일시 중지
2. `PUT /trigger/resume` → 재개

**장애 후 복구:**
1. `PUT /trigger/recover` → ERROR 상태 트리거 자동 복구
2. 필요 시 `DELETE /trigger` + `POST /trigger`로 재생성
