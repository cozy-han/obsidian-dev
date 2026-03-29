# Aurora Read/Write Splitting 적용

> AWS Advanced JDBC Wrapper의 `readWriteSplitting` 플러그인을 활용한 Aurora Reader/Writer 자동 라우팅
> 적용일: 2026-02-09

---

## 개요

### 적용 전

```
모든 쿼리 (SELECT/INSERT/UPDATE/DELETE)
        ↓
  Aurora Writer (Cluster Endpoint)
        ↓
  Reader 인스턴스 유휴 상태
```

### 적용 후

```
@Transactional(readOnly = true)     @Transactional / 기본
        ↓                                   ↓
  connection.setReadOnly(true)        connection.setReadOnly(false)
        ↓                                   ↓
  AWS JDBC Wrapper readWriteSplitting 플러그인
        ↓                                   ↓
  Aurora Reader                       Aurora Writer
```

---

## 구현 구조

### 1. application-databases.yml

`readWriteSplitting` 플러그인을 `failover` 앞에 추가 (플러그인 순서 중요).

```yaml
# local / dev / prod 공통
hikari:
  data-source-properties:
    wrapperPlugins: readWriteSplitting,failover,efm2,logQuery
    wrapperDialect: aurora-mysql
```

- JDBC URL은 **Cluster Endpoint(Writer) 하나만** 설정
- Wrapper가 `information_schema.replica_host_status` 쿼리로 Reader 인스턴스 자동 탐색
- Reader Endpoint(`cluster-ro-xxx`) 설정 불필요 — Wrapper가 개별 Reader 인스턴스에 직접 연결
- **message DataSource는 비 Aurora(일반 RDS)이므로 미적용**

### 2. MybatisConfig — TransactionManager 추가

`@Transactional(readOnly = true)` → `connection.setReadOnly(true)` 전파를 위해 `DataSourceTransactionManager` 필수.

| 모듈 | TransactionManager | 비고 |
|------|-------------------|------|
| home | `mainTransactionManager` | **신규 추가** |
| ibe | `mainTransactionManager` | **신규 추가** |
| message | `mainTransactionManager` | **신규 추가** |
| pay | `mainTransactionManager` | 기존 존재 |
| batch | `mainTransactionManager` | 기존 존재 |

### 3. 서비스 클래스 — `@Transactional` 어노테이션 패턴

**패턴**: class-level `@Transactional` + read 메서드에 `@Transactional(readOnly = true)` override

```java
@Service
@Transactional                              // 기본: Writer
public class SomeServiceImpl {

    public void insertData() { ... }        // class-level 상속 → Writer

    @Transactional(readOnly = true)         // override → Reader
    @Override
    public List<Dto> getList() { ... }
}
```

- class-level `@Transactional`: write 메서드의 트랜잭션 보장 (누락 방지)
- method-level `@Transactional(readOnly = true)`: Spring이 class-level을 **완전히 override** (속성 병합 아님)
- write 메서드는 class-level 상속으로 별도 `@Transactional` 선언 불필요 (중복 제거)
- 외부 API만 호출하는 메서드는 DB 연결을 사용하지 않으므로 제외
- **BATCH 모듈**: 다중 TransactionManager 사용으로 method-level에도 `transactionManager` 명시 필수

#### 프록시 기반 동작 주의

Spring `@Transactional`은 프록시 AOP 기반이므로 **동일 클래스 내부 호출(`this.method()`)은 프록시를 우회**한다.

| 호출 방식 | readOnly 적용 | 라우팅 |
|----------|--------------|--------|
| 외부 → readOnly 메서드 | 적용 | Reader |
| 외부 → write 메서드 → 내부 readOnly 메서드 | 무시 (호출자 트랜잭션 유지) | Writer |
| 외부 → readOnly 메서드 → 내부 write 메서드 | 무시 → **Reader에서 write 시도 → 에러** | Reader |

---

## 적용 파일 목록

### HOME 모듈 — 12개

| 파일 | class-level | readOnly 메서드 |
|------|-------------|----------------|
| `CommonCodeServiceImpl` | `@Transactional` | getCdBas, getAllCdBas |
| `CommonCountryServiceImpl` | `@Transactional` | getCntry |
| `CommonHolidayServiceImpl` | `@Transactional` | getHolidayList, getCalendar |
| `CommonMenuServiceImpl` | `@Transactional` | getMenuList, getMoreMenuList |
| `AirlineScheduleServiceImpl` | `@Transactional` | getScheduleList, getDepArrScheduleList, getDepArrScheduleConfirm |
| `AirlineCommonServiceImpl` | `@Transactional` | searchDepRoute, searchArrRoute, searchAriportCd |
| `AirlineCommonFareServiceImpl` | `@Transactional` | getFareGroupListByIntrlYn |
| `MainAdsServiceImpl` | `@Transactional` | getMainSlider, getMainPopup, getMainTopNotice |
| `MainSpecialServiceImpl` | `@Transactional` | getLwprcTicket, getBestPick |
| `CommonTermsServiceImpl` | `@Transactional` | getTemrsList, getAgreeCstmr, getAgreeCorp |
| `MainPageNoticeServiceImpl` | `@Transactional` | getMainNotice |
| `MypageReservationServiceImpl` | `@Transactional` | getMypageBookingList, searchCheckin, bookingSearchYn, getBookingUserSearch, getMainBannerSearch, getMypageSsrList, boardConfirm, getPayDetailHistory, getApproveCancelList, getPayReceiptList, getPassengerItinerary |

