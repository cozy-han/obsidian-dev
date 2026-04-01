# App: USS-WS (USS WebSocket 서버)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 9300 |
| **JAR명** | uss-ws |
| **패키지** | `com.kac.utm.uss.websocket` |
| **클래스 수** | 26개 |

---

## 역할

USS(드론운영사) 전용 **WebSocket 서버**이다. [[app-ws]](일반 WebSocket, 포트 8300)와 유사한 구조이지만, USS 전용 실시간 데이터를 처리한다.

> **언제 이 코드를 보게 되나?**
> USS 관제 화면의 실시간 데이터 표시에 문제가 있을 때. [[app-ws]]와 구조가 거의 동일하므로 ws를 이해하면 이 모듈도 쉽게 파악 가능.

---

## 패키지 구조

```
com.kac.utm.uss.websocket/
├── api/
│   ├── controller/      ← SocketReceiverController
│   └── service/         ← SocketReceiverService
├── collection/          ← 컬렉션 관리
├── common/              ← 공통 기능
├── config/              ← WebSocket 설정
├── context/             ← 컨텍스트
├── gsl/
│   └── model/           ← 지리정보 모델
├── handler/             ← 메시지 핸들러
├── model/               ← 데이터 모델
└── scheduler/
    └── ctr/
        └── service/     ← CtrCntrlTaskService
```

---

## 설정

| 환경 | USS 호스트 |
|------|-----------|
| local | `http://127.0.0.1:9000` |
| prod | `http://10.10.160.11:9000` |

---

## ws와의 차이점

| 항목 | [[app-ws]] | uss-ws |
|------|---------|--------|
| 포트 | 8300 | 9300 |
| 대상 | 일반 사용자/관리자 | USS(드론운영사) |
| 데이터 원천 | [[app-engine]] | [[app-uss-pub]] |
| Redis | 사용 | 미사용 |

---

## 관련 문서
- [[app-ws]] - 일반 WebSocket (유사 구조)
- [[app-uss-pub]] - USS 퍼블릭 서비스
