# 07. 보안 & 인증/인가

> JWT 토큰 구조, 인증 Flow, 소셜 로그인, 암호화, 접근 권한 제어

---

## 목차

1. [JWT 토큰 구조](#1-jwt-토큰-구조)
2. [인증 Flow](#2-인증-flow)
3. [Spring Security 필터 체인](#3-spring-security-필터-체인)
4. [접근 권한 제어](#4-접근-권한-제어)
5. [소셜 로그인 (OAuth2)](#5-소셜-로그인-oauth2)
6. [암호화 체계](#6-암호화-체계)
7. [보안 주의사항](#7-보안-주의사항)

---

## 1. JWT 토큰 구조

### 1.1 토큰 사양

| 항목 | Access Token | Refresh Token |
|------|-------------|---------------|
| Subject | `auth` | `refresh` |
| 유효기간 | 30분 | 30일 |
| 알고리즘 | HS256 | HS256 |
| Issuer | `sumair` | `sumair` |

**구현:** `home/config/security/jwt/JwtUtils.java`

### 1.2 Claims 구조

```json
{
  "sub": "auth",
  "cstmrNo": "C00001",         // 고객번호
  "cstmrDivCd": "INDIVIDUAL",  // 고객구분 (INDIVIDUAL, CORP, GUEST, VIP)
  "loginId": "user@test.com",
  "corpSno": null,              // 법인 일련번호
  "corpCd": null,               // 법인 코드
  "profileId": "PRF001",        // IBS 프로필 ID
  "loyaltyNumber": "LY001",     // IBS 마일리지 번호
  "clientId": "base64encoded",  // Base64 인코딩된 고객번호
  "iat": 1711094400,
  "exp": 1711096200,
  "iss": "sumair"
}
```

### 1.3 토큰 전달 헤더

```
[요청] Authorization: Bearer {accessToken}
[응답] Auth-Token: {newAccessToken}
       Refresh-Token: {newRefreshToken}
```

### 1.4 Secret Key 관리

- `${security.jwt.secret}` 프로퍼티에서 로드
- Jasypt ENC() 형식으로 암호화 저장
- 초기화 시 Base64 인코딩하여 서명키로 사용

---

## 2. 인증 Flow

### 2.1 이메일/비밀번호 로그인

```
POST /customer/login
Body: { loginId, password }

→ JwtLoginFilter.attemptAuthentication()
  ├─ AuthUserService.loadUserByUsername(loginId)
  │   └─ DB에서 고객 정보 조회 → AuthUser 생성
  ├─ passwordEncoder.matches(password, user.password)
  │   ├─ 성공 → successfulAuthentication()
  │   │   ├─ 계정 상태 확인 (DORMANT, LOCK 체크)
  │   │   ├─ JWT Access Token 생성 (30분)
  │   │   ├─ JWT Refresh Token 생성 (30일)
  │   │   ├─ 접속 이력 DB 저장
  │   │   └─ 응답 헤더: Auth-Token + Refresh-Token
  │   └─ 실패 → unsuccessfulAuthentication()
  │       ├─ 실패 횟수 +1 (IP, 에러코드 기록)
  │       ├─ 5회 초과 → 계정 잠금 (LOCK)
  │       └─ ErrorRS 반환: CM403
```

### 2.2 토큰 갱신

```
POST /customer/login
Body: { refreshToken }

→ JwtLoginFilter
  ├─ Refresh Token 유효성 검증
  ├─ Claims에서 고객번호 추출
  ├─ 새 Access Token + Refresh Token 발급
  └─ 응답 헤더에 새 토큰 설정
```

### 2.3 비회원 로그인

```
JwtGuestLoginFilter
  ├─ /customer/login/guest 엔드포인트
  ├─ cstmrDivCd: "GUEST"
  └─ ROLE_GUEST 권한 부여
```

### 2.4 토큰 검증 (매 요청)

```
JwtCheckFilter (모든 인증 필요 경로)
  ├─ Authorization 헤더에서 "Bearer " 제거
  ├─ JwtUtils.validateToken() 호출
  │   ├─ 서명 검증
  │   ├─ 만료 확인
  │   ├─ 성공 → SecurityContext에 Authentication 설정
  │   └─ 실패 →
  │       ├─ ExpiredJwtException → CM420 (토큰 만료)
  │       └─ 기타 → CM421 (토큰 무효)
  └─ 토큰 없으면 → 다음 필터로 (permitAll 경로면 통과)
```

---

## 3. Spring Security 필터 체인

### 3.1 필터 순서

```
요청
 │
 ├─ SimpleCorsFilter (CORS 헤더)
 │
 ├─ CorsFilter (Spring Security CORS)
 │
 ├─ [/customer/login] → JwtLoginFilter
 │   (UsernamePasswordAuthenticationFilter 대체)
 │
 ├─ [/customer/login/guest] → JwtGuestLoginFilter
 │   (BasicAuthenticationFilter 대체)
 │
 ├─ [인증 필요 경로] → JwtCheckFilter
 │   (BasicAuthenticationFilter 대체)
 │
 └─ Controller
```

### 3.2 Security 설정 요약

```
SecurityConfiguration.java:
  ├─ csrf().disable()
  ├─ sessionManagement() → STATELESS
  ├─ formLogin().disable()
  ├─ httpBasic().disable()
  ├─ exceptionHandling()
  │   ├─ authenticationEntryPoint → BaseAuthenticationEntryPoint (401)
  │   └─ accessDeniedHandler → BaseAccessDeniedHandler (403)
  └─ passwordEncoder → BCryptPasswordEncoder
```

---

## 4. 접근 권한 제어

### 4.1 역할 (Role) 체계

| 역할 | 코드 | 설명 |
|------|------|------|
| ROLE_USER | INDIVIDUAL | 일반 회원 |
| ROLE_CORP | CORP | 법인 회원 |
| ROLE_GUEST | GUEST | 비회원 |
| ROLE_VIP | VIP | VIP 회원 |

역할은 `cstmrDivCd`로부터 자동 생성: `ROLE_{cstmrDivCd}`

### 4.2 URL 기반 접근 제어

```java
// SecurityConfiguration.java (antMatchers)

// USER + CORP + GUEST + VIP 모두 접근 가능
"/common/terms/list/cstmr"
"/airline/booking/adjustfilghtinventory/**"
"/airline/booking/create"
"/mypage/reservation/bookingSearch"
"/mypage/reservation/mainBannerSearch"

// USER + CORP + VIP만 접근 가능 (GUEST 제외)
"/customer/common/find/info"
"/customer/update**"
"/customer/corp/**"
"/mypage/point/**"

// 나머지 전부 → permitAll
"/**"
```

### 4.3 계정 상태 관리

| 상태      | 코드          | 로그인 가능 | 설명                                  |
| ------- | ----------- | :----: | ----------------------------------- |
| 활성      | ACTIVE      |   O    | 정상                                  |
| 휴면      | DORMANT     |   X    | `isAccountNonExpired()` = false     |
| 잠금      | LOCK        |   X    | `isAccountNonLocked()` = false      |
| 비밀번호 만료 | PWD_EXPIRED |   X    | `isCredentialsNonExpired()` = false |
| 탈퇴      | WTHDR       |   X    | `isEnabled()` = false               |

### 4.4 로그인 실패 잠금

- 비밀번호 5회 연속 실패 시 계정 자동 잠금
- 실패 이력: IP, 에러코드, 시간 DB 저장 (`insertCstmrHist()`)
- 잠금 해제: 관리자 수동 처리 또는 비밀번호 재설정

---

## 5. 소셜 로그인 (OAuth2)

### 5.1 지원 Provider

| Provider | 프로토콜 | 주요 클래스 |
|----------|---------|------------|
| Kakao | OAuth2 | BaseOAuth2UserService |
| Naver | OAuth2 | BaseOAuth2UserService |
| Google | OIDC | BaseOidcUserService |

### 5.2 OAuth2 Flow

```
1. GET /oauth2/authorization/{provider}
   │
   ├─ BaseOAuth2AuthorizationRequestResolver
   │   └─ state 파라미터 생성 (Base64 암호화)
   │       { accessToken, redirectUrl, cstmrNo }
   │
2. Provider 인증 → 콜백
   │
3. BaseOAuth2UserService / BaseOidcUserService
   │   └─ Provider에서 사용자 정보 조회
   │
4. BaseOAuth2SuccessHandler
   │
   ├─ [신규] → joinCode 암호화 반환
   │   └─ FE에서 추가 정보 입력 후 POST /customer/join
   │
   ├─ [기존 회원] → JWT 토큰 발급
   │   └─ 리다이렉트 URL에 토큰 포함
   │
   ├─ [중복] → DUPLICATE 상태 + 기존 계정 정보
   │
   └─ [계정 연결 REL] → SNS 연결 처리
       └─ authUserService.snsLoginRel()
```

### 5.3 OAuth2 쿠키 설정

| 항목 | 값 |
|------|-----|
| 쿠키 이름 | `oauth2_auth_request` |
| HttpOnly | true |
| Secure | HTTPS일 때만 |
| Max-Age | 180초 |
| Path | `/` |

### 5.4 SNS 계정 연결

- 테이블: `PTY_SNS_LOGIN_REL`
- 정책: 1 SNS ID = 1 Sumair 계정 (재연결 시 기존 해제)
- 연결: `linkYn='Y'`, `linkDt` 기록
- 해제: `linkYn='N'`, `wthdrDt` 기록

---

## 6. 암호화 체계

### 6.1 Jasypt - 프로퍼티 암호화

| 항목 | 값 |
|------|-----|
| Config 클래스 | `common/core/security/jasypt/JasyptConfig.java` |
| 알고리즘 | PBEWithMD5AndDES |
| 키 | `SUMAIR-JASYPT-KEY` (application-common.yml) |
| 형식 | `ENC(암호화문자열)` |

암호화 대상: DB 비밀번호, JWT Secret, AES 키, Redis 비밀번호

### 6.2 BCrypt - 비밀번호 해싱

```java
// SecurityUtils.java
BCrypt.hashpw(password, BCrypt.gensalt())   // 해싱
BCrypt.checkpw(password, hash)              // 검증
```

- Spring Security의 `BCryptPasswordEncoder` 사용
- 단방향 해시 (복호화 불가)

### 6.3 AES - 데이터 암호화

| 항목         | 값                                               |
| ---------- | ----------------------------------------------- |
| Config 클래스 | `common/core/security/crypto/AESEncryptor.java` |
| 알고리즘       | AES/ECB/PKCS5Padding                            |
| 키 크기       | 128-bit                                         |
| 키 소스       | `${security.crypto.aes.key}`                    |

용도: OAuth2 사용자 정보 암호화 (joinCode 생성 시)

### 6.4 SHA-256 - 정보 해시

```java
// SecurityUtils.java
MessageDigest.getInstance("SHA-256").digest(str.getBytes())
```

용도: 사용자 신원 확인용 해시 (이름, 생년월일, 성별, 전화번호)
- Salt 없는 고정 해시 (중복 검출 목적)

### 6.5 암호화 계층 요약

```
프로퍼티 암호화     → Jasypt (PBEWithMD5AndDES)
비밀번호 저장       → BCrypt (Spring Security)
데이터 암호화/복호화 → AES (ECB/PKCS5Padding)
토큰 서명           → HMAC-SHA256 (JWT)
정보 해시           → SHA-256 (단방향)
```

---

## 7. 보안 주의사항

### 인수인계 시 참고할 보안 현황

| 항목 | 현 상태 | 비고 |
|------|---------|------|
| Jasypt 알고리즘 | PBEWithMD5AndDES | MD5/DES는 취약 알고리즘. 업그레이드 검토 필요 |
| AES 모드 | ECB | ECB는 패턴 노출 위험. CBC/GCM 전환 권장 |
| CORS | 전체 Origin 허용 | 프로덕션에서 특정 도메인으로 제한 권장 |
| Jasypt 키 | yml 파일에 평문 저장 | 환경변수로 분리 권장 |
| Method-Level Security | 미적용 | URL 패턴 기반만 사용. @PreAuthorize 미사용 |
| `/**` permitAll | 활성 | 명시적 경로만 인증 필요, 나머지 전체 허용 |
| JWT subject 검증 | 비활성화 | `validateTokenAndSub()` 호출 주석 처리됨 |
| OAuth2 클라이언트 키 | yml 파일에 평문 | 환경별 다르지만 소스코드에 포함 |

### 핵심 보안 파일 위치

| 파일 | 경로 |
|------|------|
| Security 설정 | `home/config/security/SecurityConfiguration.java` |
| JWT 유틸 | `home/config/security/jwt/JwtUtils.java` |
| 로그인 필터 | `home/config/security/jwt/JwtLoginFilter.java` |
| 토큰 체크 필터 | `home/config/security/jwt/JwtCheckFilter.java` |
| 비회원 필터 | `home/config/security/jwt/JwtGuestLoginFilter.java` |
| Jasypt 설정 | `common/core/security/jasypt/JasyptConfig.java` |
| AES 암호화 | `common/core/security/crypto/AESEncryptor.java` |
| 해시 유틸 | `common/core/utils/SecurityUtils.java` |
| OAuth2 성공 핸들러 | `home/config/security/oauth/BaseOAuth2SuccessHandler.java` |
| CORS 설정 | `home/config/security/CorsConfig.java` |
| 공통 보안 프로퍼티 | `common/src/main/resources/application-common.yml` |
