# 모듈별 패키지(폴더) 구조

> 각 모듈의 Java 소스 패키지 구조와 resources 구성

---

## 공통 패턴

모든 모듈은 **Controller → Service → ServiceImpl → Mapper** 계층 구조를 따른다.
기능 단위 패키지 안에 `controller/`, `service/`, `service/impl/`, `mapper/`, `model/`을 두는 **기능 기반 패키지 구조**를 사용한다.

---

## 1. sumair-be-common

> 기본 패키지: `com.sumair.common`

공유 라이브러리 모듈. 단독 실행 불가, 다른 모듈에서 의존성으로 참조.

```
com.sumair.common
├── core/                          # 핵심 프레임워크
│   ├── annotaion/                 # 커스텀 어노테이션
│   │   ├── extractor/
│   │   └── model/
│   ├── exception/                 # 전역 예외 처리
│   │   ├── handler/
│   │   ├── model/
│   │   └── type/
│   ├── http/                      # HTTP 클라이언트
│   │   └── webclient/
│   │       ├── model/
│   │       └── service/
│   ├── message/                   # 메시지 유틸
│   ├── properties/                # 프로퍼티 바인딩
│   ├── rest/                      # REST 통신
│   │   ├── config/ (impl/)
│   │   ├── model/
│   │   └── service/ (impl/)
│   ├── security/                  # 보안 모듈
│   │   ├── crypto/                # 암호화 (AES 등)
│   │   │   ├── advice/
│   │   │   ├── annotaion/
│   │   │   └── type/
│   │   ├── jasypt/                # Jasypt 설정 암호화
│   │   ├── masking/               # 개인정보 마스킹
│   │   │   ├── advice/
│   │   │   ├── annotation/
│   │   │   ├── config/
│   │   │   ├── model/
│   │   │   └── type/
│   │   └── secur/                 # 인증/인가
│   │       └── annotation/
│   ├── seq/                       # 채번(시퀀스) 생성
│   │   └── generate/service/
│   ├── slack/                     # Slack 알림
│   └── utils/                     # 공통 유틸리티
│
├── domain/                        # 도메인 모델 (DTO/VO)
│   ├── base/                      # 기본 도메인
│   │   ├── adm/                   # 관리자
│   │   ├── cns/ (type/)           # 약관(Consent)
│   │   ├── com/ (type/)           # 공통 코드
│   │   ├── message/               # 메시지
│   │   ├── ord/ (type/)           # 주문
│   │   ├── prd/ (type/)           # 상품
│   │   ├── pty/ (type/)           # 포인트
│   │   ├── pym/                   # 결제
│   │   └── qrtz/                  # Quartz 스케줄러
│   ├── common/                    # 공통 도메인
│   ├── ibe/                       # IBE 연동 도메인
│   │   ├── booking/               # 예약 SOAP 요청/응답
│   │   │   ├── adjustflightinventory/
│   │   │   ├── cancelbooking/
│   │   │   ├── createbooking/
│   │   │   ├── createcustomerprofile/
│   │   │   ├── getFlightSchedule/
│   │   │   ├── getTimeTableInfo/
│   │   │   ├── listbaggageservice/
│   │   │   ├── ListSaleableAncillaryService/
│   │   │   ├── modifybooking/
│   │   │   ├── retrieveairavail/
│   │   │   ├── retrievebooking/
│   │   │   ├── retrieveenhncdairavail/
│   │   │   ├── retrieveprice/
│   │   │   ├── savemodifybooking/
│   │   │   ├── splitpnr/
│   │   │   └── syncbooking/
│   │   ├── checkin/               # 체크인 SOAP 도메인
│   │   │   ├── assignseatsformultipleguests/
│   │   │   ├── checkinguest/
│   │   │   ├── collectapis/
│   │   │   ├── searchguest/
│   │   │   └── uncheckguest/
│   │   ├── comn/                  # IBE 공통
│   │   ├── ota/                   # OTA 규격
│   │   │   ├── dto/
│   │   │   ├── request/
│   │   │   ├── response/
│   │   │   └── tax/
│   │   └── payment/               # IBE 결제
│   ├── message/                   # 메시지 도메인
│   │   ├── email/ (members/, reservation/, schedule/, webcheckin/)
│   │   ├── infobip/
│   │   └── sms/ (members/)
│   └── pay/                       # 결제 도메인
│       └── Toss/                  # Toss Payments
│
└── mapper/                        # 공유 MyBatis 매퍼
    └── infobip/
```

**resources:**
- `application-common.yml` / `application-databases.yml` / `application-ibe.yml` / `application-init.yml`
- `masking.properties` — 마스킹 규칙
- `messages/` — 다국어 메시지

