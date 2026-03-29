# Outbox 패턴

> 작성일: 2026-02-22
> 관련 문서: [[아키텍처 설계서]], [[애플리케이션 구동 흐름]], [[Spring Modulith EventPublication 대안 분석]]

---

## 1. 해결하려는 문제: Dual Write

서비스에서 "DB 저장"과 "이벤트 발행"을 동시에 해야 하는 상황이 문제다.

```
주문 생성 로직:
  1. orderRepository.save(order)     ← DB에 쓰기
  2. kafkaTemplate.send(orderEvent)  ← 메시지 브로커에 쓰기
```

이 두 작업은 **서로 다른 시스템에 쓰기(Dual Write)**이며, 하나의 트랜잭션으로 묶을 수 없다.

### 실패 시나리오

```
시나리오 1: DB 저장 성공 → 이벤트 발행 실패
  → 주문은 생겼는데 다른 서비스는 모름 (데이터 불일치)

시나리오 2: DB 저장 실패 → 이벤트는 이미 발행됨
  → 주문은 없는데 알림이 감 (유령 이벤트)

시나리오 3: DB 저장 성공 → 이벤트 발행 직전 서버 크래시
  → 주문은 있는데 이벤트는 영원히 유실
```

핵심: **DB 트랜잭션과 메시지 브로커(Kafka/RabbitMQ)는 하나의 트랜잭션으로 묶을 수 없다.**

---

## 2. Outbox 패턴의 해결 방법

**"이벤트도 같은 DB에 저장하자"**가 핵심 아이디어다.

### 기본 원리

```kotlin
@Transactional  // ← 하나의 DB 트랜잭션
fun createOrder() {
    orderRepository.save(order)           // ① 비즈니스 데이터 저장
    outboxRepository.save(outboxEvent)    // ② 이벤트도 같은 DB에 저장
}
// COMMIT → ①②가 원자적으로 보장됨
```

별도의 프로세스가 outbox 테이블을 폴링하여 이벤트를 외부로 발행한다:

```
┌─ 비즈니스 로직 (트랜잭션) ─┐      ┌─ 별도 프로세스 ─────────────┐
│                            │      │                              │
│  orders 테이블 INSERT      │      │  outbox_events 폴링          │
│  outbox_events INSERT      │      │    → PENDING 이벤트 조회      │
│                            │      │    → Kafka/RabbitMQ 발행      │
│  COMMIT (원자성 보장)       │      │    → PROCESSED로 상태 변경    │
│                            │      │                              │
└────────────────────────────┘      └──────────────────────────────┘
```

### 왜 안전한가

| 상황 | 결과 |
|------|------|
| DB 저장 실패 | outbox도 함께 롤백 → 유령 이벤트 없음 |
| DB 저장 성공 | outbox도 반드시 저장됨 → 이벤트 유실 없음 |
| 서버 크래시 | 폴링 프로세스가 재시작 후 PENDING 이벤트를 다시 처리 |

---

## 3. Outbox 테이블 구조

```sql
CREATE TABLE outbox_events (
    id            BIGSERIAL PRIMARY KEY,
    event_id      VARCHAR(255) NOT NULL,         -- 이벤트 고유 ID (멱등성 키)
    event_type    VARCHAR(255) NOT NULL,         -- "TradeCompletedEvent" 등
    payload       TEXT NOT NULL,                 -- JSON 직렬화된 이벤트 본문
    status        VARCHAR(20) DEFAULT 'PENDING', -- PENDING → PROCESSED / FAILED
    created_at    TIMESTAMP NOT NULL,
    processed_at  TIMESTAMP,
    retry_count   INT DEFAULT 0,
    last_error    TEXT
);
```

상태 전이:

```
PENDING → PROCESSED  (정상 처리)
PENDING → FAILED     (처리 실패)
FAILED  → PROCESSED  (재시도 성공, 최대 3회)
```

---

## 4. 이벤트 발행 방식 비교

### 방식 1: 폴링 (Polling Publisher)

```
스케줄러 (60초마다)
  → SELECT * FROM outbox_events WHERE status = 'PENDING' LIMIT 100
  → 각 이벤트를 메시지 브로커로 발행
  → status = 'PROCESSED'로 변경
```

