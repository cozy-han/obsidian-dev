# 토스 이지페이(간편결제) 이슈 분석

> 분석일: 2026-03-11
> 최종 확인: 2026-03-12
> 상태: 전체 수정 완료

## 배경

토스 결제 응답에서 **카드 결제**는 `card` 객체에 정보가 담기고 `easyPay`는 null이다.
반면 **이지페이(간편결제)**는 `card`가 null이고 `easyPay` 객체에 정보가 담긴다.

### 토스 응답 비교

| 항목 | 카드 결제 | 이지페이 |
|------|----------|---------|
| `type` | BRANDPAY | NORMAL |
| `method` | 카드 | 간편결제 |
| `card` | `{ issuerCode, number, approveNo, ... }` | **null** |
| `easyPay` | null | `{ provider, amount, discountAmount }` |

---

## 1. NPE 에러 발생 지점

### 1-1. OrdServiceImpl — 비복합결제, 포인트 외 결제 ✅ 수정완료

**파일:** `sumair-be-ibe/.../ord/service/impl/OrdServiceImpl.java`
**라인:** 2902-2936

```java
if(payRS.getCard() != null) {
    pymDtlTxn.setPymMnCd(payRS.getCard().getIssuerCode());
    pymDtlTxn.setPymNoValue(payRS.getCard().getNumber());
    pymDtlTxn.setAprvlNo(payRS.getCard().getApproveNo());
    pymDtlTxn.setInstlMonthCnt(payRS.getCard().getInstallmentPlanMonths());
} else if(payRS.getEasyPay() != null) {
    // 이지페이 provider → 코드 변환 (TP/KP/NP/ET)
}
```

- null 체크 추가 및 이지페이 provider 코드화 완료

### 1-2. OrdServiceImpl — 복합결제(CP) ✅ 수정완료

**라인:** 2837-2864

```java
if (payRS.getCard() != null) {
    // 카드 / 이지페이 - 카드
    pymDtlTxn.setAprvlNo(payRS.getCard().getApproveNo());
    pymDtlTxn.setPymMnCd(payRS.getCard().getIssuerCode());
    ...
} else if (payRS.getEasyPay() != null) {
    // 이지페이 provider → 코드 변환 (TP/KP/NP/ET)
}
```

- null 체크 추가 및 이지페이 provider 코드화 완료

### 1-2-1. OrdServiceImpl — 복합결제 X, 결제 건 여러개 ✅ 수정완료

**라인:** 2961-2976

```java
if (payRS.getCard() != null) {
    pymDtlTxn.setAprvlNo(payRS.getCard().getApproveNo());
}
```

- null 체크 추가 완료

### 1-3. OrdServiceImpl — SSR 복합결제 ✅ 수정완료

**라인:** 3150-3215

```java
// 복합결제인 경우
if(rq.getGuestPaymentInfo().size() >=2 && iscp) {
    if(payRS.getCard() != null) { ... }
    else if(payRS.getEasyPay() != null) { ... }
} else if (dtlPymAmnt > 0) {
    if(payRS.getCard() != null) { ... }
    else if(payRS.getEasyPay() != null) { ... }
}
```

- 두 분기 모두 null 체크 + 이지페이 provider 코드화 완료

### 1-4. Slack 에러 알림

**IbepBookingCreateBookingServiceImpl.java** — 2곳 ✅ 수정완료

| 라인 | 상황 | 상태 |
| --- | --- | --- |
| 491-496 | 결제 취소 실패 catch | ✅ null 체크 + easyPay provider 추가 |
| 627-629 | PNR 생성 실패 후 결제 취소 실패 catch | ✅ null 체크 추가 |

**IbepBookingModifyBookingServiceImpl.java** — 2곳 ✅ 수정완료

| 라인 | 상황 | 상태 |
| --- | --- | --- |
| 658-663 | 포인트 에러 → 결제 취소 실패 catch | ✅ null 체크 + easyPay provider 추가 |
| 751-752 | PNR 변경 실패 → 결제 취소 실패 catch | ✅ null 체크 추가 |

---

## 2. 데이터 누락 (복합결제 이지페이 분기) ✅ 수정완료

기존 TODO 상태였던 이지페이 분기에 provider 코드 매핑 로직 추가됨.

다만 이지페이 특성상 아래 컬럼은 **의도적으로 비어있음** (카드 결제에만 해당하는 데이터):

| DB 컬럼 | 이지페이 시 | 비고 |
|----------|-----------|------|
| PYM_NO_VALUE | null | 카드번호 없음 (정상) |
| APRVL_NO | null | 승인번호 없음 (정상) |
| INSTL_MONTH_CNT | null | 할부 없음 (정상) |

채워지는 값:

| DB 컬럼 | 매핑 소스 | 비고 |
|----------|----------|------|
| PYM_MN_CD | provider → TP/KP/NP/ET | ✅ 코드화 완료 |
| PYM_KEY | paymentKey | ✅ |
| APRVL_DT | approvedAt | ✅ |
| ORDER_ID | orderId | ✅ |
| DELNG_NO | lastTransactionKey | ✅ |
| PYM_AMNT | payInfo.getPaymentAmount() | ✅ |

---

## 3. 안전한 지점 (변경 없음)

| 위치 | 이유 |
|------|------|
| `GuestPaymentInfoTypeDto.java:317, 397` | `FormOfPaymentCode == CC`일 때만 `getCard()` 호출. 이지페이는 ET로 분기되어 안전 |
| `BookingPaymentServiceImpl.java:364` | `if(tossPayment.getCard() != null)` null 체크 + easyPay 분기 추가됨 ✅ |

---

## 4. 남은 작업

없음 — 전체 수정 완료 ✅
