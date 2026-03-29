# Sumair BE - 인프라 & 배포 구조

## 1. AWS 인프라 구성

### 1.1 전체 인프라 토폴로지

```
┌──────────────────────────────────────────────────────────────────┐
│                    AWS ap-northeast-2 (Seoul)                    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │                    ECR (Container Registry)             │     │
│  │  072062626370.dkr.ecr.ap-northeast-2.amazonaws.com      │     │
│  │                                                         │     │
│  │  dev-sumair-common:latest   prod-sumair-common:latest   │     │
│  │  dev-sumair-home:latest     prod-sumair-home:latest     │     │
│  │  dev-sumair-ibe:latest      prod-sumair-ibe:latest      │     │
│  │  dev-sumair-pay:latest      prod-sumair-pay:latest      │     │
│  │  dev-sumair-batch:latest    prod-sumair-batch:latest    │     │
│  │  dev-sumair-message:latest  prod-sumair-message:latest  │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  ┌───────────────────────┐    ┌────────────────────────────┐     │
│  │  Compute (Container)  │    │    Data Stores              │     │
│  │                       │    │                              │     │
│  │  HOME    :8000        │    │  ┌──────────────────────┐   │     │
│  │  IBE     :8100        │───►│  │  Aurora MySQL         │   │     │
│  │  PAY     :8200        │    │  │  (Writer + Reader)    │   │     │
│  │  BATCH   :8300        │    │  │  sumair-{env}-aurora  │   │     │
│  │  MESSAGE :8400        │    │  └──────────────────────┘   │     │
│  │                       │    │                              │     │
│  │  내부 DNS:            │    │  ┌──────────────────────┐   │     │
│  │  {svc}.dev.internal   │    │  │  Valkey (ElastiCache) │   │     │
│  │  {svc}.prod.internal  │───►│  │  Serverless, SSL     │   │     │
│  └───────────────────────┘    │  │  sumair-{env}-valkey  │   │     │
│                               │  └──────────────────────┘   │     │
│                               └────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 AWS 서비스별 구성

| AWS 서비스 | 리소스명 | 용도 |
|-----------|---------|------|
| **ECR** | `072062626370.dkr.ecr.ap-northeast-2.amazonaws.com` | Docker 이미지 레지스트리 |
| **Aurora MySQL** | `sumair-dev-aurora.cluster-chh3noeqbmvd.apn2` | Dev 메인 DB 클러스터 |
| **Aurora MySQL** | `sumair-prod-aurora.cluster-chh3noeqbmvd.apn2` | Prod 메인 DB 클러스터 |
| **Aurora MySQL** | `sumair-dev-admin.cluster-chh3noeqbmvd.apn2` | Dev 메시지 전용 DB |
| **ElastiCache** | `sumair-dev-elasticashe-valkey-test-dtj6c6.serverless.apn2` | Dev Redis(Valkey) |
| **ElastiCache** | `sumair-prod-valkey-dtj6c6.serverless.apn2` | Prod Redis(Valkey) |

### 1.3 네트워크 구성

**내부 서비스 디스커버리 (DNS 기반):**

| 환경 | 서비스 | 내부 주소 |
|------|--------|----------|
| Dev | home | `http://home.dev.internal:8000` |
| Dev | ibe | `http://ibe.dev.internal:8100` |
| Dev | pay | `http://pay.dev.internal:8200` |
| Dev | batch | `http://batch.dev.internal:8300` |
| Dev | message | `http://message.dev.internal:8400` |
| Prod | home | `http://home.prod.internal:8000` |
| Prod | ibe | `http://ibe.prod.internal:8100` |
| Prod | pay | `http://pay.prod.internal:8200` |
| Prod | batch | `http://batch.prod.internal:8300` |
| Prod | message | `http://message.prod.internal:8400` |

**외부 도메인:**

