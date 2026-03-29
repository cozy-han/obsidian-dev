# Spring Modulith EventPublication 대안 분석

## 조사 배경
- **프로젝트**: Bitcoin Auto Trading System
- **기술 스택**: Kotlin, Spring Boot 3.4.x, Gradle 멀티모듈
- **현재 상황**: Spring ApplicationEvent 사용 중 (`ApplicationEventPublisher.publishEvent()`)
- **목표**: Event 처리의 신뢰성 보장 (트랜잭션 경계, 실패 복구, 완료 추적)

---

## 1. Spring Modulith EventPublication 핵심 기능

### 1.1 제공하는 기능
Spring Modulith EventPublication은 **트랜잭셔널 아웃박스 패턴**을 자동으로 구현한 기능입니다.

#### 핵심 메커니즘
```kotlin
// 1. 이벤트 발행 (트랜잭션 내부)
@Transactional
fun createOrder() {
    orderRepository.save(order)
    // 이벤트 발행
    eventPublisher.publishEvent(OrderCreatedEvent(order))
    // ⬇ DB에 event_publication 테이블에 자동 저장
}

// 2. 이벤트 리스너 (트랜잭션 완료 후 비동기 처리)
@ApplicationModuleListener
fun onOrderCreated(event: OrderCreatedEvent) {
    // 처리 성공 시 → event_publication에서 completed 표시
    // 처리 실패 시 → event_publication에 남아서 재시도 대상
}
```

#### 제공 기능 상세

| 기능             | 설명                                                |
| -------------- | ------------------------------------------------- |
| **트랜잭셔널 아웃박스** | 이벤트를 `event_publication` 테이블에 저장. 발행 트랜잭션과 원자성 보장 |
| **자동 완료 추적**   | 리스너가 성공하면 `completed_at` 타임스탬프 기록                 |
| **재시도 메커니즘**   | 미완료 이벤트를 주기적으로 재처리 (`@Scheduled` 방식)              |
| **모듈 간 경계 보장** | `@ApplicationModuleListener`로 모듈 간 이벤트만 필터링       |
| **테스트 지원**     | `PublishedEvents` API로 이벤트 발행 검증 가능               |

#### 자동 생성 테이블
```sql
CREATE TABLE event_publication (
    id UUID PRIMARY KEY,
    completion_date TIMESTAMP,
    event_type VARCHAR(255),
    listener_id VARCHAR(255),
    publication_date TIMESTAMP,
    serialized_event TEXT
);
```

### 1.2 장점
- ✅ **Zero Config**: `spring-modulith-starter-jpa` 의존성만 추가하면 자동 활성화
- ✅ **트랜잭션 안정성**: DB 트랜잭션과 이벤트 발행이 원자적으로 처리
- ✅ **자동 재시도**: 실패한 이벤트를 백그라운드에서 재처리
- ✅ **완료 추적**: 어떤 이벤트가 처리되었는지 명확히 확인 가능
- ✅ **Spring 네이티브**: Spring 생태계와 완벽히 통합 (Spring Boot, Spring Data JPA)

### 1.3 단점
- ❌ **Polling 기반 재시도**: 실시간성이 떨어질 수 있음 (기본 5분 간격)
- ❌ **JPA 종속**: JDBC 직접 사용 시 지원 안됨
- ❌ **모듈 경계 강제**: 모든 이벤트가 아닌 모듈 간 이벤트만 추적
- ❌ **커스터마이징 제한**: 재시도 정책, 저장 포맷 등 세밀한 제어 어려움
- ❌ **스토리지 증가**: 모든 이벤트를 DB에 저장하므로 정기적 삭제 필요

---

## 2. 대안 1: Spring 기본 `@TransactionalEventListener` + `@Async`

### 2.1 구현 방법
```kotlin
// 이벤트 발행 (기존과 동일)
@Transactional
fun createOrder() {
    orderRepository.save(order)
    eventPublisher.publishEvent(OrderCreatedEvent(order))
}

// 트랜잭션 커밋 후 비동기 처리
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
fun onOrderCreated(event: OrderCreatedEvent) {
    // 비동기 처리
    notificationService.sendEmail(event)
}
```

### 2.2 장단점

| 장점 | 단점 |
|------|------|
| ✅ Spring 기본 기능 (추가 의존성 불필요) | ❌ **실패 시 재시도 없음** (이벤트 소실 위험) |
| ✅ 실시간 처리 (비동기 즉시 실행) | ❌ **완료 추적 불가** (처리 여부 확인 불가) |
| ✅ 간단한 설정 (`@EnableAsync`만 추가) | ❌ 트랜잭션 롤백 시 이벤트 미발행 (원자성 부족) |
| ✅ 경량 (DB 저장 오버헤드 없음) | ❌ 애플리케이션 재시작 시 처리 중 이벤트 소실 |

