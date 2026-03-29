# Upbit API Reference (v1.6.1)

> 공식 문서: https://docs.upbit.com/kr/reference
> Base URL: `https://api.upbit.com/v1`
> WebSocket: `wss://api.upbit.com/websocket/v1`
> TLS: 1.2 이상 (1.3 권장)
> Content-Type: `application/json; charset=utf-8`

---

## 인증 (Authentication)

### JWT 토큰 구조

```
Header: { "alg": "HS256" }  // HS512 권장
Payload: {
  "access_key": "{ACCESS_KEY}",
  "nonce": "{UUID}",
  "query_hash": "{HASH}",        // 파라미터 있을 때만
  "query_hash_alg": "SHA512"     // 파라미터 있을 때만
}
Signature: HMAC-SHA512(Header.Payload, SECRET_KEY)
```

### Authorization 헤더

```
Authorization: Bearer {JWT_TOKEN}
```

### Query Hash 생성

- GET/DELETE: 쿼리스트링 그대로 SHA512 해싱 (`param1=value1&param2=value2`)
- POST: JSON body를 쿼리스트링 형식으로 변환 후 해싱
- 배열 파라미터: `states[]=wait&states[]=watch`
- 파라미터 순서 변경 금지

### API Key 관리

- Access Key + Secret Key 쌍
- Secret Key는 발급 시에만 확인 가능
- IP 화이트리스트: 최대 10개 IP 등록
- 권한 그룹: 자산조회, 주문하기, 주문조회, 출금하기, 출금조회, 입금하기, 입금조회

---

## Rate Limit

### 그룹별 제한

| 카테고리 | 그룹 | 제한 | 측정 기준 |
|---------|------|------|----------|
| Quotation | market | 10/sec | IP 기반 |
| Quotation | candle | 10/sec | IP 기반 |
| Quotation | trade | 10/sec | IP 기반 |
| Quotation | ticker | 10/sec | IP 기반 |
| Quotation | orderbook | 10/sec | IP 기반 |
| Exchange | default | 30/sec | 계정 기반 |
| Exchange | order | 8/sec | 계정 기반 |
| Exchange | order-test | 8/sec | 계정 기반 |
| Exchange | order-cancel-all | 1/2sec | 계정 기반 |
| WebSocket | websocket-connect | 5/sec | IP/계정 |
| WebSocket | websocket-message | 5/sec, 100/min | IP/계정 |

### Origin 헤더 제한

- Origin 헤더 포함 시: Quotation REST API, WebSocket 모두 **10초당 1회**

### Remaining-Req 응답 헤더

```
Remaining-Req: group=default; min=1800; sec=29
```

- `group`: Rate Limit 그룹명
- `min`: 사용 중단 예정 (고정값, 무시)
- `sec`: 남은 요청 수 (0이면 요청 불가)

### 초과 시 응답

- **429**: Too Many Requests
- **418**: IP/계정 일시 차단 (차단 기간 정보 포함)
- 429 이후 계속 요청 시 차단 기간 점진적 증가

---

## Quotation API (Public, 인증 불필요)

### 페어(마켓) 목록

```
GET /market/all
```

- `is_details` (선택): 유의종목 필드 포함 여부 (Boolean)

응답 필드:
- `market`: 마켓 코드 (예: KRW-BTC)
- `korean_name`: 한글 이름
- `english_name`: 영문 이름
- `market_event` (is_details=true): 유의 종목 정보

### 캔들 (OHLCV)

```
GET /candles/seconds                # 초봉
GET /candles/minutes/{unit}         # 분봉 (1, 3, 5, 10, 15, 30, 60, 240)
GET /candles/days                   # 일봉
GET /candles/weeks                  # 주봉
GET /candles/months                 # 월봉
GET /candles/years                  # 연봉
```

공통 파라미터:
- `market` (필수): 마켓 코드 (예: KRW-BTC)
- `to` (선택): 마지막 캔들 시각 (ISO 8601, yyyy-MM-dd'T'HH:mm:ss)
- `count` (선택): 캔들 개수 (최대 200)

