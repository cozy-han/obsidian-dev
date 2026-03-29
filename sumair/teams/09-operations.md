# Sumair BE - 운영 & 모니터링 가이드

## 1. 로깅 구조

### 1.1 로깅 아키텍처 개요

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │  HOME    │ │   IBE    │ │   PAY    │ │  BATCH   │ ...   │
│  │ Logback  │ │ Logback  │ │ Logback  │ │ Logback  │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘       │
│       │             │            │             │             │
│       ▼             ▼            ▼             ▼             │
│  ┌──────────────────────────────────────────────────┐       │
│  │              Log Output Targets                  │       │
│  │                                                  │       │
│  │  Console (stdout) ← 컨테이너 로그 수집용          │       │
│  │  File (rolling)   ← 디스크 보관용                 │       │
│  │  SOAP File Log    ← CXF SOAP 통신 전문 별도 보관  │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 로그 파일 경로 및 롤링 정책

| 환경 | 모듈 | 로그 파일 경로 | 아카이브 경로 |
|------|------|---------------|-------------|
| Local | 전체 | `./logs/sumair-{module}.log` | `./logs/archived/` |
| Dev/Prod | home | `/app/home/logs/sumair-home.log` | `/app/home/logs/archived/` |
| Dev/Prod | ibe | `/app/ibe/logs/sumair-ibe.log` | `/app/ibe/logs/archived/` |
| Dev/Prod | pay | `/app/pay/logs/sumair-pay.log` | `/app/pay/logs/archived/` |
| Dev/Prod | batch | `/app/batch/logs/sumair-batch.log` | `/app/batch/logs/archived/` |
| Dev/Prod | message | `/app/message/logs/sumair-message.log` | `/app/message/logs/archived/` |

**롤링 정책:**
- 파일 크기 기반: 최대 **10MB** per file
- 파일 패턴: `{module}-api-%d{yyyy-MM-dd}.%i.log`
- 보관 기간: **90일** (자동 삭제)
- SOAP 전문 로그: **365일** 보관 (`logs/soap-files/`)

### 1.3 로그 포맷

```
# 콘솔/파일 공통 포맷
%d{ISO8601} [MODULE]-[%thread] %-5level %logger{36} :: %msg%n

# 실제 출력 예시
2026-03-22T10:30:15.123+0900 [HOME]-[http-nio-8000-exec-1] INFO  c.s.home.controller.BookingCtrl :: 예약 조회 완료 PNR=ABC123
```

### 1.4 로그 레벨 설정

| 패키지/카테고리 | Dev | Prod | 비고 |
|---------------|-----|------|------|
| `com.sumair` | DEBUG | INFO | 애플리케이션 코드 |
| `com.zaxxer.hikari` | DEBUG | WARN | DB 커넥션 풀 |
| `com.zaxxer.hikari.HikariConfig` | DEBUG | INFO | 풀 설정 로그 |
| `org.springframework` | INFO | WARN | Spring 프레임워크 |
| `org.apache.cxf` | INFO | WARN | SOAP 프레임워크 |
| `org.mybatis` | DEBUG | INFO | SQL 매퍼 |

### 1.5 특수 로깅

#### SOAP 전문 로깅 (IBE/Message 모듈)
- CXF `SoapFileLoggerConfig`를 통해 SOAP 요청/응답 전문을 파일로 저장
- 경로: `logs/soap-files/`
- 보관: 365일
- 용도: 항공사 시스템(IBES/IBEP) 통신 감사 추적

#### Aurora 인스턴스 로깅
- `AuroraInstanceLogInterceptor` (MyBatis Plugin): 어떤 Aurora 인스턴스(Writer/Reader)에서 쿼리가 실행되었는지 추적
- Read/Write Splitting이 정상 동작하는지 모니터링 가능

#### AOP 기반 로깅
- `LoggingAspect`: 서비스 계층 메서드 호출/응답을 AOP로 자동 로깅

## 2. 모니터링 포인트

### 2.1 핵심 모니터링 매트릭스

#### 애플리케이션 레벨

| 모니터링 항목 | 대상 모듈 | 임계값 제안 | 확인 방법 |
|-------------|----------|-----------|----------|
| HTTP 응답 시간 | 전체 | p95 > 3s 경고, p99 > 10s 위험 | 액세스 로그 분석 |
| HTTP 에러율 | 전체 | 5xx > 1% 경고, > 5% 위험 | 에러 로그 카운트 |
| JVM 힙 사용률 | 전체 | > 80% 경고, > 90% 위험 | JMX / Actuator |
| GC Pause 시간 | 전체 | > 500ms 경고 | GC 로그 |
| 스레드 수 | 전체 | > 200 경고 | JMX |
| SOAP 응답 시간 | ibe, message | > 5s 경고, > 15s 위험 | SOAP 전문 로그 |
| Quartz 실패 잡 | batch | > 0 위험 | QRTZ_ 테이블 조회 |