### 2.3 적합한 상황
- 이벤트 소실이 허용되는 경우 (로그, 알림 등 비핵심 작업)
- 실시간성이 중요한 경우
- 간단한 이벤트 처리 (재시도 로직 불필요)

### 2.4 구현 난이도
⭐ (매우 쉬움)

---

## 3. 대안 2: 직접 Outbox 패턴 구현

### 3.1 구현 방법
```kotlin
// 1. Outbox 테이블 정의
@Entity
@Table(name = "outbox_events")
class OutboxEvent(
    @Id
    val id: UUID = UUID.randomUUID(),
    val eventType: String,
    val payload: String, // JSON 직렬화
    val createdAt: LocalDateTime = LocalDateTime.now(),
    var processedAt: LocalDateTime? = null,
    var retryCount: Int = 0,
    var status: EventStatus = EventStatus.PENDING
)

enum class EventStatus { PENDING, PROCESSING, COMPLETED, FAILED }

// 2. 이벤트 발행 시 Outbox에 저장
@Transactional
fun createOrder() {
    orderRepository.save(order)

    // Outbox에 저장 (같은 트랜잭션)
    val event = OrderCreatedEvent(order)
    outboxRepository.save(OutboxEvent(
        eventType = event::class.simpleName!!,
        payload = objectMapper.writeValueAsString(event)
    ))
}

// 3. 백그라운드 처리 스케줄러
@Scheduled(fixedDelay = 5000) // 5초마다
@Transactional
fun processOutboxEvents() {
    val pendingEvents = outboxRepository.findByStatus(EventStatus.PENDING)

    for (outboxEvent in pendingEvents) {
        try {
            // 이벤트 역직렬화
            val event = deserializeEvent(outboxEvent)

            // 실제 리스너 호출
            eventPublisher.publishEvent(event)

            // 완료 처리
            outboxEvent.status = EventStatus.COMPLETED
            outboxEvent.processedAt = LocalDateTime.now()
            outboxRepository.save(outboxEvent)

        } catch (e: Exception) {
            // 재시도 로직
            outboxEvent.retryCount++
            if (outboxEvent.retryCount >= 3) {
                outboxEvent.status = EventStatus.FAILED
            }
            outboxRepository.save(outboxEvent)
        }
    }
}

// 4. 이벤트 리스너 (일반 동기 처리)
@EventListener
fun onOrderCreated(event: OrderCreatedEvent) {
    // 실제 비즈니스 로직
}
```

### 3.2 장단점

| 장점                                                          | 단점                             |
| ----------------------------------------------------------- | ------------------------------ |
| ✅ **완전한 제어**: 재시도 정책, 저장 포맷, 스케줄링 모두 커스터마이징                 | ❌ **구현 복잡도 높음** (100+ 줄 코드 필요) |
| ✅ **유연한 재시도**: Exponential backoff, Circuit breaker 등 적용 가능 | ❌ 직렬화/역직렬화 로직 직접 구현            |
| ✅ **모니터링 용이**: 실패 이벤트 대시보드, 알림 등 커스텀 가능                     | ❌ 유지보수 부담 (버그, 성능 최적화)         |
| ✅ **배치 처리**: 대량 이벤트를 한번에 처리 가능                              | ❌ 테스트 복잡도 증가                   |

### 3.3 적합한 상황
- 재시도 정책을 세밀하게 제어해야 하는 경우
- 이벤트 처리 실패를 모니터링/알림해야 하는 경우
- Spring Modulith를 사용하지 않는 경우

### 3.4 구현 난이도
⭐⭐⭐⭐ (높음)

---

## 4. 대안 3: Debezium (CDC 기반)

### 4.1 구현 방법
Debezium은 데이터베이스의 **변경 로그(WAL, binlog)를 읽어** 이벤트로 변환하는 CDC(Change Data Capture) 도구입니다.

```yaml
# Docker Compose 예시
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: trading
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    command: ["postgres", "-c", "wal_level=logical"]

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0

  kafka:
    image: confluentinc/cp-kafka:7.5.0

  debezium:
    image: debezium/connect:2.5
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
```

