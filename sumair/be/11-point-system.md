# 포인트 시스템

## 포인트 코드 종류

| 코드     | 의미      | 방향 (I/D) | 설명                 |
| ------ | ------- | -------- | ------------------ |
| **AC** | 적립      | I (수입)   | 탑승 완료 후 실제 적립      |
| **PM** | 회원가입 지급 | I (수입)   | 가입 시 고정 2000포인트    |
| **SA** | 적립 예정   | I (수입)   | 예약 시 예상 적립 (보류 상태) |
| **US** | 사용      | D (지출)   | 결제 시 포인트 차감        |
| **EX** | 소멸      | D (지출)   | 3년 만료 소멸           |
| **CP** | 취소환불    | I (수입)   | 예약 취소 시 포인트 복원     |
| **SC** | 예정 취소   | D (지출)   | SA 포인트 배치 취소       |

**상수 정의:**
- `sumair-be-common/.../domain/pay/PointConstants.java`
- `sumair-be-pay/.../common/constants/PointConstants.java` (중복)

---

## DB 테이블

### PYM_POINT_BAS (포인트 원장)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| POINT_NO | Long (PK) | 포인트 고유번호 (시퀀스) |
| CSTMR_NO | String (FK) | 고객번호 |
| POINT_CD | String | 포인트 코드 (AC, PM, SA 등) |
| POINT_AMOUNT | Double | 포인트 금액 |
| REMAIN_AMOUNT | Double | 잔여 금액 |
| POINT_CREAT_DT | LocalDateTime | 생성일 |
| POINT_EXPIRD_DT | LocalDateTime | 만료일 (생성일 + 3년) |
| CREAT_DT | LocalDateTime | 레코드 생성일 |
| UPDT_DT | LocalDateTime | 레코드 수정일 |
| CREAT_USER_ID | String | 생성자 |
| UPDT_USER_ID | String | 수정자 |

### PYM_POINT_LOG_HIST (포인트 이력)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| POINT_USE_NO | Long (PK) | 이력 고유번호 (시퀀스) |
| POINT_NO | Long (FK) | PYM_POINT_BAS 참조 |
| ORD_NO | String | 주문번호 |
| CSTMR_NO | String | 고객번호 |
| POINT_CD | String | 거래 유형 |
| POINT_AMOUNT | Double | 거래 금액 |
| POINT_CREAT_DT | LocalDateTime | 포인트 생성일 (스냅샷) |
| POINT_EXPIRD_DT | LocalDateTime | 포인트 만료일 (스냅샷) |
| REMAIN_AMOUNT | Double | 거래 시점 잔여 금액 |
| CREAT_DT | LocalDateTime | 레코드 생성일 |
| UPDT_DT | LocalDateTime | 레코드 수정일 |
| CREAT_USER_ID | String | 생성자 |
| UPDT_USER_ID | String | 수정자 |

### PYM_POINT_CD (포인트 설정)

| 컬럼 | 타입 | 설명 |
|------|------|------|
| POINT_CD_NO | Long (PK) | 설정 고유번호 |
| POINT_CD | String | 포인트 코드 |
| POINT_CD_NAME | String | 코드명 (적립, 회원가입 지급 등) |
| POINT_PROP_CD | String | I(수입) / D(지출) |
| POINT_CREAT_RATE | Double | 적립률 (%) - AC/SA에서 사용 |
| POINT_CREAT_AMNT | Double | 고정 금액 - PM에서 2000 |

### POINT_VIEW (DB 뷰)

포인트 조회에 사용되는 뷰.

| 컬럼 | 설명 |
|------|------|
| CSTMR_NO | 고객번호 |
| AVL_POINT | 사용 가능 포인트 (SA 제외, 미만료, REMAIN > 0 합산) |
| EARN_POINT | 적립 예정 포인트 (SA의 REMAIN 합산) |

```sql
-- selectPoint: 뷰 조회
SELECT AVL_POINT, EARN_POINT, CSTMR_NO FROM POINT_VIEW
WHERE CSTMR_NO = #{cstmrNo}
```

---

## DTO 정의

### Request 객체

#### AddPointRQ
```java
// sumair-be-common/.../domain/pay/AddPointRQ.java
Long pointNo;
String cstmrNo;
String pointCd;
Double pointAmount;  // INSERT 할 포인트가 명확할때만 사용
Double totAmnt;      // 적립할때 필수 (DB 적립률로 포인트 금액 산정)
String creatUserId;
String ordNo;
```

> [!note] pointAmount vs totAmnt
> - `pointAmount` 설정: 해당 금액 그대로 저장
> - `totAmnt` 설정: `totAmnt * (POINT_CREAT_RATE / 100)` 계산하여 저장
> - 둘 다 null: `PYM_POINT_CD.POINT_CREAT_AMNT` 고정금액 사용 (PM용)

#### UsePointRQ
```java
// sumair-be-common/.../domain/pay/UsePointRQ.java
Number pointAmount;  // 결제 포인트 금액
String cstmrNo;
String ordNo;
String pointCd;      // "US"
```

#### CancelPointRQ
```java
// sumair-be-common/.../domain/pay/CancelPointRQ.java
String ordNo;
String creatUserId;
String cstmrNo;
Double pointAmount = 0.0;  // 0이면 전체 취소
```

#### RollbackPointRQ
```java
// sumair-be-common/.../domain/pay/RollbackPointRQ.java
String pointCd;       // "SC"
String creatUserId;
String ordNo;
Double pointAmount;   // null이면 전체 취소
```

#### SearchPointRQ / ExpectPointRQ
```java
// SearchPointRQ
String cstmrNo;
int pageNo;
String langDivCd;

// ExpectPointRQ
Double pymAmnt;  // 결제 금액
```

### Response 객체

#### SearchPointRS
```java
int avlPoint;                    // 사용 가능 포인트
int earnPoint;                   // 적립 예정 포인트
String cstmrNo;
List<PointDetail> pointDetailList;  // 거래 이력 (10건 페이징)
```

