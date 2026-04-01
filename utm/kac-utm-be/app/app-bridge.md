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

## 통신 로직 상세

### 불법드론 스캐너 (IllegalClientManager)

```java
// ConcurrentHashMap으로 클라이언트 관리
Map<String, BridgeClient> clientMap

// 30초마다 연결 해제된 클라이언트 자동 재연결
@Scheduled(fixedDelay = 30, timeUnit = SECONDS)
illegalReconnection() {
    getDisConnectionClient()  // 연결 해제 클라이언트 필터링
    → 각 클라이언트 reconnect() 호출
}
```

### MQTT 클라이언트 (MqttUtmClient)

```
클라이언트 ID: "BRIDGE_CLIENT_" + UUID
QoS: 1 (최소 1회 전송)
Clean Session: true

send(IllegalInboundSocketModel)
→ JSON 직렬화
→ mqttClient.publish(topic, payload, qos=1, retained=false)
→ Schedulers.boundedElastic() (비동기)
```

### TCP 클라이언트 (TcpUtmClient)

```
Serializer: ByteArrayCrLfSerializer
Keep-Alive: true
Single Use: false (연결 재사용)

send(IllegalInboundSocketModel)
→ SocketPayload 변환
   - AuthKey: "35ea4080-a3f2-4e34-8361-78db06bac6fc"
   - TerminalId: "ILLEGAL_DRONE"
   - Command: "ILLEGAL"
→ JSON 직렬화
→ TCP 메시지 핸들러 전송 (IOScheduler)
```

### 데이터 모델

```
IllegalInboundSocketModel:
- droneID, droneLat, droneLon
- droneAltitude, droneSpeed, droneAngle
```

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[04_앱간 연관관계]] - 앱 간 통신 상세
- [[app-ws]] - WebSocket 서버 (데이터 전달 대상)
- [[03_인프라 및 환경설정]] - MQTT/RabbitMQ 설정