```kotlin
// 1. DB 변경 (Outbox 테이블에 저장)
@Transactional
fun createOrder() {
    orderRepository.save(order)

    // Outbox 테이블에 이벤트 저장
    outboxRepository.save(OutboxEvent(
        aggregateType = "Order",
        aggregateId = order.id,
        eventType = "OrderCreated",
        payload = objectMapper.writeValueAsString(order)
    ))
}

// 2. Debezium이 outbox 테이블 변경 감지 → Kafka로 발행

// 3. Kafka Consumer가 이벤트 소비
@KafkaListener(topics = ["order.events"])
fun consumeOrderEvent(event: OrderEvent) {
    when (event.type) {
        "OrderCreated" -> handleOrderCreated(event)
        else -> log.warn("Unknown event type: {}", event.type)
    }
}
```

### 4.2 장단점

| 장점 | 단점 |
|------|------|
| ✅ **At-least-once 보장**: Kafka의 offset 관리로 이벤트 소실 없음 | ❌ **인프라 복잡도 매우 높음** (Kafka, Zookeeper, Debezium Connect) |
| ✅ **확장성**: Kafka 기반으로 대용량 이벤트 처리 가능 | ❌ 운영 부담 증가 (Kafka 클러스터 관리, 모니터링) |
| ✅ **이벤트 순서 보장**: Partition 내에서 순서 유지 | ❌ 로컬 개발 환경 설정 복잡 |
| ✅ **재처리 가능**: Kafka offset을 되돌려 재처리 | ❌ 오버엔지니어링 위험 (중소 규모 프로젝트에는 과도) |
| ✅ **마이크로서비스 친화적**: 다른 서비스와 이벤트 공유 | ❌ 비용 증가 (Kafka 인프라 비용) |

### 4.3 적합한 상황
- 마이크로서비스 아키텍처 (여러 서비스 간 이벤트 공유)
- 대용량 이벤트 처리 (초당 1000+ 이벤트)
- 이벤트 히스토리 보존 필요 (Kafka retention policy)
- 이미 Kafka 인프라가 구축된 경우

### 4.4 구현 난이도
⭐⭐⭐⭐⭐ (매우 높음)

---

## 5. 대안 4: 기타 라이브러리

### 5.1 Axon Framework
- **특징**: CQRS/Event Sourcing 전용 프레임워크
- **장점**: Event Store, Saga 패턴 기본 제공
- **단점**: 러닝 커브 높음, 프레임워크 종속성 강함
- **적합성**: 🔴 Bitcoin 프로젝트에는 오버엔지니어링

### 5.2 Spring Cloud Stream
- **특징**: 메시지 브로커 추상화 (Kafka, RabbitMQ)
- **장점**: 다양한 브로커 지원, Binder 개념으로 느슨한 결합
- **단점**: 외부 메시지 브로커 필수
- **적합성**: 🟡 멀티모듈 모노리스에는 과도할 수 있음

### 5.3 Eventuate Tram
- **특징**: 트랜잭셔널 아웃박스 패턴 전용 라이브러리
- **장점**: Spring Modulith보다 성숙, CDC 지원
- **단점**: 별도 인프라 필요 (Eventuate CDC 서비스)
- **적합성**: 🟡 CDC 필요 시 고려 가능

---

## 6. 비교 요약표

| 대안 | 트랜잭션 보장 | 재시도 | 완료 추적 | 실시간성 | 구현 난이도 | 인프라 복잡도 |
|------|-------------|--------|----------|---------|-----------|-------------|
| **Spring Modulith EventPublication** | ✅ | ✅ (Polling) | ✅ | 🟡 (지연 5분) | ⭐ | ⭐ |
| **@TransactionalEventListener + @Async** | 🟡 | ❌ | ❌ | ✅ | ⭐ | ⭐ |
| **직접 Outbox 구현** | ✅ | ✅ (커스텀) | ✅ | 🟡 (설정 가능) | ⭐⭐⭐⭐ | ⭐ |
| **Debezium + Kafka** | ✅ | ✅ | ✅ | ✅ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## 7. Bitcoin 프로젝트 추천 방안

### 현재 상황 분석
```kotlin
// 현재 코드 (api-client/UpbitWebSocketClient.kt)
eventPublisher.publishEvent(TickerUpdatedEvent(ticker))
eventPublisher.publishEvent(SystemErrorEvent(...))
```

- ✅ 이미 Spring ApplicationEvent 사용 중
- ✅ Spring Modulith dependency 이미 포함 (`spring-modulith-starter-jpa`)
- 🔴 현재는 트랜잭션 보장 없음 (이벤트 소실 위험)

### 추천 순위

#### 1위: **Spring Modulith EventPublication** (⭐⭐⭐⭐⭐)
**추천 이유**
- 이미 의존성 포함되어 있음 (`spring-modulith-starter-jpa`)
- 코드 변경 최소화 (`@ApplicationModuleListener`만 추가)
- 트랜잭션 보장 + 재시도 + 완료 추적 모두 제공
- 멀티모듈 모노리스에 최적화된 설계

