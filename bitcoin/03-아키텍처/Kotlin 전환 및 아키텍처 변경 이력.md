# Kotlin 전환 및 아키텍처 변경 이력

> 작성일: 2026-02-16
> 관련 문서: [[아키텍처 설계서]], [[Spring Modulith Kotlin 이슈 조사]], [[Spring Modulith EventPublication 대안 분석]]

---

## 1. Java → Kotlin 전환

### 전환 일자
2026-02-12

### 전환 범위
- 전체 프로젝트의 모든 Java 소스 → Kotlin 전환 (100%)
- 테스트 코드 포함

### 주요 전환 패턴

| Java | Kotlin | 적용 위치 |
|------|--------|----------|
| `class` + Lombok `@Data` | `data class` | DTO, VO |
| `record` | `data class` | Account, MarketInfo |
| `static` utility class | `object` | QueryHashUtils, RemainingReqParser, MovingAverage |
| `static final Logger` | `companion object { val log = ... }` | 모든 서비스/컴포넌트 |
| `@Slf4j` | `companion object` 로깅 | 모든 로깅 클래스 |
| Java `switch` | Kotlin `when` | Mapper, Strategy |
| `Optional<T>` | 그대로 유지 | Repository 반환 타입 |
| `package-private` | `internal` | JPA Repository |
| Mockito `when(...).thenReturn(...)` | `whenever(...).thenReturn(...)` | mockito-kotlin |
| Java Record (내부 클래스) | `private data class` | WebSocket 메시지 |

### JPA Entity 전환 규칙

```kotlin
// kotlin-jpa 플러그인으로 no-arg + all-open 자동 적용
@Entity
class Market(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long? = null,
    // ...
) {
    // var 프로퍼티에 private set
    var isActive: Boolean = false
        private set
}
```

### 학습 사항

1. **Kotlin boolean `isActive`와 JPQL**: Kotlin `isActive` 프로퍼티는 JPQL에서 `m.isActive`로 참조해야 함 (`m.active` 아님)
2. **JPA + Port 충돌**: `JpaRepository.save()`와 Port 인터페이스의 `save()`가 충돌 → Adapter 패턴으로 해결
3. **WireMock 호환성**: Spring RestClient 테스트에서 `SimpleClientHttpRequestFactory` 사용 필수
4. **JWT Secret Key**: 256-bit (32바이트) 이상 필요

---

## 2. Spring Modulith 제거

### 제거 일자
2026-02-15

### 제거 이유

1. **Gradle 멀티모듈과 중복**: 이미 Gradle `dependencies`로 컴파일 타임 의존성이 제어되고, ArchUnit으로 테스트 타임 검증이 가능
2. **Kotlin 비호환**: Kotlin은 `package-info.java`를 무시하여 `@PackageInfo` 마커 클래스를 별도 생성해야 하는 불편함 (상세: [[Spring Modulith Kotlin 이슈 조사]])
3. **MSA 전환 불가**: `EventPublication`은 단일 프로세스 + 단일 DB 전용, 서비스 분리 시 사용 불가
4. **OPEN 모듈 제약**: `@ApplicationModule(type = Type.OPEN)`으로 설정해도 `@NamedInterface`가 하위 패키지를 노출하지 않는 문제

### 제거 항목

| 항목 | 파일 수 |
|------|--------|
| `ModuleMetadata.kt` 마커 클래스 | 5개 삭제 (core, api-client, strategy, execution, monitoring) |
| `ModulithStructureTest.kt` | 1개 삭제 |
| `spring-modulith-*` 의존성 | 7개 라이브러리 제거 (bom, api, starter-core, starter-jpa, runtime, events-api, starter-test) |
| `gradle/libs.versions.toml` | spring-modulith version + 7개 library 항목 삭제 |

### 대체 방안

| Spring Modulith 기능 | 대체 | 비고 |
|---------------------|------|------|
| 모듈 구조 검증 | Gradle `dependencies` 블록 | 컴파일 타임 강제 |
| 모듈 의존성 테스트 | ArchUnit 6개 테스트 | 더 세밀한 규칙 가능 |
| `EventPublication` | Outbox 패턴 | MSA 전환 대비 |
| `@ApplicationModule` | 없음 (불필요) | Gradle 모듈이 대체 |

---

## 3. Outbox 패턴 도입

### 도입 일자
2026-02-15

### 도입 이유