일봉 추가 파라미터:
- `converting_price_unit` (선택): 종가 환산 화폐 (예: KRW)

응답 필드:
- `market`: 마켓 코드
- `candle_date_time_utc`: 캔들 기준 시각 (UTC)
- `candle_date_time_kst`: 캔들 기준 시각 (KST)
- `opening_price`: 시가
- `high_price`: 고가
- `low_price`: 저가
- `trade_price`: 종가
- `candle_acc_trade_volume`: 누적 거래량
- `candle_acc_trade_price`: 누적 거래대금
- `timestamp`: 타임스탬프

### 최근 체결

```
GET /trades/recent
```

- `market` (필수): 마켓 코드
- `count` (선택): 체결 수

### 현재가 (Ticker)

#### 페어 단위 현재가 조회

```
GET /ticker
```

- `markets` (필수): 마켓 코드 (콤마 구분, 예: KRW-BTC,KRW-ETH)

#### 마켓 단위 현재가 조회

```
GET /ticker/all
```

- `quote_currencies` (필수): 마켓 통화 코드 (콤마 구분, 예: KRW,BTC)

Ticker 응답 필드:
- `market`: 페어 코드
- `trade_date` / `trade_time`: 최근 체결 일시 (UTC, yyyyMMdd / HHmmss)
- `trade_date_kst` / `trade_time_kst`: 최근 체결 일시 (KST)
- `trade_timestamp`: 체결 타임스탬프 (ms)
- `opening_price`: 시가
- `high_price`: 고가
- `low_price`: 저가
- `trade_price`: 종가 (현재가)
- `prev_closing_price`: 전일 종가 (UTC 0시 기준)
- `change`: 가격 변동 상태 (RISE/EVEN/FALL)
- `change_price`: 전일 대비 변동가 (절대값)
- `change_rate`: 전일 대비 변동률 (절대값)
- `signed_change_price`: 부호 있는 변동가
- `signed_change_rate`: 부호 있는 변동률
- `trade_volume`: 최근 거래 수량
- `acc_trade_price`: 누적 거래 금액 (UTC 0시 기준)
- `acc_trade_price_24h`: 24시간 누적 거래 금액
- `acc_trade_volume`: 누적 거래량 (UTC 0시 기준)
- `acc_trade_volume_24h`: 24시간 누적 거래량
- `highest_52_week_price` / `highest_52_week_date`: 52주 신고가/달성일
- `lowest_52_week_price` / `lowest_52_week_date`: 52주 신저가/달성일
- `timestamp`: 현재가 반영 타임스탬프 (ms)

### 호가 (Orderbook)

```
GET /orderbook
```

- `markets` (필수): 마켓 코드 (콤마 구분)
- `level` (선택, 기본 0): 호가 모아보기 단위 (원화마켓만 지원)
- `count` (선택, 기본 30): 호가 쌍 개수 (최대 30)

응답 필드:
- `market`: 페어 코드
- `timestamp`: 조회 시각 타임스탬프 (ms)
- `total_ask_size`: 전체 매도 잔량 합계
- `total_bid_size`: 전체 매수 잔량 합계
- `orderbook_units[]`: 호가 정보 배열 (1호가~30호가)
  - `ask_price`: 매도 호가
  - `bid_price`: 매수 호가
  - `ask_size`: 매도 잔량
  - `bid_size`: 매수 잔량
  - `level`: 적용된 가격 단위

```
GET /orderbook-instruments
```

- 호가 정책 정보 (종목별 지원 모아보기 단위 등)

---

## Exchange API (Private, 인증 필수)

### 계좌 잔고

```
GET /accounts
```

응답 필드:
- `currency`: 화폐 코드
- `balance`: 주문 가능 수량
- `locked`: 주문 중 묶인 수량
- `avg_buy_price`: 평균 매수가
- `avg_buy_price_modified`: 평균 매수가 수정 여부
- `unit_currency`: 평가 화폐