#### PointDetail
```java
String pointUseNo;     // 일련번호
String pointCd;        // 코드
String pointCreatDt;   // 포인트 생성일
String creatDt;        // 거래일
String depArprtCd;     // 출발 공항코드
String depArprtNm;     // 출발 공항명
String arrArprtCd;     // 도착 공항코드
String arrArprtNm;     // 도착 공항명
String pointPropCd;    // I(증가) / D(감소)
String pointCdName;    // 코드 이름 (다국어)
String pointAmount;    // 포인트 액수
String currentAmount;  // 현재 잔액 (누적 I_POINT - D_POINT)
```

#### ExpectPointRS
```java
Double pointAmnt;  // 예상 포인트
```

### 내부 DTO

#### PointDto (Pay 모듈)
```java
// sumair-be-pay/.../point/model/PointDto.java
Long pointNo, pointUseNo;
String cstmrNo, pointCd, ordNo, creatUserId, langDivCd;
Double pointAmount, currentPointAmount, totAmnt;
Double remainAmount, currentRemainAmount, pointAmnt;
LocalDateTime pointCreatDt, pointExpirdDt, creatDt, updtDt;
int avlPoint;            // 사용가능 포인트
int pageNo;
int page4Cnt = 10;       // 한 페이지 노출 수
int offset;
```

---

## REST API 엔드포인트

### Pay 모듈 (`:8200/point`)

| Method   | URI               | 설명             | 비고                        |
| -------- | ----------------- | -------------- | ------------------------- |
| POST     | `/point/add`      | 포인트 적립 및 적립 예정 | PM, SA 등                  |
| POST     | `/point/multiAdd` | 다중 적립 (배치용)    | AC 적립 + SA→SC 자동 처리       |
| POST     | `/point/rollback` | 적립 예정 취소       | SA → SC                   |
| POST     | `/point/use`      | 포인트 사용         | US                        |
| POST     | `/point/cancel`   | 포인트 취소 (환불)    | CP                        |
| GET/POST | `/point/expiry`   | 포인트 소멸 처리      | EX (배치 호출)                |
| POST     | `/point/search`   | 포인트 조회         | avlPoint + earnPoint + 이력 |
| POST     | `/point/change`   | 적립예정→적립 전환     | **미구현 (TODO)**            |
| GET      | `/point/expect`   | 예상 포인트 금액      | SA 적립률 기반 계산              |

> [!info] `/point/change` 미구현
> 컨트롤러 주석: "실제로 CD 전환 X → 새로 생성 (일행 노쇼일 경우 때문). 배치에서 전달받은 DATA 가공 후 MULTIADD 처리하는 과정"
> 실제로는 배치에서 `/point/multiAdd`를 통해 AC 생성 + SA 자동 취소 방식으로 처리됨.

### Home 모듈 (`:8000/mypage/point`)

| Method | URI | 설명 | 비고 |
|--------|-----|------|------|
| POST | `/mypage/point/search` | 포인트 조회 | → Pay `/point/search` 호출 |
| GET | `/mypage/point/expect` | 예상 포인트 | → Pay `/point/expect` 호출 |

### Batch 모듈 (`:8300/batch/point`)

| Method | URI | 설명 | 비고 |
|--------|-----|------|------|
| POST | `/batch/point/extinct` | 포인트 소멸 배치 | → Pay `/point/expiry` 호출 |
| POST | `/batch/point/save` | 포인트 적립 배치 | SFTP → Pay `/point/multiAdd` 호출 |

---

## 모듈간 통신 (WebClient)

```
[Home :8000]                   [IBE :8100]                  [Batch :8300]
     │                              │                            │
     │  PointWebClientService       │  payWebClientService       │  PointWebClientService
     │  - searchPoint()             │  - addPoint()              │  - updatePointExtinctRest()
     │  - addPoint()                │  - usePoint()              │  - updatePointSave()
     │  - expactPoint()             │  - cancelPoint()           │    clientId="BATCH"
     │                              │  - rollBackPoint()         │
     └──────────┬───────────────────┴────────────┬───────────────┘
                │          WebClient POST/GET     │
                └────────────────┬────────────────┘
                                 ▼
                          [Pay :8200]
                       PointRestController
                       PointServiceImpl
```

### Home → Pay
- `SumairHostUrl.getPay()` 로 Pay 모듈 URL 획득
- `SessionHelper.getClientId()` 로 헤더 설정

### IBE → Pay
- `SumairHostUrl.getPay()` 로 Pay 모듈 URL 획득
- 기본 WebClientHeader 사용

### Batch → Pay
- `${sumair.host.url.pay}` 프로퍼티로 URL 설정
- `clientId = "BATCH"` 고정

---

## 1. 회원가입 포인트 (PM)

**흐름:** 회원가입 → PM 2000포인트 즉시 적립

### 관련 파일
- `sumair-be-home/.../config/security/core/service/BaseUserService.java` (Line 156-167)
- `sumair-be-home/.../core/webclient/PointWebClientService.java` (Line 48-52)

### 동작
1. `BaseUserService.join()` 에서 회원가입 처리
2. 이름+생년월일+성별+전화번호 해시로 중복 가입 확인
3. 중복 가입이 아닌 경우(`!isHashCode`)에만 실행
4. `pointWebClientService.addPoint()` 호출

```java
// BaseUserService.java Line 156-167
if (!isHashCode) {
    pointWebClientService.addPoint(AddPointRQ.builder()
        .cstmrNo(user.getCstmrNo())
        .pointCd("PM")
        .creatUserId(user.getCstmrNo())
        .build());
}
```

5. Pay 모듈에서 `pointAmount`와 `totAmnt` 모두 null이므로:
   - `PYM_POINT_CD` 테이블에서 `POINT_CREAT_AMNT` 값(2000) 조회
   - `PYM_POINT_BAS`에 PM 레코드 INSERT (만료일 = 3년 후)
   - `PYM_POINT_LOG_HIST`에 이력 기록

### INSERT SQL (동적)
```sql
INSERT INTO PYM_POINT_BAS(POINT_NO, CSTMR_NO, POINT_CD, POINT_AMOUNT, ...)
VALUES (#{pointNo}, #{cstmrNo}, #{pointCd},
    (SELECT POINT_CREAT_AMNT FROM PYM_POINT_CD WHERE POINT_CD = 'PM'), ...)
```

---

## 2. 예약 시 예상 적립 (SA)

**흐름:** PNR 생성 성공 → SA 포인트 적립 (운임 x 적립률)

