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

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[app-web]] - 일반 사용자 포털
- [[app-batch]] - 배치 모듈
- [[module-common]] - 공통 도메인
