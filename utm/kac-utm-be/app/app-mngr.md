# App: Mngr (관리자 포털)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 8100 |
| **JAR명** | utm-mngr |
| **패키지** | `com.kac.utm.mngr` |
| **메인 클래스** | `MngrApplication.java` |
| **프로필** | db, security, mngr, gsl |
| **사이트 ID** | MNGR |
| **클래스 수** | 355개 |

---

## 역할

**시스템 관리자**가 사용하는 관리 포털 API 서버이다.
사용자 관리, 권한 설정, 비행계획 검토/승인, 코드 관리, 배치 작업 관리, 통계 조회 등 관리 업무를 처리한다.

> **언제 이 코드를 보게 되나?**
> 관리자 화면 기능을 개발하거나, 비행계획 승인 프로세스를 수정할 때. 프로젝트에서 가장 많은 기능을 담고 있는 모듈(355개 클래스).

---

## 패키지 구조

```
com.kac.utm.mngr/
├── module/
│   ├── aerl/      ← 항공정보 관리
│   ├── batch/     ← 배치 작업 관리
│   ├── common/    ← 공통 관리 (사용자, 권한, 코드, 메뉴, 파일)
│   ├── conts/     ← 콘텐츠 관리
│   ├── fpl/       ← 비행계획 관리
│   ├── gsl/       ← 공간정보 관리
│   ├── psty/      ← 관제 관리
│   └── stats/     ← 통계 관리
├── config/
│   ├── ApiLogInterceptor       ← API 로그 인터셉터
│   ├── MngrFeignExceptionHandler ← Feign 예외 처리
│   ├── MngrJpaConfig           ← JPA 설정
│   └── MngrQuerydslConfig      ← QueryDSL 설정
└── MngrApplication.java
```

---

## 주요 컨트롤러

| 컨트롤러 | 기능 |
|----------|------|
| AclController | 접근 제어(ACL) 관리 |
| AuthrtController | 권한 관리 |
| UserController | 사용자 관리 |
| MngrController | 관리자 정보 관리 |
| CodeController | 공통 코드 관리 |
| FileController | 파일 관리 |
| AlrtController | 알림 관리 |
| BatchController | 배치 작업 관리 (Quartz) |
| SolAplyController | 솔루션 신청 관리 |

---

## Feign 연동

| 대상 | URL | 용도 |
|------|-----|------|
| [[app-web]] | localhost:8000 | 사용자 포털 연동 |
| [[app-engine]] | localhost:8200 | 엔진 기능 연동 |
| [[app-sol]] | localhost:8500 | 솔루션 포털 연동 |
| [[app-batch]] | localhost:8800 | 배치 작업 관리 |

---

## 특이사항

- **기본 비밀번호 리셋값**: `kacutm12!@` (관리자가 사용자 비밀번호 초기화 시 사용)
- **ApiLogInterceptor**: 모든 API 호출을 로깅하는 인터셉터가 설정되어 있음
- 가장 많은 Feign 연동을 가진 모듈 (4개 서비스와 통신)

---

## API 엔드포인트 상세

### 관리자 인증 (`/api/v1/mngr/common/authrt`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `/login` | 관리자 로그인 |
| GET | `/logout` | 로그아웃 |
| POST | `/sign-up` | 관리자 회원가입 |
| POST | `/valid/user` | 중복 체크 |
| POST | `/cert/send/no` | 인증번호 전송 |
| POST | `/cert/valid/no` | 인증번호 확인 |
| GET | `/refresh` | 토큰 갱신 |
| GET | `/profile` | 프로필 조회 |

### 사용자 관리 (`/api/v1/mngr/common/user`)

| HTTP | URL | 설명 |
|------|-----|------|
| GET | `/plist` | 전체 회원 목록 |
| PATCH | `/reset-pswd` | 비밀번호 초기화 (→ kacutm12!@) |
| PATCH | `/ent/aprv` | 기업 담당자 신청 승인/반려 |
| GET | `/inactive/plist` | 휴면 회원 목록 |
| PATCH | `/inactive/chg-status` | 휴면 해제 |
| DELETE | `/inactive` | 휴면 회원 탈퇴 |
| GET | `/stng` | 회원가입 정책 조회 |
| PUT | `/stng` | 사이트 설정 수정 |

### 관리자 계정 (`/api/v1/mngr/common/mngr`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST/PUT/DELETE | `` | 관리자 CRUD |
| PATCH | `/reset-pswd` | 비밀번호 초기화 |
| GET | `/plist` | 관리자 목록 |
| GET | `/lgn/hstry/plist` | 접속 로그 |
| GET | `/use/hstry/plist` | 이용 로그 |

### 공통 코드 (`/api/v1/mngr/common/code`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST/PUT/DELETE | `/group` | 코드 그룹 CRUD |
| POST/PUT/DELETE | `` | 코드 CRUD |

### ACL (`/api/v1/mngr/common/acl`) - 접속 IP 관리

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `/aply` | IP 접근 신청 |
| POST/PUT/DELETE | `` | 접속 IP CRUD |

### 배치 관리 (`/api/v1/mngr/batch`) → BatchClient 호출

| HTTP | URL | 설명 |
|------|-----|------|
| GET | `/job/list` | Job 상세 목록 |
| PUT | `/trigger/cron` | Cron 스케줄 변경 |
| POST | `/job/run/{jobNm}` | Job 즉시 실행 |

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[04_앱간 연관관계]] - 앱 간 통신 상세
- [[app-web]] - 일반 사용자 포털
- [[app-batch]] - 배치 모듈
- [[module-common]] - 공통 도메인
