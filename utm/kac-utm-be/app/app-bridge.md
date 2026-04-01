# App: Bridge (외부 시스템 브릿지)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 8900 |
| **JAR명** | utm-bridge |
| **패키지** | `com.kac.utm.bridge` |
| **메인 클래스** | `BridgeApplication.java` |
| **클래스 수** | 21개 |

---

## 역할

**외부 물리적 장비와의 통신을 중계**하는 브릿지 모듈이다. 불법 드론 스캐너와 WebSocket으로 통신하고, UTM 시스템과 MQTT/TCP로 연결한다.

> **언제 이 코드를 보게 되나?**
> 외부 드론 스캐너 연동 문제, MQTT 메시지 수신 문제가 발생할 때. 새로운 외부 장비를 연동해야 할 때.

---

## 패키지 구조

```
com.kac.utm.bridge/
├── api/
│   └── controller/      ← HealthController
├── config/              ← 일반 설정
├── illegal/
│   ├── config/          ← 불법 드론 스캐너 클라이언트 설정
│   └── model/           ← 불법 드론 데이터 모델
├── utm/
│   ├── client/          ← UTM MQTT/TCP 클라이언트
│   └── config/          ← UTM 클라이언트 설정
└── BridgeApplication.java
```

---

## 통신 구조

### 불법 드론 스캐너 (WebSocket)

```
드론 스캐너 장비 ──WebSocket──→ Bridge ──→ WS(8300) ──→ 관제 화면
```

| 환경 | 스캐너 URL |
|------|-----------|
| local | `ws://localhost:8300/ws` (2개) |
| prod | `ws://10.10.170.50:22002/[1-3]/sone/scanner` (3개 채널) |

### UTM MQTT 클라이언트

```
드론 장비 ──MQTT──→ MQTT Broker ──→ Bridge ──→ 내부 시스템
```

| 환경 | MQTT 브로커 |
|------|-----------|
| local | `localhost:31883` |
| prod | `10.10.50.250:12900` (kac), `52.78.183.35:12900` (palnet) |

---

## 의존성

- Spring Boot WebFlux
- Spring WebSocket
- Spring Integration MQTT
- Spring Integration Core
- Spring Integration IP (TCP)

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[app-ws]] - WebSocket 서버 (데이터 전달 대상)
- [[03_인프라 및 환경설정]] - MQTT/RabbitMQ 설정
