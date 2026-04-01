# App: Sol (솔루션 제공자 포털)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 8500 |
| **JAR명** | utm-sol |
| **패키지** | `com.kac.utm.sol` |
| **메인 클래스** | `SolApplication.java` |
| **프로필** | db, security, web, redis |
| **사이트 ID** | SOL |
| **클래스 수** | 298개 |

---

## 역할

**솔루션 제공 업체**가 사용하는 포털 API 서버이다. 드론 관련 솔루션을 제공하는 업체가 비행계획을 조회하고, 자사 서비스를 관리하는 기능을 제공한다. 구조적으로 [[app-web]]과 매우 유사하다.

> **언제 이 코드를 보게 되나?**
> 솔루션 업체 전용 화면 기능을 개발할 때. web 모듈과 구조가 유사하므로 web에서 작업한 경험이 있다면 익숙하게 작업할 수 있다.

---

## 패키지 구조

```
com.kac.utm.sol/
├── module/
│   ├── aerl/      ← 항공정보
│   ├── com/       ← 공통 기능 (알림, 인증, 코드, 파일, 메뉴)
│   ├── conts/     ← 콘텐츠
│   ├── fpl/       ← 비행계획
│   ├── gsl/       ← 공간정보
│   ├── psty/      ← 관제
│   └── stats/     ← 통계
├── config/
│   ├── SolFeignExceptionHandler
│   ├── SolFeignHeaderConfig
│   ├── SolJpaConfig
│   ├── SolMessageConfig
│   └── SolQuerydslConfig
└── SolApplication.java
```

---

## 주요 컨트롤러

| 컨트롤러 | 기능 |
|----------|------|
| ComAuthrtController | 인증/인가 |
| ComCodeController | 공통 코드 |
| ComFileController | 파일 관리 |
| ComMenuController | 메뉴 관리 |
| ComPopupController | 팝업 관리 |
| NiceValidController | NICE 본인인증 |
| AerlSnsSnrController | 센서 정보 |
| AerlWthrController | 기상 정보 |

---

## Feign 연동

| 대상 | URL | 용도 |
|------|-----|------|
| [[app-web]] | localhost:8000 | 사용자 포털 연동 |
| [[app-mngr]] | localhost:8100 | 관리자 포털 연동 |
| [[app-engine]] | localhost:8200 | 엔진 기능 연동 |

---

## web과의 차이점

- **사이트 ID**: SOL (web은 WEB)
- **대상 사용자**: 솔루션 업체 (web은 일반 사용자)
- **추가 기능**: 솔루션 신청/관리 기능
- **나머지 구조**: web과 거의 동일

---

## 관련 문서
- [[app-web]] - 일반 사용자 포털 (유사 구조)
- [[app-mngr]] - 관리자 포털