---

## 2. sumair-be-home (Port 8000)

> 기본 패키지: `com.sumair.home`

사용자 대면 포털. 프론트엔드 API 진입점.

```
com.sumair.home
├── airline/                       # 항공 예약 기능
│   ├── booking/                   # 예약 관리
│   │   ├── adustfilghtinventory/  # 좌석 재고 조정
│   │   ├── ancillaryservices/     # 부가서비스 (수하물 등)
│   │   ├── avail/                 # 항공편 조회
│   │   ├── cancel/                # 예약 취소
│   │   ├── create/                # 예약 생성
│   │   ├── modifiy/               # 예약 변경
│   │   └── retrieve/              # 예약 조회
│   ├── checkin/                   # 웹 체크인
│   │   ├── checkinguest/          # 체크인 처리
│   │   ├── printboardingpass/     # 탑승권 출력
│   │   ├── searchguest/           # 승객 검색
│   │   └── showseatmap/           # 좌석 배치도
│   ├── common/                    # 항공 공통
│   │   ├── fare/                  # 운임 관리
│   │   ├── passenger/             # 승객 정보
│   │   ├── rglt/                  # 규정 관리
│   │   ├── route/                 # 노선 관리
│   │   └── sclpst/                # 스케줄/게시
│   └── schedule/                  # 운항 스케줄
│
├── common/                        # 포털 공통 기능
│   ├── certification/             # 본인인증
│   ├── chc/                       # 헬스체크
│   ├── code/                      # 공통 코드
│   ├── cookie/                    # 쿠키 관리
│   ├── country/                   # 국가 정보
│   ├── holiday/                   # 공휴일
│   ├── menu/                      # 메뉴 관리
│   ├── pay/                       # 결제 연동
│   ├── terms/                     # 약관
│   └── wallet/                    # 지갑 (Apple/Google Wallet)
│       └── config/
│
├── config/                        # 설정
│   └── security/                  # Spring Security
│       ├── core/                  # 인증/인가 핵심
│       │   ├── controller/
│       │   ├── exception/
│       │   ├── mapper/
│       │   ├── model/
│       │   └── service/
│       ├── jwt/                   # JWT 토큰
│       └── oauth/                 # OAuth2
│
├── core/                          # 핵심 인프라
│   ├── aop/                       # AOP (로깅 등)
│   ├── app/                       # 앱 설정
│   ├── redis/                     # Redis 캐시
│   ├── session/                   # 세션 관리
│   └── webclient/                 # WebClient 설정
│
├── customer/                      # 회원 관리
│   ├── common/                    # 회원 공통
│   ├── corp/                      # 법인 회원
│   ├── history/                   # 이용 이력
│   ├── profile/                   # 프로필 (IBEP 연동)
│   └── update/                    # 회원 정보 수정
│
├── main/                          # 메인 페이지 콘텐츠
│   ├── ads/                       # 광고/배너
│   ├── hashtag/                   # 해시태그
│   ├── intro/                     # 인트로
│   ├── notice/                    # 공지사항
│   ├── special/                   # 특가
│   └── weather/                   # 날씨 정보
│
├── mypage/                        # 마이페이지
│   ├── coupon/                    # 쿠폰
│   ├── point/                     # 포인트
│   └── reservation/               # 예약 내역
│
└── share/                         # 공유 기능
    └── controller/
```

**resources:**
- `application.yml`
- `certs/` — SSL 인증서
- `config/` — 환경별 설정
- `static/` — 정적 파일
- `template/` — 이메일/문서 템플릿
- `wallet/` — Apple/Google Wallet 설정

---

## 3. sumair-be-ibe (Port 8100)

> 기본 패키지: `com.sumair.ibe`

항공 예약 엔진. IBES(Sabre)/IBEP(Amadeus) SOAP 연동 전담.