### 주문 생성

```
POST /orders
```

파라미터:
- `market` (필수): 마켓 코드 (예: KRW-BTC)
- `side` (필수): `bid` (매수) / `ask` (매도)
- `ord_type` (필수): `limit` / `price` / `market` / `best`
- `volume` (조건): 주문 수량
- `price` (조건): 주문 가격
- `identifier` (선택): 클라이언트 지정 주문 식별자
- `time_in_force` (선택): `ioc` / `fok` / `post_only`
- `smp_type` (선택): 자전거래 방지 (`reduce` / `cancel_maker` / `cancel_taker`)

주문 유형별 필수 파라미터:

| ord_type | volume | price | 설명 |
|----------|--------|-------|------|
| limit | O | O | 지정가 주문 |
| price | X | O | 시장가 매수 (총액 기준) |
| market | O | X | 시장가 매도 (수량 기준) |
| best | O | X | 최유리 지정가 주문 (time_in_force 필수) |

time_in_force 옵션:
- `ioc` (Immediate or Cancel): 즉시 체결 가능한 수량만 체결, 잔량 취소
- `fok` (Fill or Kill): 전체 수량 즉시 체결 가능 시에만 체결, 불가 시 전체 취소
- `post_only`: 메이커 주문만 허용, 즉시 체결되는 경우 주문 취소

### 주문 생성 테스트

```
POST /orders/test
```

- 실제 주문 없이 유효성만 검증 (Paper Trading에 활용 가능)

### 주문 조회 (개별)

```
GET /order
```

파라미터 (하나 필수):
- `uuid` (선택): 주문 UUID
- `identifier` (선택): 클라이언트 지정 식별자

응답 필드:
- `market`, `uuid`, `side`, `ord_type`, `price`, `state`
- `created_at`: 주문 생성 시각 (KST, yyyy-MM-ddTHH:mm:ss+09:00)
- `volume`: 주문 요청 수량
- `remaining_volume`: 체결 후 남은 수량
- `executed_volume`: 체결된 수량
- `reserved_fee`, `remaining_fee`, `paid_fee`: 수수료 관련
- `locked`: 거래에 사용 중인 비용
- `time_in_force`: ioc/fok/post_only
- `smp_type`: reduce/cancel_maker/cancel_taker
- `prevented_volume`: SMP로 취소된 수량
- `prevented_locked`: SMP로 해제된 자산
- `identifier`: 클라이언트 식별자 (2024-10-18 이후 주문만)
- `trades_count`: 체결 건수
- `trades[]`: 체결 목록
  - `market`, `uuid`, `price`, `volume`, `funds`
  - `trend`: up (매수 체결) / down (매도 체결)
  - `created_at`, `side`

주문 상태 (`state`):
- `wait`: 체결 대기
- `watch`: 예약 주문 대기
- `done`: 체결 완료
- `cancel`: 주문 취소

### 주문 취소

```
DELETE /order
```

파라미터 (하나 필수):
- `uuid` (선택): 주문 UUID
- `identifier` (선택): 클라이언트 지정 식별자

### 취소 후 재주문

```
POST /orders/cancel-and-new
```

### 입금 내역 조회 (알림용)

```
GET /deposits
```

### 출금 내역 조회 (알림용)

```
GET /withdraws
```

---

## WebSocket API

### 연결

```
wss://api.upbit.com/websocket/v1
```

### 구독 요청 형식

```json
[
  { "ticket": "unique-ticket-id" },
  {
    "type": "ticker",
    "codes": ["KRW-BTC", "KRW-ETH"],
    "is_only_snapshot": false,
    "is_only_realtime": false
  },
  { "format": "DEFAULT" }
]
```

### 스트림 타입