#### 인프라 레벨

| 모니터링 항목 | AWS 서비스 | 임계값 제안 | CloudWatch 메트릭 |
|-------------|-----------|-----------|------------------|
| DB CPU 사용률 | Aurora MySQL | > 70% 경고, > 90% 위험 | `CPUUtilization` |
| DB 커넥션 수 | Aurora MySQL | > 80% of max 경고 | `DatabaseConnections` |
| DB 복제 지연 | Aurora MySQL | > 100ms 경고 | `AuroraReplicaLag` |
| DB Deadlock | Aurora MySQL | > 0 경고 | `Deadlocks` |
| Cache 히트율 | Valkey | < 80% 경고 | `CacheHitRate` |
| Cache 메모리 | Valkey | > 80% 경고 | `BytesUsedForCache` |
| Cache 지연시간 | Valkey | > 5ms 경고 | `EngineCPUUtilization` |
| 컨테이너 CPU | ECS/EKS | > 80% 경고 | `CPUUtilization` |
| 컨테이너 메모리 | ECS/EKS | > 85% 경고 | `MemoryUtilization` |

### 2.2 서비스별 핵심 헬스체크 포인트

```
┌─────────────────────────────────────────────────────────┐
│  HOME (:8000)                                           │
│  ├── DB 커넥션 정상 여부                                  │
│  ├── Redis(Valkey) 연결 상태                              │
│  ├── JWT 토큰 발급/검증 정상 여부                          │
│  └── OAuth2 Provider 접근 가능 여부                       │
│                                                         │
│  IBE (:8100)                                            │
│  ├── DB 커넥션 정상 여부                                  │
│  ├── IBES/IBEP SOAP 엔드포인트 응답 여부                   │
│  └── CXF 서비스 상태                                     │
│                                                         │
│  PAY (:8200)                                            │
│  ├── DB 커넥션 정상 여부                                  │
│  └── Toss Payments API 접근 가능 여부                     │
│                                                         │
│  BATCH (:8300)                                          │
│  ├── DB 커넥션 정상 여부                                  │
│  ├── Quartz 스케줄러 상태 (STARTED/STANDBY)               │
│  └── SFTP 서버 접근 가능 여부                              │
│                                                         │
│  MESSAGE (:8400)                                        │
│  ├── DB 커넥션 정상 여부 (별도 datasource)                 │
│  ├── Infobip API 접근 가능 여부                           │
│  ├── Slack Webhook 응답 여부                              │
│  └── IRES SOAP 엔드포인트 상태                            │
└─────────────────────────────────────────────────────────┘
```

### 2.3 Spring Boot Actuator 활용 (권장)

현재 Actuator 설정이 제한적이므로, 다음 엔드포인트 활성화를 권장:

```yaml
# 권장 추가 설정
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
  health:
    db:
      enabled: true
    redis:
      enabled: true
```

## 3. 알람 체계

### 3.1 현재 알람 채널

```
┌──────────────────────────────┐
│  Slack Webhook               │
│  채널: 운영 알림 채널          │
│  URL: hooks.slack.com/...    │
│                              │
│  발송 모듈: MESSAGE (:8400)   │
│  트리거:                      │
│  ├── 시스템 오류 발생 시       │
│  ├── 배치 작업 실패 시         │
│  └── 중요 이벤트 발생 시       │
└──────────────────────────────┘
```

### 3.2 권장 알람 구성

| 알람 등급 | 조건 | 채널 | 대응 시간 |
|----------|------|------|----------|
| **P1 (Critical)** | 서비스 다운, DB 접속 불가, 결제 실패율 급증 | Slack + SMS(Infobip) + 전화 | 즉시 (15분 이내) |
| **P2 (High)** | 응답 지연 급증, 에러율 5% 초과, Cache 장애 | Slack + SMS | 1시간 이내 |
| **P3 (Medium)** | 배치 실패, 디스크 80% 초과, 커넥션 풀 경고 | Slack | 업무시간 내 |
| **P4 (Low)** | 로그 경고, 비정상 트래픽 패턴 | Slack | 다음 업무일 |

### 3.3 CloudWatch 알람 설정 (권장)

