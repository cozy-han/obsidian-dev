# Bitcoin Auto Trader — TODO 리스트

> 작성일: 2026-02-08
> 최종 갱신: 2026-03-25 (2차)
> 관련 문서: [[요구사항 정의서]], [[아키텍처 설계서]], [[자동매매 시스템 설계 질문]]

---

## Phase 1: 프로젝트 스켈레톤 + Core 도메인 + DB ✅

### 빌드 설정
- [x] Gradle 멀티모듈 구조 (settings.gradle.kts, build.gradle.kts)
- [x] 버전 카탈로그 (gradle/libs.versions.toml)
- [x] 모듈별 build.gradle.kts (core, api-client, strategy, execution, monitoring, app)
- [x] Gradle Wrapper (8.12.1)
- [x] .gitignore

### Core 도메인 모델
- [x] Value Object: Money, Percentage, MarketCode
- [x] Entity: Market, Candle, Order, Trade, Position
- [x] Record: Account, MarketInfo
- [x] Enum: CandleType, OrderSide, OrderType, OrderState, PositionStatus, TradingSignal

### 이벤트
- [x] AbstractDomainEvent (base)
- [x] Market: TickerUpdatedEvent, MarketDataReceivedEvent
- [x] Trading: TradingSignalEvent, OrderExecutedEvent, TradeCompletedEvent
- [x] Risk: RiskBreachedEvent
- [x] Position: PositionOpenedEvent, PositionClosedEvent
- [x] Account: DepositDetectedEvent, WithdrawalDetectedEvent
- [x] System: SystemErrorEvent, AlertEvent

### 포트 인터페이스
- [x] ExchangeClient, CandleRepository, OrderRepository, NotificationSender
- [x] DomainEventPublisher, OutboxEventRepository (Outbox 패턴)
- [x] PositionRepository, TradeRepository (Phase 4에서 추가)

### 예외
- [x] DomainException, InsufficientFundsException, OrderExecutionException, RiskLimitExceededException, ExchangeApiException

### App 모듈
- [x] BitcoinAutoTraderApplication.kt (@SpringBootApplication)
- [x] application.yml (공통/dev/prod/test)
- [x] SpringDoc OpenAPI 설정 (Swagger UI)

### DB
- [x] docker-compose.yml (PostgreSQL 16)
- [x] V1__create_tables.sql (12개 테이블 + 파티셔닝 + 인덱스)
- [x] V2__create_outbox_events.sql (Outbox 패턴 테이블 + event_publication 제거)

### 아키텍처 변경
- [x] Spring Modulith 제거 (Gradle 멀티모듈 + ArchUnit이 모듈 격리 담당)
- [x] Outbox 패턴 적용 (OutboxEvent 엔티티, OutboxDomainEventPublisher, OutboxEventProcessor)

### 테스트 (3개 클래스, 17개 테스트, 100% 통과)
- [x] ArchitectureTest — ArchUnit 6개 (core 의존성, monitoring 의존성, strategy 의존성, execution 의존성, 도메인→인프라, 커스텀 예외)
- [x] MoneyTest — 6개
- [x] MarketCodeTest — 5개

### 미해결
- [ ] Flyway 마이그레이션 실행 확인 (앱 시작 시 테이블 자동 생성 검증)
- [ ] `/actuator/health` 200 OK 확인

---

## Phase 2: Upbit API Client + 캔들 수집 ✅

> 관련 요구사항: FR-01 (마켓 스캔), FR-03 (시세 데이터 수집), FR-09 (계좌 조회)
> 모듈: `api-client`

### JWT 인증 (FR-09)
- [x] UpbitApiProperties — API Key/Secret 설정 관리
- [x] UpbitJwtTokenGenerator — JWT 토큰 생성 (HS256, query_hash 포함)
- [x] 인증 헤더 자동 생성 (Authorization: Bearer {JWT})

### REST Client
- [x] RestClientConfig — Spring RestClient 설정
- [x] UpbitRestClient — ExchangeClient 인터페이스 구현
  - [x] `GET /market/all` — KRW 마켓 종목 목록 조회 (FR-01-01)
  - [x] `GET /candles/days` — 일봉 캔들 조회 (FR-03-01)
  - [x] `GET /candles/minutes/{unit}` — 시간봉/분봉 캔들 조회 (FR-03-02, FR-03-03)
  - [x] `GET /ticker` — 현재가 조회
  - [x] `GET /accounts` — 계좌 잔고 조회 (FR-09-01)
  - [x] `POST /orders` — 주문 실행 (FR-04-01, FR-04-02)
  - [x] `GET /order` — 주문 조회 (FR-04-04)
  - [x] `DELETE /order` — 주문 취소
  - [ ] `GET /deposits` — 입금 내역 조회 *(제외: 입출금 알림으로 대체)*
  - [ ] `GET /withdraws` — 출금 내역 조회 *(제외: 입출금 알림으로 대체)*

