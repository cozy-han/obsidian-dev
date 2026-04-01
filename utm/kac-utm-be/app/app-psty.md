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

## 관련 문서
- [[00_프로젝트 개요]] - 전체 아키텍처
- [[app-ws]] - WebSocket 서버
- [[03_인프라 및 환경설정]] - RabbitMQ 설정
