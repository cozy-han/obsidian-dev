# sumair-be-ibe

항공 예약 엔진. IBES(조회)/IBEP(변경) 시스템과 SOAP/JAXB로 통신하여 항공편 검색, 예약, 변경, 결제를 처리한다.

## 기본 정보
- ArtifactId: `sumair-ibe`
- Port: **8100**
- Java 파일: 154개
- 의존: `sumair-common`
- SOAP: Apache CXF 3.5.5

## 패키지 구조

```
com.sumair.ibe
├── ibes/                        # IBES (Read-Only 조회)
│   ├── retrieveairavail/        # 항공편 가용성 검색
│   ├── retrievetimetable/       # 시간표 조회
│   ├── retrievebaggage/         # 수하물 규정
│   ├── retrieveprice/           # 가격 조회
│   ├── retrieveancillary/       # 부가서비스 조회
│   └── retrievebooking/         # 예약 조회
├── ibep/                        # IBEP (Write 변경)
│   └── booking/
│       ├── create/              # 예약 생성
│       ├── cancel/              # 예약 취소
│       ├── modifybooking/       # 예약 변경
│       ├── savemodifybooking/   # 변경 저장
│       ├── splitbooking/        # 예약 분리
│       ├── syncbooking/         # 예약 동기화
│       ├── payment/             # 결제 처리
│       ├── adjustFlightInventory/ # 재고 조정
│       └── flight/
│           ├── schedule/        # 운항 스케줄
│           └── retrievepassenger/ # 탑승객 조회
├── common/                      # 공통 기능
│   ├── ord/                     # 주문 관련
│   │   ├── bas/                 # 주문 기본
│   │   ├── book/                # 예약 데이터
│   │   ├── psngr/               # 탑승객 데이터
│   │   ├── pym/                 # 결제 데이터
│   │   └── ssr/                 # 특별 서비스 요청
│   ├── airport/                 # 공항 정보
│   ├── app/                     # 앱 이벤트
│   └── chc/                     # 헬스체크
└── core/                        # 인프라
    ├── soap/                    # SOAP 서비스 (CXF)
    ├── jaxb/                    # XML/JAXB 마샬링
    ├── error/                   # IBSException 처리
    ├── handler/                 # 이벤트 핸들러
    ├── http/webclient/          # 결제 서비스 클라이언트
    └── utils/                   # ErrorUtils
```

## IBES vs IBEP 구분

| 구분 | IBES | IBEP |
|------|------|------|
| 역할 | 조회 (Read) | 변경 (Write) |
| 작업 | 항공편/가격/수하물/시간표 조회 | 예약 생성/변경/취소/결제 |
| 통신 | SOAP Request-Response | SOAP Request-Response |

## 주요 컨트롤러 (17개)
- `IbesRetrieveAirAvailController` - 항공편 가용성
- `IbesRetrieveTimeTableController` - 시간표
- `IbesRetrieveBaggageController` - 수하물
- `IbesRetrievePriceController` - 가격
- `IbepBookingFlightScheduleController` - 운항 스케줄
- `IbepBookingSaveModifyBookingController` - 예약 변경 저장

## 주요 서비스 (52개)
- `SoapService` - SOAP 통신 핵심
- `IbesRetrieveAirAvailService` - 항공편 검색
- `IbesRetrieveEnhancedAirAvailService` - 향상된 검색
- `IbepBookingCancelService` - 예약 취소
- `IbepBookingModifyBookingService` - 예약 변경

## 설정
- CXF SOAP 경로: `/ires`
- IBES 인증: `IBS_DEV` (dev) / `IBS_PRD` (prod)
- SOAP 메시지 로깅 인터셉터 (`MessageChangetInterceptor`)

## 핵심 패턴
- Apache CXF SOAP 클라이언트
- JAXB 마샬링으로 XML 변환
- IBES(조회) / IBEP(변경) 양방향 분리
- SOAP 메시지 파일 로깅 (`logs/soap-files/`)
- 복합 도메인 모델 (항공편/예약/결제)
