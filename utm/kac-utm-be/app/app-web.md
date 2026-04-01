# App: Web (일반 사용자 포털)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 8000 |
| **JAR명** | utm-web |
| **패키지** | `com.kac.utm.web` |
| **메인 클래스** | `WebApplication.java` |
| **프로필** | db, security, web, redis |
| **사이트 ID** | WEB |
| **클래스 수** | 278개 |

---

## 역할

일반 사용자(드론 조종사, 기업 회원 등)가 사용하는 **웹 포털 API 서버**이다.
비행계획 조회, 기상정보 확인, 공지사항 열람, 회원가입/로그인 등 사용자 대면 기능을 제공한다.

> **언제 이 코드를 보게 되나?**
> 사용자 화면에서 호출하는 API를 개발하거나 수정할 때. 프론트엔드 개발자가 "이 API 응답이 이상해요"라고 할 때 가장 먼저 보는 모듈.

---

## 의존성

### 외부 라이브러리
- Spring Boot Web, WebFlux
- Spring Data JPA, Redis
- QueryDSL
- Spring Security, OAuth2 (카카오/네이버)
- JWT (jjwt 0.12.6)
- GeoTools, Hibernate Spatial (공간 데이터)
- Thymeleaf (이메일 템플릿 등)
- Spring REST Docs

### 프로젝트 내부 모듈
```
:module:module-common   ← 사용자, 권한, 코드 등
:module:module-conts    ← 게시판, FAQ, Q&A
:common:common-sys      ← 시스템 알림
:common:common-app      ← Firebase
:common:common-connector ← 외부 API (기상, 이메일)
:common:common-config   ← 환경 설정
:common:common-exception ← 예외 처리
:common:common-utils    ← 유틸리티
:common:common-redis    ← Redis
:common:common-models   ← 공통 모델
```

---

## 패키지 구조

```
com.kac.utm.web/
├── aerl/              ← 항공정보 (센서, 날씨)
│   ├── api/
│   ├── service/
│   └── model/
├── common/            ← 공통 기능 (알림, 코드, 기업, 인증)
│   ├── api/
│   ├── service/
│   ├── model/
│   └── util/
├── conts/             ← 콘텐츠 (게시판, FAQ, Q&A)
│   ├── api/
│   ├── service/
│   └── model/
├── fpl/               ← 비행계획
│   ├── api/
│   ├── service/
│   └── model/
├── gsl/               ← 공간정보 (공역)
│   ├── api/
│   ├── service/
│   └── model/
├── psty/              ← 관제 (이상상황, 사고)
│   ├── api/
│   ├── service/
│   └── model/
├── config/            ← 설정 클래스
└── WebApplication.java
```

---

## 주요 컨트롤러

| 컨트롤러 | 기능 |
|----------|------|
| WebAuthrtController | 인증/인가 (로그인, 토큰) |
| ComCodeController | 공통 코드 조회 |
| ComEntController | 기업 정보 관리 |
| ComAlrtController | 알림 관리 |
| FileController | 파일 업로드/다운로드 |
| NiceValidController | NICE 본인인증 |
| AerlSnsSnrController | 센서 정보 |
| AerlWthrController | 기상 정보 (METAR, TAF 등) |

---

## Feign 연동

| 대상 | URL | 용도 |
|------|-----|------|
| web (자기 자신) | localhost:8000 | 내부 호출 |
| [[app-mngr]] | localhost:8100 | 관리자 기능 연동 |
| [[app-engine]] | localhost:8200 | 엔진 기능 연동 |

---

## API 엔드포인트 상세

### 인증/사용자 (`/api/v1/web/authrt`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `/login` | 로그인 → WebLgnRs (토큰 발급) |
| POST | `/login/mobile/sns` | 모바일 SNS 로그인 (카카오/네이버) |
| GET | `/logout` | 로그아웃 |
| POST | `/sign-up` | 회원가입 |
| DELETE | `/whdwl` | 회원탈퇴 |
| POST | `/valid/user` | 중복 체크 (ID, 전화번호, 이메일, 사업자번호) |
| POST | `/valid/pswd` | 비밀번호 검증 |
| GET | `/refresh` | Access Token 갱신 (Refresh Token 사용) |
| GET | `/valid/token` | 토큰 유효성 검증 |
| POST | `/find/id` | 아이디 찾기 |
| PATCH | `/find/pswd/chg` | 비밀번호 재설정 |
| POST | `/chg/pswd` | 비밀번호 변경 |
| POST | `/extend/pswd` | 비밀번호 다음에 변경하기 |
| GET | `/profile` | 프로필 조회 |
| PUT | `/upd/info` | 사용자 정보 수정 |
| PUT | `/upd/mbltelno` | 휴대폰 번호 변경 |
| GET | `/menu/list` | 메뉴 조회 |
| POST | `/sns-link` | SNS 연동 |
| DELETE | `/sns-link` | SNS 연동 해제 |
| GET | `/mypage/current` | 마이페이지 현황 |
| GET | `/mypage/hstry` | 마이페이지 이력 |
| GET | `/mainpage` | 메인페이지 데이터 |
| PATCH | `/chg-authrt/temp` | 권한(구독) 변경 |

