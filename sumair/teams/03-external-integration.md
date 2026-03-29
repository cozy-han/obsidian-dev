# 03. 외부 연동 가이드

> Sumair 백엔드가 연동하는 외부 시스템의 구조, 설정, 에러 처리, 장애 영향범위를 정리한 문서

---

## 목차

1. [IBES/IBEP - 항공 예약 SOAP](#1-ibesibep---항공-예약-soap)
2. [Toss Payments - 결제](#2-toss-payments---결제)
3. [Infobip - SMS/이메일/카카오 알림톡](#3-infobip---sms이메일카카오-알림톡)
4. [IRES - 메시지 SOAP](#4-ires---메시지-soap)
5. [data.go.kr - 공휴일/날씨 API](#5-datagoкр---공휴일날씨-api)
6. [소셜 로그인 (OAuth2)](#6-소셜-로그인-oauth2)
7. [장애 영향범위 매트릭스](#7-장애-영향범위-매트릭스)

---

## 1. IBES/IBEP - 항공 예약 SOAP

### 1.1 개요

| 항목 | 내용 |
|------|------|
| **프로토콜** | SOAP (Apache CXF 3.5.5) |
| **사용 모듈** | `sumair-be-ibe` (:8100), `sumair-be-common` |
| **공급사** | IBS Software (iFly) |
| **항공사 코드** | XU |
| **설정 파일** | `sumair-be-common/src/main/resources/application-ibe.yml` |

### 1.2 SOAP 엔드포인트

| 환경 | Booking WS | Check-in WS |
|------|-----------|-------------|
| **DEV/Local** | `https://xustg.ibsplc.aero/iRes_Booking_WS/services/` | `https://xustg.ibsplc.aero/iRes_CheckIn_WS/services/` |
| **PROD** | `https://xu.ibsplc.aero/iRes_Booking_WS/services/` | `https://xu.ibsplc.aero/iRes_CheckIn_WS/services/` |

### 1.3 IBES vs IBEP 구분

**IBES (IBS Enterprise Search)** - 조회 전용 서비스:
- `IbesRetrieveAirAvailService` - 항공편 가용성 조회
- `IbesRetrievePriceService` - 요금 조회
- `IbesRetrieveBookingService` - 예약 조회
- `IbesRetrieveAncillaryService` - 부가 서비스 조회
- `IbesRetrieveBaggageService` - 수하물 정보 조회
- `IbesRetrieveTimeTableService` - 시간표 조회

**IBEP (IBS Enterprise Process)** - 처리/변경 서비스:
- `IbepBookingCreateBookingService` - 예약 생성
- `IbepBookingModifyBookingService` - 예약 변경
- `IbepBookingSaveModifyBookingService` - 예약 변경 저장
- `IbepBookingCancelService` - 예약 취소
- `IbepBookingSplitBookingService` - PNR 분리
- `IbepBookingSyncBookingService` - 예약 동기화
- `IbepBookingAdjustFlightInventoryService` - 좌석 재고 조정
- `IbepBookingFlightScheduleService` - 운항 스케줄
- `IbepBookingFlightRetrievePassengerService` - 탑승객 목록 조회
- `IbepBookingCheckinService` - 체크인 처리
- `IbepProfileService` - 고객 프로필 관리

### 1.4 SOAP Port 구조

```
AvailabilityPort    → 항공편 가용성 (getAirAvailability, getEnhancedAirAvailability, getTimeTable)
PricePort           → 요금 (confirmPrice)
ReservationsPort    → 예약 (saveCreateBooking, retrieveBooking, modifyBooking, cancelBooking, splitPnr)
AncillaryPort       → 부가서비스
CustomerProfilePort → 고객 프로필
FlightPort          → 운항 정보
CheckinPort         → 체크인
```

### 1.5 인증 방식

SOAP Header에 `api_access_key` 삽입:

| 환경 | 채널 | API Access Key |
|------|------|---------------|
| DEV/Local (OTA) | OTAWEB | `XU-OTAWEB-webtour-webtoursumair` |
| DEV/Local (IBS) | WEBKRD | `XU-WEBKRD-iflysumibe-iflysumair` |
| PROD | - | 환경별 별도 키 |

세션 관리: `CLIENT_SESSION_ID` HTTP 헤더로 요청 간 세션 유지

### 1.6 핵심 구현 클래스

| 파일 | 역할 |
|------|------|
| `ibe/core/soap/SoapService.java` | SOAP RQ/RS 마샬링, HTTP/HTTPS 처리, JAXB 직렬화 |
| `ibe/core/Code.java` | Port 타입, 채널 코드 상수 정의 |
| `ibe/config/BeanConfig.java` | 채널별 설정 Bean 등록 |
| `common/core/properties/IbsConfig.java` | 설정값 매핑 |
| `common/core/utils/SoapFileLogger.java` | SOAP 메시지 파일 로깅 |

### 1.7 RQ/RS 모델

WSDL에서 자동 생성된 클래스 위치:
- `sumair-be-common/src/main/generated/com/ibsplc/ires/simpletypes/` (200+ 클래스)
- `sumair-be-common/src/main/generated/com/ibsplc/wsdl/` (Port 인터페이스)

주요 모델:
- `AirAvailabilityRQ/RS` - 항공편 조회
- `CreateBookingRQ/RS` - 예약 생성
- `RetrieveBookingRQ/RS` - 예약 조회
- `ModifyBookingRQ/RS` - 예약 변경
- `CancelBookingRQ/RS` - 예약 취소
- `ConfirmPriceRQ/RS` - 요금 확정
- `SplitPnrRQ/RS` - PNR 분리

### 1.8 에러 처리

```java
// 1) SOAP 응답에서 에러 확인
AirAvailabilityRS rs = SOAP_SERVICE.getResponse(rq);
if (rs.getErrorType() != null) {
    throw new IBSException(rs.getErrorType());
}

// 2) Controller에서 IBSException 매핑
// "AIRLINE_615" → ErrorCode.IBE_AVAIL_INVALID_PROMO_CODE
```

- `IBSException` 클래스가 SOAP ErrorType(errorCode, errorValue)을 래핑
- Controller에서 IBS 에러 코드를 내부 ErrorCode enum으로 매핑

### 1.9 SOAP 로깅

- 경로: `logs/soap-files/`
- 보존 기간: 365일
- 프로파일: local에서는 비활성화, dev/prod에서 활성화
- `SoapFileLogger.logRequest()` / `logResponse()`로 RQ/RS XML 전문 저장

---

## 2. Toss Payments - 결제

### 2.1 개요

| 항목 | 내용 |
|------|------|
| **프로토콜** | REST API (WebClient) |
| **사용 모듈** | `sumair-be-pay` (:8200) |
| **API 버전** | v1 |
| **Base URL** | `https://api.tosspayments.com/v1` |
| **설정 파일** | `sumair-be-pay/src/main/resources/application.yml` |

### 2.2 결제 유형 & 인증 키

| 유형 | 코드 | MID (DEV) | MID (PROD) |
|------|------|----------|-----------|
| **BrandPay** (브랜드페이) | BP | cp_sumairsecret | cp_sumair |
| **Direct** (일반결제) | WD | - | sumair2op |

- 인증: `Authorization: Basic {base64(secretKey + ":")}` 헤더
- 멱등성: POST 요청에 `Idempotency-Key: {UUID}` 헤더 자동 생성

### 2.3 API 엔드포인트

| 엔드포인트 | 메서드 | 용도 |
|-----------|--------|------|
| `/brandpay/authorizations/access-token` | POST | BrandPay 토큰 발급 |
| `/payments/confirm` | POST | 일반 결제 승인 |
| `/brandpay/payments/confirm` | POST | BrandPay 결제 승인 |
| `/payments/{paymentKey}` | GET | paymentKey로 조회 |
| `/payments/orders/{orderId}` | GET | orderId로 조회 |
| `/payments/{paymentKey}/cancel` | POST | 결제 취소/환불 |
| `/settlements` | GET | 정산 조회 |
| `/brandpay/customers/remove` | POST | BrandPay 고객 삭제 |

### 2.4 결제 승인 Flow

```
1. FE → POST /booking/payment/confirm (ConfirmPaymentRQ)
2. BE: Toss API로 결제 상태 조회 → 유효성 검증
   ├─ 금액 일치 확인 (rq.amount == payment.totalAmount)
   ├─ 상태 확인 (READY/IN_PROGRESS만 승인 가능)
   └─ paymentKey 중복 확인
3. BE: DB에 결제 정보 INSERT (PYM_PYM_BAS + PYM_PYM_LOG_HIST)
4. BE: Toss API POST /payments/confirm 호출
5. 승인 성공:
   ├─ DB UPDATE (상태=DN, 카드정보, 승인번호 등)
   └─ 응답: TossPayment + pymNo
6. 승인 성공 but DB 저장 실패:
   └─ 자동 cancelPayment() 호출 → Toss 결제 취소 (일관성 보장)
```

### 2.5 결제 취소/환불 Flow

```
1. FE → POST /booking/payment/cancel (CancelPaymentRQ)
2. BE: DB에서 결제 유형 조회 (paymentKey 기반)
3. BE: Toss API POST /payments/{key}/cancel
   └─ payload: { cancelReason, cancelAmount (0이면 전액취소) }
4. 취소 성공:
   ├─ DB UPDATE (총 취소금액, 상태)
   ├─ PymRefndCanclTxn INSERT (환불/취소 트랜잭션)
   └─ PYM_PYM_LOG_HIST INSERT (이력)
```

### 2.6 결제 상태 코드

| 코드 | Toss 상태 | 설명 |
|------|----------|------|
| RD | READY | 결제 준비 |
| IP | IN_PROGRESS | 진행 중 |
| DN | DONE | 승인 완료 |
| CC | CANCELED | 전액 취소 |
| PC | PARTIAL_CANCELED | 부분 취소 |
| AR | ABORTED | 중단 (승인 실패) |
| EP | EXPIRED | 만료 (30분 초과) |
| FL | FAILED | 실패 |

### 2.7 지원 결제 수단

Card, EasyPay, VirtualAccount, Phone, AccountTransfer, CultureGiftCertificate

### 2.8 Controller 엔드포인트

| 경로 | 메서드 | 설명 |
|------|--------|------|
| `/booking/payment/confirm` | POST | 결제 승인 |
| `/booking/payment/getAccessToken` | POST | BrandPay 토큰 |
| `/booking/payment/search` | POST | 결제 조회 |
| `/booking/payment/cancel` | POST | 결제 취소/환불 |
| `/booking/payment/getSettlements` | POST | 정산 조회 |
| `/booking/payment/remove` | POST | BrandPay 고객 삭제 |

### 2.9 에러 처리

- **실패 로깅**: `SaveFailLog` 서비스가 `PROPAGATION_NOT_SUPPORTED`로 독립 트랜잭션 사용 → 메인 트랜잭션 롤백 시에도 실패 이력 보존
- **에러 메시지 형식**: `{uri} / {status} / {code} / {message}`
- **금액 불일치**: `NOT_REGIST_ERROR_CODE` 에러
- **DB 실패 시**: Toss 승인 자동 취소 (데이터 정합성 보장)

### 2.10 DB 테이블

| 테이블 | 용도 | 주요 컬럼 |
|--------|------|----------|
| `PYM_PYM_BAS` | 결제 마스터 | PYM_NO, PYM_KEY, ORD_NO, TOT_AMNT, PYM_STATUS_CD |
| `PYM_PYM_LOG_HIST` | 결제 이력 | PYM_NO(FK), PYM_STATUS_CD, CREAT_DT |
| `PYM_REFND_CANCL_TXN` | 환불/취소 | CANCL_DELNG_NO, PYM_NO(FK), REFND_AMNT |

---

## 3. Infobip - SMS/이메일/카카오 알림톡

### 3.1 개요

| 항목 | 내용 |
|------|------|
| **프로토콜** | REST API (WebClient) |
| **사용 모듈** | `sumair-be-message` (:8400) |
| **Base URL** | `https://dmxnp8.api.infobip.com` |
| **인증** | `App {infobip.key}` Bearer 토큰 |
| **설정 파일** | `sumair-be-message/src/main/resources/application.yml` |

### 3.2 채널별 API 엔드포인트

| 채널 | API Path | 설명 |
|------|----------|------|
| **SMS** | `/sms/2/text/advanced` | 문자 발송 |
| **이메일** | `/email/4/messages` | 이메일 발송 (JSON v4, 템플릿 지원) |
| **카카오 알림톡** | `/kakao-alim/1/messages` | 알림톡 발송 |
| **카카오 리포트** | `/kakao-alim/1/reports?messageId={id}` | 발송 상태 확인 |

### 3.3 발신 정보

| 항목         | 값                                          |
| ---------- | ------------------------------------------ |
| SMS 발신번호   | `18774325`                                 |
| 이메일 발신     | `sumair@mail.sumair.kr`                    |
| 카카오 발신 프로필 | `6815f20134df8ec6e9c37e27c97fc72bd7c53494` |

### 3.4 통합 템플릿 시스템

통합 API: `POST /infobip/send` (InfobipTemplateController)

요청:
```json
{
  "templateKey": "pss_ticket_guide",
  "variables": "{\"email\":\"user@test.com\", \"pnrNo\":\"ABC123\", ...}"
}
```

주요 템플릿:

| 템플릿 키 | 채널 | 용도 |
|-----------|------|------|
| `reg_welcome` | 멀티채널 | 회원가입 완료 |
| `pss_ticket_guide` | 이메일 | 왕복 여정 안내 |
| `pss_ow_ticket_guide` | 이메일 | 편도 여정 안내 |
| `pss_cancel_guide` | 이메일 | 예약 취소 안내 |
| `pss_change_guide` | 이메일+카카오 | 스케줄 변경 안내 |
| `pss_checkin_guide` | 이메일+카카오 | 웹 체크인 안내 |
| `ibe_email_verify` | 이메일 | 이메일 인증 |
| `abnormal_operation_*` | 이메일 | 비정상 운항 (지연/결항) |

### 3.5 카카오 알림톡 상세

- 템플릿 DB: `kakao_alimtalk_templates_extended` 테이블
- 변수 치환: `#{variableName}` 패턴
- 버튼 지원: URL 타입, 최대 5개 (buttonName, urlMobile, urlPc)
- **SMS 폴백**: 카카오 발송 실패(REJECTED/FAILED) 시 자동 SMS 대체 발송

### 3.6 레거시 API (개별 채널)

| 경로 패턴 | 채널 |
|-----------|------|
| `POST /sms/members/send` | SMS 인증번호 |
| `POST /sms/reservation/*` | 예약 관련 SMS |
| `POST /email/members/*` | 회원 관련 이메일 |
| `POST /email/reservation/*` | 예약 관련 이메일 |

### 3.7 에러 처리

| 에러 코드 | 설명 |
|----------|------|
| `ERR_TEMPLATE_NOT_FOUND` | 템플릿 미존재 |
| `ERR_INVALID_JSON` | 변수 JSON 파싱 실패 |
| `ERR_INTERNAL` | 서버/API 오류 |

- 예외를 throw하지 않고 `InfobipTemplateRS`에 에러 코드/메시지 담아 반환
- 카카오 알림톡: 카카오+SMS 모두 실패 시 RuntimeException throw
- Slack 알림: dev 환경에서 `SlackNotificationService`로 에러 알림

---

## 4. IRES - 메시지 SOAP

### 4.1 개요

| 항목 | 내용 |
|------|------|
| **프로토콜** | SOAP (Apache CXF 3.5.5, 서버 역할) |
| **사용 모듈** | `sumair-be-message` (:8400) |
| **WSDL** | `http://{host}:8400/ires/message?wsdl` |
| **용도** | IBS(iFly)에서 Sumair로 메시지 발송 요청을 수신 |

IRES는 다른 연동과 달리 **Sumair가 SOAP 서버 역할**을 수행합니다.
IBS 시스템이 여정 안내, 스케줄 변경, 체크인 안내 등의 메시지 발송을 Sumair에 요청합니다.

### 4.2 SOAP Operations

| Operation            | 용도              | 트리거        |
| -------------------- | --------------- | ---------- |
| `sendItr`            | 전자항공권 여정 안내 이메일 | 예약 확정/취소 시 |
| `sendScheduleChange` | 스케줄 변경 안내       | 운항 변경 시    |
| `chkinOpen`          | 웹 체크인 안내        | 체크인 오픈 시   |

### 4.3 인증

SOAP Header에 UserInfo 요소 필수:

| 환경   | Username  | Credential             |
| ---- | --------- | ---------------------- |
| DEV  | `IBS_DEV` | `jyb8SzKkJPgDm6vm`     |
| PROD | `IBS_PRD` | `wN9vKz2mPL8qXj4tB1sY` |

`AuthInterceptor`가 SOAP 헤더에서 인증 정보를 추출/검증

### 4.4 SOAP 인터셉터

| 인터셉터 | 역할 |
|----------|------|
| `SoapCaseFixInterceptor` | 요청 XML 정규화 (header→Header, body→Body) |
| `SoapCaseFixOutInterceptor` | 응답 XML 정규화 |
| `AuthInterceptor` | SOAP 헤더 인증 검증 |

### 4.5 처리 흐름 (sendItr 예시)

```
1. IBS → SOAP POST /ires/message (sendItr)
2. AuthInterceptor: 인증 검증
3. SoapCaseFixInterceptor: XML 정규화
4. IresWebServiceImpl.sendItr():
   ├─ PNR 번호로 예약 정보 조회
   ├─ 편도(OW)/왕복(RT) 판별
   ├─ 템플릿 선택: pss_ow_ticket_guide / pss_ticket_guide / pss_cancel_guide
   ├─ Infobip 이메일 API 호출
   └─ 카카오 알림톡 발송 (해당되는 경우)
5. SOAP 응답 반환
```

---

## 5. data.go.kr - 공휴일/날씨 API

### 5.1 개요

| 항목 | 내용 |
|------|------|
| **프로토콜** | REST API (WebClient) |
| **사용 모듈** | `sumair-be-batch` (:8300) |
| **설정 파일** | `sumair-be-batch/src/main/resources/application.yml` |

### 5.2 API 정보

| API | URL | 용도 |
|-----|-----|------|
| 공휴일 | `https://apis.data.go.kr/B090041/openapi/service/SpcdeInfoService/getRestDeInfo` | 공휴일 목록 |
| 날씨 | `http://apis.data.go.kr/1360000/VilageFcstInfoService_2.0` | 동네 예보 |

- 응답 형식: XML
- 성공 코드: resultCode = `"00"`

### 5.3 배치 실행

- **스케줄러**: Quartz (클러스터링 활성화)
- **공휴일 Job**: `HolidayJob` - 매월 1일 05:00 실행
- **날씨 Job**: `WeatherJob` - 주기적 실행

Job/Trigger는 `ADM_BATCH_BAS` 테이블에서 동적으로 관리되며, `SumairScheduler`가 DB에서 읽어 Quartz에 등록

### 5.4 공휴일 데이터 처리

```
1. HolidayJob 실행
2. 당해년도 + 다음해 2년치 데이터 요청
3. XML 파싱 → HolidayHeader/HolidayBody 구조
4. 기존 공휴일 데이터 삭제 (DELETE)
5. 신규 데이터 INSERT (PTY_SPECIAL_DAY)
```

### 5.5 핵심 파일

| 파일 | 역할 |
|------|------|
| `batch/job/HolidayJob.java` | 공휴일 배치 Job |
| `batch/job/WeatherJob.java` | 날씨 배치 Job |
| `batch/holiday/service/impl/HolidayServiceImpl.java` | 공휴일 처리 로직 |
| `batch/core/webclient/HolidayWebClientService.java` | API 호출 WebClient |
| `batch/config/quartz/QuartzConfiguration.java` | Quartz 설정 |
| `batch/SumairScheduler.java` | 동적 Job/Trigger 등록 |

---

## 6. 소셜 로그인 (OAuth2)

### 6.1 개요

| 항목 | 내용 |
|------|------|
| **프레임워크** | Spring Security 5.x OAuth2 Login |
| **사용 모듈** | `sumair-be-home` (:8000) |
| **지원 Provider** | Kakao, Naver, Google |

### 6.2 OAuth2 로그인 Flow

```
1. FE → GET /oauth2/authorization/{provider}
2. BE: BaseOAuth2AuthorizationRequestResolver
   └─ state 파라미터 생성 (accessToken, redirectUrl, cstmrNo 포함, Base64 암호화)
3. 사용자 → Provider 인증 페이지
4. Provider → Callback with authorization code
5. BE: BaseOAuth2UserService / BaseOidcUserService
   └─ Provider에서 사용자 정보 조회
6. BE: BaseOAuth2SuccessHandler
   ├─ [신규 사용자] → joinCode 암호화 반환 (추가 정보 입력 필요)
   ├─ [기존 사용자] → JWT 토큰 발급
   ├─ [중복 계정] → DUPLICATE 상태 + 기존 계정 정보 반환
   └─ [계정 연결(REL)] → SNS 계정 연결 처리
7. FE → 리다이렉트 URL로 이동
```

### 6.3 Provider별 사용자 데이터 매핑

| 필드   | Kakao                    | Naver                  | Google                     |
| ---- | ------------------------ | ---------------------- | -------------------------- |
| ID   | `id`                     | `response.id`          | `sub`                      |
| 이메일  | `kakao_account.email`    | `email`                | `email`                    |
| 이름   | `kakao_account.nickname` | `name`                 | `given_name + family_name` |
| 성별   | `kakao_account.gender`   | `gender`               | -                          |
| 생년월일 | `birthday + birthyear`   | `birthday + birthyear` | -                          |
| 전화번호 | `phone_number`           | `mobile`               | -                          |

### 6.4 계정 상태 코드

| 상태      | 설명              | 로그인 가능     |
| ------- | --------------- | ---------- |
| ACTIVE  | 활성              | O          |
| DORMANT | 휴면              | X (활성화 필요) |
| LOCK    | 잠금 (5회 비밀번호 실패) | X          |
| WTHDR   | 탈퇴              | X          |

### 6.5 SNS 계정 연결

- DB 테이블: `PTY_SNS_LOGIN_REL`
- 기존 연결 해제 후 새 연결 생성 (1 SNS ID = 1 계정)
- 연결 해제 시 `linkYn='N'`, 탈퇴일시 기록

### 6.6 핵심 파일

| 파일 | 역할 |
|------|------|
| `home/config/security/SecurityConfiguration.java` | Spring Security 설정 |
| `home/config/security/oauth/BaseOAuth2AuthorizationRequestResolver.java` | state 파라미터 커스텀 |
| `home/config/security/oauth/BaseOAuth2SuccessHandler.java` | 로그인 성공 처리 |
| `home/config/security/oauth/BaseOAuth2FailureHandler.java` | 로그인 실패 처리 |
| `home/config/security/core/model/AuthOAuth2User.java` | Provider별 데이터 매핑 |
| `home/config/security/core/service/AuthUserService.java` | SNS 로그인 연결 |
| `home/config/security/core/service/BaseUserService.java` | 회원가입 (join) |

---

## 7. 장애 영향범위 매트릭스

### 외부 시스템 장애 시 영향

| 외부 시스템 | 장애 시 영향 범위 | 영향 수준 | 대체 수단 |
|------------|-----------------|----------|----------|
| **IBES/IBEP** | 항공편 조회, 예약, 변경, 취소, 체크인 **전면 불가** | **Critical** | 없음 (핵심 시스템) |
| **Toss Payments** | 결제 승인/취소 불가. 기존 예약 조회는 정상 | **High** | 없음 |
| **Infobip** | SMS/이메일/알림톡 발송 불가. 예약/결제 기능은 정상 | **Medium** | 없음 (발송 실패 로깅) |
| **IRES** | IBS→Sumair 메시지 수신 불가. 여정안내/체크인안내 미발송 | **Medium** | Infobip 직접 발송으로 우회 가능 |
| **data.go.kr** | 공휴일/날씨 갱신 실패. 기존 데이터 유지 | **Low** | 기존 DB 데이터 사용 |
| **Kakao OAuth** | 카카오 소셜 로그인 불가. 이메일/네이버/구글 가능 | **Low** | 다른 Provider 또는 이메일 로그인 |
| **Naver OAuth** | 네이버 소셜 로그인 불가 | **Low** | 다른 Provider 또는 이메일 로그인 |
| **Google OAuth** | 구글 소셜 로그인 불가 | **Low** | 다른 Provider 또는 이메일 로그인 |

### 모듈별 외부 의존성

```
sumair-be-home   (:8000) ─── Kakao/Naver/Google OAuth
sumair-be-ibe    (:8100) ─── IBES/IBEP SOAP
sumair-be-pay    (:8200) ─── Toss Payments REST
sumair-be-batch  (:8300) ─── data.go.kr REST
sumair-be-message(:8400) ─── Infobip REST, IRES SOAP (서버)
```

### 연쇄 장애 시나리오

| 시나리오 | 영향 |
|----------|------|
| IBES 장애 | 항공편 조회 불가 → 예약 불가 → 결제 불가 (전체 예약 플로우 중단) |
| Toss 장애 | 결제 불가 → 예약 확정 불가 (조회/변경/취소는 정상) |
| Infobip 장애 | 알림 미발송 → 고객 여정안내 누락 (핵심 기능에는 영향 없음) |
| DB(Aurora) 장애 | 전 모듈 서비스 중단 (모든 데이터 접근 불가) |
| Redis(Valkey) 장애 | 세션/캐시 실패 → 성능 저하 (DB 폴백으로 기능은 유지) |