```
Aurora MySQL:
├── CPUUtilization > 90% → P2 알람
├── DatabaseConnections > 80% of max → P3 알람
├── AuroraReplicaLag > 200ms → P2 알람
└── FreeableMemory < 500MB → P2 알람

Valkey (ElastiCache):
├── EngineCPUUtilization > 80% → P2 알람
├── CurrConnections > 500 → P3 알람
└── ReplicationLag > 100ms → P2 알람

Container:
├── CPUUtilization > 90% for 5min → P2 알람
├── MemoryUtilization > 90% → P1 알람
└── HealthCheckFailed → P1 알람
```

## 4. 장애 대응 절차

### 4.1 장애 탐지 → 대응 플로우

```
[장애 탐지]
    │
    ├── 모니터링 알람 수신 (Slack/SMS)
    ├── 사용자 리포트
    └── 배치 실패 알림
    │
    ▼
[1단계: 초기 진단] (5분 이내)
    │
    ├── 어떤 서비스가 영향받는지 확인
    │   → 각 서비스 헬스체크 확인 (GET /{service}/health)
    │
    ├── 에러 로그 확인
    │   → /app/{module}/logs/sumair-{module}.log
    │   → docker logs {container}
    │
    └── 인프라 상태 확인
        → AWS Console: Aurora, ElastiCache, 컨테이너 상태
    │
    ▼
[2단계: 원인 분류]
    │
    ├── DB 장애 ──────────────► 4.2 DB 장애 대응
    ├── Cache 장애 ────────────► 4.3 Cache 장애 대응
    ├── 외부 API 장애 ──────────► 4.4 외부 연동 장애 대응
    ├── 애플리케이션 버그 ────────► 4.5 롤백 절차
    └── 인프라(컨테이너) 장애 ───► 4.6 컨테이너 장애 대응
    │
    ▼
[3단계: 복구 확인]
    ├── 서비스 정상 응답 확인
    ├── 에러율 정상화 확인
    └── 후속 조치 (RCA 작성, 재발 방지)
```

### 4.2 DB 장애 대응

```
[Aurora MySQL 장애]
    │
    ├── Writer 노드 다운
    │   → AWS JDBC Wrapper의 failover 플러그인이 자동 장애조치
    │   → efm2(Enhanced Failure Monitoring)가 빠른 감지 수행
    │   → Reader가 Writer로 승격 (보통 30초 이내)
    │   → 수동 조치: AWS Console에서 Failover 트리거
    │
    ├── Reader 노드 다운
    │   → readWriteSplitting 플러그인이 Writer로 읽기 전환
    │   → 성능 저하 가능 (Writer에 읽기 부하 집중)
    │
    ├── 커넥션 풀 고갈
    │   → HikariCP max-pool-size: 5 (현재 매우 보수적)
    │   → 긴급 시 풀 사이즈 증가 후 재배포
    │   → 장기적: slow query 최적화, 커넥션 누수 점검
    │
    └── Deadlock 발생
        → Aurora Deadlock 로그 확인
        → 관련 트랜잭션 분석 (MyBatis 매퍼 확인)
        → 인덱스/쿼리 최적화
```

### 4.3 Cache 장애 대응

```
[Valkey (ElastiCache) 장애]
    │
    ├── 연결 불가
    │   → HOME 모듈에만 영향 (유일한 Redis 클라이언트)
    │   → 캐시 미사용 fallback으로 DB 직접 조회 (성능 저하)
    │   → ElastiCache Serverless 자동 복구 대기
    │
    ├── 응답 지연
    │   → 캐시 키 패턴 확인 (대량 키 스캔 여부)
    │   → TTL 설정 확인 (현재 30분)
    │   → Hot Key 확인
    │
    └── 메모리 부족
        → Serverless는 자동 스케일링
        → 불필요한 캐시 키 정리
        → TTL 단축 검토
```

### 4.4 외부 연동 장애 대응

| 외부 시스템 | 장애 시 영향 | 대응 |
|-----------|------------|------|
| IBES/IBEP | 항공 예약/검색 불가 | SOAP 전문 로그 확인, iFly 측 문의, 재시도 로직 확인 |
| Toss Payments | 결제 불가 | Toss 상태 페이지 확인, 결제 큐잉 검토, 고객 안내 |
| Infobip | SMS/이메일 발송 실패 | 대체 채널 검토, 발송 큐에 적체된 메시지 재처리 |
| data.go.kr | 배치 데이터 미수집 | 다음 배치 주기에 자동 재시도, 수동 트리거 가능 |
| SFTP (ibsplc) | 정산 데이터 미수신 | SFTP 접속 확인, 수동 다운로드, iFly 측 문의 |

### 4.5 롤백 절차