### 관련 파일
- `sumair-be-ibe/.../IbepBookingCreateBookingServiceImpl.java` (Line 403-406, 633-666)
- `sumair-be-pay/.../point/service/Impl/PointServiceImpl.java` (Line 39-66)
- `sumair-be-pay/.../point/mapper/PointMapper.xml` (Line 189-231)

### 동작
1. 예약(PNR) 생성 성공 후 실행 (PNR ACTIVE 상태)
2. `PYM_POINT_CD`에서 `POINT_CD='SA'`의 적립률 조회 (현재 5%)
3. 예약 응답의 FARE 합산 x 적립률로 예상 포인트 계산
4. `payService.addPoint()` 호출
5. `PYM_POINT_BAS`에 SA 레코드 INSERT (만료일 = 3년 후)

### 적립액 산출 로직

#### 산출 공식
```
적립 포인트 = FARE 합산액 x (POINT_CREAT_RATE / 100)
```

#### 적립률 조회 (Line 403-406)
```java
PymPointCd pymPointCd = new PymPointCd();
pymPointCd.setPointCd("SA");
List<PymPointCd> getPointRate = basePymPointCdMapper.selectAllBySearch(pymPointCd);
```

#### 적립 대상 금액 산출 (Line 634-644)
예약 응답(`createBookingRs`)의 결제 정보를 순회하며 FARE만 합산:

```java
for (GuestPaymentInfoType payInfo : createBookingRs.getGuestPaymentInfo()) {
    // 포인트 결제(PA)는 합산에서 제외
    if (!payInfo.getFormOfPaymentCode().toString().equals("PA")) {
        for (PaymentElementDetailsType payDetails : payInfo.getPaymentElementType()) {
            // FARE 항목만 합산 (TAX 등 제외)
            if (payDetails.getElementType().equals("FARE")) {
                totalFareAmnt += payDetails.getAmount();
            }
        }
    }
}
```

#### 최종 산출 및 저장 (Line 648-665)
```java
if (totalFareAmnt > 0) {
    addRQ.setCreatUserId(rq.getTossInfo().getUserId());
    addRQ.setCstmrNo(rq.getTossInfo().getUserId());
    addRQ.setOrdNo(rq.getOrdNo());
    addRQ.setPointAmount(totalFareAmnt * getPointRate.get(0).getPointCreatRate() / 100);
    addRQ.setPointCd("SA");
    payService.addPoint(addRQ);
}
```

#### 산출 규칙
| 항목 | 규칙 |
|------|------|
| 적립 대상 | `ElementType == "FARE"` 항목만 (TAX, 유류할증료 등 제외) |
| 포인트 결제 제외 | `FormOfPaymentCode == "PA"` 결제분은 합산 제외 |
| 적립률 | `PYM_POINT_CD` 테이블 `POINT_CD='SA'`의 `POINT_CREAT_RATE` (현재 5%) |
| 소수점 처리 | 미정 (TODO 주석 - 현재 5%라 문제없으나 정책 미수립) |

#### 에러 처리
적립 실패 시 예외를 catch하고 Slack 메시지 발송 (주문번호, 사용자ID, 금액 포함).
예약 자체는 실패하지 않음.

### 예시
```
운임(FARE) 100,000원 + TAX 20,000원 = 총 결제 120,000원
-> 적립 대상: FARE 100,000원만
-> 적립 포인트: 100,000 x 5% = 5,000 포인트 (SA로 저장)
```

### 예상 포인트 조회 API (PointServiceImpl.java Line 352-359)
- `GET /mypage/point/expect`
- Request: `{ pymAmnt: 100000 }`
- Response: `{ pointAmnt: 5000 }`

```java
PymPointCd cd = pointMapper.selectPointCd("SA");
Double pointCreatRate = cd.getPointCreatRate();
if (pointCreatRate == null) pointCreatRate = 1.0;  // null 방어 (기본 1%)
Double pointAmnt = rq.getPymAmnt() * (pointCreatRate / 100);
```

> [!warning] 예약 시 산출 vs 조회 API 차이
> 예약 시에는 IBS 응답의 FARE만 필터링하여 합산하지만, 조회 API(`/mypage/point/expect`)는 클라이언트가 전달한 `pymAmnt` 값을 그대로 사용한다. 프론트엔드에서 FARE 금액만 전달해야 정확한 예상치가 나온다.

---

## 3. 포인트 사용 (US)

**흐름:** 예약 결제 시 포인트 결제 선택 -> US 포인트 차감

### 관련 파일
- `sumair-be-ibe/.../IbepBookingCreateBookingServiceImpl.java` (Line 444-466)
- `sumair-be-pay/.../point/service/Impl/PointServiceImpl.java` (Line 261-302)
- `sumair-be-pay/.../point/mapper/PointMapper.xml` (Line 6-31)

### IBE 측 호출 로직 (Line 444-466)

```java
// 결제 수단에서 PA(포인트) 금액 합산
for (GuestPaymentInfoTypeDto payInfo : rq.getGuestPaymentInfo()) {
    if (payInfo.getFormOfPaymentCode().toString().equals("PA")) {
        totalPointAmnt += payInfo.getPaymentAmount();
    }
}

// 포인트 사용 호출
if (totalPointAmnt > 0) {
    useRQ.setCstmrNo(rq.getTossInfo().getUserId());
    useRQ.setOrdNo(rq.getOrdNo());
    useRQ.setPointAmount(totalPointAmnt);
    useRQ.setPointCd("US");
    payService.usePoint(useRQ);
}
```

### Pay 측 처리 로직 (PointServiceImpl.usePoint)

1. 사용 가능 포인트 조회 (SA 제외, REMAIN_AMOUNT > 0, 만료 전)
2. `POINT_VIEW`의 `AVL_POINT` 확인 → 부족 시 예외 발생
3. **만료일 가까운 순서로 차감** (FIFO)
4. 각 포인트 레코드의 `REMAIN_AMOUNT` 갱신
5. `PYM_POINT_LOG_HIST`에 US 이력 기록