| type | 설명 | 인증 |
|------|------|------|
| `ticker` | 현재가 | 불필요 |
| `trade` | 체결 | 불필요 |
| `orderbook` | 호가 | 불필요 |
| `candle` | 캔들 | 불필요 |
| `myOrder` | 내 주문/체결 | 필요 |
| `myAsset` | 내 자산 | 필요 |

### 공통 옵션

- `is_only_snapshot`: true면 스냅샷만 수신
- `is_only_realtime`: true면 실시간만 수신
- `format`: `DEFAULT` (풀네임) / `SIMPLE_LIST` (축약) / `JSON_LIST`

### Ticker (현재가)

요청: `{ "type": "ticker", "codes": ["KRW-BTC"] }`

| 필드 | 축약 | 설명 | 타입 |
|------|------|------|------|
| `type` | `ty` | 타입 (ticker) | String |
| `code` | `cd` | 페어 코드 | String |
| `opening_price` | `op` | 시가 | Double |
| `high_price` | `hp` | 고가 | Double |
| `low_price` | `lp` | 저가 | Double |
| `trade_price` | `tp` | 현재가 | Double |
| `prev_closing_price` | `pcp` | 전일 종가 | Double |
| `change` | `c` | 변동 방향 (RISE/EVEN/FALL) | String |
| `change_price` | `cp` | 변동가 (절대값) | Double |
| `signed_change_price` | `scp` | 부호 있는 변동가 | Double |
| `change_rate` | `cr` | 변동률 (절대값) | Double |
| `signed_change_rate` | `scr` | 부호 있는 변동률 | Double |
| `trade_volume` | `tv` | 최근 거래량 | Double |
| `acc_trade_volume` | `atv` | 누적 거래량 (UTC 0시) | Double |
| `acc_trade_volume_24h` | `atv24h` | 24시간 누적 거래량 | Double |
| `acc_trade_price` | `atp` | 누적 거래대금 (UTC 0시) | Double |
| `acc_trade_price_24h` | `atp24h` | 24시간 누적 거래대금 | Double |
| `trade_date` | `tdt` | 최근 거래일 (yyyyMMdd) | String |
| `trade_time` | `ttm` | 최근 거래시각 (HHmmss) | String |
| `trade_timestamp` | `ttms` | 체결 타임스탬프 (ms) | Long |
| `ask_bid` | `ab` | ASK(매도) / BID(매수) | String |
| `acc_ask_volume` | `aav` | 누적 매도량 | Double |
| `acc_bid_volume` | `abv` | 누적 매수량 | Double |
| `highest_52_week_price` | `h52wp` | 52주 최고가 | Double |
| `highest_52_week_date` | `h52wdt` | 52주 최고가 달성일 | String |
| `lowest_52_week_price` | `l52wp` | 52주 최저가 | Double |
| `lowest_52_week_date` | `l52wdt` | 52주 최저가 달성일 | String |
| `market_state` | `ms` | 거래상태 (PREVIEW/ACTIVE/DELISTED) | String |
| `timestamp` | `tms` | 타임스탬프 (ms) | Long |
| `stream_type` | `st` | SNAPSHOT / REALTIME | String |

### Trade (체결)

요청: `{ "type": "trade", "codes": ["KRW-BTC"] }`

| 필드 | 축약 | 설명 | 타입 |
|------|------|------|------|
| `type` | `ty` | 타입 (trade) | String |
| `code` | `cd` | 페어 코드 | String |
| `trade_price` | `tp` | 체결 가격 | Double |
| `trade_volume` | `tv` | 체결량 | Double |
| `ask_bid` | `ab` | ASK(매도) / BID(매수) | String |
| `prev_closing_price` | `pcp` | 전일 종가 | Double |
| `change` | `c` | 변동 방향 (RISE/EVEN/FALL) | String |
| `change_price` | `cp` | 변동가 (절대값) | Double |
| `trade_date` | `td` | 체결 일자 (yyyy-MM-dd, UTC) | String |
| `trade_time` | `ttm` | 체결 시각 (HH:mm:ss, UTC) | String |
| `trade_timestamp` | `ttms` | 체결 타임스탬프 (ms) | Long |
| `timestamp` | `tms` | 타임스탬프 (ms) | Long |
| `sequential_id` | `sid` | 체결 번호 (Unique) | Long |
| `best_ask_price` | `bap` | 최우선 매도 호가 | Double |
| `best_ask_size` | `bas` | 최우선 매도 잔량 | Double |
| `best_bid_price` | `bbp` | 최우선 매수 호가 | Double |
| `best_bid_size` | `bbs` | 최우선 매수 잔량 | Double |
| `stream_type` | `st` | SNAPSHOT / REALTIME | String |