```
com.sumair.ibe
├── common/                        # IBE 공통
│   ├── airport/                   # 공항 정보
│   ├── app/
│   ├── chc/                       # 헬스체크
│   └── ord/                       # 주문 처리
│       ├── bas/                   # 주문 기본
│       ├── book/                  # 예약 주문
│       ├── psngr/                 # 승객 정보
│       ├── pym/                   # 결제 정보
│       ├── service/               # 주문 공통 서비스
│       └── ssr/                   # 특별 서비스 요청 (SSR)
│
├── config/                        # 설정
│
├── core/                          # 핵심 인프라
│   ├── aop/
│   ├── error/                     # 에러 처리
│   ├── handler/                   # 핸들러
│   ├── http/webclient/            # HTTP 클라이언트
│   ├── jaxb/adapter/              # XML 바인딩 (JAXB)
│   ├── soap/                      # SOAP 클라이언트 (Apache CXF)
│   └── utils/
│
├── ibep/                          # IBEP (Amadeus) 연동
│   ├── booking/                   # 예약 SOAP 호출
│   │   ├── adjustFlightInventory/ # 좌석 재고 조정
│   │   ├── cancel/                # 예약 취소
│   │   ├── create/                # 예약 생성
│   │   │   ├── booking/           # 예약 생성 상세
│   │   │   └── flight/            # 항공편 생성
│   │   ├── flight/                # 항공편 조회
│   │   │   ├── retrievepassenger/ # 승객 목록 조회
│   │   │   └── schedule/          # 스케줄 조회
│   │   ├── modifybooking/         # 예약 변경
│   │   ├── payment/               # 결제 처리
│   │   ├── savemodifybooking/     # 변경 저장
│   │   ├── splitbooking/          # PNR 분리
│   │   └── syncbooking/           # 예약 동기화
│   ├── checkin/                   # 체크인 SOAP 호출
│   └── profile/                   # 고객 프로필
│
├── ibes/                          # IBES (Sabre) 연동
│   ├── retrieveairavail/          # 항공편 가용성 조회
│   ├── retrieveancillary/         # 부가서비스 조회
│   ├── retrievebaggage/           # 수하물 조회
│   ├── retrievebooking/           # 예약 조회
│   ├── retrieveprice/             # 운임 조회
│   └── retrievetimetable/         # 시간표 조회
│
└── service/                       # 비즈니스 오케스트레이션
    ├── booking/
    │   ├── avail/                 # 가용성 서비스
    │   │   ├── createcustomerprofile/
    │   │   ├── getairavailability/
    │   │   ├── getalloanddpairs/
    │   │   ├── getenhancedairavailability/
    │   │   ├── gettimetable/
    │   │   ├── listbaggages/
    │   │   └── listsaleableancillary/
    │   ├── flight/                # 항공편 서비스
    │   │   ├── getflightinfomation/
    │   │   ├── getflightschedule/
    │   │   └── retrievepassengermanifest/
    │   ├── price/confirmprice/    # 운임 확정
    │   └── rsrvt/                 # 예약 서비스
    │       ├── adjustflightinventory/
    │       ├── cancelbooking/
    │       ├── modifybooking/
    │       ├── retrievebooking/
    │       ├── retrievereservationsummary/
    │       ├── savecreatebooking/
    │       ├── savemodifybooking/
    │       └── splitreservation/
    └── checkin/                   # 체크인 서비스
        ├── assignseats/
        ├── checkinguest/
        ├── collectapis/
        ├── flightinfodisplay/
        ├── printboardingpass/
        ├── searchguest/
        ├── showseatmap/
        └── uncheckguest/
```

**resources:**
- `application.yml`
- `cert/` — SOAP 통신용 SSL 인증서
- `config/` — 환경별 설정
- `templates/` — SOAP 요청 템플릿

---

## 4. sumair-be-pay (Port 8200)

> 기본 패키지: `com.sumair.pay`

결제/쿠폰/포인트 처리 모듈. 가장 간결한 구조.

```
com.sumair.pay
├── booking/                       # 예약 결제
│   └── pay/                       # Toss Payments 연동
│       ├── controller/
│       ├── mapper/
│       ├── model/
│       └── service/ (Impl/)
│
├── common/                        # 결제 공통
│   ├── chc/                       # 헬스체크
│   └── constants/                 # 상수
│
├── config/                        # 설정
│
├── controller/                    # 공통 컨트롤러
│
├── core/                          # 핵심 인프라
│   ├── aop/
│   └── rest/                      # REST 설정
│
├── coupon/                        # 쿠폰 관리
│   ├── controller/
│   ├── mapper/
│   ├── model/
│   └── service/ (Impl/)
│
└── point/                         # 포인트 관리
    ├── controller/
    ├── mapper/
    ├── model/
    └── service/ (Impl/)
```

**resources:**
- `application.yml`
- `config/` — 환경별 설정

---

## 5. sumair-be-batch (Port 8300)

> 기본 패키지: `com.sumair.batch`

배치/스케줄링 모듈. Quartz 스케줄러 사용.

