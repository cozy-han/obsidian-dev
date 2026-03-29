# db-scheduler 라이브러리 조사

> 참조: [GitHub - kagkarlsson/db-scheduler](https://github.com/kagkarlsson/db-scheduler) | [JavaDoc](https://javadoc.io/doc/com.github.kagkarlsson/db-scheduler/latest/index.html)
> 최신 버전: 16.7.x (2025년 1월 기준)

---

## 1. 개요

db-scheduler는 **Java용 영속적(persistent), 클러스터 친화적(cluster-friendly) 태스크 스케줄러**이다. `java.util.concurrent.ScheduledExecutorService`의 클러스터 버전이 필요하지만 Quartz처럼 복잡한 것은 원하지 않을 때 사용한다.

### 해결하는 문제
- **단일 DB 테이블**만으로 영속적 스케줄링 구현
- **분산 환경**에서 동일 태스크의 **중복 실행 방지** (DB 레벨 잠금)
- 스케줄러 인스턴스 장애 시 **자동 감지 및 복구**
- 외부 메시징 시스템(Kafka, RabbitMQ) 없이 **신뢰성 있는 작업 스케줄링**

### 핵심 특징
- Apache 2.0 라이선스
- 최소 의존성 (slf4j만 필요)
- 2,000~10,000 executions/sec 처리량 (테스트 기준)
- PostgreSQL, MySQL, Oracle, MSSQL 지원

---

## 2. 핵심 개념

### 2.1 Task 종류

#### RecurringTask (반복 태스크)
- 정해진 스케줄에 따라 **반복 실행**
- 실행 완료 후 자동으로 다음 실행 스케줄링
- 인스턴스가 하나 (task_name + "DEFAULT" 인스턴스)
- 예: 매시간 캔들 데이터 수집, 매일 포트폴리오 정산

#### OneTimeTask (일회성 태스크)
- 지정된 시간에 **한 번만 실행**
- 고유 ID와 커스텀 데이터 부착 가능
- 실행 완료 후 DB에서 제거
- 예: 주문 실행, 알림 발송, 지연된 작업 처리

#### CustomTask
- RecurringTask/OneTimeTask로 커버되지 않는 **커스텀 동작** 정의
- 실행 완료 후 동작을 `CompletionHandler`로 직접 제어

### 2.2 TaskInstance
- 특정 Task의 **실행 가능한 인스턴스**
- `task_name` + `task_instance` 조합이 Primary Key
- 커스텀 데이터(`task_data`)를 직렬화하여 저장 가능

### 2.3 Execution
- TaskInstance의 **실제 실행 기록**
- `execution_time`, `picked`, `picked_by`, `last_heartbeat` 등 상태 추적
- `version` 필드로 낙관적 락 구현

### 2.4 Scheduler
- 태스크 실행을 관리하는 **메인 엔트리포인트**
- 스레드풀 관리, 폴링, 하트비트, 데드 실행 감지 수행
- `SchedulerClient` 인터페이스로 런타임에 태스크 스케줄링/취소 가능

---

## 3. 동작 원리

### 3.1 DB 폴링 방식

스케줄러는 주기적으로(기본 10초) DB를 폴링하여 실행할 태스크를 조회한다.

**두 가지 폴링 전략:**

| 전략 | 설명 | 적합한 경우 |
|------|------|-------------|
| `fetch-and-lock-on-execute` (기본) | 실행 예정 태스크를 먼저 조회하고, 실행 시점에 락 획득 경쟁 | 일반적인 사용 (초당 수백 건 이하) |
| `lock-and-fetch` | `SELECT FOR UPDATE ... SKIP LOCKED`로 조회와 동시에 락 획득 | 고처리량 환경 (초당 1,000건 이상) |

### 3.2 낙관적 락 (Optimistic Locking)

```
1. 스케줄러가 execution_time이 지난 태스크를 SELECT
2. UPDATE scheduled_tasks
   SET picked = true, picked_by = '<scheduler-name>', version = version + 1
   WHERE task_name = ? AND task_instance = ? AND version = ?
3. UPDATE가 1 row 영향 → 락 획득 성공 → 실행
4. UPDATE가 0 row 영향 → 다른 인스턴스가 선점 → 스킵
```

- `version` 컬럼을 통한 **CAS(Compare-And-Swap)** 방식
- 별도의 분산 락 시스템 불필요

### 3.3 하트비트 (Heartbeat)

- 실행 중인 태스크는 주기적으로(기본 5분) `last_heartbeat` 갱신
- 스케줄러가 살아있고 태스크가 정상 실행 중임을 증명

### 3.4 데드 실행 감지 (Dead Execution Detection)

```
데드 실행 판단 기준:
  picked = true
  AND last_heartbeat < now() - (heartbeat_interval * missed_heartbeats_limit)

기본값: 5분 * 6 = 30분 동안 하트비트 없으면 "데드"로 판단
```

- 데드 실행 감지 시 기본적으로 `ReviveDeadExecution` 핸들러가 **즉시 재스케줄링**
- 커스텀 `DeadExecutionHandler`로 동작 변경 가능

---

## 4. 미스파이어 처리

"미스파이어"란 스케줄러 다운타임 등으로 예정된 실행 시간을 놓친 경우를 말한다.

### 감지 메커니즘
1. **폴링 시점**: `execution_time <= now()` 조건으로 조회하므로, 과거 시간의 미실행 태스크도 자동으로 잡힌다
2. **데드 실행 감지**: picked 상태인데 하트비트가 끊긴 태스크를 찾아냄

### 보상 전략
- **RecurringTask**: 스케줄러 재시작 시 `execution_time`이 과거인 태스크를 즉시 실행하고, 다음 스케줄로 재등록
- **OneTimeTask**: 실행 시간이 지났더라도 DB에 남아있으므로 폴링 시 즉시 실행
- **데드 실행**: `ReviveDeadExecution` 핸들러가 `execution_time = now()`로 갱신하여 즉시 재실행

### 중요 참고
- db-scheduler는 Quartz의 `MISFIRE_INSTRUCTION_*` 같은 세밀한 미스파이어 정책은 제공하지 않는다
- "놓친 실행은 가능한 빨리 1번 실행한다"는 단순한 정책

---

## 5. 분산 환경 지원

### 중복 방지 원리

```
[App Instance A]  ──┐
[App Instance B]  ──┤──> [PostgreSQL: scheduled_tasks 테이블]
[App Instance C]  ──┘
                         version 기반 낙관적 락으로
                         단 하나의 인스턴스만 실행
```

- **별도의 클러스터링 프레임워크 불필요** (ZooKeeper, Redis 등 필요 없음)
- DB가 **Single Source of Truth** 역할
- 각 인스턴스는 `picked_by` 필드에 자신의 이름(기본: hostname) 기록
- 모든 인스턴스가 동일한 DB 테이블을 공유하면 자동으로 클러스터 형성

### 운영 고려사항
- DB 연결이 끊기면 스케줄링도 중단 (DB가 SPOF)
- `lock-and-fetch` 전략은 PostgreSQL의 `SKIP LOCKED` 활용으로 경합 최소화
- 인스턴스 수 증가에 따른 DB 폴링 부하 고려 필요

---

## 6. 실패/재시도

### 기본 동작

| 태스크 유형 | 실패 시 기본 동작 |
|------------|------------------|
| RecurringTask | 원래 스케줄대로 다음 실행 (예: 1시간 후) |
| OneTimeTask | **5분 후 재시도** |

### 내장 FailureHandler

```kotlin
// 1. 최대 재시도 횟수 제한
Tasks.oneTime("my-task", MyData::class.java)
    .onFailure(MaxRetriesFailureHandler(maxRetries = 5))
    .execute { inst, ctx -> /* ... */ }

// 2. 지수 백오프 (Exponential Backoff)
Tasks.oneTime("my-task", MyData::class.java)
    .onFailure(ExponentialBackoffFailureHandler(
        Duration.ofSeconds(1),  // 초기 딜레이
        2.0                      // 배수
    ))
    .execute { inst, ctx -> /* ... */ }

// 3. 커스텀 FailureHandler
Tasks.oneTime("my-task", MyData::class.java)
    .onFailure { executionComplete, executionOperations ->
        val failures = executionComplete.execution.consecutiveFailures
        if (failures >= 10) {
            executionOperations.stop()  // 재시도 중단
        } else {
            executionOperations.reschedule(
                executionComplete, Instant.now().plusSeconds(30)
            )
        }
    }
    .execute { inst, ctx -> /* ... */ }
```

### DeadExecutionHandler

```kotlin
// 기본: 즉시 재스케줄링
Tasks.recurring("my-task", FixedDelay.ofHours(1))
    .onDeadExecution(ReviveDeadExecution())  // 기본값
    .execute { inst, ctx -> /* ... */ }

// 커스텀: 로깅 후 재스케줄링
Tasks.recurring("my-task", FixedDelay.ofHours(1))
    .onDeadExecution { execution, executionOperations ->
        logger.error("Dead execution detected: ${execution.taskInstance}")
        executionOperations.reschedule(
            ExecutionComplete.simulated(execution), Instant.now()
        )
    }
    .execute { inst, ctx -> /* ... */ }
```

---

## 7. Spring Boot 연동

### 7.1 의존성 추가

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.github.kagkarlsson:db-scheduler-spring-boot-starter:16.7.0")
}
```

### 7.2 application.yml 설정

```yaml
db-scheduler:
  enabled: true                          # 스케줄러 활성화 (기본: true)
  threads: 10                            # 실행 스레드 수 (기본: 10)
  polling-interval: 10s                  # 폴링 주기 (기본: 10s)
  polling-strategy: fetch                # fetch 또는 lock-and-fetch
  polling-strategy-lower-limit-fraction-of-threads: 0.5
  polling-strategy-upper-limit-fraction-of-threads: 3.0
  heartbeat-interval: 5m                 # 하트비트 간격 (기본: 5m)
  missed-heartbeats-limit: 6             # 데드 판단 임계값 (기본: 6)
  table-name: scheduled_tasks            # 테이블명 (기본: scheduled_tasks)
  immediate-execution-enabled: false     # 즉시 실행 활성화
  scheduler-name:                        # 스케줄러 이름 (기본: hostname)
  priority-enabled: false                # 우선순위 기능
  delay-startup-until-context-ready: false  # 컨텍스트 준비까지 대기
  shutdown-max-wait: 30m                 # 종료 시 최대 대기 (기본: 30m)