### DTO + Mapper
- [x] CandleResponse, TickerResponse, MarketResponse, OrderResponse, AccountResponse
- [x] CandleResponseMapper, TickerResponseMapper, MarketResponseMapper, OrderResponseMapper, AccountResponseMapper

### Rate Limiter (FR-04-06, NFR-02-05)
- [x] ResilienceConfig — Resilience4j 설정
- [x] RateLimiter 3종 — Quotation 10/sec(IP), Exchange 30/sec(계정), Order 8/sec(계정)
- [x] RemainingReqParser — Remaining-Req 응답 헤더 파싱
- [x] UpbitApiErrorHandler — 429/418 에러 핸들링

### Circuit Breaker (NFR-02-05)
- [x] CircuitBreakerConfig — Resilience4j 설정 (ResilienceConfig 내)
- [x] API 장애 시 안전 모드 전환 (CircuitBreaker OPEN → 요청 차단)
- [x] SystemErrorEvent 발행 (OPEN 전환 시)

### 캔들 수집 스케줄러 (FR-03-01, FR-03-02)
- [x] CandleCollectionService — 캔들 수집 로직
  - [x] 일봉 수집 스케줄러 (매일 KST 00:05) — **전체 KRW 마켓 대상**
  - [x] 시간봉 수집 스케줄러 (매시간 :05분) — 활성 마켓만
  - [x] 최초 실행 시 과거 데이터 백필 (최대 200개씩)
  - [x] 중복 저장 방지 (existsBy... 체크)
  - [x] collectAllMarketCandles / backfillAllMarketCandles — 전체 마켓 수집 메서드
- [x] MarketDataReceivedEvent 발행
- [x] CandleJpaRepository + CandleRepositoryAdapter — CandleRepository 구현

### 거래량 기반 마켓 선정 (FR-01-02)
- [x] MarketSelectionService — 거래대금 기반 자동 마켓 선정
  - [x] 최근 N일 평균 거래대금 상위 N개 활성화 (매일 00:10 KST)
  - [x] 최소 거래대금 임계값 필터링
  - [x] MarketSelectionChangedEvent 발행
- [x] MarketSelectionProperties — @ConfigurationProperties (topN, minTradePriceKrw, averageDays)
- [x] CandleRepository.findByCandleTypeAndDateRange — 날짜 범위 전체 캔들 조회
- [x] MarketSelectionServiceTest — 7개 테스트

### 마켓 동기화 (FR-01-01)
- [x] MarketSyncService — Upbit 종목 목록 동기화 (@PostConstruct + @Scheduled)
- [x] MarketJpaRepository + MarketRepositoryAdapter — Market 저장소 구현

### 계좌 조회 (FR-09-01, FR-09-02, FR-09-03)
- [x] AccountQueryService — 계좌/자산 조회

### WebSocket Client (FR-03-04, NFR-01-01)
- [x] WebSocketConfig — WebSocket 설정
- [x] UpbitWebSocketClient — Ticker 실시간 수신
  - [x] 구독 요청 포맷 구성 (ticket, type, codes)
  - [x] Ticker 실시간 수신 → TickerUpdatedEvent 발행
  - [ ] MyOrder 실시간 수신 (인증 WebSocket) *(Phase 4에서 구현)*
- [x] WebSocketReconnectManager — 자동 재연결 (NFR-02-02)
  - [x] 지수 백오프 재연결 (1s → 2s → 4s → ... → 300s cap, 최대 10회)
  - [x] 연결 끊김 시 SystemErrorEvent 발행

### 캐시 설정
- [x] Caffeine Cache 적용 (마켓 목록 10분 TTL, 현재가 5초 TTL)

