# Sumair Backend Overview

## 프로젝트 개요
항공사 예약 시스템(Airline Reservation System)의 백엔드 프로젝트

| 항목 | 내용 |
|------|------|
| 프레임워크 | Spring Boot 2.7.10 |
| Java 버전 | 1.8 |
| 빌드 도구 | Maven |
| DB | MySQL (MyBatis) |
| 캐시 | Redis (Valkey 호환, AWS ElastiCache) |
| 총 Java 파일 | ~1,262개 |
| 환경 프로파일 | local / dev / prod |

## 모듈 구성

```
sumair-be-common   <- 공통 라이브러리 (모든 모듈이 의존)
    |
    +-- sumair-be-home     (port 8000)  메인 API / 인증 / 고객관리
    +-- sumair-be-ibe      (port 8100)  항공 예약 엔진 (IBES/IBEP SOAP)
    +-- sumair-be-pay      (port 8200)  결제 (Toss Payments)
    +-- sumair-be-batch    (port 8300)  배치 작업 (Quartz)
    +-- sumair-be-message  (port 8400)  SMS/이메일 발송 (Infobip)
```

## 모듈별 요약

| 모듈 | 파일 수 | 역할 | 상세 |
|------|---------|------|------|
| common | 526 | 공통 인프라 | [[01-module-common]] |
| home | 387 | 메인 API/인증 | [[02-module-home]] |
| ibe | 154 | 예약 엔진 | [[03-module-ibe]] |
| pay | 29 | 결제 | [[04-module-pay]] |
| batch | 83 | 배치 작업 | [[05-module-batch]] |
| message | 83 | 메시징 | [[06-module-message]] |

## 핵심 기술 스택

### 인증/보안
- JWT + OAuth2 (Naver, Kakao, Google)
- AES-256 암호화
- Jasypt 프로퍼티 암호화
- 커스텀 데이터 마스킹 (AOP 어노테이션 기반)

### 외부 연동
- IBES/IBEP - Apache CXF SOAP (항공 예약 시스템)
- Toss Payments (결제 게이트웨이)
- Infobip (SMS/이메일)
- data.go.kr (공휴일/날씨 API)
- IBS FTP (secureftp-apn2.ibsplc.aero)

### 내부 통신
- Spring WebClient (서비스간 비동기 통신)
- Redis (세션/캐시)

### 개발 도구
- Swagger (SpringFox 3.0) API 문서
- Docker Compose (로컬 Valkey)
- log4jdbc SQL 디버깅

## 의존성 관계도

```
               sumair-be-common
              /    |    |    \    \
           home   ibe  pay  batch  message
            |      |    |     |      |
            +------+----+----+------+
            |   WebClient 기반 통신   |
            +------------------------+
```

- home -> ibe : 항공편 조회/예약 요청
- home -> pay : 결제 요청
- batch -> ibe : 스케줄 동기화
- batch -> message : 알림 발송 트리거
- message -> ibe : 예약 정보 조회 (SOAP)

## Aurora 마이그레이션

| 문서 | 설명 |
|------|------|
| [[07-mysql-to-aurora-migration]] | MySQL → Aurora 마이그레이션 가이드 |
| [[08-aws-jdbc-wrapper-guide]] | AWS Advanced JDBC Wrapper 적용 |
| [[09-aurora-mysql-function-compatibility]] | Aurora MySQL 함수 호환성 체크리스트 |
| [[10-aurora-read-write-splitting]] | Aurora Read/Write Splitting 적용 |
