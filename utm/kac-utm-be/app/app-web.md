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

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[app-mngr]] - 관리자 포털
- [[module-common]] - 공통 도메인
- [[module-fpl]] - 비행계획 도메인