### Orderbook (호가)

요청: `{ "type": "orderbook", "codes": ["KRW-BTC"], "level": 10000 }`

호가 개수 지정: codes에서 `KRW-BTC.15` 형식 (지원 단위: 1, 5, 15, 30)

| 필드 | 축약 | 설명 | 타입 |
|------|------|------|------|
| `type` | `ty` | 타입 (orderbook) | String |
| `code` | `cd` | 페어 코드 | String |
| `total_ask_size` | `tas` | 매도 총 잔량 | Double |
| `total_bid_size` | `tbs` | 매수 총 잔량 | Double |
| `orderbook_units` | `obu` | 호가 목록 | List |
| `orderbook_units[].ask_price` | `obu[].ap` | 매도 호가 | Double |
| `orderbook_units[].bid_price` | `obu[].bp` | 매수 호가 | Double |
| `orderbook_units[].ask_size` | `obu[].as` | 매도 잔량 | Double |
| `orderbook_units[].bid_size` | `obu[].bs` | 매수 잔량 | Double |
| `timestamp` | `tms` | 타임스탬프 (ms) | Long |
| `level` | `lv` | 모아보기 단위 (KRW 마켓만) | Double |
| `stream_type` | `st` | SNAPSHOT / REALTIME | String |

### Candle (캔들)

요청 type 종류:
- `candle.1s`: 초봉
- `candle.1m` ~ `candle.240m`: 분봉 (1, 3, 5, 10, 15, 30, 60, 240)

전송 주기: 1초. 체결 발생 시에만 데이터 생성/전송.

| 필드 | 축약 | 설명 | 타입 |
|------|------|------|------|
| `type` | `ty` | 캔들 타입 | String |
| `code` | `cd` | 마켓 코드 | String |
| `candle_date_time_utc` | `cdttmu` | 캔들 시각 (UTC) | String |
| `candle_date_time_kst` | `cdttmk` | 캔들 시각 (KST) | String |
| `opening_price` | `op` | 시가 | Double |
| `high_price` | `hp` | 고가 | Double |
| `low_price` | `lp` | 저가 | Double |
| `trade_price` | `tp` | 종가 | Double |
| `candle_acc_trade_volume` | `catv` | 누적 거래량 | Double |
| `candle_acc_trade_price` | `catp` | 누적 거래 금액 | Double |
| `timestamp` | `tms` | 타임스탬프 (ms) | Long |
| `stream_type` | `st` | SNAPSHOT / REALTIME | String |

### MyOrder (내 주문/체결) — 인증 필요

요청: `{ "type": "myOrder" }` (codes 생략 시 전체 마켓 구독)

주문/체결 발생 시에만 실시간 전송.