```

### 7.3 DbSchedulerCustomizer

`application.yml`로 설정 불가능한 항목은 `DbSchedulerCustomizer` Bean으로 설정한다.

```kotlin
@Configuration
class SchedulerConfig {

    @Bean
    fun dbSchedulerCustomizer(): DbSchedulerCustomizer {
        return object : DbSchedulerCustomizer {
            override fun serializer(): Optional<Serializer> {
                return Optional.of(JacksonSerializer())  // JSON 직렬화
            }

            override fun executorService(): Optional<ExecutorService> {
                return Optional.empty()  // 기본 스레드풀 사용
            }
        }
    }
}
```

### 7.4 Task 정의 (Spring Bean)

```kotlin
@Configuration
class TaskConfiguration {

    // RecurringTask: 1시간마다 캔들 수집
    @Bean
    fun collectCandlesTask(): RecurringTask<Void> {
        return Tasks.recurring("collect-candles", FixedDelay.ofHours(1))
            .execute { taskInstance, executionContext ->
                // 캔들 수집 로직
                logger.info("Collecting candles...")
            }
    }

    // RecurringTask: 크론 스케줄
    @Bean
    fun dailyReportTask(): RecurringTask<Void> {
        return Tasks.recurring("daily-report", Cron("0 0 9 * * *"))
            .execute { taskInstance, executionContext ->
                // 일일 리포트 생성
            }
    }

