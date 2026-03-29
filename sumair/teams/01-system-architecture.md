# Sumair BE - 시스템 아키텍처 개요

## 1. 전체 시스템 구성도

```
                              ┌─────────────────────────────────────────────┐
                              │              Client Layer                   │
                              │                                             │
                              │  ┌──────────────┐    ┌──────────────────┐   │
                              │  │ Next.js Mobile│    │ External Partners│   │
                              │  │ (sumair.kr)   │    │ (IBES/IBEP SOAP)│   │
                              │  └──────┬───────┘    └────────┬─────────┘   │
                              └─────────┼─────────────────────┼─────────────┘
                                        │                     │
                              ┌─────────▼─────────────────────▼─────────────┐
                              │           AWS Cloud (ap-northeast-2)         │
                              │                                              │
                              │  ┌──────────────────────────────────────┐    │
                              │  │         Microservices Layer          │    │
                              │  │                                      │    │
                              │  │  ┌──────────┐    ┌──────────┐       │    │
                              │  │  │   HOME   │◄──►│   IBE    │       │    │
                              │  │  │  :8000   │    │  :8100   │       │    │
                              │  │  └────┬─────┘    └────┬─────┘       │    │
                              │  │       │               │             │    │
                              │  │  ┌────▼─────┐    ┌────▼─────┐      │    │
                              │  │  │   PAY    │    │  BATCH   │      │    │
                              │  │  │  :8200   │    │  :8300   │      │    │
                              │  │  └────┬─────┘    └────┬─────┘      │    │
                              │  │       │               │             │    │
                              │  │       │    ┌──────────┤             │    │
                              │  │       │    │ MESSAGE  │             │    │
                              │  │       │    │  :8400   │             │    │
                              │  │       │    └──────────┘             │    │
                              │  │       │                             │    │
                              │  │  ┌────▼────────────────────────┐    │    │
                              │  │  │        COMMON (Library)     │    │    │
                              │  │  │  (공유 모듈 - JAR 의존성)    │    │    │
                              │  │  └─────────────────────────────┘    │    │
                              │  └──────────────────────────────────────┘    │
                              │                                              │
                              │  ┌──────────────────────────────────────┐    │
                              │  │           Data Layer                 │    │
                              │  │                                      │    │
                              │  │  ┌──────────────┐  ┌─────────────┐  │    │
                              │  │  │ Aurora MySQL  │  │   Valkey    │  │    │
                              │  │  │ (RDS Cluster) │  │(ElastiCache)│  │    │
                              │  │  └──────────────┘  └─────────────┘  │    │
                              │  └──────────────────────────────────────┘    │
                              └──────────────────────────────────────────────┘
```

## 2. 마이크로서비스 구성

### 2.1 모듈별 역할

| 모듈 | 포트 | 역할 | 주요 기능 |
|------|------|------|----------|
| **common** | - | 공유 라이브러리 | 보안(JWT/Jasypt/AES), REST/WebClient 추상화, 도메인 객체, MyBatis 설정, 유틸리티 |
| **home** | 8000 | 메인 API 서버 | 회원 관리, OAuth2 인증, 예약 조회, Apple/Samsung Wallet, Redis 캐시 관리 |
| **ibe** | 8100 | 항공 예약 엔진 연동 | IBES/IBEP SOAP 통신 (Apache CXF), 스케줄 검색, 예약 생성/수정/취소 |
| **pay** | 8200 | 결제 처리 | Toss Payments PG 연동, 결제/환불/출금 처리 |
| **batch** | 8300 | 배치/스케줄링 | Quartz 클러스터링, 공휴일/날씨 API 수집, FTP 데이터 연동, 정산 배치 |
| **message** | 8400 | 메시징 서비스 | Infobip SMS/이메일, IRES SOAP 엔드포인트, Slack 알림, 카카오톡 알림 |

### 2.2 Common 모듈 의존 관계

```
  home ──┐
  ibe  ──┤
  pay  ──┼──► common (JAR dependency)
  batch ─┤
  message┘
```

모든 서비스 모듈은 `sumair-common`을 Maven 의존성으로 포함하며, `scanBasePackages = "com.sumair"`로 공통 빈을 스캔한다.

### 2.3 서비스 간 통신

```
              ┌──────────┐
              │   HOME   │──── WebClient ────► IBE (스케줄/예약 조회)
              │  :8000   │──── WebClient ────► PAY (결제 요청)
              │          │──── WebClient ────► MESSAGE (알림 발송)
              └──────────┘

              ┌──────────┐
              │   IBE    │──── CXF SOAP ─────► IBES/IBEP (외부 항공시스템)
              │  :8100   │──── WebClient ────► HOME (내부 API 호출)
              └──────────┘

              ┌──────────┐
              │   PAY    │──── REST ──────────► Toss Payments (PG)
              │  :8200   │──── WebClient ────► MESSAGE (결제 알림)
              └──────────┘

              ┌──────────┐
              │  BATCH   │──── WebClient ────► 각 서비스 (배치 트리거)
              │  :8300   │──── REST ──────────► data.go.kr (공공API)
              │          │──── SFTP ──────────► ibsplc.aero (FTP 연동)
              └──────────┘

              ┌──────────┐
              │ MESSAGE  │──── REST ──────────► Infobip (SMS/이메일)
              │  :8400   │──── CXF SOAP ─────► IRES (SOAP 엔드포인트)
              │          │──── Webhook ───────► Slack (알림)
              └──────────┘
```