```java
// 사용만료 가까운 포인트부터 사용
for (int i = 0; i < ppb.size(); i++) {
    if (pointAmnt - ppb.get(i).getRemainAmount().intValue() > 0) {
        // 현재 포인트 전체 소진
        pointAmnt -= ppb.get(i).getRemainAmount().intValue();
        ppb2.setPointAmount(ppb.get(i).getRemainAmount());
        ppb2.setRemainAmount(0.0);
    } else {
        // 현재 포인트에서 잔여분만 차감
        ppb2.setPointAmount((double) pointAmnt);
        ppb2.setRemainAmount(ppb.get(i).getRemainAmount() - pointAmnt);
        break;  // 차감 완료
    }
}
```

### 사용 가능 포인트 조회 SQL
```sql
SELECT POINT_NO, A.CSTMR_NO, POINT_CD, POINT_AMOUNT, POINT_EXPIRD_DT,
       REMAIN_AMOUNT, B.AVL_POINT
FROM PYM_POINT_BAS A, POINT_VIEW B
WHERE POINT_CD != 'SA'
  AND A.CSTMR_NO = B.CSTMR_NO
  AND A.CSTMR_NO = #{cstmrNo}
  AND REMAIN_AMOUNT != 0
  AND POINT_EXPIRD_DT > NOW()
ORDER BY POINT_EXPIRD_DT ASC
```

### 포인트 사용 실패 시 (PNR 생성 전)
포인트 사용 실패 시 Toss 결제 취소 처리 후 `PAYBACK` 상태 반환.

---

## 4. 예약 취소 시 포인트 환불 (CP)

**흐름:** 예약 취소 -> 해당 주문의 US 이력 조회 -> CP로 환불

### 관련 파일
- `sumair-be-ibe/.../IbepBookingCreateBookingServiceImpl.java` (Line 571-589)
- `sumair-be-pay/.../point/service/Impl/PointServiceImpl.java` (Line 121-182)
- `sumair-be-pay/.../point/mapper/PointMapper.xml` (Line 58-86)

### 동작
1. 주문번호로 US(사용) 이력 조회 (`selectPointLogHist4OrdNo`)
2. 취소 가능 금액 검증: `TOT_AMNT >= cancelAmount`
3. `pointAmount == 0` 이면 전체 취소 (남은 사용분 전체 환불)
4. US 이력 역순으로 순회하며 CP 레코드 생성
5. 원래 포인트 레코드의 `REMAIN_AMOUNT` 복원
6. `PYM_POINT_LOG_HIST`에 CP 이력 기록

### 취소 가능 금액 산출 SQL
```sql
-- TOT_AMNT: US 금액 합 - CP 금액 합 (이미 환불된 부분 차감)
SELECT SUM(IF(POINT_CD = 'US', POINT_AMOUNT, POINT_AMOUNT * -1)) AS TOT_AMNT
FROM PYM_POINT_LOG_HIST
WHERE POINT_CD IN ('US', 'CP') AND ORD_NO = #{ordNo}
```

### PNR 생성 실패 시 자동 취소
PNR 생성 실패 시 포인트 사용 건이 있다면 전체 취소 호출:
```java
if (totalPointAmnt > 0) {
    CancelPointRQ cancelRQ = new CancelPointRQ();
    cancelRQ.setCreatUserId(rq.getTossInfo().getUserId());
    cancelRQ.setOrdNo(rq.getOrdNo());
    // pointAmount 기본값 0.0 = 전체 취소
    payService.cancelPoint(cancelRQ);
}
```
실패 시 Slack 알림 발송.

---

## 5. 적립 예정 취소 - Rollback (SC)

**흐름:** SA 포인트 취소 요청 -> SC 코드로 이력 기록 -> REMAIN_AMOUNT = 0

### 관련 파일
- `sumair-be-pay/.../point/service/Impl/PointServiceImpl.java` (Line 191-258)
- `sumair-be-pay/.../point/mapper/PointMapper.xml` (Line 160-187)

### 동작

#### pointAmount 지정 시 (부분 취소)
1. 주문번호로 SA 포인트 조회 (`selectRollbackPoint`)
2. 취소 가능 금액 검증: `TOT_AMNT >= cancelAmount`
3. 각 SA 레코드 순회하며 잔액에서 차감
4. `PYM_POINT_BAS.REMAIN_AMOUNT` 갱신
5. `PYM_POINT_LOG_HIST`에 SC 이력 기록

#### pointAmount == null 시 (전체 취소)
1. 모든 SA 레코드의 `REMAIN_AMOUNT = 0` 처리
2. `pointAmount = 0`, `remainAmount = 0` 으로 BAS 갱신
3. SC 이력 기록

### Rollback 대상 조회 SQL
```sql
SELECT PPLH.*, PPB.POINT_AMOUNT AS CURRENT_POINT_AMOUNT,
       PPB.REMAIN_AMOUNT AS CURRENT_REMAIN_AMOUNT,
       (SELECT SUM(B.REMAIN_AMOUNT) FROM PYM_POINT_LOG_HIST A, PYM_POINT_BAS B
        WHERE A.POINT_NO = B.POINT_NO AND B.REMAIN_AMOUNT != 0
        AND A.POINT_CD = 'SA' AND A.ORD_NO = #{ordNo}) AS TOT_AMNT
FROM PYM_POINT_LOG_HIST PPLH, PYM_POINT_BAS PPB
WHERE PPLH.POINT_NO = PPB.POINT_NO
  AND PPB.REMAIN_AMOUNT != 0
  AND PPLH.POINT_CD = 'SA' AND PPLH.ORD_NO = #{ordNo}
ORDER BY POINT_USE_NO DESC
```

---

## 6. 배치 - 예상적립 -> 실제적립 변환

**흐름:** IBS Revenue Report -> 탑승 확인 -> AC 적립 + SA 취소

### 관련 파일
- `sumair-be-batch/.../job/PointSaveJob.java`
- `sumair-be-batch/.../point/service/impl/PointServiceImpl.java`
- `sumair-be-batch/.../point/mapper/PointMapper.xml`
- `sumair-be-batch/.../core/webclient/PointWebClientService.java`

### 동작

#### Step 1: IBS Revenue Report 다운로드
- IBS SFTP 서버에서 **3일 전** 날짜 기준 엑셀 다운로드
- 날짜 포맷: `ddMMMyyyy` (예: `22FEB2026`)
- 파일명: `DOC_LEVEL_REVENUE_REPORT_ID{date}_Part_1.xls`
- SFTP 설정: `ibs.ftp.host`, `ibs.ftp.user-name`, `ibs.ftp.password`, `ibs.ftp.port`, `ibs.ftp.path`