| 환경 | 도메인 | 용도 |
|------|--------|------|
| Dev | `https://dev-ibe.sumair.co.kr` | Dev 프론트엔드 |
| Dev | `https://xu-staging.sumair.co.kr` | Dev 스테이징 프론트 |
| Prod | `https://ibe.sumair.co.kr` | Prod 프론트엔드 |
| Prod | `https://sumair.kr` | Prod 메인 도메인 |

## 2. Docker 빌드 구조

### 2.1 멀티스테이지 빌드 전략

모든 서비스 모듈은 동일한 3단계 멀티스테이지 빌드를 사용한다.

```
┌─────────────────────────────────────────────────────────┐
│  Stage 1: common (Maven 캐시 레이어)                     │
│                                                         │
│  FROM ECR/{env}-sumair-common:latest AS common          │
│  → 사전 빌드된 Common 모듈의 Maven 로컬 저장소 활용        │
│  → ~/.m2/repository 캐시로 빌드 시간 단축                  │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Stage 2: MAVEN_BUILD                                   │
│                                                         │
│  FROM maven:3.8.7-eclipse-temurin-8 AS MAVEN_BUILD      │
│  COPY --from=common ~/.m2  (캐시된 의존성 복사)           │
│  COPY pom.xml + src/                                    │
│  RUN mvn package -DskipTests                            │
│  → 테스트 스킵으로 빌드 속도 최적화                        │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│  Stage 3: final (런타임 이미지)                           │
│                                                         │
│  FROM openjdk:8                                         │
│  COPY --from=MAVEN_BUILD target/*.jar ./app.jar         │
│  EXPOSE {port}                                          │
│  ENTRYPOINT java -jar ./app.jar (+ JVM 옵션)            │
└─────────────────────────────────────────────────────────┘
```

### 2.2 Dockerfile 환경별 분리

각 모듈에 두 개의 Dockerfile이 존재:

| 파일                | 용도         | Common 이미지 소스               |
| ----------------- | ---------- | --------------------------- |
| `Dockerfile.dev`  | 개발/스테이징 빌드 | `dev-sumair-common:latest`  |
| `Dockerfile.prod` | 운영 빌드      | `prod-sumair-common:latest` |

### 2.3 JVM 실행 옵션

모든 모듈의 ENTRYPOINT에 적용되는 JVM 설정:

```bash
java \
  -Duser.timezone=Asia/Seoul \          # 타임존 고정
  -Dspring.profiles.active=prod,swagger \ # 프로파일 활성화
  -XX:+UseContainerSupport \            # 컨테이너 리소스 인식
  -XX:MaxRAMPercentage=75.0 \           # 컨테이너 메모리의 75% 사용
  -XX:+UseG1GC \                        # G1 가비지 컬렉터
  -jar ./app.jar
```

**주요 설정 해설:**
- `UseContainerSupport`: Docker cgroup 메모리 제한을 JVM이 인식
- `MaxRAMPercentage=75.0`: 컨테이너 메모리 할당의 75%를 JVM 힙으로 사용 (나머지 25%는 메타스페이스/네이티브 메모리)
- `UseG1GC`: 대용량 힙에서도 낮은 지연시간 유지

### 2.4 로컬 개발 환경 (docker-compose.yml)

```yaml
services:
  valkey:
    image: valkey/valkey:latest
    ports:
      - "7379:6379"        # 로컬에서 7379로 매핑
    command: valkey-server --appendonly yes --save ""
    volumes:
      - /data/valkey-data/:/data
```

로컬 개발 시 Valkey(Redis)만 Docker로 구동하며, 서비스 모듈은 IDE에서 직접 실행.

## 3. 배포 프로세스

### 3.1 배포 흐름

```
[개발자 코드 푸시]
       │
       ▼
[Docker 이미지 빌드]
  1. Common 모듈 빌드 → ECR 푸시 ({env}-sumair-common:latest)
  2. 서비스 모듈 빌드 → ECR 푸시 ({env}-sumair-{module}:latest)
       │
       ▼
[ECR에 이미지 저장]
  072062626370.dkr.ecr.ap-northeast-2.amazonaws.com
       │
       ▼
[컨테이너 배포]
  - Graceful Shutdown (30초 타임아웃)
  - 새 컨테이너 기동
  - 헬스체크 확인
```

