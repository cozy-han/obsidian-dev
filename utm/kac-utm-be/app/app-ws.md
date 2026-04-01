# App: WS (WebSocket 서버)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 8300 |
| **JAR명** | utm-ws |
| **패키지** | `com.kac.utm.ws` |
| **메인 클래스** | (WebFlux 기반) |
| **클래스 수** | 37개 |

---

## 역할

일반 사용자/관리자 화면에 **실시간 데이터를 스트리밍**하는 WebSocket 서버이다.
드론 위치, 관제 상태, 센서 데이터 등을 실시간으로 클라이언트에 전달한다.

> **언제 이 코드를 보게 되나?**
> 실시간 지도 위 드론 위치 표시, 관제 화면 실시간 업데이트 등 WebSocket 관련 기능을 개발하거나 디버깅할 때.

---

## 패키지 구조

```
com.kac.utm.ws/
├── api/
│   ├── controller/      ← CommonController, SocketReceiverController
│   └── service/         ← SocketReceiverService
├── collection/          ← 데이터 컬렉션 관리
├── config/              ← Redis, WebSocket 설정
├── context/             ← 컨텍스트 관리
├── gsl/
│   └── model/           ← 지리정보 모델
├── handler/             ← WebSocket 메시지 핸들러
├── model/               ← 데이터 모델
├── redis/
│   └── model/           ← Redis 데이터 모델
└── scheduler/
    └── ctr/
        └── service/     ← CtrCntrlTaskService (제어 작업)
```

---

## 기술 스택

- **Spring WebFlux**: 비동기/논블로킹 기반
- **Redis (Reactive)**: Lettuce 기반 리액티브 Redis 클라이언트
- **Spring Boot Actuator**: 모니터링

---

## Redis 환경별 설정

| 환경 | 호스트 | 모드 |
|------|--------|------|
| local | localhost:16379 | Standalone |
| dev | 54.180.88.137:6379 | Standalone |
| prod | 10.10.160.26-28:26379 | Sentinel |

---

## 연동

| 대상 | 용도 |
|------|------|
| [[app-engine]] (8200) | 엔진에서 처리된 데이터 수신 |
| Redis | 실시간 데이터 캐싱/조회 |
| 클라이언트 (브라우저) | WebSocket 연결 |

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[app-engine]] - 데이터 원천
- [[app-uss-ws]] - USS용 WebSocket (유사한 구조)