| 장점 | 단점 |
|------|------|
| 구현 단순 | 폴링 주기만큼 지연 (최대 60초) |
| 추가 인프라 불필요 | DB 부하 (주기적 SELECT) |
| 디버깅 쉬움 | 대량 이벤트 시 병목 |

### 방식 2: CDC (Change Data Capture) — Debezium + Kafka

```
outbox_events 테이블 INSERT
  → Debezium이 DB WAL(Write-Ahead Log) 감지
  → 즉시 Kafka 토픽으로 발행
  → 거의 실시간 (밀리초 단위)
```

| 장점 | 단점 |
|------|------|
| 실시간 (ms 단위) | Debezium + Kafka 인프라 필요 |
| 폴링 부하 없음 | 운영 복잡도 높음 |
| 대량 이벤트에 강함 | 설정/트러블슈팅 난이도 |

### 선택 기준

```
소규모 / 모놀리스 / 시작 단계 → 폴링
대규모 / MSA / 실시간 필수   → CDC (Debezium)
```

---

## 5. 멱등성 (Idempotency)

Outbox 패턴은 **at-least-once delivery** (최소 1회 전달)를 보장한다. 이벤트가 **중복 발행**될 수 있다:

- 폴링 프로세서가 이벤트를 발행한 후 PROCESSED 마킹 전에 크래시
- 재시작 후 같은 이벤트를 다시 발행

따라서 소비자(Consumer)는 반드시 멱등성을 보장해야 한다:

```kotlin
@EventListener
fun onTradeCompleted(event: TradeCompletedEvent) {
    // eventId로 이미 처리한 이벤트인지 확인
    if (alreadyProcessed(event.eventId)) return

    processEvent(event)
    markAsProcessed(event.eventId)
}
```

### 전달 보장 수준 비교

| 수준 | 설명 | 구현 난이도 |
|------|------|------------|
| At-most-once | 최대 1회 전달, 유실 가능 | 쉬움 |
| **At-least-once** | **최소 1회 전달, 중복 가능** | **Outbox 패턴** |
| Exactly-once | 정확히 1회 전달 | 매우 어려움 (사실상 불가능) |

---

## 6. 현재 프로젝트 적용 현황

### 아키텍처 위치

```
core/
├── event/outbox/
│   ├── OutboxEvent          (도메인 모델)
│   └── OutboxStatus         (PENDING / PROCESSED / FAILED)
├── port/outbound/
│   ├── DomainEventPublisher (포트 인터페이스)
│   └── OutboxEventRepository (포트 인터페이스)

app/
└── infrastructure/outbox/
    ├── OutboxEventEntity         (JPA 엔티티)
    ├── OutboxEventEntityMapper   (Entity ↔ Domain 변환)
    ├── OutboxJpaRepository       (Spring Data JPA)
    ├── OutboxRepositoryAdapter   (OutboxEventRepository 구현)
    ├── OutboxDomainEventPublisher (DomainEventPublisher 구현)
    └── OutboxEventProcessor      (폴링 스케줄러)
```

### 동작 방식

현재는 모놀리스이므로 Outbox의 역할이 특수하다:

```
OutboxDomainEventPublisher.publish(event)
  │
  │  @Transactional (같은 트랜잭션)
  │
  ├─ ① outboxEventRepository.save()              ← DB에 이벤트 저장
  └─ ② applicationEventPublisher.publishEvent()   ← 같은 JVM 내 즉시 전달
```

- **②가 실시간 처리**를 담당 (같은 JVM 내 @EventListener로 즉시 전달)
- **①은 감사 추적(audit trail)** + MSA 전환 대비

### 이벤트 분류

#### Outbox 대상 (비즈니스 이벤트 — DomainEventPublisher 사용)

| 이벤트 | 이유 |
|--------|------|
| TradingSignalEvent | 매매 신호 — 유실 시 거래 기회 상실 |
| OrderExecutedEvent | 주문 실행 — 기록 필수 |
| TradeCompletedEvent | 거래 완료 — 포지션 생성의 트리거 |
| RiskBreachedEvent | 리스크 발동 — 강제 매도의 트리거 |
| PositionOpenedEvent | 포지션 개설 — 기록 필수 |
| PositionClosedEvent | 포지션 종료 — 손익 기록 |
| DepositDetectedEvent | 입금 감지 — 알림 필수 |
| WithdrawalDetectedEvent | 출금 감지 — 알림 필수 |