    // OneTimeTask: 주문 실행
    @Bean
    fun executeOrderTask(): OneTimeTask<OrderData> {
        return Tasks.oneTime("execute-order", OrderData::class.java)
            .onFailure(MaxRetriesFailureHandler(3))
            .execute { taskInstance, executionContext ->
                val orderData = taskInstance.data
                // 주문 실행 로직
                logger.info("Executing order: ${orderData.orderId}")
            }
    }
}

// 데이터 클래스 (직렬화 필요)
data class OrderData(
    val orderId: String,
    val symbol: String,
    val amount: BigDecimal,
    val side: String  // BUY or SELL
) : Serializable
```

### 7.5 런타임에서 OneTimeTask 스케줄링

```kotlin
@Service
class OrderService(
    private val scheduler: Scheduler  // 자동 주입
) {
    fun placeOrder(orderId: String, symbol: String, amount: BigDecimal) {
        val orderData = OrderData(orderId, symbol, amount, "BUY")

        // 즉시 실행
        scheduler.schedule(
            Tasks.oneTime("execute-order", OrderData::class.java)
                .instance(orderId, orderData)
                .scheduledTo(Instant.now())
        )
    }

    fun placeDelayedOrder(orderId: String, delay: Duration) {
        val orderData = OrderData(orderId, "KRW-BTC", BigDecimal("100000"), "BUY")

        // 지연 실행
        scheduler.schedule(
            Tasks.oneTime("execute-order", OrderData::class.java)
                .instance(orderId, orderData)
                .scheduledTo(Instant.now().plus(delay))
        )
    }
}
```

### 7.6 트랜잭션과 함께 사용 (Transactionally Staged Job)

```kotlin
@Service
class TransactionalOrderService(
    private val orderRepository: OrderRepository,
    private val schedulerClient: SchedulerClient
) {
    @Transactional
    fun createAndScheduleOrder(request: CreateOrderRequest) {
        // 1. DB에 주문 저장 (같은 트랜잭션)
        val order = orderRepository.save(Order.from(request))

        // 2. 스케줄러에 태스크 등록 (같은 트랜잭션)
        // 트랜잭션이 커밋되어야만 태스크가 실행됨
        schedulerClient.schedule(
            Tasks.oneTime("execute-order", OrderData::class.java)
                .instance(order.id.toString())
                .data(OrderData.from(order))
                .scheduledTo(Instant.now())
        )
    }
}
```

---

## 8. DB 테이블 구조

### PostgreSQL DDL

```sql
CREATE TABLE IF NOT EXISTS scheduled_tasks (
    task_name       TEXT        NOT NULL,
    task_instance   TEXT        NOT NULL,
    task_data       BYTEA       NULL,
    execution_time  TIMESTAMPTZ NOT NULL,
    picked          BOOLEAN     NOT NULL,
    picked_by       TEXT        NULL,
    last_success    TIMESTAMPTZ NULL,
    last_failure    TIMESTAMPTZ NULL,
    consecutive_failures INT    NULL,
    last_heartbeat  TIMESTAMPTZ NULL,
    version         BIGINT      NOT NULL,
    PRIMARY KEY (task_name, task_instance)
);