### 3.2 빌드 순서 (의존성 기반)

```
Step 1: sumair-be-common    ──► ECR 푸시 (다른 모듈의 캐시 레이어로 사용)
        │
Step 2: (병렬 가능)
        ├── sumair-be-home
        ├── sumair-be-ibe
        ├── sumair-be-pay     ──► 각각 ECR 푸시
        ├── sumair-be-batch
        └── sumair-be-message
```

**중요**: Common 모듈은 반드시 먼저 빌드해야 하며, 나머지 5개 서비스 모듈은 병렬 빌드 가능.

### 3.3 Graceful Shutdown 설정

모든 서비스에 적용:
```yaml
server:
  shutdown: graceful

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```

- SIGTERM 수신 시 진행 중인 요청 완료까지 최대 30초 대기
- 신규 요청은 즉시 거부 (503 응답)
- 30초 초과 시 강제 종료

## 4. 환경별 설정 구조

### 4.1 환경 구분

| 환경 | Spring Profile | 용도 |
|------|---------------|------|
| **Local** | `default` (프로파일 미지정) | 개발자 로컬 머신 |
| **Dev** | `dev` | AWS 개발/스테이징 서버 |
| **Prod** | `prod` | AWS 운영 서버 |

### 4.2 환경별 주요 차이점

| 항목 | Local | Dev | Prod |
|------|-------|-----|------|
| DB 호스트 | localhost / dev Aurora | dev Aurora cluster | prod Aurora cluster |
| Redis 호스트 | localhost:7379 | ElastiCache (SSL) | ElastiCache (SSL) |
| 서비스 호스트 | localhost:{port} | {svc}.dev.internal:{port} | {svc}.prod.internal:{port} |
| IBES 엔드포인트 | xustg.ibsplc.aero | xustg.ibsplc.aero | xu.ibsplc.aero |
| Toss API 키 | test_sk_* | test_sk_* | live_sk_* |
| 로그 경로 | `./logs/` | `/app/{module}/logs/` | `/app/{module}/logs/` |
| 로그 레벨 | debug | debug | info |
| Swagger | 활성화 | 활성화 | 활성화 (prod,swagger) |
| 프론트 도메인 | localhost:3000 | xu-staging.sumair.co.kr | sumair.kr |

## 5. 설정 파일 로드 순서

### 5.1 Spring Boot 설정 로드 메커니즘

Spring Boot 2.7.x에서 설정은 아래 순서로 로드되며, **나중에 로드된 설정이 우선**한다.

```
[로드 순서: 낮은 우선순위 → 높은 우선순위]

 1. application.yml (각 모듈 자체)
    │  → 서버 포트, 모듈 전용 설정, spring.config.import 선언
    │
 2. spring.config.import로 가져오는 Common 설정들:
    │  ├── application-databases.yml    (DB 접속 정보)
    │  ├── application-common.yml       (JWT, 암호화 키, 호스트 URL)
    │  └── application-ibe.yml          (IBES/IBEP 엔드포인트) — ibe/home 모듈
    │
 3. 프로파일별 활성화 (spring.config.activate.on-profile):
    │  ├── on-profile: dev   → Dev용 설정 블록 활성화
    │  └── on-profile: prod  → Prod용 설정 블록 활성화
    │
 4. JVM 시스템 프로퍼티 (-D 옵션):
    │  └── -Dspring.profiles.active=prod,swagger
    │
 5. Jasypt 복호화:
       └── ENC(...) 값들이 런타임에 복호화됨 (키: SUMAIR-JASYPT-KEY)
```

### 5.2 모듈별 설정 파일 맵