### 테스트 (12개 테스트 클래스, 64개 테스트, 100% 통과)
- [x] WireMock 통합 테스트 (UpbitRestClientTest — 9개 테스트)
- [x] WebSocket 재연결 테스트 (WebSocketReconnectManagerTest — 5개 테스트)
- [x] RemainingReqParser 단위 테스트 (5개 테스트)
- [x] JWT 토큰 생성 테스트 (UpbitJwtTokenGeneratorTest — 5개, QueryHashUtilsTest — 6개)
- [x] Mapper 테스트 (CandleResponseMapperTest — 3개, MarketResponseMapperTest — 3개)
- [x] Service 테스트 (CandleCollectionServiceTest — 7개, MarketSyncServiceTest — 4개, MarketSelectionServiceTest — 7개)
- [x] Repository 테스트 (CandleJpaRepositoryTest — 5개, MarketJpaRepositoryTest — 5개, Testcontainers)

---

## Phase 3: 전략 엔진 + 백테스팅 ✅

> 관련 요구사항: FR-02 (트레이딩 전략), FR-07 (백테스팅)
> 모듈: `strategy`

### 전략 인터페이스 (FR-02-02)
- [x] TradingStrategy 인터페이스 — `analyze(marketCode, candles) → StrategyResult`
- [x] StrategyResult — signal(BUY/SELL/HOLD) + confidence + reason
- [x] StrategyConfig JPA 엔티티 — 전략 설정 (strategy_configs 테이블, JSONB 파라미터)

### 기술적 지표
- [x] MovingAverage — SMA (단순이동평균) 계산, smaList
- [x] RSI — Relative Strength Index 계산 (Wilder 평활화)
- [x] EMA — 지수이동평균 (MACD 내부에서 사용)
- [x] BollingerBands — 볼린저밴드 (%B, 밴드폭)
- [x] MACD — MACD Line, Signal Line, Histogram

### 기본 전략 (FR-02-01)
- [x] MovingAverageCrossoverStrategy — 이동평균 크로스오버
  - [x] 골든크로스 (단기MA > 장기MA 교차) → BUY
  - [x] 데드크로스 (단기MA < 장기MA 교차) → SELL
  - [x] 단기/장기 기간 파라미터 설정 (기본 5/20)
  - [x] confidence 계산 (교차 강도 기반)

### 복합 전략 (FR-02-03, FR-02-04)
- [x] CompositeStrategy — 복합 전략 컨테이너
- [x] CombineMode Enum — AND / OR / WEIGHTED
  - [x] AND: 전체 일치 시 신호 발생
  - [x] OR: 하나라도 신호 시 발생 (충돌 시 HOLD)
  - [x] WEIGHTED: 가중치 합산 + 임계값(0.3) 초과 시 신호

### 전략 관리 (FR-02-05, FR-02-06)
- [x] StrategyFactory — 전략 타입 → 인스턴스 생성 (JSONB 파라미터 파싱)
- [x] StrategyService — 활성 전략으로 마켓 분석
- [x] StrategyConfigJpaRepository — 전략 설정 저장소

### 전략 평가 (이벤트 연동) ✅
- [x] StrategyEvaluationService (StrategyAnalysisListener 대체)
  - [x] TickerUpdatedEvent 수신 → 전략 평가 (쓰로틀링, 중복 신호 방지)
  - [x] MarketDataReceivedEvent 수신 → 전략 재평가
  - [x] TradingSignalEvent 발행 (BUY/SELL/HOLD + 신뢰도)
  - [x] StrategyEvaluationConfig — 설정 프로퍼티 (평가 간격, 캔들 타입, 활성화 여부)

### 백테스팅 엔진 (FR-07-01 ~ FR-07-06)
- [x] BacktestEngine — 시뮬레이션 엔진
  - [x] 과거 캔들 데이터 기반 전략 실행 (FR-07-01)
  - [x] 기간/종목/전략 지정 시뮬레이션 (FR-07-02)
  - [x] 손절/익절 자동 적용
  - [x] 수수료(feeRate) 반영
  - [x] 미청산 포지션 종가 정리
- [x] BacktestRequest — 백테스팅 요청 파라미터
- [x] BacktestResult — 성과 지표 포함 결과
- [x] BacktestTradeRecord — 개별 거래 기록

### 성과 지표 (FR-07-04)
- [x] 총 수익률 (totalReturn %)
- [x] 최대 낙폭 MDD (maxDrawdown %)
- [x] 승률 (winRate %)
- [x] 총 거래 횟수, 승/패 횟수
- [x] 자산 곡선 (EquityPoint 리스트)