**적용 방법**
```kotlin
// 1. 이벤트 리스너 변경
// Before
@EventListener
fun onTickerUpdated(event: TickerUpdatedEvent) { ... }

// After
@ApplicationModuleListener
fun onTickerUpdated(event: TickerUpdatedEvent) { ... }

// 2. application.yml 설정 (선택사항)
spring:
  modulith:
    republish-outstanding-events-on-restart: true
    delete-completed-events-after: 7d
```

**주의사항**
- 모든 이벤트가 DB에 저장되므로 정기적으로 완료된 이벤트 삭제 필요
- 실시간성이 중요한 이벤트(Ticker 업데이트)는 `@Async`와 병행 고려

---

#### 2위: **하이브리드 (Modulith + @Async)** (⭐⭐⭐⭐)
**추천 이유**
- 핵심 이벤트는 Modulith로 신뢰성 보장
- 실시간 이벤트는 `@Async`로 즉시 처리

**적용 예시**
```kotlin
// 핵심 비즈니스 이벤트 → Modulith
@ApplicationModuleListener
fun onOrderExecuted(event: OrderExecutedEvent) {
    // 주문 완료 처리 (반드시 성공해야 함)
}

// 실시간 알림 → @Async
@Async
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
fun onTickerUpdated(event: TickerUpdatedEvent) {
    // 실시간 시세 업데이트 (소실 허용)
}
```

---

#### 3위: **직접 Outbox 구현** (⭐⭐)
**비추천 이유**
- Spring Modulith이 이미 잘 작동하는 상황에서 직접 구현은 비효율적
- 유지보수 부담만 증가

**고려할 상황**
- Spring Modulith의 재시도 정책을 커스터마이징해야 하는 경우
- 이벤트 처리 실패를 Discord로 실시간 알림해야 하는 경우

---

## 8. 실무 적용 가이드

### Phase 1: Spring Modulith 도입 (1-2일)
```kotlin
// 1. 핵심 이벤트 리스너에 @ApplicationModuleListener 추가
@ApplicationModuleListener
fun onOrderExecuted(event: OrderExecutedEvent) { ... }

@ApplicationModuleListener
fun onRiskBreached(event: RiskBreachedEvent) { ... }

// 2. application.yml 설정
spring:
  modulith:
    republish-outstanding-events-on-restart: true
    delete-completed-events-after: 7d
```

### Phase 2: 모니터링 추가 (1일)
```kotlin
@Component
class EventPublicationMonitor(
    private val repository: EventPublicationRepository
) {
    @Scheduled(cron = "0 0 * * * *") // 매시간
    fun checkPendingEvents() {
        val pending = repository.findIncompletePublications()
        if (pending.isNotEmpty()) {
            log.warn("Pending events: {}", pending.size)
            // Discord 알림 발송
        }
    }
}
```

### Phase 3: 정리 작업 (지속적)
```kotlin
@Scheduled(cron = "0 0 3 * * *") // 매일 새벽 3시
@Transactional
fun cleanupCompletedEvents() {
    val weekAgo = LocalDateTime.now().minusDays(7)
    val deleted = repository.deleteCompletedPublicationsBefore(weekAgo)
    log.info("Deleted {} completed events", deleted)
}
```

---

## 9. 결론

### Bitcoin 프로젝트에 가장 적합한 방안
**Spring Modulith EventPublication (+ 선택적으로 @Async 병행)**

### 핵심 근거
1. ✅ **이미 의존성 포함** → 추가 설정 불필요
2. ✅ **트랜잭션 안정성** → 주문/리스크 이벤트 소실 방지
3. ✅ **간단한 도입** → 리스너 어노테이션만 변경
4. ✅ **멀티모듈 모노리스 최적화** → 모듈 간 경계 자동 관리

### 도입 시 체크리스트
- [ ] `@ApplicationModuleListener` 어노테이션 추가
- [ ] `event_publication` 테이블 자동 생성 확인
- [ ] 완료된 이벤트 정기 삭제 스케줄러 구현
- [ ] 미처리 이벤트 모니터링 추가
- [ ] 실시간 이벤트는 `@Async` 병행 고려

### 추후 고려사항
- 마이크로서비스 전환 시 → Debezium + Kafka 검토
- 대용량 이벤트 발생 시 → 배치 처리 최적화

---

**문서 작성일**: 2026-02-15
**프로젝트**: Bitcoin Auto Trading System
**버전**: v0.1.0-SNAPSHOT
