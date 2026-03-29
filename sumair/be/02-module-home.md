# sumair-be-home

메인 API 게이트웨이. 사용자 인증, 고객 관리, 예약 조회 등 프론트엔드 대면 API 제공.

## 기본 정보
- ArtifactId: `sumair-home`
- Port: **8000**
- Java 파일: 387개
- 의존: `sumair-common`

## 패키지 구조

```
com.sumair.home
├── config/
│   ├── security/
│   │   ├── core/        # 사용자 인증 (BaseUserController, AuthUserMapper)
│   │   ├── oauth/       # OAuth2 (Naver, Kakao, Google)
│   │   └── jwt/         # JWT 토큰 관리
│   ├── RedisConfig
│   ├── MyBatisConfig
│   ├── SwaggerConfig
│   └── CorsConfig
├── core/webclient/      # 서비스간 HTTP 클라이언트
├── airline/
│   ├── booking/
│   │   ├── avail/       # 항공편 조회
│   │   ├── create/      # 예약 생성
│   │   ├── cancel/      # 예약 취소
│   │   └── ancillaryservices/  # 부가서비스
│   ├── checkin/
│   │   └── showseatmap/ # 좌석 선택
│   └── common/
│       ├── fare/        # 운임 계산
│       ├── passenger/   # 탑승객 관리
│       ├── route/       # 노선 관리
│       └── sclpst/      # 스케줄/비용
├── customer/            # 고객 프로필 / 계정 관리
├── main/
│   ├── intro/           # 메인 페이지
│   ├── notice/          # 공지사항
│   └── special/         # 특가 상품
├── common/
│   ├── holiday/         # 공휴일 정보
│   ├── terms/           # 약관
│   ├── pay/             # 결제 연동
│   ├── code/            # 코드 관리
│   ├── menu/            # 메뉴 관리
│   ├── certification/   # 본인인증
│   └── country/         # 국가/지역 데이터
└── mypage/              # 마이페이지
```

## 주요 컨트롤러 (40개)
- `BaseUserController` - 사용자 인증
- `AirlineBookingAvailController` - 항공편 조회
- `AirlineBookingCreateController` - 예약 생성
- `AirlineBookingCancelController` - 예약 취소
- `MypagePointController` - 포인트 관리
- `CommonTermsController` - 약관
- `CommonCertificationController` - 본인인증
- `CommonCountryController` - 국가 데이터

## 주요 WebClient 서비스 (서비스간 통신)
- `IbesWebClientService` -> ibe (항공편 조회)
- `PayWebClientService` -> pay (결제)
- `PointWebClientService` -> pay (포인트)
- `SmsWebClientService` -> message (SMS)
- `InfobipWebClientService` -> message (Infobip)
- `CouponWebClientService` -> pay (쿠폰)
- `EmailWebClientService` -> message (이메일)

## 주요 의존성

| 라이브러리 | 용도 |
|-----------|------|
| spring-boot-starter-security | Spring Security |
| spring-boot-starter-data-redis | Redis 캐시 |
| spring-boot-starter-oauth2-client | OAuth2 |
| jjwt 0.9.1 | JWT |
| pagehelper 1.4.6 | 페이징 |

## 인증 체계
- **JWT**: 토큰 기반 인증
- **OAuth2**: Naver, Kakao, Google 소셜 로그인
- **Redis**: 세션/토큰 캐시 (@EnableCaching)

## 설정
- `application.yml` - OAuth2 클라이언트 설정 포함
- Redis: AWS ElastiCache (Valkey 호환)
- 프로파일: local / dev / prod