```
sumair-be-common/src/main/resources/
├── application-databases.yml    # DB 접속 (Aurora, HikariCP)
├── application-common.yml       # JWT, AES, 호스트 URL, 공통 설정
└── application-ibe.yml          # IBES/IBEP SOAP 엔드포인트/인증

sumair-be-home/src/main/resources/
├── application.yml              # 포트:8000, Redis, OAuth2, Wallet, 모듈 설정
└── config/mybatis/mybatis-config.xml

sumair-be-ibe/src/main/resources/
├── application.yml              # 포트:8100, CXF 설정, SOAP 로깅
└── config/mybatis/mybatis-config.xml

sumair-be-pay/src/main/resources/
├── application.yml              # 포트:8200, Toss Payments 키/MID
└── config/mybatis/mybatis-config.xml

sumair-be-batch/src/main/resources/
├── application.yml              # 포트:8300, 공공API 키, FTP 설정
├── application-quartz.yml       # Quartz 스케줄러 (클러스터, JDBC 저장소)
└── config/mybatis/mybatis-config.xml

sumair-be-message/src/main/resources/
├── application.yml              # 포트:8400, Infobip, Slack, IRES 인증
└── config/mybatis/mybatis-config.xml
```

### 5.3 설정 암호화 체계

```
┌──────────────────────────────────────────┐
│  application.yml 내 암호화된 값           │
│                                          │
│  password: ENC(pZ8SuzYVK/TxFUX3...)     │
│  jwt.secret: ENC(v+6cGl20xfWghQ...)     │
│  aes.key: ENC(Ik/c4Z38C5jRR3yf...)      │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│  JasyptConfig.java                       │
│                                          │
│  Encryption Key: SUMAIR-JASYPT-KEY       │
│  Algorithm: PBEWithMD5AndDES (default)   │
│  → 런타임 시 자동 복호화                   │
└──────────────────────────────────────────┘
```

**주의**: 일부 Prod 설정에서 암호화되지 않은 평문 비밀번호가 존재함 (DB, Infobip, Toss 키 등). 보안 강화를 위해 Jasypt 암호화 적용 또는 AWS Secrets Manager 도입을 권장.

## 6. 로깅 구조

### 6.1 로깅 설정 (Logback)

| 항목 | 설정값 |
|------|-------|
| 프레임워크 | Logback (Spring Boot 내장) |
| 콘솔 패턴 | `%d{ISO8601} [MODULE]-[%thread] %-5level %logger{36} :: %msg%n` |
| 파일 패턴 | 동일 |
| 롤링 정책 | 시간 + 크기 기반 (`%d{yyyy-MM-dd}.%i.log`) |
| 최대 파일 크기 | 10MB |
| 보관 기간 | 90일 |
| SOAP 로그 | 파일 기반 별도 저장, 365일 보관 (`logs/soap-files/`) |

### 6.2 로그 경로

| 환경 | 모듈 | 경로 |
|------|------|------|
| Local | 전체 | `./logs/sumair-{module}.log` |
| Dev/Prod | home | `/app/home/logs/sumair-home.log` |
| Dev/Prod | ibe | `/app/ibe/logs/sumair-ibe.log` |
| Dev/Prod | pay | `/app/pay/logs/sumair-pay.log` |
| Dev/Prod | batch | `/app/batch/logs/sumair-batch.log` |
| Dev/Prod | message | `/app/message/logs/sumair-message.log` |

### 6.3 로그 레벨 설정

| 패키지 | Local/Dev | Prod |
|--------|----------|------|
| `com.sumair` | DEBUG | INFO |
| `com.zaxxer.hikari` | DEBUG | WARN |
| Spring Framework | INFO | WARN |

## 7. 데이터베이스 인프라

### 7.1 Aurora MySQL 클러스터 구성

