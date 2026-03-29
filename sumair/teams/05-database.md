# 05. DB 스키마 & 데이터 구조

> MySQL Aurora 데이터베이스 구성, 테이블 관계도, MyBatis 매퍼 구조, 커넥션 풀, 암호화 설정

---

## 목차

1. [데이터소스 구성](#1-데이터소스-구성)
2. [HikariCP 커넥션 풀](#2-hikaricp-커넥션-풀)
3. [AWS Advanced JDBC Wrapper](#3-aws-advanced-jdbc-wrapper)
4. [MyBatis 설정 & 매퍼 구조](#4-mybatis-설정--매퍼-구조)
5. [테이블 목록 & 도메인 분류](#5-테이블-목록--도메인-분류)
6. [테이블 관계도](#6-테이블-관계도)
7. [Jasypt 프로퍼티 암호화](#7-jasypt-프로퍼티-암호화)
8. [시퀀스 관리](#8-시퀀스-관리)

---

## 1. 데이터소스 구성

**설정 파일:** `sumair-be-common/src/main/resources/application-databases.yml`

### 데이터소스 개요

| 데이터소스 | 풀 이름 | 용도 | 사용 모듈 |
|-----------|---------|------|----------|
| **main** | main-pool | 핵심 비즈니스 DB | home, ibe, pay, batch |
| **message** | message-pool | 메시지 전용 DB | message |
| **infobip** | (message 재사용) | Infobip 템플릿 | message |
| **quartz** | (main 재사용) | Quartz 스케줄러 | batch |

### 환경별 DB 호스트

| 환경 | Main DB | Message DB |
|------|---------|-----------|
| **Local/Dev** | `sumair-dev-aurora.cluster-chh3noeqbmvd.ap-northeast-2.rds.amazonaws.com` | `sumair-dev-admin.cluster-chh3noeqbmvd.ap-northeast-2.rds.amazonaws.com` |
| **Prod** | `sumair-prod-aurora.cluster-chh3noeqbmvd.ap-northeast-2.rds.amazonaws.com` | `sumair-prod-admin.cluster-chh3noeqbmvd.ap-northeast-2.rds.amazonaws.com` |

- 포트: 3306
- 스키마: `sumair`
- 드라이버: `software.amazon.jdbc.Driver` (AWS Advanced JDBC Wrapper)

---

## 2. HikariCP 커넥션 풀

### Main Pool 설정

| 설정 | Local | Dev/Prod |
|------|-------|----------|
| maximum-pool-size | 3 | 5 |
| minimum-idle | 1 | 3 |
| max-lifetime | 600,000ms (10분) | 600,000ms |
| idle-timeout | 60,000ms (1분) | 60,000ms |
| connection-timeout | 30,000ms (30초) | 30,000ms |
| validation-timeout | 5,000ms (5초) | 5,000ms |

### Message Pool 설정

| 설정 | 값 |
|------|-----|
| maximum-pool-size | 3 |
| idle-timeout | 60,000ms |
| max-lifetime | 300,000ms (5분) |
| connection-timeout | 30,000ms |
| validation-timeout | 5,000ms |

> **참고**: Main Pool이 max-lifetime 10분, Message Pool이 5분으로 차이 있음. Aurora 권장 설정에 맞춰 조정 가능.

---

## 3. AWS Advanced JDBC Wrapper

```
JDBC URL 형식: jdbc:aws-wrapper:mysql://{host}:3306/sumair
```

### 활성화된 플러그인

| 플러그인 | 기능 |
|----------|------|
| `readWriteSplitting` | Reader/Writer 인스턴스 자동 라우팅 |
| `failover` | Aurora 장애 시 자동 페일오버 |
| `efm2` | Enhanced Failure Monitoring v2 |
| `logQuery` | SQL 쿼리 로깅 |

- **Wrapper Dialect**: `aurora-mysql`
- **Connection Pool 타입**: `hikari`
- **적용 모듈**: home, ibe, batch, message (pay, common 제외)

### AuroraInstanceLogInterceptor

- MyBatis 인터셉터로 Aurora 인스턴스 정보 로깅
- 모든 모듈의 MybatisConfig에서 등록

---

## 4. MyBatis 설정 & 매퍼 구조

### 공통 MyBatis 설정

**파일:** `sumair-be-*/src/main/resources/config/mybatis/mybatis-config.xml`

```xml
<setting name="mapUnderscoreToCamelCase" value="true"/>  <!-- DB 컬럼 → Java 필드 자동 매핑 -->
<setting name="callSettersOnNulls" value="true"/>         <!-- NULL도 setter 호출 -->
<setting name="jdbcTypeForNull" value="NULL"/>            <!-- NULL 파라미터 처리 -->
```

### 모듈별 DataSource Config 클래스

| 모듈 | Config 클래스 | 매퍼 경로 패턴 |
|------|--------------|--------------|
| **home** | `MybatisConfig.java` | `classpath:com/sumair/home/**/**/mapper/*Mapper.xml` |
| **ibe** | `MybatisConfig.java` | `classpath:com/sumair/ibe/**/**/mapper/*Mapper.xml` |
| **pay** | `MybatisConfig.java` | 동적 경로 |
| **batch** | `QuartsDataSource.java` | `classpath:com/sumair/batch/**/mapper/*Mapper.xml` |
| **message** | `MybatisConfig.java` (main) + `InfobipDbConfig.java` (infobip) | 모듈별 분리 |

### 모듈별 매퍼 파일 수

```
sumair-be-common   : 99개 (Base 매퍼 - 모든 모듈에서 참조)
sumair-be-home     : 33개
sumair-be-ibe      :  6개
sumair-be-pay      :  3개
sumair-be-batch    :  6개
sumair-be-message  :  5개
────────────────────────
총 합계             : 152개+
```

### Message 모듈 듀얼 데이터소스

```
MybatisConfig (mainDataSource)
  └─ IRES, SMS, Email 매퍼

InfobipDbConfig (infobipDataSource = message DB)
  └─ InfobipTemplateMapper만 사용
  └─ 테이블: infobip_templates, kakao_alimtalk_templates_extended
```

---

## 5. 테이블 목록 & 도메인 분류

### PTY_ (Party/고객) - 15개

| 테이블 | 설명 |
|--------|------|
| PTY_CSTMR_BAS | 고객 기본정보 |
| PTY_CSTMR_DTL | 고객 상세정보 |
| PTY_CSTMR_CONECT_HIST | 고객 접속 이력 |
| PTY_CSTMR_INFO_HIST | 고객 정보 변경 이력 |
| PTY_CSTMR_STATUS_CNG_HIST | 고객 상태 변경 이력 |
| PTY_CORP_BAS | 법인 기본정보 |
| PTY_CORP_TERMS_AGREE_TXN | 법인 약관 동의 |
| PTY_CRTFC_TXN | 인증 트랜잭션 (SMS) |
| PTY_EMAIL_CRTFC_TXN | 이메일 인증 트랜잭션 |
| PTY_SNS_LOGIN_REL | SNS 로그인 연결 |
| PTY_SEARCH_HIST | 검색 이력 |
| PTY_TERMS_BAS | 약관 기본 |
| PTY_TERMS_DTL | 약관 상세 |
| PTY_TERMS_AGREE_TXN | 약관 동의 트랜잭션 |
| PTY_TOKEN_BAS | 토큰 기본 |

### ORD_ (Order/주문) - 16개

| 테이블 | 설명 |
|--------|------|
| ORD_BAS | 주문 기본 |
| ORD_BOOK_BAS | 예약 기본 |
| ORD_BOOK_ERROR_TXN | 예약 에러 |
| ORD_BOOK_PRSNT_TXN | 예약 프레젠테이션 |
| ORD_BOOK_TRSF_TXN | 예약 이관 |
| ORD_BOOKER_BAS | 예약자 기본 |
| ORD_PSNGR_BAS | 탑승객 기본 |
| ORD_PSNGR_CHRG_TXN | 탑승객 요금 |
| ORD_PSNGR_TAX_TXN | 탑승객 세금 |
| ORD_SEGMNT_BAS | 구간 기본 |
| ORD_SSR_CHRG_BAS | SSR 요금 기본 |
| ORD_SSR_CHRG_TXN | SSR 요금 트랜잭션 |
| ORD_PYM_TXN | 주문 결제 트랜잭션 |
| ORD_PYM_DTL_TXN | 주문 결제 상세 |
| ORD_PYM_CANCL_TXN | 주문 결제 취소 |
| ORD_PYM_CANCL_PROC_TXN | 주문 결제 취소 처리 |
| ORD_TICKET_TXN | 티켓 트랜잭션 |
| ORD_COUPON_TXN | 쿠폰 트랜잭션 |
| ORD_CHKIN_TXN | 체크인 트랜잭션 |
| ORD_CHKIN_TIME | 체크인 시간 |
| ORD_RGLT_AGREE_TXN | 약관 동의 트랜잭션 |

### PRD_ (Product/상품) - 20개+

| 테이블 | 설명 |
|--------|------|
| PRD_SCHDUL_BAS | 운항 스케줄 기본 |
| PRD_SCHDUL_DTL | 운항 스케줄 상세 |
| PRD_SCHDUL_CNG_BAS | 스케줄 변경 기본 |
| PRD_SCHDUL_CNG_TXN | 스케줄 변경 트랜잭션 |
| PRD_SCHDUL_BLCK_TXN | 스케줄 블록 |
| PRD_ROUTE_BAS | 노선 기본 |
| PRD_ARPRT_CD_BAS | 공항 코드 기본 |
| PRD_ARPRT_CD_DTL | 공항 코드 상세 |
| PRD_ARPRT_TRMINL_BAS | 공항 터미널 |
| PRD_ARPRT_ATENT_MTTR_BAS | 공항 주의사항 |
| PRD_FARE_GROUP_BAS | 요금 그룹 기본 |
| PRD_FARE_GROUP_DTL | 요금 그룹 상세 |
| PRD_FARE_DSCNT_REL | 요금 할인 관계 |
| PRD_FARE_RGLT_REL | 요금 약관 관계 |
| PRD_FARE_CLASS_CD | 요금 클래스 코드 |
| PRD_RGLT_BAS | 약관 기본 |
| PRD_RGLT_DTL | 약관 상세 |
| PRD_HOLIDY_BAS | 공휴일 |
| PRD_LWPRC_BAS | 최저가 |
| PRD_REVENUE_REPORT_TXN | 매출 보고서 |

### PYM_ (Payment/결제) - 9개

| 테이블 | 설명 |
|--------|------|
| PYM_PYM_BAS | 결제 기본 |
| PYM_PYM_LOG_HIST | 결제 로그 이력 |
| PYM_POINT_BAS | 포인트 기본 |
| PYM_POINT_CD | 포인트 코드 |
| PYM_POINT_LOG_HIST | 포인트 이력 |
| PYM_CASH_RCPTS_TXN | 현금영수증 |
| PYM_REFND_CANCL_TXN | 환불/취소 |
| PYM_COUPON_BAS | 쿠폰 기본 |
| PYM_COUPON_USE_HIST | 쿠폰 사용 이력 |

### CNS_ (Contents/콘텐츠) - 6개

| 테이블 | 설명 |
|--------|------|
| CNS_NOTICE_BAS | 공지사항 |
| CNS_ACTIVITY_BAS | 액티비티 |
| CNS_ACTIVITY_HASHTAG_REL | 액티비티-해시태그 관계 |
| CNS_HASHTAG_BAS | 해시태그 |
| CNS_ADS_BAS | 광고 |
| CNS_INTRO_BAS | 인트로 |

### COM_ (Common/공통) - 14개

| 테이블 | 설명 |
|--------|------|
| COM_CD_BAS | 공통 코드 |
| COM_CD_GROUP_BAS | 공통 코드 그룹 |
| COM_CD_LANG_CTG | 코드 다국어 |
| COM_CNTRY_CD_BAS | 국가 코드 |
| COM_CNTRY_CD_DTL | 국가 코드 상세 |
| COM_CNTRY_CD_LANG_REL | 국가 코드 다국어 관계 |
| COM_LANG_DIV_CD | 언어 구분 코드 |
| COM_MENU_BAS | 메뉴 |
| COM_MENU_LANG_CTG | 메뉴 다국어 |
| COM_SEQ | 시퀀스 관리 |
| COM_TRNSMIS_TXN | 전송 트랜잭션 |
| COM_TRNSMIS_RESULT_TXN | 전송 결과 |
| COM_WEATHER_BAS | 날씨 정보 |
| COM_WEATHER_COORD_BAS | 날씨 좌표 |

### ADM_ (Admin) / Infobip / QRTZ_

| 테이블 | 설명 |
|--------|------|
| ADM_BATCH_BAS | 배치 작업 관리 |
| infobip_templates | Infobip 메시지 템플릿 |
| kakao_alimtalk_templates_extended | 카카오 알림톡 확장 템플릿 |
| QRTZ_* (11개) | Quartz 스케줄러 테이블 |
| POINT_VIEW | 포인트 뷰 |

---

## 6. 테이블 관계도

### 고객-주문-결제 핵심 관계

```
PTY_CSTMR_BAS (고객)
  │
  ├── PTY_CSTMR_DTL (고객 상세) ─── 1:1
  ├── PTY_SNS_LOGIN_REL (SNS 연결) ─── 1:N
  ├── PTY_TOKEN_BAS (토큰) ─── 1:N
  ├── PTY_TERMS_AGREE_TXN (약관 동의) ─── 1:N
  │
  └── ORD_BAS (주문)
        │
        ├── ORD_BOOKER_BAS (예약자) ─── 1:1
        ├── ORD_BOOK_BAS (예약) ─── 1:N
        │     └── ORD_SEGMNT_BAS (구간) ─── 1:N
        │
        ├── ORD_PSNGR_BAS (탑승객) ─── 1:N
        │     ├── ORD_PSNGR_CHRG_TXN (요금)
        │     └── ORD_PSNGR_TAX_TXN (세금)
        │
        ├── ORD_PYM_TXN (결제) ─── 1:N
        │     └── ORD_PYM_DTL_TXN (결제 상세)
        │
        ├── ORD_SSR_CHRG_BAS (SSR 요금) ─── 1:N
        ├── ORD_TICKET_TXN (티켓) ─── 1:N
        └── ORD_CHKIN_TXN (체크인) ─── 1:N
```

### 결제 도메인

```
PYM_PYM_BAS (결제 마스터)
  │
  ├── PYM_PYM_LOG_HIST (결제 이력) ─── 1:N
  ├── PYM_REFND_CANCL_TXN (환불/취소) ─── 1:N
  └── PYM_CASH_RCPTS_TXN (현금영수증) ─── 1:1

PYM_POINT_BAS (포인트 마스터)
  └── PYM_POINT_LOG_HIST (포인트 이력) ─── 1:N

PYM_COUPON_BAS (쿠폰 마스터)
  └── PYM_COUPON_USE_HIST (쿠폰 사용) ─── 1:N
```

### 상품/스케줄 도메인

```
PRD_ROUTE_BAS (노선)
  └── PRD_SCHDUL_BAS (스케줄)
        ├── PRD_SCHDUL_DTL (스케줄 상세) ─── 1:N
        └── PRD_SCHDUL_CNG_BAS (스케줄 변경) ─── 1:N

PRD_ARPRT_CD_BAS (공항)
  ├── PRD_ARPRT_CD_DTL (공항 상세)
  └── PRD_ARPRT_TRMINL_BAS (터미널)

PRD_FARE_GROUP_BAS (요금 그룹)
  └── PRD_FARE_GROUP_DTL (요금 상세)
```

---

## 7. Jasypt 프로퍼티 암호화

### 설정

**Config 클래스:** `common/core/security/jasypt/JasyptConfig.java`

| 항목 | 값 |
|------|-----|
| 알고리즘 | PBEWithMD5AndDES |
| Bean 이름 | jasyptStringEncryptor |
| Pool Size | 1 |
| 암호화 키 | `${security.jasypt.key}` → `SUMAIR-JASYPT-KEY` |

### 암호화된 프로퍼티 목록

| 프로퍼티 | 위치 |
|---------|------|
| Main DB 비밀번호 | application-databases.yml |
| JWT Secret | application-common.yml |
| AES 암호화 키 | application-common.yml |
| Redis 비밀번호 | application-common.yml |

**사용 형식:** `ENC(암호화된문자열)`

```yaml
# 예시
spring:
  datasource:
    password: ENC(pZ8SuzYVK/TxFUX3kS2FfJUOFO2hau85THuXIanbbZo=)
security:
  jwt:
    secret: ENC(v+6cGl20xfWghQH+u0K5orHwCFNt6/GbtAyXe3hBf2M=)
```

---

## 8. 시퀀스 관리

### COM_SEQ 테이블 기반 시퀀스

- Spring Data Sequence가 아닌 `COM_SEQ` 테이블로 직접 시퀀스 관리
- `BaseComSeqMapper.xml`에서 SELECT + UPDATE로 다음 번호 조회
- 주문번호, 고객번호, 결제번호 등의 채번에 사용

### Quartz 테이블 DDL

- **파일:** `sumair-be-batch/src/main/resources/schema.sql`
- 11개 QRTZ_ 테이블 CREATE 문 포함
- 엔진: InnoDB
- 17개 인덱스 정의
- `spring.sql.init.mode: never` (자동 실행 비활성화, 수동 적용 필요)
