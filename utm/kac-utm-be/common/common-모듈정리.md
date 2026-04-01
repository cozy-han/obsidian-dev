# Common 모듈 정리

#kac-utm #backend #common

> 9개의 공통 모듈을 한 곳에서 정리한다. 모든 app 모듈이 공유하는 기반 라이브러리이다.

---

## 모듈 목록

| 모듈 | Gradle | 클래스 수 | 핵심 기능 |
|------|--------|----------|----------|
| [app](#common-app) | `:common:common-app` | 1 | Firebase 초기화 |
| [config](#common-config) | `:common:common-config` | 0 | 환경별 설정 프로퍼티 |
| [connector](#common-connector) | `:common:common-connector` | 252 | 외부 API 연동 (Feign) |
| [exception](#common-exception) | `:common:common-exception` | 14 | 전역 예외 처리, XSS 방어 |
| [models](#common-models) | `:common:common-models` | 7 | 페이징, 소켓 페이로드 |
| [redis](#common-redis) | `:common:common-redis` | 4 | Redis 설정 및 컴포넌트 |
| [security](#common-security) | `:common:common-security` | 5 | JWT 토큰, 인증/인가 |
| [sys](#common-sys) | `:common:common-sys` | 5 | 시스템 알림, 설정 변수 |
| [utils](#common-utils) | `:common:common-utils` | 10 | 암호화, 좌표 변환, PDF 등 |

---

## common-app

**Firebase 초기화 모듈**

```
common/app/
└── FirebaseInitializer.java
    - @PostConstruct로 Firebase Admin SDK 초기화
    - serviceAccountKey.json 기반 GoogleCredentials 로드
    - Firebase 앱 싱글톤 관리
```

**의존성**: firebase-admin 9.2.0

> **용도**: 푸시 알림(FCM) 발송을 위한 Firebase 초기화

---

## common-config

**환경별 설정 프로퍼티 모듈** (Java 클래스 없음, yml만 존재)

```
common/config/src/main/resources/
├── application-db.yml        ← MySQL 환경별 설정
├── application-redis.yml     ← Redis 환경별 설정
├── application-rabbitmq.yml  ← RabbitMQ 환경별 설정
├── application-security.yml  ← JWT 토큰 설정
└── application-gsl.yml       ← Hibernate Spatial 설정
```

> 상세 내용은 [[03_인프라 및 환경설정]] 참조

---

## common-connector

**외부 API 연동 모듈** (252개 클래스 - 가장 큼)

Spring Cloud OpenFeign (3.1.8) 기반으로 외부 시스템과 HTTP 통신한다.

### Feign 클라이언트

| 클라이언트 | 용도 | 주요 API |
|-----------|------|---------|
| `CtrlWthrClient` | 제어기상 정보 | `/getVilageFcst` (읍면단위 예보) |
| `WthrClient` | 항공기상 정보 | `/getMetar`, `/getTaf`, `/getSigmet`, `/getAirmet` |
| `TsClient` | 항공교통서비스 (KOTSA) | eDrone API |
| `SnrSnsClient` | 슈어SNS 센서 | 센서 데이터 조회 |
| `DirectSendClient` | 이메일 발송 | DirectSend API |

### 패키지 구조
```
connector/external/
├── client/       ← Feign 클라이언트 인터페이스
└── model/        ← 요청/응답 모델
    ├── ctrlWthr/ ← 제어기상
    ├── email/    ← 이메일
    ├── snrsns/   ← SNS
    ├── ts/       ← 항공교통서비스
    └── wthr/     ← 기상
```

---

## common-exception

**전역 예외 처리 모듈**

### 핵심 클래스

| 클래스 | 역할 |
|--------|------|
| `BaseException` | 기본 예외 클래스 (커스텀 예외의 부모) |
| `BaseAuthenticationException` | 인증 예외 |
| `BaseExceptionHandler` | `@ControllerAdvice` 전역 예외 핸들러 |
| `ErrorCode` | 에러 코드 정의 (Enum) |
| `ErrorResponse` | 표준 에러 응답 형식 |

### 부가 기능

| 클래스 | 역할 |
|--------|------|
| `ValidPattern` / `ValidPatternValidator` | 커스텀 검증 애너테이션 |
| `XssConfig` / `HeaderXssFilter` | XSS 필터링 |
| `LocaleConfig` / `MessageSourceConfig` | 다국어 에러 메시지 |

---

## common-models

**공통 데이터 모델 모듈**

### 페이징

| 클래스 | 용도 |
|--------|------|
| `BasePageRq` | 페이지 기반 요청 (pageNo, pageSize) |
| `BasePageRs` | 페이지 기반 응답 (totalCount, data) |
| `BaseScrollRq` | 스크롤 기반 요청 |
| `BaseScrollRs` | 스크롤 기반 응답 |

### 기타

| 클래스 | 용도 |
|--------|------|
| `GpModel` | 지점(좌표) 모델 |
| `GPHistoryModel` | 지점 이력 모델 |
| `SocketPayload` | WebSocket 통신 페이로드 |

---

## common-redis

**Redis 설정 및 컴포넌트 모듈**

### RedisConfig
- LettuceConnectionFactory 설정
- Standalone / Sentinel 자동 선택
- StringRedisTemplate, RedisTemplate 빈 등록

### RedisComponent (핵심)
```java
// 주요 메서드
getData(key)          // 단일 키 조회
getData(keys)         // 다중 키 조회
setData(key, value)   // 저장
setData(key, value, ttl) // TTL 지정 저장
removeData(key)       // 삭제
isConnect()           // 연결 상태 확인
```

### 모델
| 클래스 | 용도 |
|--------|------|
| `RedisPath` | Redis 키 경로 생성 헬퍼 |
| `RedisValue` | 직렬화 가능 인터페이스 |

---

## common-security

**보안 모듈 (JWT + BCrypt)**

### TokenProvider (핵심)
```java
// 토큰 생성
generateAccessToken(payload)
generateRefreshToken(userNo)

// 토큰 검증
validateAccessToken(token)
validateRefreshToken(token)

// 토큰에서 정보 추출
getPayload(token)
getUserNoByRefreshToken(token)
```

**토큰 스펙:**
- 알고리즘: PBES2_HS512_A256KW (패스워드 기반 암호화)
- 인코딩: A256GCM
- Access Token: 2시간
- Refresh Token: 24시간

### 기타

| 클래스 | 역할 |
|--------|------|
| `SecurityComponent` | BCrypt PasswordEncoder 빈 |
| `SecurityHelper` | 보안 헬퍼 메서드 |
| `BaseUserDto` | 사용자 기본 DTO |
| `BaseUserDetailsDto` | Spring Security UserDetails 구현 |

---

## common-sys

**시스템 알림/설정 모듈**

| 클래스 | 역할 |
|--------|------|
| `SysAlrtService` | 시스템 알림 발송 서비스 |
| `SaveAlrtModel` | 알림 저장 모델 |
| `SysAlrtTypeCd` | 시스템 알림 유형 코드 (Enum) |
| `SystemStngType` | 시스템 설정 유형 |
| `SystemVariable` | 시스템 변수 관리 |

---

## common-utils

**범용 유틸리티 모듈**

| 클래스 | 기능 | 언제 사용? |
|--------|------|-----------|
| `AESEncryptorUtils` | AES 암호화/복호화 | 민감 데이터 암호화 |
| `AreaUtils` | 면적 계산 | 공역 면적 계산 |
| `CoordinateUtils` | 좌표 변환 (WGS84, 웹메르카토르 등) | 좌표계 변환 |
| `DmsUtils` | 도분초(DMS) ↔ 십진도 변환 | 좌표 표시 형식 변환 |
| `JsonUtil` | JSON 직렬화/역직렬화 | JSON 처리 |
| `MaskingUtil` | 개인정보 마스킹 | 이름, 전화번호 마스킹 |
| `PdfUtils` | PDF 생성 (iTextPDF) | 보고서 PDF 출력 |
| `RandomUtil` | 난수 생성 | 임시 비밀번호, 인증코드 |
| `RequestUtil` | HTTP 요청 유틸 | 클라이언트 IP 추출 등 |
| `CryptoUtil` | 암호화 관련 | 기타 암호화 처리 |

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[02_프로젝트 구조]] - 모듈 구조
- [[03_인프라 및 환경설정]] - 환경별 설정 상세