```
[배포 후 장애 발생 시 롤백]
    │
    ▼
Step 1: 장애 확인
    - 에러 로그에서 신규 배포 관련 오류 식별
    - 배포 전후 에러율 비교
    │
    ▼
Step 2: 이전 이미지로 롤백
    - ECR에서 이전 정상 이미지 태그 확인
      (latest가 아닌 이전 빌드 이미지)
    │
    ▼
Step 3: 컨테이너 교체
    - 이전 이미지로 컨테이너 재배포
    - Graceful Shutdown 대기 (30초)
    │
    ▼
Step 4: 검증
    - 헬스체크 통과 확인
    - 에러율 정상화 확인
    - 주요 기능 수동 테스트
    │
    ▼
Step 5: 후속 조치
    - 장애 원인 분석 (RCA)
    - 수정 후 재배포 계획 수립
```

**롤백 시 주의사항:**
- DB 스키마 변경이 포함된 배포는 롤백이 복잡할 수 있음 (현재 DB 마이그레이션 도구 미사용)
- Quartz 배치 잡 변경 시 QRTZ_ 테이블 데이터도 확인 필요
- Common 모듈 변경 시 의존하는 모든 서비스 모듈도 함께 롤백 필요

### 4.6 컨테이너 장애 대응

```
[컨테이너 레벨 장애]
    │
    ├── OOM (Out of Memory)
    │   → MaxRAMPercentage=75.0 설정 확인
    │   → 힙 덤프 분석 (가능한 경우)
    │   → 컨테이너 메모리 제한 증가 또는 메모리 누수 수정
    │
    ├── CPU 과부하
    │   → 스레드 덤프 확인
    │   → Slow Query / SOAP 타임아웃 확인
    │   → 스케일 아웃 또는 원인 코드 최적화
    │
    └── 컨테이너 재시작 반복
        → 애플리케이션 시작 로그 확인
        → DB/Cache 연결 실패 여부 확인
        → 설정 파일 오류 (Jasypt 복호화 실패 등) 확인
```

## 5. 운영 체크리스트

### 5.1 일일 점검 항목

- [ ] 전체 서비스 헬스체크 정상 확인
- [ ] 배치 작업 실행 결과 확인 (QRTZ_ 테이블 또는 배치 로그)
- [ ] 에러 로그 이상 패턴 확인
- [ ] DB 커넥션 풀 상태 확인
- [ ] Slack 알림 채널 미처리 알람 확인

### 5.2 주간 점검 항목

- [ ] Aurora MySQL 성능 지표 리뷰 (CPU, 커넥션, 복제 지연)
- [ ] Valkey 캐시 히트율 및 메모리 사용량 확인
- [ ] 디스크 사용량 확인 (로그 파일 축적)
- [ ] SOAP 전문 로그 용량 확인 (365일 보관)
- [ ] Slow Query 리뷰

### 5.3 월간 점검 항목

- [ ] 인증서 만료일 확인 (IBES TLS 인증서, Apple Wallet 서명 인증서)
- [ ] API 키 / 토큰 만료 확인 (Samsung Wallet, Infobip 등)
- [ ] HikariCP 커넥션 풀 효율성 리뷰
- [ ] 로그 보관 정책 준수 확인 (90일/365일)
- [ ] 보안 패치 적용 여부 확인

## 6. 유용한 운영 명령어

### 6.1 로그 조회

```bash
# 특정 모듈 실시간 로그 (컨테이너 내부)
tail -f /app/{module}/logs/sumair-{module}.log

# 에러만 필터링
grep "ERROR" /app/{module}/logs/sumair-{module}.log | tail -50

# 특정 시간대 로그
grep "2026-03-22T10:3" /app/{module}/logs/sumair-{module}.log

# SOAP 전문 로그 (IBE 모듈)
ls -la /app/ibe/logs/soap-files/
```

### 6.2 DB 상태 확인

```sql
-- Aurora 현재 커넥션 수 확인
SHOW STATUS LIKE 'Threads_connected';

-- 실행 중인 쿼리 확인
SHOW FULL PROCESSLIST;

-- Deadlock 정보 확인
SHOW ENGINE INNODB STATUS;

-- Quartz 스케줄러 상태 확인
SELECT * FROM QRTZ_SCHEDULER_STATE;

-- 실행 중인 배치 잡 확인
SELECT * FROM QRTZ_FIRED_TRIGGERS;
```

### 6.3 Redis(Valkey) 상태 확인

```bash
# 연결 테스트
redis-cli -h {valkey-endpoint} -p 6379 --tls ping

# 키 수 확인
redis-cli -h {valkey-endpoint} -p 6379 --tls dbsize

# 메모리 사용량
redis-cli -h {valkey-endpoint} -p 6379 --tls info memory
```

---

> **문서 버전**: 2026-03-22 | **작성**: 시스템 아키텍트
