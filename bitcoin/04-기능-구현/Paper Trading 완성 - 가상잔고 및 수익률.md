# Paper Trading 완성 — 가상 잔고 및 수익률

> 작성일: 2026-03-22
> 관련 모듈: core, execution, app
> 관련 문서: [[Paper Trading 파이프라인 구현]], [[실시간 티커 캐시 구현]]

---

## 배경

Paper Trading 기본 구조(즉시 체결, 이벤트 파이프라인)는 완성되어 있었으나, 가상 잔고 관리/수익률 산출이 미구현이었다. DB에 `paper_orders`, `paper_positions`, `paper_balances` 테이블이 이미 생성되어 있었으나 사용되지 않고 있었다.

## 구현 범위

| 항목       | 설명                              | 상태      |
| -------- | ------------------------------- | ------- |
| FR-05-02 | 가상 잔고 관리 (초기 금액, 매수 차감, 매도 증가)  | ✅       |
| FR-05-03 | 모의매매 수익률 산출 (총 수익률, 승률, 미실현 손익) | ✅       |
| FR-05-04 | 종목별 독립 모드 전환                    | ⏳ 별도 진행 |

## 핵심 설계 결정

### 1. 데이터 격리: paper_* 테이블 분리

Live 데이터와 Paper 데이터를 완전 격리하여 오염 방지.

| 모드 | 주문 | 거래 | 포지션 | 잔고 |
|------|------|------|--------|------|
| Live | `orders` | `trades` | `positions` | Upbit API |
| Paper | `paper_orders` | `paper_trades` | `paper_positions` | `paper_balances` |

### 2. @ConditionalOnProperty 기반 Repository 교체

```
PAPER 모드 → PaperTradingConfiguration 활성화
  → paper_* Repository를 @Primary로 등록
  → 기존 서비스 코드 변경 없이 저장소 교체
```

### 3. BalanceProvider 추상화

`RiskManagementService.calculateTotalBalance()`가 모드에 무관하게 동작하도록 인터페이스 도입.

```
BalanceProvider (interface)
├─ LiveBalanceProvider: ExchangeClient.getAccounts() 래핑
└─ PaperBalanceService: paper_balances 테이블 기반
```

## 파일 구조

### 신규 파일

```
core/port/outbound/BalanceProvider.kt          — 잔고 조회 포트

execution/application/
├─ LiveBalanceProvider.kt                      — Live 잔고 (Upbit API)
├─ PaperBalanceService.kt                      — Paper 잔고 (DB)
├─ PaperTradingStatsService.kt                 — 수익률 산출
execution/api/
└─ PaperTradingController.kt                   — REST API

execution/config/
└─ PaperTradingConfiguration.kt                — 모드별 빈 교체

execution/infrastructure/paper/
├─ PaperOrderEntity.kt, PaperTradeEntity.kt, PaperPositionEntity.kt, PaperBalanceEntity.kt
├─ PaperOrderJpaRepository.kt, ... (JPA 4개)
├─ PaperOrderEntityMapper.kt, ... (Mapper 3개)
└─ PaperOrderRepositoryAdapter.kt, ... (Adapter 3개)

app/src/main/resources/db/migration/
└─ V7__paper_trading_completion.sql
```

### 수정 파일

| 파일 | 변경 |
|------|------|
| `TradingConfig.kt` | `paper.initialBalance` 추가 |
| `RiskManagementService.kt` | `exchangeClient` → `BalanceProvider` 교체 |
| `OrderExecutionService.kt` | Paper 잔고 차감/증가 추가 |

## REST API

| 메서드 | 경로 | 기능 |
|--------|------|------|
| GET | `/api/paper/stats` | 종합 수익률/통계 |
| GET | `/api/paper/balance` | 현재 잔고 |
| POST | `/api/paper/balance/reset` | 잔고 초기화 |
| GET | `/api/paper/positions` | 포지션 목록 |
| GET | `/api/paper/positions/open` | 열린 포지션 |
| GET | `/api/paper/orders` | 주문 내역 |
| GET | `/api/paper/trades` | 거래 내역 |

## 대시보드 UI

- 접속: `http://localhost:9000/paper`
- Thymeleaf + Chart.js (backtest.html 패턴 재사용)
- 메트릭 카드 8개, 도넛 차트 2개 (자산 구성, 승/패), 포지션/거래 테이블
- 잔고 초기화 기능

## 설정

```yaml
trading:
  mode: PAPER            # PAPER 또는 LIVE
  fee-rate: 0.0005
  paper:
    initial-balance: 10000000  # 1,000만원
```
