# App: Engine (UTM 핵심 엔진)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 8200 |
| **JAR명** | utm-engine |
| **패키지** | `com.kac.utm.engine` |
| **메인 클래스** | `EngineApplication.java` |
| **프로필** | connector, db, gsl, engine, redis |
| **사이트 ID** | ENGINE |
| **클래스 수** | 88개 |

---

## 역할

UTM 시스템의 **핵심 처리 엔진**이다. 비행계획 검증, 공역 충돌 검사, 센서 데이터 처리, 통계 생성 등 핵심 비즈니스 로직을 담당한다. 다른 모듈(web, mngr, sol, uss-pub)이 Feign으로 호출하는 **백엔드의 백엔드** 역할.

> **언제 이 코드를 보게 되나?**
> 비행계획 검증 로직, 공역 충돌 판단, 통계 처리 등 핵심 비즈니스 로직을 수정할 때. "왜 비행계획이 반려되었는지" 디버깅할 때.

---

## 패키지 구조

```
com.kac.utm.engine/
├── common/
│   └── api/             ← CommonController (헬스체크)
├── config/              ← 설정
├── domain/
│   ├── aerl/            ← 항공정보 처리
│   │   ├── api/
│   │   └── service/
│   ├── dos/             ← DOS 비행계획서 처리
│   │   ├── api/
│   │   ├── model/
│   │   └── service/
│   ├── fpl/             ← 비행계획 검증/처리
│   │   ├── api/
│   │   ├── model/
│   │   ├── scheduler/   ← 비행 스케줄러
│   │   └── service/
│   ├── gsl/             ← 공간정보/공역 처리
│   │   ├── api/
│   │   └── service/
│   ├── psty/            ← 관제 처리
│   │   ├── api/
│   │   └── service/
│   └── stats/           ← 통계 처리
│       ├── api/
│       └── service/
└── EngineApplication.java
```

---

## 주요 컨트롤러

| 컨트롤러 | 기능 |
|----------|------|
| FplBscController | 비행계획 기본 처리 |
| FplAcrController | 항공기 관리 |
| FplItrpController | 비행 경로 처리 |
| DosBscController | DOS 비행계획서 |
| GslAirspaceController | 공역 관리/검색 |
| AerlSnrSnsController | 센서 데이터 처리 |
| AerlWthrController | 기상 정보 처리 |
| PstyAbnController | 이상상황 처리 |
| PstyCntrlController | 관제 제어 |
| StatsAbnSitController | 이상상황 통계 |
| StatsFlyngController | 비행 통계 |

---

## 특별 기능

- **@EnableScheduling**: 비행 스케줄러 활성화 (자동 비행 상태 갱신)
- **@EnableAsync**: 비동기 처리 활성화
- **GeoTools**: 공역 기하학적 연산 (포함, 교차, 버퍼 등)
- **Naver API**: 역지오코딩 (좌표 → 주소 변환)
- **VWorld API**: 주소 변환

---

## 외부 연동

| 서비스 | 용도 |
|--------|------|
| KOTSA eDrone | 항공교통서비스 |
| KMA 기상 | METAR, TAF, SIGMET, AIRMET |
| 제어기상 | 읍면단위 예보 |
| Naver Map | 역지오코딩 |
| VWorld | 주소 변환 |

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[app-ws]] - WebSocket (실시간 데이터 전달)
- [[module-fpl]] - 비행계획 도메인
- [[module-gsl]] - 공간정보 도메인