### 결과 저장 (FR-07-06)
- [x] BacktestResultEntity — JPA 엔티티 (backtest_results 테이블)
- [x] BacktestResultJpaRepository — 결과 저장/조회
- [x] BacktestService — 백테스팅 실행 + 결과 저장 서비스

### API + UI (FR-07-03)
- [x] BacktestApiController — REST API
  - [x] `POST /api/backtest/run` — 백테스팅 실행 + 결과 저장
  - [x] `GET /api/backtest/results` — 전체 결과 목록
  - [x] `GET /api/backtest/results/{id}` — 개별 결과 조회
- [x] BacktestViewController — Thymeleaf 페이지 (`GET /backtest`)
- [x] backtest.html — 백테스팅 UI
  - [x] 설정 폼 (마켓/전략/기간/투자금/손절/익절/MA기간)
  - [x] Chart.js 자산곡선 (Equity Curve)
  - [x] Chart.js 도넛 차트 (Win/Loss 분포)
  - [x] 성과 지표 카드 (수익률, MDD, 승률, 거래수)
  - [x] 거래 내역 테이블
  - [x] 과거 백테스팅 결과 히스토리

### 테스트 (4개 클래스, 21개 테스트, 100% 통과)
- [x] MovingAverageTest — SMA 계산 4개
- [x] MovingAverageCrossoverStrategyTest — 골든/데드크로스, HOLD, 에러 5개
- [x] CompositeStrategyTest — AND/OR/WEIGHTED 모드 7개
- [x] BacktestEngineTest — 수익, 빈 캔들, 손절, 익절, MDD 5개

---

## Phase 4: 주문 실행 + 리스크 관리 ✅

> 관련 요구사항: FR-04 (주문 실행), FR-06 (리스크 관리)
> 모듈: `execution`

### 설정 (FR-06-08)
- [x] RiskConfig — @ConfigurationProperties 리스크 파라미터 설정
  - [x] 1회 최대 투자 금액 (기본: 100,000원) (FR-06-01)
  - [x] 손절 기준 (기본: -3%) (FR-06-02)
  - [x] 익절 기준 (기본: +5%) (FR-06-03)
  - [x] 전체 자산 하드 리밋 (기본: -15%) (FR-06-05)
  - [x] 단일 종목 최대 포지션 비율 (기본: 50%) (FR-06-06)
  - [x] 모든 파라미터 application.yml에서 설정 가능 (FR-06-08)
- [x] TradingConfig — 매매 모드(LIVE/PAPER) + 수수료율 설정
- [x] ExecutionConfigProperties — @EnableConfigurationProperties

### 주문 실행 (FR-04-01 ~ FR-04-06)
- [x] TradingCoordinator — 이벤트 → 주문 실행 조율
  - [x] TradingSignalEvent 수신 (@EventListener)
  - [x] 리스크 체크 → 주문 실행 또는 거부
  - [x] LIVE/PAPER 모드 분기
  - [x] RiskBreachedEvent 수신 → 강제 매도 실행
- [x] OrderExecutionService — 실제 주문 실행
  - [x] 지정가 매수/매도 주문 (FR-04-03)
  - [x] LIVE: ExchangeClient.placeOrder() 호출
  - [x] PAPER: 즉시 체결 시뮬레이션 (수수료 반영)
  - [x] 주문 실패 시 cancel + 예외 전파 (FR-04-05)
  - [x] OrderExecutedEvent 발행
  - [x] TradeCompletedEvent 발행

### 리스크 관리 (FR-06-01 ~ FR-06-08)
- [x] RiskCheckResult — sealed class (Approved / Rejected)
- [x] RiskManagementService — 리스크 검증
  - [x] 주문 전 리스크 체크 (투자금 한도, 포지션 비율, 매매 중지 상태)
  - [x] 실시간 손절/익절 모니터링 (TickerUpdatedEvent 수신)
  - [x] 하드 리밋 도달 시 전체 매매 중지 (tradingHalted)
  - [x] RiskBreachedEvent 발행 (STOP_LOSS, TAKE_PROFIT, HARD_LIMIT_TOTAL)
  - [x] resetHalt() — 매매 중지 해제

### 포지션 관리
- [x] PositionManagementService
  - [x] TradeCompletedEvent(BID) 수신 → Position 생성 + PositionOpenedEvent 발행
  - [x] 기존 포지션 있으면 addVolume() 물타기
  - [x] TradeCompletedEvent(ASK) 수신 → Position 종료 + PositionClosedEvent 발행
  - [x] RiskBreachedEvent(STOP_LOSS/TAKE_PROFIT) 수신 → 강제 청산 대기

