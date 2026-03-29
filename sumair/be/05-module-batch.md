# sumair-be-batch

배치 작업 서비스. Spring Batch + Quartz를 사용하여 스케줄 기반 작업 수행.

## 기본 정보
- ArtifactId: `sumair-batch`
- Port: **8300**
- Java 파일: 83개
- 의존: `sumair-common`

## 패키지 구조

```
com.sumair.batch
├── config/
│   └── quartz/              # Quartz 스케줄러 설정
├── core/
│   ├── aop/                 # 배치 로깅 AOP
│   ├── properties/          # 설정 프로퍼티
│   └── webclient/           # 외부 서비스 클라이언트
│       ├── WebcheckinWebClientService
│       ├── HolidayWebClientService
│       ├── PointWebClientService
│       ├── CustomerWebClientService
│       ├── ScheduleWebClientService
│       └── WeatherWebClientService
├── holiday/                 # 공휴일 동기화
├── schedule/                # 운항 스케줄 배치
├── point/                   # 포인트 처리
├── admin/                   # 관리 배치 작업
├── reservation/             # 예약 처리
├── webcheckin/              # 웹 체크인 자동화
├── weather/                 # 날씨 데이터 수집
├── customer/                # 고객 데이터 동기화
├── job/                     # 배치 잡 정의
└── common/chc/              # 헬스체크
```

## 배치 작업 목록

| 작업 | 설명 |
|------|------|
| holiday | data.go.kr 공휴일 API에서 데이터 동기화 |
| weather | data.go.kr 날씨 API에서 기상 데이터 수집 |
| schedule | IBS FTP에서 운항 스케줄 동기화 |
| reservation | 예약 상태 처리 및 업데이트 |
| webcheckin | 웹 체크인 자동 처리 |
| point | 포인트 적립/만료 처리 |
| customer | 고객 데이터 동기화 |
| admin | 관리 작업 |

## 외부 API 연동

| API | URL |
|-----|-----|
| 공휴일 | `apis.data.go.kr/B090041/openapi/service/SpcdeInfoService` |
| 날씨 | `apis.data.go.kr/1360000/VilageFcstInfoService_2.0` |
| IBS FTP | `secureftp-apn2.ibsplc.aero:22` |

## 주요 의존성

| 라이브러리 | 용도 |
|-----------|------|
| spring-boot-starter-batch | Spring Batch |
| spring-boot-starter-quartz | Quartz 스케줄러 |
| c3p0 0.9.2.1 | 커넥션 풀 (Quartz용) |

## 설정 파일
- `application.yml` - 메인 설정
- `application-quartz.yml` - Quartz 스케줄러 크론 설정