#### 비-Outbox (실시간/시스템 이벤트 — ApplicationEventPublisher 직접 사용)

| 이벤트 | 이유 |
|--------|------|
| TickerUpdatedEvent | 초당 수십 건, 유실 허용 |
| MarketDataReceivedEvent | 정보성, 유실 허용 |
| SystemErrorEvent | 일시적, 유실 허용 |

### 스케줄러 설정

```
process-outbox-events:       60초 간격, PENDING → PROCESSED (배치 100건)
retry-failed-outbox-events:  5분 간격, FAILED 재시도 (최대 3회)
```

---

## 7. MSA 전환 시 변경 사항

현재 모놀리스에서 MSA로 전환할 때, **Port-Adapter 패턴 덕분에 인터페이스 변경 없이 구현체만 교체**하면 된다.

### DomainEventPublisher 구현체 교체

```kotlin
// 현재 (모놀리스) — OutboxDomainEventPublisher
outboxEventRepository.save(outboxEvent)            // DB 저장
applicationEventPublisher.publishEvent(event)       // 같은 JVM 즉시 전달

// MSA 전환 후 — 방법 A: Kafka 직접 발행
outboxEventRepository.save(outboxEvent)            // DB 저장 (감사 추적)
kafkaTemplate.send("domain.events", event)          // Kafka로 외부 발행

// MSA 전환 후 — 방법 B: CDC (Debezium)
outboxEventRepository.save(outboxEvent)            // DB 저장만
// Debezium이 WAL 감지 → 자동으로 Kafka 발행
```

### 소비자 변경

```kotlin
// 현재 (모놀리스)
@EventListener
fun onTradeCompleted(event: TradeCompletedEvent) { ... }

// MSA 전환 후
@KafkaListener(topics = ["trading.trade-completed"])
fun onTradeCompleted(event: TradeCompletedEvent) { ... }
```

### 전환 단계 요약

```
1단계: 현재 (모놀리스 + Outbox audit trail)
  └─ ApplicationEventPublisher로 즉시 전달

2단계: 메시지 브로커 도입 (Kafka)
  └─ Outbox → 폴링 → Kafka 발행
  └─ 소비자: @KafkaListener로 변경

3단계: CDC 도입 (선택)
  └─ Debezium이 outbox 테이블 변경 감지 → Kafka 자동 발행
  └─ 폴링 프로세서 제거
```

---

## 8. 관련 패턴 비교

| 패턴 | 방식 | 장점 | 단점 |
|------|------|------|------|
| **Outbox** | 이벤트를 같은 DB에 저장 후 별도 발행 | 원자성 보장, 구현 단순 | 지연 발생 (폴링), DB 부하 |
| **Event Sourcing** | 상태 대신 이벤트 자체를 저장 | 완벽한 이력, 이벤트가 곧 데이터 | 학습 곡선, 복잡도 높음 |
| **Saga** | 분산 트랜잭션을 보상 트랜잭션으로 관리 | 서비스 간 일관성 | 보상 로직 복잡 |
| **2PC (Two-Phase Commit)** | 분산 트랜잭션 코디네이터 | 강한 일관성 | 성능 저하, 가용성 낮음 |

---

## 9. 핵심 정리

| 항목 | 내용 |
|------|------|
| **문제** | DB 저장과 이벤트 발행의 원자성 불가 (Dual Write) |
| **해결** | 이벤트를 같은 DB 트랜잭션으로 저장, 별도 프로세스가 발행 |
| **보장 수준** | at-least-once delivery (최소 1회 전달, 중복 가능) |
| **소비자 책임** | 멱등성 처리 (eventId로 중복 감지) |
| **발행 방식** | 폴링 (단순) 또는 CDC/Debezium (실시간) |
| **현재 프로젝트** | 모놀리스 — 즉시 전달 + audit trail 용도 |
| **MSA 전환 시** | 구현체만 교체 (Port-Adapter 패턴) |