| 필드 | 축약 | 설명 | 타입 |
|------|------|------|------|
| `type` | `ty` | myOrder | String |
| `code` | `cd` | 페어 코드 | String |
| `uuid` | `uid` | 주문 UUID | String |
| `ask_bid` | `ab` | ASK(매도) / BID(매수) | String |
| `order_type` | `ot` | limit/price/market/best | String |
| `state` | `s` | wait/watch/trade/done/cancel/prevented | String |
| `trade_uuid` | `tuid` | 체결 UUID | String |
| `price` | `p` | 주문/체결 가격 | Double |
| `avg_price` | `ap` | 평균 체결 가격 | Double |
| `volume` | `v` | 주문/체결량 | Double |
| `remaining_volume` | `rv` | 주문 잔량 | Double |
| `executed_volume` | `ev` | 체결된 수량 | Double |
| `trades_count` | `tc` | 체결 건수 | Integer |
| `reserved_fee` | `rsf` | 예약 수수료 | Double |
| `remaining_fee` | `rmf` | 남은 수수료 | Double |
| `paid_fee` | `pf` | 사용된 수수료 | Double |
| `locked` | `l` | 사용 중인 비용 | Double |
| `executed_funds` | `ef` | 체결 금액 | Double |
| `time_in_force` | `tif` | ioc/fok/post_only | String |
| `trade_fee` | `tf` | 체결 시 수수료 (trade 시) | Double |
| `is_maker` | `im` | 메이커 여부 (trade 시) | Boolean |
| `identifier` | `id` | 클라이언트 식별자 | String |
| `smp_type` | `smpt` | reduce/cancel_maker/cancel_taker | String |
| `prevented_volume` | `pv` | SMP 취소 수량 | Double |
| `prevented_locked` | `pl` | SMP 해제 자산 | Double |
| `trade_timestamp` | `ttms` | 체결 타임스탬프 (ms) | Long |
| `order_timestamp` | `otms` | 주문 타임스탬프 (ms) | Long |
| `timestamp` | `tms` | 타임스탬프 (ms) | Long |
| `stream_type` | `st` | SNAPSHOT / REALTIME | String |

### MyAsset (내 자산) — 인증 필요

요청: `{ "type": "myAsset" }` (codes 파라미터 미지원, 포함 시 WRONG_FORMAT 에러)

자산 변동 발생 시에만 실시간 전송. 최초 이용 시 수분간 미수신 가능 (재연결 후 정상).

| 필드 | 축약 | 설명 | 타입 |
|------|------|------|------|
| `type` | `ty` | myAsset | String |
| `asset_uuid` | `astuid` | 자산 고유 식별자 | String |
| `assets` | `ast` | 자산 목록 | List |
| `assets[].currency` | `ast[].cu` | 화폐 코드 | String |
| `assets[].balance` | `ast[].b` | 주문 가능 수량 | Double |
| `assets[].locked` | `ast[].l` | 주문 중 묶인 수량 | Double |
| `asset_timestamp` | `asttms` | 자산 타임스탬프 (ms) | Long |
| `timestamp` | `tms` | 타임스탬프 (ms) | Long |
| `stream_type` | `st` | REALTIME | String |

### WebSocket 제한 사항

- websocket-connect: 5/sec
- websocket-message: 5/sec, 100/min
- 인증 없이 연결: IP 기반 제한
- 인증 후 연결: 계정 기반 제한
- 연결 유지: Ping/Pong 프레임 활용

---

## HTTP 상태 코드

| 코드 | 설명 |
|------|------|
| 200 | 정상 |
| 201 | 생성 완료 |
| 400 | 요청 오류 |
| 401 | 인증 실패 |
| 404 | 데이터 없음 |
| 418 | 과도한 요청 (차단) |
| 429 | Rate Limit 초과 |
| 500 | 서버 오류 |

## 주요 에러 코드

| 코드 | 설명 |
|------|------|
| `invalid_query_payload` | JWT 페이로드 오류 |
| `insufficient_funds` | 잔고 부족 |
| `validation_error` | 필수 파라미터 누락 |
| `expired_access_key` | API Key 만료 |
| `out_of_scope` | API Key 권한 부족 |

---

## 미사용 API (보안상 제외)

| API | 사유 |
|-----|------|
| `POST /withdrawals` | 코인/원화 출금 — 보안 위험 |
| `DELETE /withdrawals/{id}` | 출금 취소 — 불필요 |
| `POST /deposits/addresses` | 입금 주소 생성 — 불필요 |
| `POST /deposits` | 원화 입금 — 불필요 |