-- 필수 인덱스
CREATE INDEX IF NOT EXISTS execution_time_idx
    ON scheduled_tasks (execution_time);
CREATE INDEX IF NOT EXISTS last_heartbeat_idx
    ON scheduled_tasks (last_heartbeat);

-- 우선순위 기능 사용 시 (선택)
ALTER TABLE scheduled_tasks ADD COLUMN priority SMALLINT DEFAULT NULL;
```

### 컬럼 설명

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `task_name` | TEXT (PK) | 태스크 식별자 (예: "collect-candles") |
| `task_instance` | TEXT (PK) | 인스턴스 식별자 (RecurringTask는 "DEFAULT") |
| `task_data` | BYTEA | 직렬화된 태스크 데이터 (JSON 등) |
| `execution_time` | TIMESTAMPTZ | 다음 실행 예정 시간 |
| `picked` | BOOLEAN | 현재 실행 중 여부 |
| `picked_by` | TEXT | 실행 중인 스케줄러 인스턴스 이름 |
| `last_success` | TIMESTAMPTZ | 마지막 성공 시간 |
| `last_failure` | TIMESTAMPTZ | 마지막 실패 시간 |
| `consecutive_failures` | INT | 연속 실패 횟수 |
| `last_heartbeat` | TIMESTAMPTZ | 마지막 하트비트 시간 |
| `version` | BIGINT | 낙관적 락용 버전 (CAS) |
| `priority` | SMALLINT | 실행 우선순위 (선택적) |

---

## 9. ShedLock / Quartz와의 비교

### 비교표

| 항목 | db-scheduler | ShedLock | Quartz |
|------|-------------|----------|--------|
| **유형** | 완전한 스케줄러 | 분산 락 라이브러리 | 완전한 스케줄러 |
| **목적** | 영속적 태스크 스케줄링 | `@Scheduled`의 중복 실행 방지 | 엔터프라이즈급 스케줄링 |
| **DB 테이블** | 1개 | 1개 | 11개+ |
| **설정 복잡도** | 낮음 | 매우 낮음 | 높음 |
| **동적 태스크 생성** | O (OneTimeTask) | X | O (API로 가능) |
| **태스크 데이터 전달** | O (task_data) | X | O (JobDataMap) |
| **클러스터링** | DB 레벨 락 | DB/Redis/ZK 등 | DB + 추가 설정 |
| **미스파이어 정책** | 단순 (즉시 실행) | 없음 (락만 관리) | 세밀한 정책 |
| **실패 재시도** | O (내장) | X | O (내장) |
| **의존성** | 최소 (slf4j) | Spring 의존 | 많음 |
| **처리량** | 높음 (2k-10k/s) | 해당 없음 | 중간 |
| **API 복잡도** | 단순 (Builder) | 매우 단순 (@Annotation) | 복잡 (Verbose) |
| **저장소 지원** | RDBMS만 | RDBMS, Redis, Mongo, ZK 등 | RDBMS만 |

### 선택 가이드

```
"@Scheduled로 충분한데 클러스터에서 중복만 방지하고 싶다"
  → ShedLock