- Spring Modulith `EventPublication` 제거 후 이벤트 영속화/재시도 기능 대체
- 비즈니스 이벤트의 원자성 보장 (DB 변경 + 이벤트 발행이 같은 트랜잭션)
- MSA 전환 시 Outbox 테이블을 Polling하여 외부 서비스로 전달 가능
- 이벤트 감사 추적 (audit trail)

### 아키텍처

```
┌─────────────────────────────────────────────────────┐
│  Service (같은 트랜잭션)                              │
│    1. 비즈니스 로직 실행                              │
│    2. DomainEventPublisher.publish(event)            │
│       ├─ OutboxEvent INSERT (outbox_events 테이블)   │
│       └─ ApplicationEventPublisher (즉시 전달)       │
│    3. COMMIT                                         │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  OutboxEventProcessor (스케줄러)                      │
│    - 1분 간격: PENDING → PROCESSED                   │
│    - 5분 간격: FAILED 재시도 (최대 3회)                │
│    - MSA 전환 시: HTTP/MQ 전송으로 변경                │
└─────────────────────────────────────────────────────┘
```

### 구현 파일

| 위치 | 파일 | 역할 |
|------|------|------|
| core | `OutboxEvent.kt` | JPA 엔티티 (outbox_events 테이블) |
| core | `OutboxStatus.kt` | 상태 Enum (PENDING, PROCESSED, FAILED) |
| core | `OutboxEventRepository.kt` | Port 인터페이스 |
| core | `DomainEventPublisher.kt` | 이벤트 발행 Port 인터페이스 |
| app | `OutboxJpaRepository.kt` | JPA Repository |
| app | `OutboxRepositoryAdapter.kt` | Repository Adapter 구현 |
| app | `OutboxDomainEventPublisher.kt` | Publisher 구현 (Outbox + Spring Event) |
| app | `OutboxEventProcessor.kt` | 스케줄러 (처리 + 재시도) |
| app | `V2__create_outbox_events.sql` | Flyway 마이그레이션 |

### 이벤트 분류 기준

| 분류 | 사용 인터페이스 | 이벤트 | 기준 |
|------|---------------|--------|------|
| **Outbox 대상** | `DomainEventPublisher` | OrderExecuted, TradeCompleted, PositionOpened/Closed, RiskBreached, TradingSignal, Deposit/Withdrawal | 비즈니스 핵심, 유실 불가 |
| **비-Outbox** | `ApplicationEventPublisher` 직접 | TickerUpdated, MarketDataReceived, SystemError, Alert | 실시간/고빈도/일시적, 유실 허용 |

### outbox_events 테이블 스키마

```sql
CREATE TABLE outbox_events (
    id            BIGSERIAL PRIMARY KEY,
    event_id      VARCHAR(36) NOT NULL UNIQUE,  -- AbstractDomainEvent.eventId
    event_type    VARCHAR(100) NOT NULL,         -- 이벤트 클래스명
    payload       TEXT NOT NULL,                 -- JSON 직렬화
    status        VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at  TIMESTAMP,
    retry_count   INT NOT NULL DEFAULT 0,
    last_error    TEXT
);
```

### MSA 전환 시 변경 지점

현재 (모놀리스):
- `OutboxDomainEventPublisher`가 Outbox INSERT + Spring ApplicationEvent 동시 발행
- `OutboxEventProcessor`는 PENDING → PROCESSED 마킹만 수행

MSA 전환 시:
- `OutboxEventProcessor`에서 PENDING 이벤트를 HTTP/MQ로 외부 서비스에 전송
- 전송 성공 시 PROCESSED, 실패 시 FAILED + retryCount 증가
- `ApplicationEventPublisher` 호출 제거 (외부 서비스가 직접 구독)

---

## 4. 타임라인 요약

| 일자 | 변경 사항 | 커밋 |
|------|----------|------|
| 2026-02-08 | Phase 1: 프로젝트 스켈레톤 (Java) | `1c3e1b4` |
| 2026-02-12 | Java → Kotlin 전환 완료 | `a800f43` |
| 2026-02-12 | Spring Modulith + Kotlin 호환 해결 | `bf56217` |
| 2026-02-15 | Spring Modulith 제거 + Outbox 패턴 적용 | `06c980f` |
| 2026-02-15 | Phase 3-6 (전략/실행/모니터링) 구현 | `ffa06b2` |
