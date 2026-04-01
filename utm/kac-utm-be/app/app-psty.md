# App: PSTY (메시지/알림 엔진)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 8400 |
| **JAR명** | utm-psty |
| **패키지** | `com.kac.utm.psty` |
| **메인 클래스** | `PstyApplication.java` |
| **프로필** | rabbitmq |
| **클래스 수** | 21개 |

---

## 역할

**메시지 큐 처리 및 알림 발송 엔진**이다. RabbitMQ를 통해 메시지를 수신하고, WebSocket 서버나 엔진에 데이터를 전달한다. Netty 기반 TCP 소켓 서버도 포함하고 있어 외부 장비와의 TCP 통신도 처리한다.

> **언제 이 코드를 보게 되나?**
> 알림이 안 오거나, 메시지 큐 처리가 지연될 때. RabbitMQ 메시지 흐름을 디버깅할 때.

---

## 패키지 구조

```
com.kac.utm.psty/
├── amqp/
│   └── model/           ← RabbitMQ 메시지 모델
├── cache/               ← 캐시 관리
├── collection/          ← 컬렉션 관리
├── command/             ← 명령어 처리 패턴
├── config/              ← RabbitMQ 설정
├── controller/
│   ├── CacheController  ← 캐시 조회/관리
│   ├── CommonController ← 헬스체크
│   └── SendController   ← 메시지 발송
├── gp/                  ← General Purpose
└── model/               ← 데이터 모델
```

---

## 연동

| 대상 | 프로토콜 | 용도 |
|------|---------|------|
| RabbitMQ | AMQP | 메시지 수신/발송 |
| [[app-ws]] (8300) | HTTP | WebSocket 서버에 데이터 전달 |
| [[app-engine]] (8200) | HTTP | 엔진 데이터 조회 |
| 외부 장비 | TCP (Netty, 포트 8082) | 장비 데이터 수신 |

---

## Command 패턴 상세

PSTY는 **Command 패턴**으로 드론 데이터를 처리한다. `SocketCommand` 클래스가 핵심.

### Command 종류

| Command | 데이터 소스 | 설명 |
|---------|-----------|------|
| `smatiiCommand` | q_smatii (RabbitMQ) | LTEM 기반 SMATII 드론 데이터 |
| `illegalCommand` | q_illegal (RabbitMQ) | 불법 드론 탐지 데이터 |
| `sandBoxCommand` | 테스트 시스템 | 드론 테스트 데이터 |

### sendData() 핵심 흐름 (Mono 파이프라인)

```
① 데이터 유효성 검사 (null/empty 체크)
   ↓
② Control ID 캐시 조회
   ├── 캐시 히트 → Control ID 설정
   └── 캐시 미스 → Engine API 비동기 호출
       POST /api/v1/engine/psty/ctrl/crt/id
       → 비행계획 매칭, 허가 여부 확인
       → Control ID 발급 + SyncMap 캐시 저장
   ↓
③ GPModel 속성 설정
   - controlId, mainControlId
   - typeCd ("START"), prmsnYn (허가 여부)
   - acrStatus (불법드론 여부)
   ↓
④ GPCollection에 추가 (메모리 버퍼)
   → 20초마다 Engine으로 배치 업로드
   ↓
⑤ WebSocket 전송 (mainControlId == null만)
   → WsClient.post(/api/ws/receiver)
```

### RabbitMQ Consumer 상세

```java
// q_smatii: SMATII 데이터 (리스트 단위 수신)
@RabbitListener(queues = "q_smatii")
queueSmatii(List<SmatiiRq>, Channel, Message)
→ SmatiiRq에서 UAS ID, 좌표, 속도, 방향 추출
→ smatiiCommand() 호출

// q_illegal: 불법드론 데이터 (단건 수신)
@RabbitListener(queues = "q_illegal")
illegalSmatii(String message, Channel, Message)
→ IllegalInboundSocketRq 역직렬화
→ illegalCommand() 호출
```

### 좌표 유효성 검사
- 한국 범위 내 좌표인지 확인 (`latlonCheck()`)
- 범위 외 좌표는 무시

---

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[04_앱간 연관관계]] - 앱 간 통신 상세
- [[app-ws]] - WebSocket 서버
- [[03_인프라 및 환경설정]] - RabbitMQ 설정
