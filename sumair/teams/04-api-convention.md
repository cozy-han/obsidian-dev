# 04. API 통신 규약

> FE↔BE 간 통신 패턴, 토큰 관리, 응답 포맷, 에러 코드, 주요 엔드포인트 정리

---

## 목차

1. [공통 규약](#1-공통-규약)
2. [인증/토큰 관리](#2-인증토큰-관리)
3. [응답 포맷](#3-응답-포맷)
4. [에러 코드 체계](#4-에러-코드-체계)
5. [요청 검증 패턴](#5-요청-검증-패턴)
6. [필터/인터셉터 체인](#6-필터인터셉터-체인)
7. [주요 엔드포인트](#7-주요-엔드포인트)

---

## 1. 공통 규약

### 1.1 기본 설정

| 항목 | 값 |
|------|-----|
| Content-Type | `application/json` |
| 세션 관리 | STATELESS (JWT 기반) |
| CORS | 모든 Origin 허용 (`allowedOriginPattern: *`) |
| CSRF | 비활성화 |
| Form Login | 비활성화 |
| HTTP Basic | 비활성화 |

### 1.2 모듈별 Base URL

| 모듈 | 포트 | 용도 |
|------|------|------|
| home | `:8000` | 인증, 회원, 예약 조회, 마이페이지 |
| ibe | `:8100` | 항공편 검색, 예약 SOAP 프록시 |
| pay | `:8200` | 결제 처리 |
| message | `:8400` | 메시지 발송 |

### 1.3 네이밍 컨벤션

- **요청 DTO**: `*RQ` (예: `ConfirmPaymentRQ`, `LoginFormRQ`)
- **응답 DTO**: `*RS` (예: `CustomerInfoRS`, `GetAvailRS`)
- **Controller**: `@RestController` + `@RequestMapping`
- **HTTP 메서드**: 대부분 `POST` 사용 (조회 포함)
- **Swagger**: `@Api(tags)`, `@ApiOperation`, `@ApiModelProperty` 어노테이션

### 1.4 CORS 설정

```java
// CorsConfig.java
CorsConfiguration config = new CorsConfiguration();
config.setAllowCredentials(true);
config.addAllowedOriginPattern("*");
config.addAllowedHeader("*");
config.addExposedHeader("*");        // Auth-Token, Refresh-Token 노출
config.addAllowedMethod("*");

// SimpleCorsFilter.java (서블릿 필터 - 이중 적용)
Access-Control-Allow-Methods: POST, GET, OPTIONS, DELETE, PUT, PATCH
Access-Control-Max-Age: 3600
```

---

## 2. 인증/토큰 관리

### 2.1 JWT 토큰 사양

| 항목 | 값 |
|------|-----|
| 알고리즘 | HS256 |
| Issuer | `sumair` |
| Access Token 유효기간 | 30분 |
| Refresh Token 유효기간 | 30일 |

### 2.2 토큰 페이로드 (Claims)

```json
{
  "sub": "auth",           // "auth" 또는 "refresh"
  "cstmrNo": "C00001",    // 고객번호
  "cstmrDivCd": "INDIVIDUAL", // 고객구분 (INDIVIDUAL, GUEST)
  "loginId": "user@test.com",
  "corpSno": null,         // 법인 일련번호
  "corpCd": null,          // 법인 코드
  "profileId": "PRF001",   // IBS 프로필 ID
  "loyaltyNumber": "LY001", // IBS 마일리지 번호
  "clientId": "base64encrypted", // Base64 암호화된 고객번호
  "iat": 1711094400,
  "exp": 1711096200,
  "iss": "sumair"
}
```

### 2.3 토큰 전달 방식

**요청 시 (FE → BE):**
```
Authorization: Bearer {accessToken}
```

**응답 시 (BE → FE):**
```
Auth-Token: {newAccessToken}
Refresh-Token: {newRefreshToken}
```

### 2.4 로그인 Flow

```
[이메일 로그인]
POST /customer/login
Body: { "loginId": "user@test.com", "password": "..." }
→ 성공: 200 + Auth-Token/Refresh-Token 헤더 + CustomerInfoRS body
→ 실패: ErrorRS { code: "CM403", message: "로그인 실패" }

[토큰 갱신]
POST /customer/login
Body: { "refreshToken": "{refreshToken}" }
→ 성공: 200 + 새 Auth-Token/Refresh-Token 헤더

[비회원 로그인]
JwtGuestLoginFilter를 통한 별도 처리
→ cstmrDivCd: "GUEST" 권한 부여

[소셜 로그인]
GET /oauth2/authorization/{kakao|naver|google}
→ Provider 인증 후 콜백 → JWT 발급 또는 joinCode 반환
```

### 2.5 로그인 실패 처리

- 비밀번호 5회 연속 실패 → 계정 잠금 (LOCK)
- 실패 이력 DB 저장 (IP, 에러 코드 포함)
- 휴면 계정(DORMANT) → 활성화 필요 안내
- 탈퇴 계정(WTHDR) → 로그인 불가

### 2.6 토큰 검증 실패 응답

```json
// 토큰 만료
{ "code": "CM420", "message": "토큰 만료" }

// 토큰 무효
{ "code": "CM421", "message": "토큰 무효" }
```

---

## 3. 응답 포맷

### 3.1 성공 응답

단일 객체:
```json
// ResponseEntity.ok(object)
// HTTP 200
{
  "cstmrNo": "C00001",
  "loginId": "user@test.com",
  "cstmrNm": "홍길동"
}
```

리스트:
```json
// ResponseEntity.ok(list)
// HTTP 200
[
  { "bookNo": "BK001", "depDate": "2026-04-01" },
  { "bookNo": "BK002", "depDate": "2026-04-15" }
]
```

빈 성공:
```
// ResponseEntity.ok().build()
// HTTP 200 (No Body)
```

### 3.2 에러 응답

```json
// HTTP 500 (대부분의 비즈니스 에러)
{
  "code": "CM403",
  "message": "로그인 실패",
  "params": {}         // 선택적 추가 정보
}
```

### 3.3 응답 래퍼 클래스

| 클래스              | 용도                            | 위치                                  |
| ---------------- | ----------------------------- | ----------------------------------- |
| `ErrorRS`        | 에러 응답 (code, message, params) | `common/core/exception/model/`      |
| `RestResltVO`    | 유연한 key-value 결과              | `common/core/rest/model/`           |
| `WebClientRS<T>` | 내부 WebClient 호출 응답            | `common/core/http/webclient/model/` |

> **참고**: 공통 성공 응답 래퍼 없음. 성공 시 DTO를 직접 반환하는 패턴 사용.

### 3.4 JSON 직렬화 설정

```yaml
spring:
  jackson:
    serialization:
      write-dates-as-timestamps: false  # ISO 8601 형식 사용
```

---

## 4. 에러 코드 체계

### 4.1 에러 코드 구조

```
{Prefix}{Number}
CM = Common (공통)
IBE = IBE 모듈 (항공 예약)
```

### 4.2 전체 에러 코드 목록

#### 공통 (CM)

| 코드 | 메시지 | HTTP 상태 | 설명 |
|------|--------|----------|------|
| CM000 | 성공 | 200 | 성공 |
| CM300 | 잘못된 parameter 요청 | 500 | 파라미터 오류 |
| CM301 | Not Null 체크 오류 | 500 | 필수값 누락 |
| CM302 | Not Empty 체크 오류 | 500 | 빈 값 |
| CM303 | 상태 체크 오류 | 500 | 상태 검증 실패 |
| CM400 | 해당하는 정보의 사용자를 찾을 수 없습니다 | 500 | 사용자 미존재 |
| CM401 | 접근 거부 | 500 | 권한 없음 |
| CM402 | 중복 로그인 | 500 | 이중 로그인 |
| CM403 | 로그인 실패 | 500 | 인증 실패 |
| CM420 | 토큰 만료 | 500 | Access Token 만료 |
| CM421 | 토큰 무효 | 500 | JWT 유효하지 않음 |
| CM500 | 내부 시스템 오류 | 500 | 서버 에러 |
| CM510 | 데이터 없음 | 500 | 조회 결과 없음 |
| CM511 | 데이터 중복 | 500 | 중복 데이터 |

#### IBE (항공 예약)

| 코드 | 메시지 | 설명 |
|------|--------|------|
| IBE000 | IBE 에러 | 일반 IBE 오류 |
| IBE100 | 결제 실패 | 결제 처리 실패 |
| IBE201 | 항공편 조회 실패 | 항공편 검색 오류 |
| IBE203 | 예약 생성 실패 | 예약 생성 오류 |
| IBE204 | PNR 세션이 만료되었습니다 | SOAP 세션 타임아웃 |

### 4.3 예외 처리 메커니즘

```java
// 1) 비즈니스 예외 발생
throw new BaseException(ErrorCode.FAILED_LOGIN);
throw new BaseException(ErrorCode.IBE_PAYMENT_FAILED, params, true); // params 포함

// 2) GlobalExceptionHandler 처리
@RestControllerAdvice
public class BaseExceptionHandler {
    @ExceptionHandler(BaseException.class)
    protected ResponseEntity<?> handleBaseException(BaseException e) {
        ErrorRS rs = ErrorRS.builder()
            .code(e.getCode().code())
            .message(message)
            .build();
        if (e.isSendParams()) rs.setParams(e.getParams());
        return ResponseEntity.status(status).body(rs);
    }
}
```

### 4.4 에러 코드 관련 파일

| 파일 | 역할 |
|------|------|
| `common/core/exception/type/ErrorCode.java` | 에러 코드 Enum 정의 |
| `common/core/exception/BaseException.java` | 비즈니스 예외 클래스 |
| `common/core/exception/handler/BaseExceptionHandler.java` | 전역 예외 핸들러 |
| `common/core/exception/model/ErrorRS.java` | 에러 응답 DTO |

---

## 5. 요청 검증 패턴

### 5.1 Controller 레벨 검증

```java
// @Valid 어노테이션으로 Bean Validation 적용
@PostMapping("/get-avail-list")
public ResponseEntity<GetAvailRS> getAvailList(
    @RequestBody @Valid RetrieveAirAvailRQDto rq) throws BaseException {
    // ...
}

// @Validated 사용 예
@PostMapping("/reset/password")
public ResponseEntity resetPassword(
    @RequestBody @Validated CustomerResetPasswordRQ rq) {
    // ...
}
```

### 5.2 DTO 필드 검증

```java
@Data
public class LoginFormRQ {
    @ApiModelProperty(value = "로그인ID")
    private String loginId;

    @ApiModelProperty(value = "패스워드")
    private String password;

    @ApiModelProperty(value = "재발급 토큰")
    private String refreshToken;
}
```

> **참고**: 대부분의 검증은 Service 레이어에서 비즈니스 로직으로 수행. Bean Validation 어노테이션(@NotNull, @Size 등) 사용은 제한적.

---

## 6. 필터/인터셉터 체인

### 6.1 Spring Security 필터 체인

```
요청 도착
  │
  ├─ SimpleCorsFilter (CORS 헤더 설정)
  │
  ├─ CorsFilter (Spring Security CORS)
  │
  ├─ [/customer/login] → JwtLoginFilter
  │   ├─ 이메일/비밀번호 인증
  │   ├─ Refresh Token 갱신
  │   └─ 성공: Auth-Token + Refresh-Token 헤더 설정
  │
  ├─ [비회원 로그인] → JwtGuestLoginFilter
  │
  ├─ [인증 필요 경로] → JwtCheckFilter
  │   ├─ Authorization 헤더에서 Bearer 토큰 추출
  │   ├─ 토큰 유효성 검증
  │   ├─ 성공: SecurityContext에 인증 정보 설정
  │   └─ 실패: ErrorRS 반환 (CM420/CM421)
  │
  └─ Controller 도달
```

### 6.2 인증 제외 경로 (추정)

- `/customer/login` - 로그인
- `/customer/join` - 회원가입
- `/oauth2/**` - 소셜 로그인
- `/common/**` - 공통 코드 조회
- `/main/**` - 메인 페이지 데이터

---

## 7. 주요 엔드포인트

### 7.1 Home 모듈 (:8000)

#### 인증

| 메서드 | 경로 | 설명 | 인증 |
|--------|------|------|------|
| POST | `/customer/login` | 로그인 (이메일/토큰갱신) | X |
| POST | `/customer/join` | 회원가입 | X |
| GET | `/customer/join-code` | 소셜 가입 정보 조회 | X |
| POST | `/customer/auth/social` | 네이티브 앱 소셜 로그인 | X |
| POST | `/customer/common/account/active` | 휴면 해제 | X |

#### 고객 관리

| 메서드 | 경로 | 설명 | 인증 |
|--------|------|------|------|
| POST | `/customer/common/duplicate/id` | 아이디 중복 확인 | X |
| POST | `/customer/common/duplicate/email` | 이메일 중복 확인 | X |
| POST | `/customer/common/duplicate/check` | 고객 정보 중복 확인 | X |
| POST | `/customer/common/reset/password` | 비밀번호 재설정 | X |
| GET | `/customer/common/find/info` | 고객 정보 조회 | O |
| POST | `/customer/update/*` | 고객 프로필 수정 | O |

#### 항공 예약

| 메서드 | 경로 | 설명 | 인증 |
|--------|------|------|------|
| POST | `/airline/booking/avail/get-avail-list` | 항공편 조회 | X |
| POST | `/airline/booking/create/*` | 예약 생성 | O |
| POST | `/airline/booking/retrieve/*` | 예약 조회 | O |
| POST | `/airline/booking/modify/*` | 예약 변경 | O |
| POST | `/airline/booking/cancel/*` | 예약 취소 | O |

#### 체크인

| 메서드 | 경로 | 설명 | 인증 |
|--------|------|------|------|
| POST | `/airline/checkin/checkinguest/*` | 비회원 체크인 | X |
| POST | `/airline/checkin/searchguest/*` | 탑승객 검색 | X |
| POST | `/airline/checkin/showseatmap/*` | 좌석 배치도 | X |
| POST | `/airline/checkin/printboardingpass/*` | 탑승권 출력 | X |

#### 공통 데이터

| 메서드 | 경로 | 설명 | 인증 |
|--------|------|------|------|
| GET | `/common/code` | 공통 코드 | X |
| GET | `/common/holiday` | 공휴일 목록 | X |
| GET | `/common/country` | 국가 목록 | X |
| GET | `/main/notice` | 공지사항 | X |
| GET | `/main/ads/slider` | 슬라이더 광고 | X |
| GET | `/main/ads/popup` | 팝업 | X |

#### 마이페이지

| 메서드 | 경로 | 설명 | 인증 |
|--------|------|------|------|
| POST | `/mypage/reservation/bookingSearch` | 나의 여정 목록 | O |
| POST | `/mypage/reservation/segmntSearch` | 구간 검색 | O |
| POST | `/mypage/coupon/*` | 쿠폰 관리 | O |
| POST | `/mypage/point/*` | 포인트 조회 | O |

### 7.2 IBE 모듈 (:8100)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/avail/retrieveAirAvail` | 항공편 가용성 조회 |
| POST | `/avail/retrieveEnhancedAirAvail` | 향상된 가용성 조회 |
| POST | `/price/*` | 요금 조회/확정 |
| POST | `/booking/*` | 예약 CRUD |
| POST | `/ancillary/*` | 부가서비스 |
| POST | `/baggage/*` | 수하물 |
| POST | `/checkin/*` | 체크인 |
| POST | `/profile/*` | IBS 프로필 |

### 7.3 Pay 모듈 (:8200)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/booking/payment/confirm` | 결제 승인 |
| POST | `/booking/payment/cancel` | 결제 취소/환불 |
| POST | `/booking/payment/search` | 결제 조회 |
| POST | `/booking/payment/getAccessToken` | BrandPay 토큰 |
| POST | `/booking/payment/getSettlements` | 정산 조회 |
| POST | `/booking/payment/remove` | BrandPay 고객 삭제 |

### 7.4 Message 모듈 (:8400)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| POST | `/infobip/send` | 통합 템플릿 메시지 발송 |
| POST | `/sms/members/send` | SMS 인증번호 발송 |
| POST | `/email/members/*` | 회원 이메일 |
| POST | `/email/reservation/*` | 예약 이메일 |
| SOAP | `/ires/message` | IRES SOAP 수신 (IBS→Sumair) |

---

## 부록: 페이지네이션

현재 Spring Data의 `Page`/`Pageable`을 사용하지 않습니다. 필터링 기반 조회 패턴:

```java
// Request
@Data
public class BookingSearchRQ {
    private String stDate;       // 시작일
    private String endDate;      // 종료일
    private String psngrLname;   // 탑승객 성
    private String bookStatusCd; // 예약 상태
}

// Response: 직접 List 반환
@PostMapping("/bookingSearch")
public ResponseEntity getMypageBookingList(@RequestBody BookingSearchRQ rq) {
    List<BookingSearchRS> list = service.getMypageBookingList(rq);
    return ResponseEntity.ok(list);
}
```

MyBatis 매퍼에서 동적 쿼리로 필터링 처리.