### IBE 모듈 — 1개

| 파일 | class-level | readOnly 메서드 |
|------|-------------|----------------|
| `ComnAirportServiceImpl` | `@Transactional` | selectMapAreaList, selectMapDepAirportList, selectMapArrAirportList, selectOneArprtDetail, getAirportKorCash, getAirportKorCashOne, getAirportNmKorCashOne |

### PAY 모듈 — 3개

| 파일 | class-level | readOnly 메서드 |
|------|-------------|----------------|
| `BookingPaymentServiceImpl` | `@Transactional` | getOrdNo, toss2Dto |
| `CouponServiceImpl` | `@Transactional` | searchCoupon |
| `PointServiceImpl` | `@Transactional` | changePoint, searchPoint, expactePoint |

### BATCH 모듈 — 2개

| 파일 | class-level | readOnly 메서드 |
|------|-------------|----------------|
| `HolidayServiceImpl` | `@Transactional(transactionManager = "mainTransactionManager")` | getHolidayList |
| `CustomerServiceImpl` | `@Transactional(transactionManager = "mainTransactionManager")` | getDormantAccountAlarm |

### MESSAGE 모듈 — 1개

| 파일 | class-level | readOnly 메서드 |
|------|-------------|----------------|
| `IresWebServiceImpl` | `@Transactional` | convertSendItrCreateEmailRQ, convertSendItrCancelEmailRQ, processScheduleChangeNotification, processCheckinOpenNotification |

---

## 주의사항

### Replication Lag

- Reader는 Writer 대비 수 ms ~ 수십 ms 지연 가능
- **INSERT 직후 SELECT**하는 로직이 있으면 Reader에서 데이터가 안 보일 수 있음
- 이런 경우 해당 메서드는 `readOnly = true`를 붙이지 않아야 함 (Writer로 가도록)

### readOnly 미설정 시 동작

- class-level `@Transactional`이 있으므로 모든 메서드는 기본 Writer로 라우팅
- `@Transactional(readOnly = true)` override가 있는 메서드만 Reader로 라우팅
- write 메서드에 `@Transactional` 누락해도 class-level이 보장 (안전)

### message DataSource

- 비 Aurora(일반 RDS)이므로 `readWriteSplitting` 플러그인 미적용
- `wrapperPlugins: logQuery`만 사용

### batch 모듈 Quartz DataSource

- Quartz는 별도 DataSource(`quartzDs`)를 사용하므로 readWriteSplitting 영향 없음

---

## 검증 방법

### 1. Writer/Reader 라우팅 확인

Aurora에서 현재 연결된 인스턴스 확인:

```sql
SELECT @@aurora_server_id;
```

- Writer: `sumair-dev-aurora-writer` 등
- Reader: `sumair-dev-aurora-reader` 등

### 2. AWS JDBC Wrapper 로그 확인

5개 모듈의 `logback-spring.xml`에 readWriteSplitting 로거 추가 완료 (local + dev/prod 모든 환경).

```xml
<logger level="DEBUG" name="software.amazon.jdbc.plugin.readwritesplitting"/>
```

Reader/Writer 전환 시 아래와 같은 로그 출력:
```
DEBUG readwritesplitting - Switching to reader connection: sumair-dev-aurora-reader.xxx
DEBUG readwritesplitting - Switching to writer connection: sumair-dev-aurora-writer.xxx
```

> 안정화 후 prod에서는 제거하거나 `INFO`로 변경 권장

### 3. 테스트 시나리오

| 시나리오 | 예상 결과 |
|---------|----------|
| `@Transactional(readOnly = true)` 메서드 호출 | Reader 인스턴스로 라우팅 |
| `@Transactional` 메서드 호출 | Writer 인스턴스로 라우팅 |
| `@Transactional` 없는 메서드에서 mapper 호출 | Writer 인스턴스로 라우팅 (기본값) |
| INSERT 후 같은 트랜잭션 내 SELECT | Writer에서 읽음 (정합성 보장) |