```
┌─────────────────────────────────────────────────┐
│  Aurora MySQL Cluster                           │
│  (chh3noeqbmvd.ap-northeast-2.rds.amazonaws.com)│
│                                                 │
│  ┌───────────────┐    ┌───────────────┐         │
│  │ Writer        │    │ Reader        │         │
│  │ (cluster EP)  │◄──►│ (read-only EP)│         │
│  └───────────────┘    └───────────────┘         │
│                                                 │
│  AWS JDBC Wrapper Plugins:                      │
│  ┌────────────────────┐                         │
│  │ readWriteSplitting  │ → SELECT는 Reader로     │
│  │ failover            │ → Writer 장애 시 자동    │
│  │ efm2                │ → 향상된 장애 모니터링    │
│  │ logQuery            │ → 쿼리 로깅              │
│  └────────────────────┘                         │
└─────────────────────────────────────────────────┘
```

### 7.2 커넥션 풀 설정 (HikariCP)

| 항목 | 메인 DB | 메시지 DB |
|------|--------|----------|
| max-pool-size | 5 | 3 |
| min-idle | 3 | - |
| max-lifetime | 600,000ms (10분) | 300,000ms (5분) |
| idle-timeout | 60,000ms (1분) | 60,000ms (1분) |
| connection-timeout | 30,000ms (기본) | 30,000ms (기본) |
| Driver | `software.amazon.jdbc.Driver` | `software.amazon.jdbc.Driver` |
| Dialect | `aurora-mysql` | `aurora-mysql` |

## 8. 캐시 인프라 (Valkey/Redis)

### 8.1 ElastiCache Serverless 구성

| 항목 | Dev | Prod |
|------|-----|------|
| 엔드포인트 | `sumair-dev-elasticashe-valkey-test-dtj6c6.serverless.apn2.cache.amazonaws.com` | `sumair-prod-valkey-dtj6c6.serverless.apn2.cache.amazonaws.com` |
| 포트 | 6379 | 6379 |
| SSL | 활성화 | 활성화 |
| 모드 | Serverless (자동 스케일링) | Serverless (자동 스케일링) |

### 8.2 Redis 캐시 설정 (Spring)

```
CacheManager 설정:
├── Entry TTL: 30분
├── Key Serializer: StringRedisSerializer
├── Value Serializer: GenericJackson2JsonRedisSerializer
└── Batch Strategy: Custom (캐시 클리어 시 SCAN 기반)
```

## 9. 스케줄링 인프라 (Quartz)

### 9.1 Quartz 클러스터 설정

```yaml
Quartz Configuration:
├── Instance ID: AUTO (자동 생성)
├── Job Store: JDBC (Aurora MySQL, QRTZ_ 테이블)
├── Clustering: enabled
│   └── Cluster Checkin Interval: 15초
├── Thread Pool: SimpleThreadPool (20 threads)
├── Misfire Threshold: 60초
├── Trigger Acquisition: WithinLock (동시성 안전)
└── Batch Trigger Acquisition: 최대 20개
```

### 9.2 Quartz 테이블

Quartz는 JDBC Job Store를 사용하며, Aurora MySQL에 `QRTZ_` 접두사 테이블들이 생성되어 클러스터 노드 간 작업 분배를 관리한다.

## 10. 보안 인프라 권장사항

현재 구성에서 발견된 보안 개선 포인트:

| 항목                    | 현재 상태        | 권장 조치                             |
| --------------------- | ------------ | --------------------------------- |
| Prod DB 비밀번호          | 일부 평문        | Jasypt 암호화 또는 AWS Secrets Manager |
| OAuth Client Secret   | YAML 내 평문    | AWS Secrets Manager로 외부화          |
| API 키 (Toss, Infobip) | YAML 내 평문    | 환경 변수 또는 Secrets Manager          |
| Wallet 인증서 비밀번호       | YAML 내 평문    | Secrets Manager 또는 KMS            |
| Jasypt 마스터 키          | 코드/설정 내 하드코딩 | 환경 변수로 주입                         |

---

> **문서 버전**: 2026-03-22 | **작성**: 시스템 아키텍트
