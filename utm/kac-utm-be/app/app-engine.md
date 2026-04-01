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

## API 엔드포인트 상세

### 비행계획 (`/api/v1/engine/fpl`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `` | 비행계획서 생성 (공역 충돌 검사 포함) |
| GET | `/list` | 비행계획서 풀 데이터 조회 |
| GET | `/plist` | 비행계획서 페이징 조회 |
| GET | `/{fplNo}` | 비행계획서 상세 |
| GET | `/by/idntfNo` | 식별번호별 비행영역 조회 |
| GET | `/live/by/userNo` | 사용자별 실시간 비행영역 |
| GET | `/schedule/list` | 스케줄 목록 |
| PUT | `/prgrs` | 비행계획 진행상태 변경 |
| GET | `/mngr/plist` | 관리자 자동비행승인 관리 |
| GET | `/ws/refresh/event` | WebSocket 이벤트 갱신 |
| DELETE | `/ctrl` | Redis 관제 데이터 삭제 |

### 기체 (`/api/v1/engine/fpl/acr`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `` | 기체 추가 (중복검사: 식별/신고/제조번호) |
| PUT | `` | 기체 수정 |
| DELETE | `` | 기체 삭제 |
| GET | `/by/idntfNo` | 식별번호별 기체 조회 |
| GET | `/load/ts` | KOTSA TS 기체정보 불러오기 |

### 비행 허가 (`/api/v1/engine/fpl/prmsn`)

| HTTP | URL | 설명 |
|------|-----|------|
| PUT | `/aprv` | 허가 승인 |
| PUT | `/unaprv` | 허가 미승인 |
| PUT | `/request` | 허가 요청 전송 |
| POST | `/setting` | 허가 설정 추가 |

### 관제 (`/api/v1/engine/psty/ctrl`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `/crt/id` | ★ 관제 ID 발급 (비행계획 매칭, 허가 확인) |
| GET | `/plist` | 비행 이력 목록 |
| GET | `/live/acr/dtl` | 실시간 기체 상세 |
| PUT | `/end` | 관제 종료 |

### 비정상 상황 (`/api/v1/engine/psty/abn`)

| HTTP | URL | 설명 |
|------|-----|------|
| GET | `/plist` | 비정상 상황 조회 |
| POST | `/live/list` | 실시간 비정상상황 건수 |

### 공역 (`/api/v1/engine/gsl/airspace`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `/list` | 좌표 기반 공역 조회 |
| POST | `/all/list` | 전체 공역 조회 |
| POST | `/notam/list` | NOTAM 조회 |
| POST | `/valid/control` | 영역 관제권 유효성 확인 |

---

## 핵심 비즈니스 로직 상세

### 비행계획 생성 (`FplBscService.crtFpl()`)
```
1. 비행계획 기본정보 저장
2. 비행영역(RLM) 좌표 → JTS Geometry 변환
   - RlmType: POINT, POLYGON, CIRCLE
   - Buffer(완충거리) 적용 → 다각형 생성
   - GeoJSON 변환 및 저장
3. ★ 공역 충돌 검사 ★
   getViolatedCmptncInstSpaces()
   - 비행영역 Geometry vs 관제권 Geometry 교차 판정
   - STRtree 공간 인덱스 사용
   - 충돌 영역별 검토대기(WAIT) 항목 생성
4. 조종사/기체 정보 저장
5. 동적 스케줄러 등록 (비행 시작/종료 시간)
```

### 관제 ID 발급 (`PstyCtrlService.crtId()`)
```
1. 관제 번호 생성
2. 식별번호 → 비행계획 매칭 (validYn)
3. 동시 비행 기체 확인
   - GENERAL/ETC: 기존 메인 관제번호 설정
   - ILLEGAL: 불법드론 → 캐시 제거, WebSocket 중단
4. 허가 여부 확인 (prmsnYn)
5. 관제 상태 코드 반환 (START, GROUND 등)
```

### 기체 등록 중복 검사
- 식별번호(idntfNo) 중복 → `WEB_DUPLICATED_IDNTF_NO`
- 신고번호(dclrNo) 중복 → `WEB_DUPLICATED_DCLR_NO`
- 제조번호(fbctnNo) 중복 → `WEB_DUPLICATED_FBCTN_NO`

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[04_앱간 연관관계]] - 앱 간 통신 상세
- [[app-ws]] - WebSocket (실시간 데이터 전달)
- [[module-fpl]] - 비행계획 도메인
- [[module-gsl]] - 공간정보 도메인
