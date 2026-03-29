# 시스템 구성도

## 전체 아키텍처

```mermaid
graph TB
    subgraph Client["클라이언트"]
        WEB["Web Browser"]
        APP["Mobile App"]
    end

    subgraph SUMAIR["Sumair Backend"]
        subgraph HOME["sumair-be-home :8000"]
            HOME_API["메인 API 게이트웨이<br/>컨트롤러 42개 / 서비스 56개"]
        end

        subgraph IBE["sumair-be-ibe :8100"]
            IBE_API["항공 예약 엔진<br/>컨트롤러 17개 / 서비스 52개"]
        end

        subgraph PAY["sumair-be-pay :8200"]
            PAY_API["결제 서비스<br/>컨트롤러 4개 / 서비스 4개"]
        end

        subgraph BATCH["sumair-be-batch :8300"]
            BATCH_API["배치 서비스<br/>컨트롤러 6개 / 서비스 11개"]
        end

        subgraph MSG["sumair-be-message :8400"]
            MSG_API["메시징 서비스<br/>컨트롤러 11개 / 서비스 15개"]
        end

        subgraph COMMON["sumair-be-common"]
            COMMON_LIB["공통 라이브러리<br/>Mapper 101개 / 유틸리티"]
        end
    end

    subgraph INFRA["인프라"]
        MYSQL[("MySQL / Aurora")]
        REDIS[("Redis / Valkey<br/>:7379")]
    end

    subgraph EXTERNAL["외부 시스템"]
        IBS["IBS (IBES/IBEP)<br/>SOAP"]
        TOSS["Toss Payments<br/>REST"]
        INFOBIP["Infobip<br/>SMS/Email"]
        IRES["IRES<br/>SOAP Callback"]
        DATAGOKR["data.go.kr<br/>공휴일/날씨"]
        IBS_FTP["IBS FTP/SFTP<br/>Revenue Report"]
        SLACK["Slack<br/>Webhook"]
    end

    WEB & APP --> HOME_API

    HOME_API --> IBE_API
    HOME_API --> PAY_API
    HOME_API --> MSG_API
    BATCH_API --> IBE_API
    BATCH_API --> PAY_API
    BATCH_API --> MSG_API
    IRES -->|SendITR Callback| MSG_API

    IBE_API -->|CXF SOAP| IBS
    PAY_API -->|REST| TOSS
    MSG_API -->|REST| INFOBIP
    MSG_API -->|CXF SOAP| IRES
    MSG_API -->|Webhook| SLACK
    BATCH_API -->|REST| DATAGOKR
    BATCH_API -->|SFTP| IBS_FTP

    HOME_API & IBE_API & PAY_API & BATCH_API & MSG_API --> MYSQL
    HOME_API --> REDIS

    COMMON_LIB -.->|JAR 의존성| HOME_API
    COMMON_LIB -.->|JAR 의존성| IBE_API
    COMMON_LIB -.->|JAR 의존성| PAY_API
    COMMON_LIB -.->|JAR 의존성| BATCH_API
    COMMON_LIB -.->|JAR 의존성| MSG_API
```

## 모듈 간 통신 관계

```mermaid
graph LR
    subgraph "WebClient 기반 내부 통신"
        HOME["HOME<br/>:8000"] -->|"IbesWebClientService<br/>IbepWebClientService<br/>CheckinWebClientService"| IBE["IBE<br/>:8100"]
        HOME -->|"PayWebClientService<br/>PointWebClientService<br/>CouponWebClientService"| PAY["PAY<br/>:8200"]
        HOME -->|"SmsWebClientService<br/>EmailWebClientService<br/>InfobipWebClientService"| MSG["MESSAGE<br/>:8400"]
        BATCH["BATCH<br/>:8300"] -->|"IbeWebClientService"| IBE
        BATCH -->|"PointWebClientService"| PAY
        BATCH -->|"MsgWebClientService"| MSG
    end
```



## 모듈별 포트 및 역할

| 모듈                    | 포트   | 역할                        |
| --------------------- | ---- | ------------------------- |
| **sumair-be-common**  | -    | 공통 라이브러리 (JAR)            |
| **sumair-be-home**    | 8000 | 메인 API 게이트웨이, 인증, 캐시      |
| **sumair-be-ibe**     | 8100 | 항공 예약 엔진 (IBES/IBEP SOAP) |
| **sumair-be-pay**     | 8200 | 결제/포인트/쿠폰 처리              |
| **sumair-be-batch**   | 8300 | 배치 작업 (Quartz 스케줄러)       |
| **sumair-be-message** | 8400 | SMS/이메일 메시징               |

## 환경 구성

```mermaid
graph TB
    subgraph "환경 프로파일"
        LOCAL["local<br/>로컬 개발"]
        DEV["dev<br/>개발 서버"]
        PROD["prod<br/>운영 서버"]
    end

    subgraph "설정 파일 구조 (common)"
        A["application-common.yml"]
        B["application-databases.yml"]
        C["application-ibe.yml"]
        D["application-init.yml"]
    end

    subgraph "배포"
        CB["AWS CodeBuild"]
        CD["AWS CodeDeploy"]
        CB --> CD
    end

    subgraph "인프라"
        EC["ElastiCache<br/>(Valkey)"]
        AU["Aurora MySQL"]
    end
```

## 외부 연동 상세

| 외부 시스템            | 프로토콜       | 연동 모듈   | 용도                                      |
| ----------------- | ---------- | ------- | --------------------------------------- |
| **IBES** (IBS)    | SOAP (CXF) | IBE     | 항공편 조회, 가격 확인, 시간표, 수하물, 부가서비스          |
| **IBEP** (IBS)    | SOAP (CXF) | IBE     | 예약 생성/변경/취소/분리, 체크인, 좌석 배정, 프로필         |
| **Toss Payments** | REST       | PAY     | 결제 승인/취소/조회, BrandPay 액세스토큰             |
| **Infobip**       | REST       | MESSAGE | SMS 발송, 이메일 발송, 템플릿 관리                  |
| **IRES**          | SOAP (CXF) | MESSAGE | SendITR 콜백 수신 → 이티켓 이메일 발송              |
| **data.go.kr**    | REST (XML) | BATCH   | 공휴일 API, 날씨(동네예보) API                   |
| **IBS FTP/SFTP**  | SFTP       | BATCH   | Revenue Report (Excel), Flight Schedule |
| **Slack**         | Webhook    | MESSAGE | 내부 에러 알림                                |
| **OAuth2**        | REST       | HOME    | Naver, Kakao, Google 소셜 로그인             |