### 비행계획 (`/api/v1/web/fpl/bsc`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `` | 비행계획서 등록 → Engine 호출 |
| GET | `/plist` | 비행계획서 조회 (페이징) |
| GET | `/slist` | 비행계획서 조회 (스크롤) |
| GET | `/{fplNo}` | 비행계획서 상세 |
| GET | `/schedule` | 비행 스케줄 조회 |
| GET | `/ofdoc` | 비행계획서 공문 출력 |
| POST | `/send` | 비행계획서 신청내역 전송 |

### 기체 관리 (`/api/v1/web/fpl/acr`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `` | 기체 등록 (파일 포함) |
| PUT | `` | 기체 수정 |
| DELETE | `` | 기체 삭제 |
| GET | `/plist` | 기체 목록 (페이징) |
| GET | `/slist` | 기체 목록 (스크롤) |
| GET | `/hstry/slist` | 기체 변경 이력 |
| GET | `/load/ts` | TS 기체정보 조회 (KOTSA 연동) |

### 조종사 관리 (`/api/v1/web/fpl/plt`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `/cust` | 고객 조종사 등록 |
| PUT | `/cust` | 고객 조종사 수정 |
| DELETE | `/cust` | 고객 조종사 삭제 |
| GET | `/cust/slist` | 고객 조종사 목록 |

### 비행 허가 (`/api/v1/web/fpl/prmsn`)

| HTTP | URL | 설명 |
|------|-----|------|
| GET | `/plist` | 허가 요청 목록 |
| GET | `/setting/data` | 허가 설정 데이터 |
| POST | `/setting` | 허가 설정 추가 |
| PUT | `/setting` | 허가 설정 수정 |
| PATCH | `/request` | 허가 요청 전송 |

### 비행 영역 (`/api/v1/web/fpl/rlm`)

| HTTP | URL | 설명 |
|------|-----|------|
| GET | `/list` | 나의 비행 영역 조회 |
| POST | `` | 영역 저장 |
| PUT | `` | 영역 수정/삭제 |
| GET | `/current/prmsn` | 현 시각 비행승인 영역 |

### 관제 (`/api/v1/web/psty/ctrl`)

| HTTP | URL | 설명 |
|------|-----|------|
| GET | `/plist` | 관제 이력 (페이징) |
| GET | `/live/list` | 실시간 드론 비행현황 |
| GET | `/live/acr/dtl` | 실시간 기체 상세 |
| GET | `/live/rlm/{idntfNo}` | 드론 비행계획 영역 |
| GET | `/smlt/plist` | 비행 시뮬레이션 목록 |
| GET | `/smlt/stream/{fplNo}` | 시뮬레이션 SSE 스트리밍 |

### 비정상 상황 (`/api/v1/web/psty/abn`)

| HTTP | URL | 설명 |
|------|-----|------|
| GET | `/plist` | 비정상 상황 이력 |
| GET | `/live/list` | 실시간 비정상상황 |

### 사고보고 (`/api/v1/web/psty/acdnt/rpt`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `` | 사고보고 등록 (파일 포함) |
| PUT | `` | 사고보고 수정 |
| GET | `/slist` | 사고보고 목록 |
| POST | `/cncl` | 사고보고 취소 |

### 비행 중단 (`/api/v1/web/fpl/itrp`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `` | 비행 중단 등록 |
| PUT | `` | 비행 중단 수정 |
| GET | `/slist` | 비행 중단 목록 |
| POST | `/cncl` | 비행 중단 취소 |

---

## 주요 비즈니스 흐름

### 로그인 흐름
1. `POST /login` → BCrypt 비밀번호 검증
2. 로그인 실패 5회 시 계정 잠금 (`lgnLckYn`)
3. 성공 시 JWT Access Token + Refresh Token 발급
4. 비밀번호 변경 기한 초과 시 변경 안내

### 비행계획 신청 흐름
1. 사용자가 `POST /api/v1/web/fpl/bsc` 호출
2. web → `EngineFplClient.crtFpl()` → engine 호출
3. engine에서 공역 충돌 검사 후 결과 반환
4. 상태: APPLY → REVIEW → APPROVED/REJECTED

### OAuth2 소셜 로그인
- 카카오, 네이버 지원
- `POST /login/mobile/sns` (모바일)
- SNS 연동/해제 API 별도 제공

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[04_앱간 연관관계]] - 앱 간 통신 상세
- [[app-mngr]] - 관리자 포털
- [[module-common]] - 공통 도메인
- [[module-fpl]] - 비행계획 도메인