```
com.sumair.batch
├── admin/                         # 배치 관리 (스케줄러 CRUD)
│   ├── controller/
│   ├── mapper/
│   ├── model/
│   └── service/ (Impl/)
│
├── common/                        # 배치 공통
│   └── chc/                       # 헬스체크
│
├── config/                        # 설정
│   └── quartz/                    # Quartz 스케줄러 설정
│
├── core/                          # 핵심 인프라
│   ├── aop/
│   ├── properties/                # 배치 프로퍼티
│   └── webclient/                 # 외부 API 호출
│
├── holiday/                       # 공휴일 동기화 배치
│   ├── controller/
│   ├── mapper/
│   ├── model/ (common/)
│   └── service/ (impl/)
│
├── infobip/                       # Infobip 발송 이력 배치
│   ├── config/
│   ├── controller/
│   ├── dto/
│   ├── entity/
│   ├── job/                       # Spring Batch Job
│   ├── mapper/
│   └── service/
│
├── job/                           # Quartz Job 정의
│
├── point/                         # 포인트 만료 배치
│   ├── controller/
│   ├── mapper/
│   ├── model/
│   └── service/ (impl/)
│
├── reservation/                   # 예약 상태 동기화 배치
│   ├── controller/
│   ├── mapper/
│   ├── model/
│   └── service/ (impl/)
│
├── schedule/                      # 운항 스케줄 동기화 배치
│   ├── controller/
│   ├── mapper/
│   ├── model/
│   └── service/ (impl/)
│
└── weather/                       # 날씨 정보 수집 배치
    ├── controller/
    ├── mapper/
    ├── model/ (common/)
    └── service/ (impl/)
```

**resources:**
- `application.yml`
- `application-quartz.yml` — Quartz 설정
- `config/` — 환경별 설정
- `schema.sql` — Quartz 테이블 DDL

---

## 6. sumair-be-message (Port 8400)

> 기본 패키지: `com.sumair.message`

알림 발송 모듈. 이메일/SMS/카카오 발송 전담.

```
com.sumair.message
├── common/                        # 메시지 공통
│   └── chc/                       # 헬스체크
│
├── config/                        # 설정
│   └── webclient/                 # WebClient 설정
│
├── core/                          # 핵심 인프라
│   ├── aop/
│   └── util/                      # 유틸리티
│
├── email/                         # 이메일 발송
│   ├── cstmrservice/              # 고객센터 이메일
│   ├── members/                   # 회원 관련 이메일
│   ├── reservation/               # 예약 확인 이메일
│   ├── schedule/                  # 스케줄 변경 알림
│   └── webcheckin/                # 웹체크인 안내
│
├── infobip/                       # Infobip 통합 발송
│   ├── controller/
│   ├── mapper/
│   ├── model/
│   │   ├── email/                 # 이메일 모델
│   │   └── kakao/                 # 카카오 알림톡 모델
│   └── service/ (impl/)
│
├── ires/                          # IRES 연동
│   ├── config/
│   ├── interceptor/
│   ├── mapper/
│   ├── model/
│   └── service/ (impl/)
│
├── slack/                         # Slack 알림
│
└── sms/                           # SMS 발송
    ├── cstmrservice/              # 고객센터 SMS
    ├── members/                   # 회원 관련 SMS
    ├── reservation/               # 예약 확인 SMS
    ├── schedule/                  # 스케줄 변경 SMS
    └── webcheckin/                # 웹체크인 안내 SMS
```

**resources:**
- `application.yml` / `application-init.yml` / `application.properties`
- `config/` — 환경별 설정
- `static/` — 이메일 템플릿 리소스 (이미지 등)

---

## 모듈 간 호출 관계

```
[FE] → home(8000) → ibe(8100) → IBES/IBEP (SOAP)
                   → pay(8200) → Toss Payments
                   → message(8400) → Infobip (SMS/Email/Kakao)

[Scheduler] → batch(8300) → ibe(8100) (스케줄 동기화)
                           → message(8400) (알림 발송)
                           → 외부 API (공휴일/날씨)

[common] ← 모든 모듈에서 의존
```

---

## 패키지 네이밍 참고

| 약어 | 의미 |
|------|------|
| `adm` | Admin (관리자) |
| `cns` | Consent (약관/동의) |
| `com` | Common (공통) |
| `ord` | Order (주문) |
| `prd` | Product (상품) |
| `pty` | Point (포인트) |
| `pym` | Payment (결제) |
| `qrtz` | Quartz (스케줄러) |
| `rglt` | Regulation (규정) |
| `sclpst` | Schedule/Post (스케줄/게시) |
| `psngr` | Passenger (승객) |
| `ssr` | Special Service Request |
| `rsrvt` | Reservation (예약) |
| `chc` | Health Check |
| `cstmrservice` | Customer Service (고객센터) |