**통신 패턴 요약:**
- **서비스 간 내부 통신**: Spring WebClient (비동기/논블로킹) — `InnerWebClientService` 추상화
- **외부 SOAP 통신**: Apache CXF 3.5.5 (ibe, message 모듈)
- **외부 REST 통신**: RestTemplate / WebClient (pay, batch 모듈)
- **내부 호스트 주소** (dev): `http://<service>.dev.internal:<port>`, (prod): `http://<service>.prod.internal:<port>`

## 3. 데이터 흐름도

### 3.1 항공 예약 플로우

```
[사용자] ──► [Next.js Frontend]
                 │
                 ▼
            [HOME :8000]  ── JWT 인증 확인
                 │
                 ▼
            [IBE :8100]   ── 스케줄 검색 / 예약 생성
                 │
                 ▼ (CXF SOAP)
            [IBES/IBEP]   ── 항공 인벤토리 시스템
                 │
                 ▼ (예약 완료)
            [HOME :8000]  ── 예약 정보 저장 (Aurora MySQL)
                 │
          ┌──────┴──────┐
          ▼             ▼
     [PAY :8200]   [MESSAGE :8400]
    Toss 결제 처리    SMS/이메일 발송
```

### 3.2 결제 플로우

```
[Frontend] ──► [HOME :8000] ──► [PAY :8200] ──► [Toss Payments API]
                                     │
                                     ▼ (결제 완료 콜백)
                                [PAY :8200] ──► [MESSAGE :8400] (결제 확인 알림)
                                     │
                                     ▼
                                [Aurora MySQL] (결제 이력 저장)
```

### 3.3 배치 처리 플로우

```
[Quartz Scheduler (클러스터)]
          │
          ├──► 공휴일 수집 ──► data.go.kr API ──► Aurora MySQL
          ├──► 날씨 수집   ──► data.go.kr API ──► Aurora MySQL
          ├──► 정산 리포트 ──► SFTP (ibsplc.aero) ──► Aurora MySQL
          └──► 알림 배치   ──► [MESSAGE :8400] ──► Infobip
```

## 4. 기술 스택 요약

### 4.1 핵심 기술 스택

| 구분             | 기술                        | 버전                     |
| -------------- | ------------------------- | ---------------------- |
| **Language**   | Java                      | 1.8 (OpenJDK 8)        |
| **Framework**  | Spring Boot               | 2.7.10                 |
| **Build**      | Maven                     | 3.8.7                  |
| **ORM**        | MyBatis (+ PageHelper)    | 3.5.0 / 2.3.0          |
| **Database**   | AWS Aurora MySQL          | MySQL 8.0 호환           |
| **DB Driver**  | AWS Advanced JDBC Wrapper | 3.2.0                  |
| **Cache**      | Valkey (Redis 호환)         | ElastiCache Serverless |
| **SOAP**       | Apache CXF                | 3.5.5                  |
| **Security**   | Spring Security + JWT     | jjwt 0.9.1             |
| **OAuth2**     | Spring Security OAuth2    | 2.5.2                  |
| **Scheduling** | Spring Batch + Quartz     | (Spring Boot 내장)       |
| **Container**  | Docker                    | -                      |
| **Registry**   | AWS ECR                   | -                      |
| **API Docs**   | Swagger (Springfox)       | 3.0.0                  |

### 4.2 주요 라이브러리

| 라이브러리 | 용도 |
|-----------|------|
| Jasypt 3.0.5 | application.yml 프로퍼티 암호화 |
| jBCrypt 0.4 | 비밀번호 해싱 |
| BouncyCastle 1.70 | Apple Wallet PKCS7/CMS 서명 |
| Nimbus JOSE+JWT 9.37.3 | Samsung Wallet JWE/JWS |
| Google Zxing 3.1.0 | QR코드 생성 |
| Apache POI 3.15 | Excel 파일 처리 |
| Slack API 1.30.0 | Slack 알림 |
| ModelMapper 3.1.1 | DTO 변환 |
| XStream 1.4.9 | XML 직렬화 |

### 4.3 외부 연동 시스템

