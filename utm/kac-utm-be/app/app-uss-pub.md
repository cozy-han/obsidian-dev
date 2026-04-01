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

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[app-uss-ws]] - USS WebSocket (실시간 데이터)
- [[module-fpl]] - 비행계획 도메인
- [[module-psty]] - 관제 도메인
