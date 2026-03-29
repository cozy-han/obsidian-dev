# 배치 포인트 적립 재설계

## 1. 배경

기존 PointSaveJob은 IBS FTP에서 Revenue Report XLS를 직접 다운로드하여 파싱 후 적립하는 방식.
RevenueReportJob이 별도로 Revenue 데이터를 `PRD_REVENUE_REPORT_TXN` 테이블에 적재하므로, DB 기반으로 재설계.

### 기존 문제점
- FTP 직접 접근으로 Revenue 적재와 이중 처리
- 포인트 사용분(복합결제) 미차감: `ORD_PSNGR_CHRG_TXN.FARE_AMNT`는 원본 FARE로 저장되어 포인트 사용분 반영 안됨
- Revenue에 passenger ID 없음 → 개별 탑승객 식별 불가
- SA ↔ AC 금액 불일치 발생 가능

## 2. 설계 원칙

- **적립 기준점**: 예약 기준 (ORD_NO 단위) + Revenue로 탑승 확인
- **적립 단위**: 티켓(전체 여정 완료) 단위
- **SA ↔ AC 처리**: 단순 교체 (SA → SC 전환 후 AC 신규 적립)
- **적립 기준 금액**: 탑승 확인된 FARE 합산 - 포인트 사용 비례분

## 3. 배치 처리 흐름 (Batch 서버)

### Step 1. 적립 대상 PNR 조회

**조건**:
- 마지막 여정 도착일 + N일 경과 (N=3일, Revenue 적재 지연 고려)
- PYM_POINT_BAS에 SA 존재, AC 미존재
- ORD_BOOK_BAS.LAST_TXN_YN = 'Y'

**산출물**: ORD_NO, BOOK_NO, BOOK_SNO, PNR_NO, CSTMR_NO 목록

### Step 2. 예약 정보 조회 (PNR 단위)

**조회 내용**:
- `ORD_SEGMNT_BAS`: 전체 여정 (SEGMNT_ID, FLIGHT_NO)
- `ORD_PSNGR_CHRG_TXN`: 탑승객별 × 여정별 FARE_AMNT (분리 저장 확인됨)
- `ORD_PYM_TXN`: 사용 포인트 금액 (POINT_AMNT)

### Step 3. Revenue 탑승 이력 조회

**조건**:
- `PRD_REVENUE_REPORT_TXN.TRACKING_NUMBER` = PNR
- `DOCUMENT_TYPE` = 'TKT' AND `STATUS` = 'USED'
- `POINT_APPLY_STATUS` = 'INIT' (미처리 건)

**산출물**: FLIGHT_NUMBER, DOCUMENT_AMOUNT 목록

### Step 4. 탑승객 매칭

편명 단위로 Revenue DOCUMENT_AMOUNT와 예약 FARE_AMNT를 금액 매칭.

#### 매칭 로직
1. 편명별로 예약측 FARE 목록과 Revenue측 DOCUMENT_AMOUNT 목록 생성
2. 정확히 일치하는 금액끼리 1:1 매칭 (매칭된 건은 양쪽에서 제거)
3. 동일 금액 복수 건은 순서대로 매칭 (금액 같으므로 결과 동일)

#### 결과 분류
| 분류 | 설명 | 처리 |
|------|------|------|
| 매칭 성공 | FARE = DOCUMENT_AMOUNT | 탑승 확정, 적립 대상 |
| Revenue 미매칭 | Revenue에 있으나 예약측 매칭 금액 없음 | 로그 기록, 적립 제외 |
| 예약 미매칭 | 예약에 있으나 Revenue 없음 | 미탑승 처리, 적립 제외 |

#### 미매칭 로그 기록
- ORD_NO, PNR_NO, FLIGHT_NO
- Revenue DOCUMENT_AMOUNT
- 예약측 후보 FARE_AMNT 목록
- 배치 실행일시
→ 추후 DOCUMENT_AMOUNT ↔ FARE_AMNT 불일치 패턴 분석 및 보완 로직 추가용

### Step 5. 적립 금액 산출