```java
Calendar day = Calendar.getInstance();
day.add(Calendar.DATE, -3);
String beforeDate = new SimpleDateFormat("yyyy-MM-dd").format(day.getTime());
SimpleDateFormat fmt_out = new SimpleDateFormat("ddMMMyyyy", Locale.ENGLISH);
String date = fmt_out.format(fmt_in.parse(beforeDate, new ParsePosition(0)));
String fileName = "DOC_LEVEL_REVENUE_REPORT_ID" + date + "_Part_1.xls";
```

#### Step 2: 엑셀 파싱 및 필터링

| 행/셀 | 내용 |
|--------|------|
| Row 8, Cell 4 | 항공사 코드 |
| Row 13+ | 데이터 시작 |
| Cell 9 | Type (TKT = 발권) |
| Cell 27 | Status (USED = 탑승 완료) |
| Cell 11 | PNR 번호 |
| Cell 5 | 편명 (Flight No) |

- 필터: `Type='TKT'` AND `Status='USED'`
- 같은 PNR+편명 조합은 중복제거, 승객수(cnt) 카운트

#### Step 3: DB 조회로 적립 데이터 구성
```sql
-- selectPointRq: PNR/항공편 기준 주문번호, 고객번호, 운임 합산
SELECT B.ORD_NO, OBB.CSTMR_NO, IFNULL(SUM(O.FARE_AMNT), 0) AS TOT_AMNT
FROM ORD_BOOK_BAS B
LEFT JOIN ORD_SEGMNT_BAS S ON B.BOOK_NO = S.BOOK_NO AND ...
LEFT JOIN ORD_BOOKER_BAS OBB ON B.BOOK_NO = OBB.BOOK_NO AND ...
LEFT JOIN (SELECT * FROM ORD_PSNGR_CHRG_TXN WHERE ... LIMIT #{cnt}) AS O
    ON S.BOOK_NO = O.BOOK_NO AND ...
WHERE B.LAST_TXN_YN = 'Y' AND B.PNR_NO = #{pnrNo}
```

#### NO_SHOW 대응 로직

IBS Revenue Report에서 `Status=USED`인 건만 추출하므로, NO_SHOW 승객은 자동 제외된다.

| 시나리오 | IBS 엑셀 USED 건수 | cnt | LIMIT 결과 | totAmnt |
|---|---|---|---|---|
| 성인 2명 모두 탑승 | 2건 | 2 | 2행 | 2명분 FARE |
| 성인 1명 탑승, 1명 NO_SHOW | 1건 | 1 | 1행 | 1명분 FARE |
| 전원 NO_SHOW | 0건 | - | 적립 대상 없음 | - |

> [!note] LIMIT의 한계
> `selectPointRq` SQL에서 `LIMIT #{cnt}`에 ORDER BY가 없어, 어떤 승객의 운임이 선택될지 보장되지 않는다. 현재는 동일 운임 클래스 예약이 대부분이라 문제없으나, 다른 운임이 섞이면 부정확해질 수 있다.

#### Step 4: AC 적립 및 SA 취소
- `AddPointRQ` 리스트 구성: `pointCd="AC"`, `creatUserId="BATCH"`, `totAmnt=FARE_AMNT합`
- Pay `/point/multiAdd` 호출
- **Pay 서버에서 포인트 계산**: `totAmnt × (POINT_CREAT_RATE / 100)` — 줄어든 totAmnt가 그대로 반영됨

#### MultiAddPoint 내부 동작 (Pay PointServiceImpl)
1. AC 포인트 배치 INSERT (3년 만료)
2. 자동으로 만료 SA 조회 및 취소:

```java
// 도착일 + 3일 경과한 SA 포인트 조회
List<String> ordNoList = pointMapper.selectSAPointList();

for (int i = 0; i < ordNoList.size(); i++) {
    RollbackPointRQ rrq = new RollbackPointRQ();
    rrq.setPointCd("SC");
    rrq.setOrdNo(ordNoList.get(i));
    rrq.setCreatUserId("ADMIN");
    rollbackPoint(rrq);  // SA -> SC 전환
}
```

#### 만료 SA 조회 SQL
```sql
SELECT ORD_NO FROM PYM_POINT_LOG_HIST
WHERE POINT_CD = 'SA'
  AND ORD_NO IN (
    SELECT A.ORD_NO FROM ORD_BOOK_BAS A
    LEFT JOIN (
        SELECT MAX(BOOK_NO) AS BOOK_NO, MAX(BOOK_SNO) AS BOOK_SNO,
               MAX(ARR_DATE) AS ARR_DATE
        FROM ORD_SEGMNT_BAS GROUP BY BOOK_NO, BOOK_SNO
    ) B ON A.BOOK_NO = B.BOOK_NO AND A.BOOK_SNO = B.BOOK_SNO
    WHERE A.LAST_TXN_YN = 'Y'
      AND NOW() > DATE_ADD(DATE_FORMAT(ARR_DATE, '%Y-%m-%d'), INTERVAL 3 DAY)
  )
```

> [!note] SA -> AC 변환 방식
> SA 포인트가 직접 AC로 변환되는 것이 아니라, **새로운 AC 레코드를 생성하고 기존 SA는 SC로 취소**하는 방식이다.
> `/point/change` API는 미구현 상태이고 실제로는 `/point/multiAdd`에서 일괄 처리됨.

---

## 7. 배치 - 포인트 소멸 (EX)

**흐름:** 매일 오전 5시 -> 만료된 포인트 EX 처리

### 관련 파일
- `sumair-be-batch/.../job/PointExtinctJob.java`
- `sumair-be-batch/.../core/webclient/PointWebClientService.java` (Line 44-46)
- `sumair-be-pay/.../point/service/Impl/PointServiceImpl.java` (Line 305-322)

### 동작
1. Quartz 스케줄러로 매일 실행 (`ADM_BATCH_BAS` 테이블 cron 설정)
2. Batch -> Pay `/point/expiry` 호출
3. `POINT_EXPIRD_DT < NOW()` 인 포인트 조회
4. 각 포인트에 대해:
   - `POINT_CD = 'EX'` 변경
   - `REMAIN_AMOUNT = 0` 처리
   - `CREAT_USER_ID = CommonConfig.getName()` (시스템명)
