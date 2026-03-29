# sumair-be-message

메시징 서비스. Infobip 게이트웨이와 IRES SOAP을 통한 SMS/이메일 발송.

## 기본 정보
- ArtifactId: `sumair-message`
- Port: **8400**
- Java 파일: 83개
- 의존: `sumair-common`
- SOAP: Apache CXF 3.5.5

## 패키지 구조

```
com.sumair.message
├── infobip/
│   ├── controller/      # InfobipTemplateController
│   ├── service/         # InfobipTemplateService (템플릿 관리)
│   ├── mapper/          # InfobipTemplateMapper
│   └── model/           # 이메일 모델
├── sms/
│   ├── schedule/        # 예약 SMS 발송
│   ├── reservation/     # 예약 확인 SMS
│   ├── webcheckin/      # 체크인 알림 SMS
│   ├── members/         # 회원 SMS
│   └── cstmrservice/    # 고객 서비스 SMS
├── email/
│   ├── schedule/        # 예약 이메일 발송
│   ├── reservation/     # 예약 확인 이메일
│   ├── webcheckin/      # 체크인 알림 이메일
│   ├── members/         # 회원 이메일
│   └── cstmrservice/    # 고객 서비스 이메일
├── config/webclient/
│   ├── IbesWebClientService
│   ├── InfobipSmsWebClientService
│   └── InfobipMailWebClientService
├── core/
│   ├── aop/             # 로깅/모니터링
│   └── util/            # 유틸리티
├── ires/                # IRES SOAP 연동
│   ├── IresWebService
│   └── IresMapper
└── common/chc/          # 헬스체크
```

## 메시지 유형별 처리

| 유형 | SMS | Email |
|------|-----|-------|
| 예약 확인 | SmsReservationService | EmailReservationService |
| 웹 체크인 | SmsWebcheckinService | EmailWebcheckinService |
| 스케줄 발송 | SmsScheduleService | EmailScheduleService |
| 회원 알림 | SmsMembersService | EmailMembersService |
| 고객 서비스 | SmsCstmrServiceService | EmailCstmrServiceService |

## Infobip 설정

| 항목 | 값 |
|------|---|
| 발신 이메일 | `sumair@mail.sumair.kr` |
| 발신 SMS | `8218774325` (국가코드 82) |
| 통신 방식 | REST API |

## 이중 메시징 게이트웨이
- **Infobip**: 현대적 REST API 기반 (주 채널)
- **IRES**: 레거시 SOAP 기반 (IBS 시스템 연동)

## 핵심 패턴
- 템플릿 기반 메시지 생성
- SMS/이메일 병렬 구조 (동일 비즈니스 로직, 채널만 다름)
- 트랜잭션 메시지 (예약/체크인 등 이벤트 기반)
- CXF SOAP으로 레거시 시스템 연동
- 배치 서비스에서 트리거하는 예약 발송