```
① 탑승 FARE 합산 = 매칭 성공한 탑승객들의 FARE_AMNT 합계
② 총 FARE = 전체 탑승객의 FARE_AMNT 합계
③ 사용 포인트 = OrdPymTxn.POINT_AMNT
④ 포인트 차감 = 사용 포인트 × (탑승 FARE / 총 FARE)
⑤ 적립 기준 금액 = 탑승 FARE - 포인트 차감분
⑥ AC 포인트 = floor(적립 기준 금액 × AC 적립률 / 100)
```

### Step 6. Pay 서버 호출

**전달 데이터**: ORD_NO, CSTMR_NO, pointCd='AC', 적립 금액
- 전원 미탑승 시 적립 금액 0으로 전달 → Pay에서 SC만 처리
- Pay에서 SA→SC 전환 + AC 적립 수행

### Revenue pointApplyStatus 업데이트
- 매칭 성공 건: `POINT_APPLY_STATUS` = 'SUCCESS'
- 매칭 실패 건: `POINT_APPLY_STATUS` = 'FAIL'

## 4. 데이터 구조

### ORD_PSNGR_CHRG_TXN 저장 구조
- **탑승객별 × 여정별** 분리 저장
- PK: PSNGR_CHRG_SNO (AUTO INCREMENT)
- 비즈니스 키: BOOK_NO + BOOK_SNO + SEGMNT_ID + PSNGR_ID

### PRD_REVENUE_REPORT_TXN 주요 필드
- TRACKING_NUMBER: PNR 번호
- FLIGHT_NUMBER: 편명
- DOCUMENT_AMOUNT: 금액 (FARE_AMNT 매칭용)
- DOCUMENT_TYPE: 'TKT' (티켓)
- STATUS: 'USED' (사용)
- POINT_APPLY_STATUS: INIT/SUCCESS/FAIL

## 5. 예시

```
예약: 성인+소아, 서울↔제주 왕복
  P001(성인): 가는편 50,000 / 오는편 60,000
  P002(소아): 가는편 40,000 / 오는편 48,000
  총 FARE: 198,000원, 사용 포인트: 60,000원

Revenue (TKT+USED):
  XU101: 50,000, 40,000 (2건)
  XU102: 60,000 (1건)

매칭 결과:
  P001×XU101: 50,000 → 매칭 O
  P002×XU101: 40,000 → 매칭 O
  P001×XU102: 60,000 → 매칭 O
  P002×XU102: 48,000 → Revenue 없음, 미탑승
  
산출:
  탑승 FARE = 150,000원
  포인트 차감 = 60,000 × (150,000 / 198,000) = 45,455원
  적립 기준 = 150,000 - 45,455 = 104,545원
  AC 포인트 = floor(104,545 × 5%) = 5,227P
```

## 6. 관련 파일

### Batch 모듈
- `PointSaveJob.java` - 배치 Job 엔트리
- `PointServiceImpl.java` - 포인트 적립 서비스
- `PointMapper.java` / `PointMapper.xml` - 적립 대상 조회 쿼리
- `PointWebClientService.java` - Pay 서버 호출

### Pay 모듈
- `/point/multiAdd` API - 다중 적립
- `PointServiceImpl.MultiAddPoint()` - AC 적립 + SA→SC 처리

### Common 모듈
- `AddPointRQ` - 적립 요청 DTO
- `PrdRevenueReportTxn` - Revenue 엔티티
- `OrdPsngrChrgTxn` - 탑승객 요금 엔티티
- `OrdPymTxn` - 결제 거래 엔티티








----


1. 적립 (Flown 된 탑승객의 가격에 대한 Fare 금액)
2. 저희 DB 기준으로 포인트 적립 예정인 PNR조회.(날자)
3. Revenue - 실제 탑승객 여부 확인하기 위함.
--- 트리거 및 data 구성 ----
4. 여정별 - 탑승객별 fare 금액 산출
5. 적립 포인트 산출
6. pay 서버 적립

왕복일 경우
가는편10 / 오는편5 - 오는편 기준으로 (시점)

pay 수정. ---- 10 / 5
예약시 - 포인트 예상 적립을 여정별로


전체금액 금액
여정 예약시 - 포인트 산출 확인(기존 fare금액으로 산출해야하는데 Tax 포함한 전체금액으로 하고 있음.) << 확인하고 피드백