5. `PYM_POINT_LOG_HIST`에 소멸 이력 기록

### 만료 포인트 조회 SQL
```sql
SELECT POINT_NO, CSTMR_NO, POINT_CD, POINT_AMOUNT, ...
FROM PYM_POINT_BAS
WHERE POINT_EXPIRD_DT < NOW()
```

> [!warning] 주의
> SA 포인트도 3년 만료일이 지나면 EX 처리 대상이 됨. SA의 별도 필터링 없음.

---

## 8. 포인트 조회 API

### 마이페이지 포인트 조회
- `POST /mypage/point/search` (Home) -> `POST /point/search` (Pay)
- Request: `{ cstmrNo, pageNo, langDivCd }`
- Response: `{ avlPoint, earnPoint, cstmrNo, pointDetailList }`

### searchPoint 동작 (PointServiceImpl Line 324-347)
1. `POINT_VIEW`에서 `AVL_POINT`, `EARN_POINT` 조회
2. 포인트가 없으면 기본값 `avlPoint=0, earnPoint=0` 반환
3. `selectPointDetail`로 상세 이력 조회 (페이징)
   - `page4Cnt = 10` (한 페이지 10건)
   - `offset = page4Cnt * (pageNo - 1)`

### 상세 이력 조회 SQL (selectPointDetail)

주요 특징:
- **SC 코드 제외**: `WHERE POINT_CD != 'SC'` (사용자에게 SC 이력 미노출)
- **누적 잔액 계산**: `CURRENT_AMOUNT = I_POINT - D_POINT`
  - `I_POINT`: 해당 거래 시점까지의 I(수입) 포인트 합산 (SA 제외)
  - `D_POINT`: 해당 거래 시점까지의 D(지출) 포인트 합산
- **다국어 지원**: `COM_CD_LANG_CTG` 테이블에서 `LANG_DIV_CD` 기반 코드명 조회
- **공항명**: `PRD_ARPRT_CD_DTL` 테이블에서 다국어 공항명 조회
- **SA 특별 처리**: `IF(PPC.POINT_CD = 'SA', SUM(PPB.REMAIN_AMOUNT), SUM(PPLH.POINT_AMOUNT))`
  - SA는 현재 잔여금액(REMAIN_AMOUNT) 표시, 나머지는 거래금액(POINT_AMOUNT) 표시
- **0 포인트 제외**: `WHERE POINT_AMOUNT != 0`
- **정렬**: `CREAT_DT DESC, POINT_USE_NO DESC`

```sql
SELECT *, (I_POINT - D_POINT) AS CURRENT_AMOUNT
FROM (
    SELECT MAX(PPLH.POINT_USE_NO), PPLH.POINT_CD, ...
           PPC.POINT_PROP_CD,
           (SELECT CD_NM FROM COM_CD_LANG_CTG WHERE GROUP_CD = 'POINT_CD' AND ...) AS POINT_CD_NAME,
           IF(PPC.POINT_CD = 'SA', SUM(PPB.REMAIN_AMOUNT), SUM(PPLH.POINT_AMOUNT)) AS POINT_AMOUNT,
           OSB.DEP_ARPRT_CD, OSB.ARR_ARPRT_CD,
           MAX(IFNULL((SELECT SUM(POINT_AMOUNT) FROM PYM_POINT_LOG_HIST
               WHERE CREAT_DT <= PPLH.CREAT_DT
               AND POINT_CD IN (SELECT POINT_CD FROM PYM_POINT_CD WHERE POINT_PROP_CD = 'I' AND POINT_CD != 'SA')
           ), 0)) AS I_POINT,
           MAX(IFNULL((SELECT SUM(POINT_AMOUNT) FROM PYM_POINT_LOG_HIST
               WHERE CREAT_DT <= PPLH.CREAT_DT
               AND POINT_CD IN (SELECT POINT_CD FROM PYM_POINT_CD WHERE POINT_PROP_CD = 'D')
           ), 0)) AS D_POINT
    FROM PYM_POINT_LOG_HIST PPLH
    LEFT JOIN PYM_POINT_CD PPC ON ...
    LEFT JOIN ORD_BOOK_BAS OBB ON ...
    LEFT JOIN PYM_POINT_BAS PPB ON ...
    LEFT JOIN ORD_SEGMNT_BAS OSB ON ...
    WHERE PPLH.CSTMR_NO = #{cstmrNo} AND PPLH.POINT_CD != 'SC'
    GROUP BY PPLH.ORD_NO, PPLH.POINT_CD, PPLH.CREAT_DT
    ORDER BY PPLH.CREAT_DT DESC, POINT_USE_NO DESC
) A
WHERE POINT_AMOUNT != 0
LIMIT #{page4Cnt} OFFSET #{offset}
```

---

## INSERT SQL 분기 로직

`insertPointBas`와 `insertPointLogHist`에서 포인트 금액은 조건에 따라 3가지 방식으로 결정됨:

| 조건 | POINT_AMOUNT 결정 | 사용 케이스 |
|------|-------------------|-------------|
| `totAmnt != null` | `totAmnt * (POINT_CREAT_RATE / 100)` | 배치 AC 적립 |
| `pointAmount != null` | `#{pointAmount}` 그대로 | SA 예상적립, US 사용 |
| 둘 다 null | `POINT_CREAT_AMNT` (고정금액) | PM 회원가입 |

```xml
<if test="totAmnt != null">
    (#{totAmnt} * (SELECT POINT_CREAT_RATE FROM PYM_POINT_CD WHERE POINT_CD = #{pointCd}) / 100),
</if>
<if test="pointAmount != null">
    #{pointAmount},
</if>
<if test="pointAmount == null and totAmnt == null">
    (SELECT POINT_CREAT_AMNT FROM PYM_POINT_CD WHERE POINT_CD = #{pointCd}),
</if>
```

---

## Quartz 배치 스케줄링

### 구성
- `sumair-be-batch/.../config/quartz/QuartzConfiguration.java`
- `LocalDataSourceJobStore` (Spring DataSource 공유)
- 테이블 prefix: `QRTZ_`
- 클러스터링: enabled (`isClustered=true`)
- 스레드 풀: 20개
- **autoStartup: false** — `SumairScheduler`에서 정리 후 수동 시작

