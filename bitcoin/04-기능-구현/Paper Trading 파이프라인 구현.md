# Paper Trading 파이프라인 구현

> 작성일: 2026-03-10
> 관련 모듈: api-client, strategy, execution, app

---

## 개요

WebSocket 실시간 시세 수신부터 전략 분석, 주문 실행, 포지션 관리, 리스크 관리까지의 **전체 Paper Trading 파이프라인**을 연결하고 통합 테스트를 작성했다.

## 이벤트 파이프라인 흐름

```
[WebSocket] TickerUpdatedEvent
  ├→ RiskManagementService: 손절/익절/하드리밋 체크
  └→ (시세 캐싱)

[db-scheduler] MarketDataReceivedEvent (1분봉 수집 완료)
  └→ StrategyAnalysisListener
       └→ StrategyService.analyzeMarket()
            └→ TradingSignalEvent (BUY/SELL)
                 └→ TradingCoordinator
                      ├→ RiskManagementService.checkPreOrderRisk()
                      └→ OrderExecutionService (Paper/Live 분기)
                           └→ TradeCompletedEvent
                                └→ PositionManagementService
                                     ├→ 매수 → Position 개설/추가
                                     └→ 매도 → Position 종료

[RiskBreachedEvent] (손절/익절 발동)
  └→ TradingCoordinator.onRiskBreached()
       └→ 강제 매도 → TradeCompletedEvent → PositionManagementService
```

## 신규/수정 파일 목록

### 신규 생성

| 파일 | 모듈 | 역할 |
|------|------|------|
| `WebSocketLifecycleManager.kt` | api-client | 앱 시작 시 WebSocket 자동 연결, 마켓 변경 시 재구독 |
| `WebSocketTickerRs.kt` | api-client | WebSocket 전용 DTO (`code` 필드 사용) |
| `StrategyAnalysisListener.kt` | strategy | MarketDataReceivedEvent → 전략 분석 → TradingSignalEvent 발행 |
| `V6__seed_strategy_configs.sql` | app | MA 크로스오버 (5/20) 기본 전략 시드 데이터 |
| `PaperTradingPipelineTest.kt` | app (test) | Paper 모드 전체 파이프라인 통합 테스트 6건 |

### 수정

| 파일 | 변경 내용 |
|------|-----------|
| `UpbitWebSocketClient.kt` | WebSocketTickerRs 파싱, 재연결 시 구독 복원 (`lastSubscribedMarkets`) |
| `StrategyService.kt` | 마켓코드 필터 추가, configs 빈 검사 최적화 |
| `PositionManagementService.kt` | `resolveCloseStatus()` 추가 (CLOSED/STOP_LOSS/TAKE_PROFIT 구분) |
| `RiskManagementService.kt` | WebSocket 가격 캐싱, 하드리밋 체크 60초 쓰로틀링 |

## Paper vs Live 모드

`TradingConfig.TradingMode` enum으로 구분:
- **PAPER**: `executePaperOrder()` — 즉시 체결 시뮬레이션, DB에 주문/거래 기록
- **LIVE**: `executeLiveOrder()` — `ExchangeClient.placeOrder()` 실제 API 호출

```kotlin
// execution/config/TradingConfig.kt
data class TradingConfig(
    val mode: TradingMode = TradingMode.PAPER,
    val feeRate: Double = 0.0005
) {
    enum class TradingMode { LIVE, PAPER }
}
```

## 포지션 종료 상태 구분

전략명 prefix로 종료 사유를 식별:

| strategyName prefix | PositionStatus |
|---------------------|----------------|
| `RISK_STOP_LOSS` | `STOP_LOSS` |
| `RISK_TAKE_PROFIT` | `TAKE_PROFIT` |
| 그 외 | `CLOSED` |

## 통합 테스트 (PaperTradingPipelineTest)

| 테스트 | 검증 내용 |
|--------|-----------|
| 골든크로스 감지 | SMA(5) > SMA(20) 교차 시 BUY 신호 발행 |
| BUY → 포지션 생성 | 리스크 체크 → Paper 주문 → TradeCompletedEvent → Position 개설 |
| SELL → 포지션 종료 | 열린 포지션 매도 → Position CLOSED |
| 손절 발동 | -4% 하락 시 RiskBreachedEvent → 강제 매도 → STOP_LOSS |
| 캔들 부족 | 데이터 없으면 신호 미발행 |
| 리스크 한도 초과 | 잔고 부족 시 매수 거부 |

## WebSocket 이슈 해결

**문제**: REST API의 `TickerRs`는 `market` 필드, WebSocket은 `code` 필드 사용
**해결**: `WebSocketTickerRs` DTO 분리 생성

```kotlin
@JsonIgnoreProperties(ignoreUnknown = true)
data class WebSocketTickerRs(
    @JsonProperty("code") val code: String,
    @JsonProperty("trade_price") val tradePrice: BigDecimal,
    // ...
)
```

## 배포 전 로드맵

1. ~~Paper 모드 파이프라인 연결~~ ✅
2. ~~통합 테스트 작성~~ ✅
3. ~~1분봉 파이프라인 시뮬레이션~~ ✅ → [[04-기능-구현/파이프라인 시뮬레이션 구현]]
4. Paper + Live 동시 실행 모드 (예정)
5. Live 배포