### 저장소 (Adapter 패턴)
- [x] OrderJpaRepository + OrderRepositoryAdapter — OrderRepository 구현
- [x] PositionJpaRepository + PositionRepositoryAdapter — PositionRepository 구현
- [x] TradeJpaRepository + TradeRepositoryAdapter — TradeRepository 구현

### API
- [x] OrderController — 주문 내역 조회 (최근/미체결/상세)
- [x] PositionController — 포지션 조회 (열린 포지션/상세/매매 상태)

### 테스트 (4개 클래스, 19개 테스트, 100% 통과)
- [x] RiskManagementServiceTest — 투자한도/포지션비율/매매중지/손절/익절 6개
- [x] OrderExecutionServiceTest — Paper매수/매도/수수료/이벤트발행 4개
- [x] PositionManagementServiceTest — 신규포지션/물타기/포지션종료/매도무시 4개
- [x] TradingCoordinatorTest — BUY승인/거부/SELL실행/SELL무시/HOLD무시 5개

---

## Phase 5: Paper Trading ✅

> 관련 요구사항: FR-05 (Paper Trading)
> 모듈: `execution`

### 모드 관리 (FR-05-04)
- [x] TradingConfig.TradingMode — LIVE / PAPER (Phase 4에서 구현)
- [x] 종목별 독립 모드 전환 지원 (MarketTradingSettings + TradingModeResolver)

### Paper Trading 기본 동작
- [x] OrderExecutionService.executePaperOrder() — PAPER 모드 즉시 체결
- [x] 수수료 시뮬레이션 (feeRate 설정)
- [x] Paper 주문 → paper_* 테이블에 격리 저장 (PaperTradingConfiguration)

### 가상 잔고 관리 (FR-05-02) ✅
- [x] BalanceProvider 포트 인터페이스 (core)
- [x] PaperBalanceService — paper_balances 테이블 기반 (deduct/add/reset)
- [x] LiveBalanceProvider — ExchangeClient 래핑
- [x] PaperTradingConfiguration — @ConditionalOnProperty로 PAPER 모드 시 자동 교체
- [x] OrderExecutionService — Paper 주문 시 잔고 차감/증가
- [x] TradingConfig.paper.initialBalance 설정 (기본 1,000만원)