### 스케줄러 동적 등록
- `SumairScheduler.java` - `@EventListener(ApplicationReadyEvent.class)`에서 `ADM_BATCH_BAS` 테이블 조회
- Job 클래스명 매칭으로 동적 등록/갱신/삭제
- **시작 순서**: Job/Trigger 동기화 → 불필요 항목 정리 → `scheduler.start()` 수동 호출

| Job | 설명 | Cron (예상) |
|-----|------|-------------|
| `PointExtinctJob` | 포인트 소멸 | `0 0 5 * * ?` (매일 5AM) |
| `PointSaveJob` | 포인트 적립 (IBS Report) | DB 설정 기반 |
| `RevenueReportJob` | Revenue Report 수집 | DB 설정 기반 |

### 클러스터 환경 주의사항

> [!warning] 클러스터 노드 간 Job 클래스 불일치 문제
> Quartz 클러스터에서 동일 DB를 공유할 때, 특정 노드에만 Job 클래스가 존재하면 다른 노드에서 `ClassNotFoundException` → 트리거 ERROR 상태가 된다.
>
> **해결 방식**: `autoStartup=false`로 설정하여 Quartz 스레드가 즉시 시작되지 않도록 하고, `SumairScheduler.start()`에서 정리 후 수동 시작.
> - 이 노드에 클래스가 없는 Job → 트리거 일시정지 (`pauseTrigger`)
> - 불필요 Job/Trigger → 삭제
> - 정리 완료 후 `scheduler.start()` 호출
>
> **관련 파일**:
> - `QuartzConfiguration.java` — `setAutoStartup(false)`
> - `SumairScheduler.java` — ClassNotFoundException 시 트리거 pause + 정리 후 `scheduler.start()`

### Quartz 테이블 정리 (트러블슈팅)

ERROR 상태 트리거 리셋:
```sql
UPDATE QRTZ_TRIGGERS SET TRIGGER_STATE = 'WAITING'
WHERE SCHED_NAME = 'quartzTest' AND TRIGGER_STATE = 'ERROR';
```

QRTZ_TRIGGERS ↔ QRTZ_CRON_TRIGGERS 불일치 정리:
```sql
DELETE T FROM QRTZ_TRIGGERS T
LEFT JOIN QRTZ_CRON_TRIGGERS CT
  ON T.TRIGGER_NAME = CT.TRIGGER_NAME
  AND T.TRIGGER_GROUP = CT.TRIGGER_GROUP
  AND T.SCHED_NAME = CT.SCHED_NAME
WHERE T.SCHED_NAME = 'quartzTest'
  AND T.TRIGGER_TYPE = 'CRON'
  AND CT.TRIGGER_NAME IS NULL;
```

---

## 6-1. 배치 - Revenue Report 수집 (RevenueReportJob)

**흐름:** IBS SFTP Revenue Report → PRD_REVENUE_REPORT_TXN 테이블 저장

### 관련 파일
- `sumair-be-batch/.../job/RevenueReportJob.java`
- `sumair-be-batch/.../reservation/service/impl/ReservationServiceImpl.java`
- `sumair-be-batch/.../reservation/mapper/ReservationMapper.java/.xml`

### 동작

1. **마지막 처리일 조회**: `PRD_REVENUE_REPORT_TXN` 테이블에서 파일명 기반으로 최근 처리일+Part 번호 추출
2. **날짜 범위 순회**: 마지막 처리일 ~ 오늘까지 각 날짜별 파일 다운로드 시도
3. **파일명 패턴**: `DOC_LEVEL_REVENUE_REPORT_ID{ddMMMyyyy}_Part_{idx}.xls`
4. **엑셀 파싱**: 전체 행을 `PrdRevenueReportTxn` Entity로 변환 (TKT/USED 필터 없이 전체 저장)
5. **DB 저장**: `insertAllByDuplication` (중복 시 UPDATE)

### PRD_REVENUE_REPORT_TXN 주요 컬럼

| 컬럼 | 엑셀 셀 | 설명 |
|------|---------|------|
| flightDate | Cell 6 | 운항일 |
| origin | Cell 3 | 출발지 |
| destination | Cell 4 | 도착지 |
| flightNumber | Cell 5 | 편명 |
| documentType | Cell 9 | 문서 유형 (TKT 등) |
| documentNumber | Cell 10 | 문서번호 |
| trackingNumber | Cell 11 | 추적번호 |
| status | Cell 27 | 상태 (USED, OPEN 등) |
| documentAmount | Cell 14 | 문서 금액 |
| total | Cell 17 | 합계 |
| pointApplyStatus | - | 포인트 적용 상태 (INIT/SUCCESS/FAIL) |
| boardingApplyStatus | - | 탑승 적용 상태 (INIT/SUCCESS/FAIL) |

> [!info] PointSaveJob과의 차이
> `RevenueReportJob`은 Revenue Report **전체 데이터**를 DB에 저장하는 역할이고, `PointSaveJob`은 3일 전 리포트에서 TKT+USED만 필터링하여 **포인트 적립**하는 역할이다. 두 Job은 별도로 동작한다.

---

## 전체 흐름도

```
[회원가입] ------> PM 2000포인트 즉시 적립
                        │
[예약 생성] -----> SA 예상 포인트 적립 (운임 x 5%)
                        │
              ┌─────────┴─────────┐
              │                   │
        [포인트 결제]           [탑승 완료]
              │                   │
        US 포인트 차감         IBS Revenue Report에
              │                USED로 기록
              │                   │
              │              [배치 실행 (매일)]
              │                   │
              │             AC 실제 적립 생성 (/point/multiAdd)
              │              + SA -> SC 자동 취소
              │                   │
        [예약 취소]               │
              │                   │
     ┌────────┤              [3년 경과 (매일 5AM)]
     │        │                   │
     │   CP 포인트 복원       EX 소멸 처리 (/point/expiry)
     │
  [PNR 실패]
     │
  CP 자동 환불
  + Toss 결제 취소
```

---

## 예약 생성 (createBooking) 포인트 전체 흐름