| 시스템                | 프로토콜      | 용도           | 사용 모듈   |
| ------------------ | --------- | ------------ | ------- |
| IBES/IBEP (iFly)   | SOAP/WSDL | 항공 예약/스케줄    | ibe     |
| IRES               | SOAP      | 예약 알림 수신     | message |
| Toss Payments      | REST      | 결제/환불/출금     | pay     |
| Infobip            | REST      | SMS/이메일/카카오톡 | message |
| data.go.kr         | REST      | 공휴일/날씨 데이터   | batch   |
| SFTP (ibsplc.aero) | SFTP      | 정산 리포트       | batch   |
| Google/Kakao/Naver | OAuth2    | 소셜 로그인       | home    |
| Apple Wallet       | PKPass    | 모바일 탑승권      | home    |
| Samsung Wallet     | JWE/JWS   | 모바일 탑승권      | home    |
| Slack              | Webhook   | 운영 알림        | message |

## 5. 인증/보안 아키텍처

### 5.1 인증 흐름

```
[사용자 로그인]
      │
      ├──► 자체 로그인 ──► JwtLoginFilter ──► JWT 발급
      │
      └──► 소셜 로그인 ──► OAuth2 (Google/Kakao/Naver)
                              │
                              ▼
                         OAuth2SuccessHandler ──► JWT 발급

[인증된 요청]
      │
      ▼
JwtCheckFilter ──► JWT 검증 ──► SecurityContext 설정
      │
      ▼ (게스트 예약)
JwtGuestLoginFilter ──► 비회원 임시 JWT 발급
```

### 5.2 보안 계층

| 계층 | 기술 | 설명 |
|------|------|------|
| 전송 | HTTPS/TLS | 외부 통신 암호화 |
| 인증 | JWT (jjwt) | Stateless 토큰 기반 인증 |
| 인가 | Spring Security | URL 패턴 기반 접근 제어 |
| 암호화 | AES + Jasypt | 데이터 암호화 / 설정 암호화 |
| 해싱 | jBCrypt | 비밀번호 단방향 해싱 |
| DB 접근 | AWS IAM + JDBC Wrapper | Aurora 인스턴스 접근 제어 |
| 캐시 접근 | TLS | Valkey ElastiCache SSL 통신 |

## 6. 데이터 아키텍처

### 6.1 데이터베이스 구성

```
┌─────────────────────────────────────────────┐
│          Aurora MySQL Cluster                │
│                                             │
│  ┌────────────────┐  ┌────────────────┐     │
│  │  Writer Node   │  │  Reader Node   │     │
│  │  (Read/Write)  │  │  (Read Only)   │     │
│  └────────────────┘  └────────────────┘     │
│                                             │
│  AWS JDBC Wrapper Plugins:                  │
│  - readWriteSplitting (읽기/쓰기 분리)       │
│  - failover (자동 장애 조치)                 │
│  - efm2 (Enhanced Failure Monitoring)       │
│  - logQuery (쿼리 로깅)                     │
└─────────────────────────────────────────────┘
         │                    │
    ┌────┴─────┐        ┌────┴──────┐
    │ sumair   │        │ sumair    │
    │ (메인DB) │        │ (메시지DB)│
    └──────────┘        └───────────┘
```

- **메인 DB**: home, ibe, pay, batch 모듈이 공유
- **메시지 DB**: message 모듈 전용 (별도 클러스터 엔드포인트)
- **ORM**: MyBatis (XML 매퍼 기반) — `mapUnderscoreToCamelCase` 활성화
- **커넥션 풀**: HikariCP (max-pool-size: 5, min-idle: 3)

### 6.2 캐시 구성

```
┌─────────────────────────────────────┐
│    Valkey (ElastiCache Serverless)  │
│                                     │
│  - TTL: 30분                        │
│  - Key: StringRedisSerializer       │
│  - Value: GenericJackson2Json       │
│  - SSL: enabled                     │
└─────────────────────────────────────┘
          │
          ▼
    [HOME :8000] (유일한 Redis 클라이언트)
    - 세션/인증 캐시
    - 조회 결과 캐시
    - @Cacheable 기반
```

## 7. 모듈 패키지 구조

### 7.1 Common 모듈 (공유 라이브러리)

```
com.sumair.common
├── core
│   ├── security        # JWT, Jasypt, AES, Masking
│   ├── seq             # 시퀀스 생성기
│   ├── properties      # IbsConfig, SoapFileLoggerConfig, CommonConfig
│   ├── rest            # REST 클라이언트 추상화 (AbstractRestServiceImpl)
│   ├── http.webclient  # WebClient 서비스 (Inner/Ibes/Pay/Message 등)
│   └── message         # MessageSource 설정
└── domain              # 공유 도메인 객체
```

### 7.2 서비스 모듈 (공통 패턴)

```
com.sumair.<module>
├── config              # Spring 설정 (Security, MyBatis, Redis, CORS 등)
├── controller          # REST 컨트롤러
├── service             # 비즈니스 로직
├── mapper              # MyBatis 매퍼 인터페이스
├── model / domain      # VO/DTO/Entity
└── core                # 모듈 특화 유틸리티
```

---

> **문서 버전**: 2026-03-22 | **작성**: 시스템 아키텍트