"동적 태스크 생성, 재시도, 데이터 전달이 필요하다"
  → db-scheduler

"복잡한 워크플로, 캘린더 기반 스케줄, 세밀한 미스파이어 정책이 필요하다"
  → Quartz

"Quartz가 너무 복잡하고, ShedLock은 기능이 부족하다"
  → db-scheduler (가장 균형잡힌 선택)
```

### 우리 프로젝트(암호화폐 자동매매)에서의 적합성

**db-scheduler가 적합한 이유:**
1. **주문 실행**: OneTimeTask로 동적 주문 생성 및 데이터 전달
2. **캔들 수집**: RecurringTask로 주기적 수집
3. **재시도**: 주문 실패 시 지수 백오프 재시도
4. **트랜잭션 연동**: 주문 생성과 태스크 등록을 같은 트랜잭션으로 보장
5. **분산 환경**: K8s Pod 스케일링 시 중복 실행 자동 방지
6. **단순성**: PostgreSQL만 있으면 동작, 추가 인프라 불필요
7. **Outbox 패턴 대체/보완**: 현재 Outbox 패턴과 함께 사용 가능

---

## 10. 참고 자료

- [GitHub - kagkarlsson/db-scheduler](https://github.com/kagkarlsson/db-scheduler)
- [JavaDoc (최신)](https://javadoc.io/doc/com.github.kagkarlsson/db-scheduler/latest/index.html)
- [Maven Central - Spring Boot Starter](https://mvnrepository.com/artifact/com.github.kagkarlsson/db-scheduler-spring-boot-starter)
- [Transactionally Staged Jobs with db-scheduler (Bekk Blog)](https://blogg.bekk.no/transactionally-staged-jobs-with-db-scheduler-and-spring-7c3a609c132b)
- [db-scheduler-ui (관리 UI)](https://github.com/bekk/db-scheduler-ui)
- [Task Schedulers in Java: Modern Alternatives to Quartz (foojay.io)](https://foojay.io/today/task-schedulers-in-java-modern-alternatives-to-quartz-scheduler/)
