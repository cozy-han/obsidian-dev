# App: USS-Pub (USS 퍼블릭 서비스)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 9000 |
| **JAR명** | uss-pub |
| **패키지** | `com.kac.utm.uss` |
| **메인 클래스** | `UssPubApplication.java` |
| **프로필** | db, security, uss, gsl, redis, rabbitmq |
| **사이트 ID** | USS |
| **클래스 수** | 448개 (프로젝트 최대) |

---

## 역할

**USS(Unmanned aircraft System Service provider)** 즉 드론 운영 서비스 제공자를 위한 API 서버이다. 비행계획 신청/승인/조회, 공역 관리, Inter-USS 통신 등 드론 운영의 핵심 기능을 제공한다. 프로젝트에서 **가장 큰 모듈**(448개 클래스).

> **언제 이 코드를 보게 되나?**
> 드론 운영사(USS) 관련 기능을 개발할 때. 비행계획의 전체 라이프사이클(신청→검증→승인→비행→종료), Inter-USS 통신, FIXM 표준 처리를 작업할 때.

---

## 패키지 구조

```
com.kac.utm.uss/
├── module/
│   ├── aerl/
│   │   ├── notam/       ← NOTAM (항공 공지사항)
│   │   ├── snrsns/      ← 센서 정보
│   │   └── wthr/        ← 기상 정보
│   ├── com/             ← 공통 (인증, 코드, 기업, 메뉴, 팝업)
│   ├── conts/           ← 콘텐츠
│   ├── fpl/             ← 비행계획 (핵심)
│   │   ├── acr/         ← 항공기(Aircraft) 관리
│   │   ├── bsc/         ← 기본 비행계획
│   │   ├── convert/     ← 포맷 변환 (FIXM 등)
│   │   ├── itrp/        ← 비행경로 교환(Interchange)
│   │   ├── plt/         ← 조종사(Pilot) 관리
│   │   ├── prmsn/       ← 비행 허가(Permission)
│   │   ├── rlm/         ← 비행 구간(Realm) 관리
│   │   └── valid/       ← 비행계획 검증(Validation)
│   ├── gsl/
│   │   └── airspace/    ← 공역 정보
│   ├── interuss/        ← Inter-USS 통신 (★ 핵심)
│   │   ├── in/          ← 외부 USS로부터 수신
│   │   └── out/         ← 외부 USS로 송신
│   └── psty/
│       ├── abn/         ← 이상 상황
│       ├── acdnt/       ← 사고 보고
│       ├── ctrl/        ← 관제 제어
│       └── hstry/       ← 관제 이력
├── config/
│   ├── UssFeignExceptionHandler
│   ├── UssFeignHeaderConfig
│   ├── UssJpaConfig
│   ├── UssMessageConfig
│   └── UssQuerydslConfig
└── UssPubApplication.java
```

---

## 특수 의존성

| 라이브러리 | 용도 |
|-----------|------|
| Apache POI | 엑셀 파일 처리 |
| JavaAPI for KML | 지리정보 KML 형식 |
| JAXB | XML 직렬화/역직렬화 |
| FIXM USS JAR | ICAO FIXM 표준 비행정보 교환 |

---

## Feign 연동

| 대상 | URL | 용도 |
|------|-----|------|
| [[app-web]] | localhost:8000 | 사용자 포털 |
| [[app-mngr]] | localhost:8100 | 관리자 포털 |
| [[app-engine]] | localhost:8200 | 엔진 |
| [[app-sol]] | localhost:8500 | 솔루션 포털 |

### 외부 API

| 서비스 | 용도 |
|--------|------|
| ctrlwthr-service (KMA) | 기상청 제어기상 |
| ts-service (KOTSA) | 항공교통서비스 |

---

## API 엔드포인트 상세

### USS 비행계획 (`/api/v1/web/fpl/bsc`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `` | 비행계획서 추가 |
| GET | `/plist` | 비행계획서 조회 (페이징) |
| GET | `/{fplNo}` | 비행계획서 상세 |
| GET | `/schedule` | 비행 스케줄 조회 |
| GET | `/live/by/userNo` | 실시간 비행영역/기체 |

### ★ Inter-USS API (`/utm`) - 외부 USS 연동

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `/ussapi/v1/fpl/approval` | 비행계획 승인 요청 (인바운드) |
| POST | `/ussapi/v1/fpl/callback` | 비행계획 승인 비동기 응답 |
| POST | `/openapi/v1/fpl/retrieveTotal` | 비행계획 통합 조회 |
| POST | `/openapi/v1/fpl/retrieveDetail` | 비행계획 상세 조회 |
| POST | `/ussapi/v1/fpl/registry` | 비행계획 보고(제출) |
| POST | `/ussapi/v1/fpl/updateStatus` | 비행계획 상태 업데이트 |

> Inter-USS API는 Bearer Token 인증 사용. 토큰에서 사용자 ID 추출하여 권한 검증.

### USS 관제 (`/api/v1/web/psty/ctrl`)

| HTTP | URL | 설명 |
|------|-----|------|
| GET | `/plist` | 관제 이력 (페이징) |
| GET | `/live/list` | 실시간 드론 비행현황 |
| GET | `/live/acr/dtl` | 실시간 기체 상세 |
| GET | `/smlt/stream/{fplNo}` | 시뮬레이션 SSE 스트리밍 |

---

## Inter-USS 비즈니스 로직

### 비행계획 승인 요청 흐름 (`aprvFpl`)
1. 외부 USS가 `POST /utm/ussapi/v1/fpl/approval` 호출
2. Bearer Token 검증 → 사용자 ID 추출
3. 필수 파라미터 검증 (uasFlightPlanId, Version, Type 등)
4. 비행 허가 ID 자동 생성 (UasFlightAuthorizationId)
5. FplBase + FplExtra 데이터 UssStorage에 저장
6. 승인 상태 반환 (resultCd: "01")

### SWIM 연동 (WebClient 기반)
```
SwimClient.send(body, SwimEndpoint)
→ FIXM XML 프로토콜로 항공교통 데이터 교환
→ prod: http://10.10.130.15:9500
```

### RabbitMQ 소비자
- **큐**: `q_smatii_rnd` (USS용 드론 항적)
- **처리**: SmatiiRq → GpModel 변환 → 관제 이력 저장

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[04_앱간 연관관계]] - 앱 간 통신 상세
- [[app-uss-ws]] - USS WebSocket (실시간 데이터)
- [[module-fpl]] - 비행계획 도메인
- [[module-psty]] - 관제 도메인