```
1. [적립률 조회] - SA 적립률 미리 조회 (createBooking 시작 시)
       │
2. [Toss 결제 승인] - 카드 결제 처리
       │ (실패 시 PAYBACK 반환)
       │
3. [포인트 사용] - PA(포인트) 결제분 차감 (/point/use, pointCd="US")
       │ (실패 시 Toss 결제 취소 -> PAYBACK 반환)
       │ (실패 + Toss 취소 실패 시 Slack 알림 -> FAILED 반환)
       │
4. [PNR 생성] - IBS 예약 생성
       │
       ├── 성공: DB 저장 -> 5번으로
       │
       └── 실패:
           ├── 포인트 사용 건 있으면 -> 전체 취소 (/point/cancel)
           │   └── 실패 시 Slack 알림
           ├── Toss 결제 취소
           └── PAYBACK 반환
       │
5. [SA 적립] - FARE 합산 x 적립률 -> /point/add (pointCd="SA")
       │ (실패 시 Slack 알림, 예약 자체는 유지)
       │
6. [완료 반환]
```

---

## 관련 모듈 역할 정리

| 모듈 | 포트 | 포인트 역할 |
|------|------|-------------|
| **common** | - | 도메인 객체 (Entity, DTO), Mapper XML, 상수 정의 |
| **home** | 8000 | 회원가입 시 PM 적립, 마이페이지 포인트 조회/예상 API |
| **ibe** | 8100 | 예약 생성 시 SA 적립, 포인트 결제(US), 취소(CP), Rollback(SC) |
| **pay** | 8200 | 포인트 CRUD 핵심 로직 (적립/사용/환불/소멸/조회) |
| **batch** | 8300 | SA->AC 변환 (IBS SFTP), EX 소멸 배치 |
| **message** | 8400 | (포인트 직접 관련 없음) |

---

## 전체 파일 참조

### common 모듈
| 파일 | 설명 |
|------|------|
| `domain/pay/PointConstants.java` | 포인트 상수 (AC, PM, US, EX, CP, SA) |
| `domain/pay/AddPointRQ.java` | 적립 요청 DTO |
| `domain/pay/UsePointRQ.java` | 사용 요청 DTO |
| `domain/pay/CancelPointRQ.java` | 취소 요청 DTO |
| `domain/pay/RollbackPointRQ.java` | Rollback 요청 DTO |
| `domain/pay/ChangePointRQ.java` | 전환 요청 DTO (미사용) |
| `domain/pay/ExpectPointRQ.java` | 예상 포인트 요청 DTO |
| `domain/pay/ExpectPointRS.java` | 예상 포인트 응답 DTO |
| `domain/pay/SearchPointRQ.java` | 조회 요청 DTO |
| `domain/pay/SearchPointRS.java` | 조회 응답 DTO |
| `domain/pay/PointDetail.java` | 포인트 상세 이력 DTO |
| `domain/base/pym/PymPointBas.java` | 포인트 원장 Entity |
| `domain/base/pym/PymPointLogHist.java` | 포인트 이력 Entity |
| `domain/base/pym/PymPointCd.java` | 포인트 설정 Entity |
| `domain/base/prd/type/PointApplyStatus.java` | 포인트 적용 상태 Enum (INIT, SUCCESS, FAIL) |
| `mapper/BasePymPointBasMapper.java/.xml` | 원장 기본 Mapper |
| `mapper/BasePymPointLogHistMapper.java/.xml` | 이력 기본 Mapper |
| `mapper/BasePymPointCdMapper.java/.xml` | 설정 기본 Mapper |

### pay 모듈 (:8200)
| 파일 | 설명 |
|------|------|
| `point/controller/PointRestController.java` | REST 컨트롤러 (10개 엔드포인트) |
| `point/service/PointService.java` | 서비스 인터페이스 (9개 메서드) |
| `point/service/Impl/PointServiceImpl.java` | 핵심 비즈니스 로직 구현 |
| `point/mapper/PointMapper.java` | MyBatis Mapper 인터페이스 |
| `point/mapper/PointMapper.xml` | SQL 쿼리 (14개 쿼리) |
| `point/model/PointDto.java` | 내부 전달 DTO |
| `common/constants/PointConstants.java` | 상수 (common과 중복) |

### home 모듈 (:8000)
| 파일 | 설명 |
|------|------|
| `mypage/point/controller/MypagePointController.java` | 마이페이지 포인트 컨트롤러 |
| `mypage/point/service/MypagePointService.java` | 서비스 인터페이스 |
| `mypage/point/service/Impl/MypagePointServiceImpl.java` | 서비스 구현 (WebClient 위임) |
| `core/webclient/PointWebClientService.java` | Pay 모듈 호출 WebClient |
| `config/security/core/service/BaseUserService.java` | 회원가입 시 PM 적립 |

### ibe 모듈 (:8100)
| 파일 | 설명 |
|------|------|
| `core/http/webclient/payWebClientService.java` | Pay 모듈 호출 WebClient (결제+포인트) |
| `ibep/booking/create/booking/service/impl/IbepBookingCreateBookingServiceImpl.java` | 예약 시 SA 적립, US 사용, CP/SC 호출 |

### batch 모듈 (:8300)
| 파일 | 설명 |
|------|------|
| `job/PointSaveJob.java` | SA->AC 변환 Quartz Job |
| `job/PointExtinctJob.java` | EX 소멸 Quartz Job (매일 5AM) |
| `point/controller/PointController.java` | 배치 REST 컨트롤러 |
| `point/service/PointService.java` | 배치 서비스 인터페이스 |
| `point/service/impl/PointServiceImpl.java` | SFTP 다운로드 + 엑셀 파싱 |
| `point/mapper/PointMapper.java` | 배치 Mapper 인터페이스 |
| `point/mapper/PointMapper.xml` | 배치 SQL (selectSegmnt, selectPointRq) |
| `point/model/PointDto.java` | 배치 PointDto (pnrNo, airlineCd, flightNo, cnt) |
| `core/webclient/PointWebClientService.java` | Pay 모듈 호출 WebClient |
| `config/quartz/QuartzConfiguration.java` | Quartz 설정 |
| `SumairScheduler.java` | 동적 Job 등록 |




- 예약 취소시 ORD_PYM - 음수로 들어감
- PYM_REFND_CANCL_TXN - PNR_NO 안들어감.
- 