### 수익률 산출 (FR-05-03) ✅
- [x] PaperTradingStatsService — 총수익률, 승률, 미실현 손익 산출
- [x] PaperTradingController — REST API (/api/paper/*)
- [x] Paper Trading 대시보드 UI (/paper, Thymeleaf + Chart.js)

### 인프라
- [x] V7 마이그레이션 — paper_trades 테이블, paper_orders 보완
- [x] V8 마이그레이션 — paper_orders uuid 컬럼
- [x] Paper Entity 4개 + JPA Repository 4개 + Mapper 3개 + Adapter 3개

### 미구현
- [ ] 실제 매매 결과와 비교 (FR-05-05, 선택)

### 테스트
- [x] Paper 모드 주문 테스트 (OrderExecutionServiceTest에 포함)
- [x] PaperTradingPipelineTest 통과

---

## Phase 6: 모니터링 + 알림 ✅

> 관련 요구사항: FR-08 (알림)
> 모듈: `monitoring`

### Discord Webhook (FR-08-01 ~ FR-08-05)
- [x] DiscordConfig — @ConfigurationProperties (`discord.webhook-url`, `discord.enabled`)
- [x] DiscordWebhookSender — NotificationSender 구현체
  - [x] Embed 메시지 포맷 (색상: INFO파랑/SUCCESS초록/WARNING주황/ERROR빨강 + 타임스탬프)
  - [x] 전송 실패 시 예외 없이 로그 (서비스 안정성)
  - [x] 미설정/비활성화 시 graceful skip

### 이벤트 리스너
- [x] TradingEventListener
  - [x] TradeCompletedEvent → 매수/매도 체결 INFO 알림 (FR-08-01)
- [x] RiskEventListener
  - [x] RiskBreachedEvent → STOP_LOSS(WARNING), TAKE_PROFIT(INFO), HARD_LIMIT(CRITICAL) 알림 (FR-08-02, FR-08-03)
- [x] SystemEventListener
  - [x] SystemErrorEvent → 시스템 오류 ERROR 알림 (FR-08-05)
  - [x] AlertEvent → 레벨별 알림 전송
  - [x] DepositDetectedEvent → 입금 감지 INFO 알림 (FR-08-04)
  - [x] WithdrawalDetectedEvent → 출금 감지 WARNING 알림 (FR-08-04)

### 입출금 감지 (FR-08-04)
- [ ] DepositWithdrawalMonitorService — 5분 주기 폴링
  - [ ] Upbit API로 입금/출금 내역 조회 *(제외 범위: 입출금 API 미사용)*
  - [ ] 새로운 내역 감지 시 이벤트 발행

### 알림 이력 (FR-08-06)
- [x] AlertHistory JPA 엔티티 — alert_history 테이블 (level, eventType, title, message, sent)
- [x] AlertHistoryJpaRepository — 최근 50건 조회, 레벨별 조회
- [x] AlertManagementService — 알림 발송 + DB 저장 (level별 NotificationSender 메서드 분기)
- [x] AlertController — REST API (`GET /api/alerts`, `GET /api/alerts/{id}`)
- [x] V3__create_alert_history.sql — Flyway 마이그레이션

### application.yml 설정 추가
- [x] `discord.webhook-url`, `discord.enabled` 설정
- [x] `trading.mode`, `trading.fee-rate` 설정
- [x] `trading.risk.*` 리스크 파라미터 설정 (Phase 4 연동)

### 테스트 (5개 클래스, 18개 테스트, 100% 통과)
- [x] DiscordWebhookSenderTest — 미설정/비활성화/잘못된URL/isConfigured 4개
- [x] AlertManagementServiceTest — INFO/WARNING/ERROR/CRITICAL/sent확인 5개
- [x] TradingEventListenerTest — 매수/매도 체결 알림 2개
- [x] RiskEventListenerTest — 손절/익절/하드리밋 알림 3개
- [x] SystemEventListenerTest — 시스템오류/AlertEvent/입금/출금 알림 4개

---

## Phase 7: Docker + K8s 배포 🔄

> 관련 요구사항: NFR-06 (운영)
> 관련 문서: [[Kubernetes 로컬 개발 도구 비교]], [[인프라 아키텍처 개요]], [[Woodpecker CI 설치 및 설정 가이드]], [[ArgoCD 설치 및 GitOps 가이드]]

### Docker (NFR-06-01) ✅
- [x] Dockerfile 멀티스테이지 빌드 (`docker/Dockerfile` — 로컬 개발용)
- [x] Dockerfile.ci (`docker/Dockerfile.ci` — CI용, 사전 빌드된 JAR만 COPY)
- [x] docker-compose.yml (PostgreSQL 16)
- [x] .dockerignore

### Kubernetes (NFR-06-02) ✅
- [x] 매니페스트 레포 분리 (`liooos-k8s-manifests` — 별도 Git 레포, ArgoCD 감시 대상)
- [x] Deployment (`bitcoin-trader`, 네임스페이스 `bitcoin`)
- [x] Service, ConfigMap
- [x] PostgreSQL (`database` 공용 네임스페이스에 분리 배포)
- [x] Kustomize 기반 환경 관리
- [x] 무중단 배포 (Rolling Update: maxSurge:1, maxUnavailable:0)

### Local Registry ✅
- [x] registry:2 K3s Pod (NodePort :31500, PVC 5Gi)
- [x] K3s containerd registries.yaml 미러 설정

### Sealed Secrets (NFR-03-01) ✅
- [x] Sealed Secrets Controller 설치
- [x] API Key SealedSecret 생성
- [x] DB 비밀번호 SealedSecret 생성
- [x] 가이드 문서 작성 ([[Sealed Secrets 가이드]])

### CI — Woodpecker CI (NFR-06-03) ✅
> GitHub Actions에서 Woodpecker CI로 변경 (K3s Pod, Helm 설치)
- [x] `.woodpecker.yml` 파이프라인 (3단계)
  - [x] clone (plugin-git, depth=1)
  - [x] build-image (kaniko — Docker 데몬 없이 멀티스테이지 빌드)
  - [x] update-manifest (alpine/git — 매니페스트 레포 newTag 업데이트)
- [x] 트리거: dev 브랜치 push
- [x] 이미지: `registry:5000/bitcoin-trader:${CI_COMMIT_SHA}`
- [x] Synology 리버스 프록시 (GitHub Webhook → Woodpecker)

### CD — ArgoCD (NFR-06-04) ✅
- [x] ArgoCD 설치 (K3s, Helm)
- [x] Application 매니페스트 (`bitcoin-trader` + `postgres`)
- [x] 매니페스트 레포 감시 → kustomize build → 자동 Sync
- [x] 가이드 문서 작성 ([[ArgoCD 설치 및 GitOps 가이드]])

### 데이터 백업 (NFR-06-05, FR-03-05, FR-03-06) ✅
- [x] CronJob — pg_dump 일별 백업 (`k8s/backup/cronjob.yaml`)
- [x] 월별 아카이브 (매월 1일 자동 복사)
- [x] 30일 초과 일별 백업 자동 삭제
- [x] 백업 스크립트 (`scripts/backup-postgres.sh` — 로컬/K8s 겸용)
- [ ] 매니페스트 레포에 복사 후 ArgoCD 배포 (수동 작업 필요)

---

## 횡단 관심사 (Phase 전반에 걸쳐 진행)

### 성능 (NFR-01)
- [x] WebSocket 시세 수신 지연 < 1초 (NFR-01-01) — `[PERF]` 로그 측정 추가 ✅
- [x] 전략 신호 → 주문 실행 < 3초 (NFR-01-02) — `[PERF]` 로그 측정 추가 ✅
- [x] 백테스팅 1년 일봉 < 10초 (NFR-01-03) — `[PERF]` 로그 측정 추가 ✅
- [x] 동시 10개 이상 종목 모니터링 (NFR-01-04) — `[PERF]` 로그 측정 추가 ✅

### 안정성 (NFR-02)
- [x] 24시간 무중단 운영 (NFR-02-01) — 무한 재연결 + Graceful Shutdown + HealthIndicator + EventListener try-catch ✅
- [x] WebSocket 자동 재연결 (NFR-02-02) — WebSocketReconnectManager (무한 재연결, 10회마다 Discord 알림)
- [x] 미체결 주문 상태 복구 (NFR-02-03) — PendingOrderRecoveryService (앱 시작 시 자동)
- [x] 예기치 못한 종료 시 데이터 무결성 (NFR-02-04) — Outbox 패턴 + @Transactional
- [x] API 장애 시 안전 모드 (NFR-02-05) — CircuitBreaker + RateLimiter

### 보안 (NFR-03)
- [x] API Key Sealed Secrets 관리 (NFR-03-01) — K8s Sealed Secrets 적용
- [x] 입출금 API 미호출 (NFR-03-02) — 설계상 제외 완료
- [x] IP 화이트리스트 (NFR-03-03) — 가이드 문서 작성 완료 ([[Upbit API IP 화이트리스트 설정 가이드]])
- [x] 민감 정보 로그 마스킹 (NFR-03-04) — toString() 마스킹 + Authorization 헤더 마스킹

### 확장성 (NFR-04)
- [x] 새 전략 최소 코드 변경으로 추가 (NFR-04-01) — TradingStrategy + StrategyFactory
- [x] MSA 전환 가능 구조 (NFR-04-02) — Outbox 패턴 + 멀티모듈
- [x] 알림 채널 확장 가능 (NFR-04-03) — NotificationSender 인터페이스 + DiscordWebhookSender
- [ ] TimescaleDB 전환 가능 (NFR-04-04)

### 테스트 (NFR-05)
- [x] 전략 단위 테스트 (NFR-05-01) — MA크로스오버 5개, 복합전략 7개, 백테스팅 5개
- [x] Testcontainers PostgreSQL (NFR-05-02) — CandleJpaRepositoryTest, MarketJpaRepositoryTest
- [x] ArchUnit 모듈 의존성 (NFR-05-03) — ArchitectureTest 6개 (core, monitoring, strategy, execution, 도메인→인프라, 예외)
- [ ] Paper Trading 실시간 검증 (NFR-05-04)

---

## 요구사항 추적 매트릭스

| 요구사항 | Phase | 상태 |
|---------|-------|------|
| FR-01 마켓 스캔/종목 관리 | Phase 2 | ✅ MarketSyncService + MarketTradingSettings |
| FR-02 트레이딩 전략 | Phase 3 | ✅ MA크로스오버 + 복합전략 |
| FR-03 시세 데이터 수집 | Phase 2 | ✅ 캔들 수집 + WebSocket |
| FR-04 주문 실행 | Phase 4 | ✅ TradingCoordinator + OrderExecutionService |
| FR-05 Paper Trading | Phase 4+5 | ✅ 가상잔고 + 수익률 + 대시보드 + 종목별 모드 전환 |
| FR-06 리스크 관리 | Phase 4 | ✅ RiskManagementService + 종목별 리스크 설정 |
| FR-07 백테스팅 | Phase 3 | ✅ 엔진 + UI + 결과 저장 |
| FR-08 알림 | Phase 6 | ✅ Discord Webhook + 이벤트 리스너 + 알림 이력 |
| FR-09 계좌 조회 | Phase 2 | ✅ AccountQueryService |
| NFR-01 성능 | 횡단 | ✅ PERF 로그 측정 추가 |
| NFR-02 안정성 | Phase 2 + 횡단 | ✅ 무한 재연결 + Graceful Shutdown + HealthIndicator |
| NFR-03 보안 | Phase 7 + 횡단 | ✅ Sealed Secrets + IP 가이드 + 마스킹 완료 |
| NFR-04 확장성 | 설계 시 반영 | ✅ Outbox + Strategy Pattern |
| NFR-05 테스트 | 각 Phase | 🔄 Phase 1~6 테스트 130개 통과 (27 클래스) |
| NFR-06 운영 | Phase 7 | 🔄 CI/CD 완료, 데이터 백업 미구현 |

---

## 남은 작업 (우선순위순)

### 필수 (자동매매 실행에 필요)
- [x] **StrategyEvaluationService** — 실시간 이벤트(TickerUpdated/MarketDataReceived) → 전략 평가 → TradingSignalEvent 발행 ✅
- [x] 캔들 파티션 자동 생성 — CandlePartitionService + db-scheduler (매월 1일 실행) ✅

### 운영 안정성
- [x] 데이터 백업 CronJob (pg_dump 일별/월별) ✅
- [x] 24시간 무중단 운영 강화 — 무한 재연결, Graceful Shutdown, HealthIndicator, EventListener 보호 ✅
- [x] IP 화이트리스트 가이드 작성 ✅
- [x] 성능 측정 메트릭 로깅 (`[PERF]` prefix) ✅
- [x] 민감 정보 로그 마스킹 ✅

### 기능 보완 (선택)
- [x] RSI 전략 추가 ✅
- [x] 볼린저밴드 전략 추가 ✅
- [x] MACD 전략 추가 ✅
- [x] Phase 3 전략 6종 추가 (AD, CMF, MFI, Ichimoku, SAR, Pivot Points) + Factory 등록 ✅
- [x] 백테스팅 세션 삭제 기능 ✅
- [x] 백테스팅 과거 세션 상세 조회 (거래내역/차트) ✅
- [x] 백테스팅 UI 전략 드롭다운 19종 전체 추가 ✅
- [x] 매매 트리거를 WebSocket 전용으로 변경 (REST 수집 시 매매 안 함) ✅
- [x] Resilience4j Retry 적용 (429 응답 지수 백오프) ✅
- [x] Discord 알림 RestClient JSON 직렬화 오류 수정 (RestClient.Builder 주입) ✅
- [x] WebSocket 활성화/비활성화 설정 추가 (`upbit.websocket.enabled`) ✅
- [x] Paper 미체결 주문 복구 `@Transactional` 누락 수정 ✅
- [x] 외부 IP 변경 감지 스크립트 + K8s CronJob (10분마다 Discord 알림) ✅
- [x] 백테스팅 금액 입력 콤마 포맷팅 + label 단위 표시 ✅
- [ ] 분봉 캔들 수집 스케줄러
- [ ] 종목 추가/삭제 REST API
- [ ] 전략 활성화/비활성화 REST API
- [ ] 총 자산 평가액/수익률 계산 (FR-09-02)
- [ ] 입출금 감지 자동 폴링 서비스
- [ ] 이벤트별 세부 알림 on/off
- [ ] Paper vs Live 결과 비교 (FR-05-05)

---

## 진행 순서 요약

```
Phase 1 ✅ → Phase 2 ✅ → Phase 3 ✅ → Phase 4 ✅ → Phase 5 ✅ → Phase 6 ✅ → Phase 7 🔄
스켈레톤       API연동       전략엔진    주문실행     모의매매      알림        배포
                            +백테스팅   +리스크      +가상잔고                (CI/CD ✅)
                                                   +수익률                 (백업 ⏳)
                                                   +대시보드
                                                   +종목별 모드전환
```

> 각 Phase 완료 시 이 문서의 체크박스를 업데이트한다.
