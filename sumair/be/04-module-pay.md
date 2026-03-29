# sumair-be-pay

결제 처리 서비스. Toss Payments 게이트웨이를 통한 결제, 포인트, 쿠폰 관리.

## 기본 정보
- ArtifactId: `sumair-pay`
- Port: **8200**
- Java 파일: 29개 (가장 작은 모듈)
- 의존: `sumair-common`

## 패키지 구조

```
com.sumair.pay
├── booking/pay/
│   ├── controller/   # BookingPaymentRestController
│   ├── service/      # BookingPaymentService
│   ├── mapper/       # BookingPaymentMapper
│   └── model/        # 결제 DTO
├── point/
│   ├── controller/   # PointRestController
│   ├── service/      # PointService
│   ├── mapper/       # PointMapper
│   └── model/        # 포인트 DTO
├── coupon/
│   ├── controller/   # CouponRestController
│   ├── service/      # CouponService
│   ├── mapper/       # CouponMapper
│   └── model/        # 쿠폰 DTO
├── core/
│   ├── aop/          # AOP 로깅
│   └── rest/         # TossRestService (결제 게이트웨이)
├── config/           # MyBatis 설정
└── common/
    ├── chc/          # 헬스체크
    └── constants/    # 상수
```

## 컨트롤러
- `BookingPaymentRestController` - 예약 결제
- `PointRestController` - 포인트 조회/적립/사용
- `CouponRestController` - 쿠폰 조회/사용

## Toss Payments 연동

| 환경 | 타입 | MID |
|------|------|-----|
| local | BP | cp_sumair |
| dev | BP | cp_sumairsecret |
| prod | BP | (라이브 키) |
| local/dev | WD | sumair2op |

- BP (Business Partner): 비즈니스 파트너 결제
- WD: 일반 결제

## 핵심 패턴
- 단일 책임 원칙: 결제만 담당
- Controller -> Service -> Mapper 3계층
- TossRestService를 통한 PG 연동
- Stateless 트랜잭션 처리
