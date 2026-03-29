# 주요 기능

## 모듈별 기능 목록

---

## 1. sumair-be-home (메인 API 게이트웨이, :8000)

### 1-1. 인증 / 회원 관리
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 이메일 로그인 | `POST /customer/login` | JWT 인증 (Auth 30분 + Refresh 30일), BCrypt, 5회 실패 잠금 |
| 소셜 로그인 (Web) | OAuth2 redirect | Naver, Kakao, Google. AES 암호화 joinCode 전달 |
| 소셜 로그인 (App) | `POST /customer/auth/social` | 네이티브 앱용 SNS 로그인 |
| 회원가입 | `POST /customer/join` | 가입 → IBS 프로필 생성(loyaltyNumber) → PM 포인트 → 환영 이메일 |
| 회원정보 수정 | `/customer/update/*` | 비밀번호, 프로필, 연락처 변경 |
| 아이디/비밀번호 찾기 | `/customer/common/*` | 중복 확인, 비밀번호 초기화, 휴면 활성화 |
| 인증번호 | `POST /common/certification/*` | 이메일/SMS 인증번호 생성 및 검증 |
| SNS 계정 연동 | OAuth2 (state=REL) | 기존 계정에 SNS 연결/해제 |

### 1-2. 항공편 조회
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 항공편 검색 | `POST /airline/booking/avail/get-avail-list` | IBES 조회 + Redis 캐시 (180초 TTL) |
| 가격 확정 | `POST /airline/booking/avail/confirm-price` | 선택 항공편 최종 가격 확인 |
| 귀국편 재조회 | `POST /airline/booking/avail/get-enhanced-avail-list` | 왕복 귀국편 별도 조회 |

### 1-3. 예약 관리
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 예약 생성 | `POST /airline/booking/create` | 결제 → PNR 생성 → DB 저장 → 포인트 적립 |
| 예약 조회 | `POST /airline/booking/retrieve/get-booking-info` | PNR 기반 예약 상세 조회 |
| 취소 조회 | `POST /airline/booking/modify-cancel` | 취소 유형 판별 (전체/구간/승객/분리) + 환불금액 |
| 취소 확정 | `POST /airline/booking/save-cancel` | Toss 환불 + 포인트 복원 + PNR 취소 |
| 분리 필요 확인 | `POST /airline/booking/cancel/is-split` | PNR 분리 필요 여부 사전 체크 |
| SSR 변경 조회 | `POST /airline/booking/modify-ssr-update` | 부가서비스 추가/삭제 금액 확인 |
| SSR 변경 확정 | `POST /airline/booking/save-ssr-update` | 결제 + IBEP 변경 저장 |
| 좌석 변경 조회 | `POST /airline/booking/modify-seat-update` | 사전좌석배정 변경 금액 확인 |
| 좌석 변경 확정 | `POST /airline/booking/save-seat-update` | 결제 + IBEP 좌석 변경 |
| 좌석 마킹 | `POST /airline/booking/adjustfilghtinventory/seat-mark` | 인벤토리 좌석 마킹 |

### 1-4. 부가서비스 (Ancillary)
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 변경용 부가서비스 | `POST .../saleable-service/modify/get-ancillary-list` | 기존 예약 기반 DB 조회 → IBES 부가서비스 + 수하물 |
| 예약용 부가서비스 | `POST .../saleable-service/avail/get-ancillary-list` | 프론트엔드 데이터 기반 조회 |
| 수하물 목록 | `POST .../saleable-service/get-baggage-list` | (Deprecated) 수하물 단독 조회 |

### 1-5. 체크인
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 승객 조회 | `POST /airline/checkin/search-guest` | IBEP SOAP 체크인 가능 승객 |
| 체크인 실행 | `POST /airline/checkin/checkin-guest` | APIS 수집 → 좌석 배정 → 체크인 (4단계) |
| 체크인 취소 | `POST /airline/checkin/uncheckin-guest` | 체크인 상태 → NO_INFO |
| 좌석 배치도 | `POST /airline/checkin/seat-map/get-seat-data` | DB + IBEP 좌석 맵 (Available/Block/Active) |
| 보딩패스 | `POST /airline/checkin/print-boarding-pass` | 보딩패스 데이터 + QR 바코드 |

### 1-6. 마이페이지
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 예약 목록 | `POST /mypage/reservation/bookingSearch` | DB 직접 조회, bookNo 중복 제거 |
| 구간 상세 | `POST /mypage/reservation/segmntSearch` | IBS 동기화 후 DB 조회 |
| 결제 상세 | `POST /mypage/reservation/payDetailHistory` | 승객별/구간별/SSR별 계층 구조 |
| 승인/취소 이력 | `POST /mypage/reservation/approveCancelDetail` | 결제 승인 + 취소 이력 |
| 보딩패스 조회 | `POST /mypage/reservation/boardingPassSearch` | 보딩패스 상태 확인 |
| 포인트 조회 | `POST /mypage/point/search` | PAY 모듈 위임 |
| 예상 포인트 | `GET /mypage/point/expect` | 적립율 기반 예상치 |
| 쿠폰 조회 | `POST /mypage/coupon/search` | PAY 모듈 위임 |
| 쿠폰 등록 | `POST /mypage/coupon/register` | PT 쿠폰은 포인트 자동 적립 |

### 1-7. 공통 / 메인 페이지
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 공항/노선 | `/airline/common/route/*` | 출발/도착 공항 목록 |
| 동행자 관리 | `/airline/common/passenger/*` | 즐겨찾기 승객 CRUD |
| 운임 규정 | `/airline/common/rglt/*` | 운임 규정/코드 조회 |
| 특가 | `/airline/common/sclpst/*` | 특가 목록/운임 |
| 스케줄 | `/airline/schedule/*` | 운항 스케줄 (DB 직접, 배치 갱신) |
| 공휴일 | `/common/holiday/*` | 공휴일 달력 (배치 갱신) |
| 약관 | `/common/terms/*` | 이용약관 조회/동의 |
| 국가 목록 | `/common/country/list` | 국가 코드 |
| 코드 | `/common/code/*` | 공통 코드 그룹 |
| 메뉴 | `/common/menu/*` | 메인/추가 메뉴 |
| 슬라이더/팝업 | `/main/ads/*` | 광고 배너, 팝업, 상단 공지 |
| 공지사항 | `/main/notice/*` | 공지 목록/조회수 |
| 특가/베스트 | `/main/special/*` | 특가 항공권, 베스트 상품 |
| 해시태그 | `/main/hashtag/*` | 해시태그, 액티비티 |
| 날씨 | `/main/weather/current` | 현재 날씨 (배치 갱신) |
| Apple Wallet | `POST /common/wallet/apple-pass` | Apple Wallet 보딩패스 |
| Samsung Wallet | `POST /common/wallet/samsung-pass` | Samsung Wallet 보딩패스 |

---

## 2. sumair-be-ibe (항공 예약 엔진, :8100)

### 2-1. 항공편 조회 (IBES)
| 기능          | 엔드포인트                                  | SOAP                          | 설명                          |
| ----------- | -------------------------------------- | ----------------------------- | --------------------------- |
| 항공편 조회      | `POST /avail/retrieveAirAvail`         | GetAirAvailability            | 편도/왕복 가용 항공편 (AVAIL_PORT)   |
| 단체 항공편 조회   | `POST /avail/retrieveAirAvailForGroup` | GetAirAvailability            | 단체 항공권 조회                   |
| Enhanced 조회 | `POST /avail/retrieveEnhancedAirAvail` | GetEnhancedAirAvailability    | 귀국편 재조회                     |
| 변경 항공편 조회   | `POST /avail/retrieveModifyAirAvail`   | GetAirAvailability            | 예약 변경용 조회                   |
| 가격 확인       | `POST /retrievePrice/confirmPrice`     | ConfirmPrice                  | 선택 항공편 최종 운임                |
| 시간표         | `POST /retrieveTimeTable/timeTable`    | GetTimeTableInfo              | 노선별 운항 시간표                  |
| 수하물         | `POST /retrieveBaggage/saleable`       | ListBaggageServices           | 수하물 옵션 (ANCIL_PORT)         |
| 부가서비스       | `POST /retrieveAncillary/saleable`     | ListSaleableAncillaryServices | XBAG/PETC/CBBG (ANCIL_PORT) |
| 예약 조회       | `POST /booking/retrieveBooking`        | RetrieveBooking               | PNR 기반 예약 상세                |

### 2-2. 예약 생성/변경/취소 (IBEP)
| 기능       | 엔드포인트                                        | SOAP              | 설명                 |
| -------- | -------------------------------------------- | ----------------- | ------------------ |
| 예약 생성    | `POST /booking/create/createBooking`         | CreateBooking     | PNR 생성 (RSRV_PORT) |
| 변경 조회    | `POST /booking/modify/modifyBooking`         | ModifyBooking     | 변경 시뮬레이션 (금액 확인)   |
| 변경 확인    | `POST /booking/modify/changeSegmentCheck`    | ModifyBooking     | 여정 변경 금액 확인        |
| 변경 확정    | `POST /booking/modify/changeSegment`         | SaveModifyBooking | 결제 + 변경 저장         |
| 구간 삭제    | `POST /booking/modify/deleteSegment`         | ModifyBooking     | 구간 삭제 확인           |
| 변경/취소 확정 | `POST /booking/savemodify/saveModifyBooking` | SaveModifyBooking | 변경/취소 확정           |
| 취소 확정    | `POST /booking/savemodify/cancel`            | SaveModifyBooking | 예약 취소 확정           |

### 2-3. 취소/분리
| 기능     | 엔드포인트                              | SOAP              | 설명                |
| ------ | ---------------------------------- | ----------------- | ----------------- |
| 취소 확인  | `POST /booking/cancel/cancelCheck` | ModifyBooking     | 취소 금액 확인          |
| 취소 체크  | `POST /booking/cancel/check`       | ModifyBooking     | 취소 가능 여부          |
| 취소 실행  | `POST /booking/cancel/cancel`      | SaveModifyBooking | 전체/구간 취소          |
| PNR 분리 | `POST /booking/split/splitPnr`     | SplitReservation  | 일부 승객 분리 (부분 취소용) |

### 2-4. 예약 동기화
| 기능        | 엔드포인트                                          | 설명                 |
| --------- | ---------------------------------------------- | ------------------ |
| 온라인 동기화   | `POST /booking/syncbooking/syncbooking`        | IBS 최신 상태 로컬 DB 반영 |
| 오프라인 동기화  | `POST /booking/syncbooking/syncbookingOffline` | 오프라인 예약 동기화        |
| 나의 PNR 조회 | `POST /booking/syncbooking/syncViewMyBooking`  | 오프라인 PNR 조회        |
| 나의 PNR 저장 | `POST /booking/syncbooking/syncSaveMyBooking`  | 오프라인 PNR 가져오기      |

### 2-5. 좌석 인벤토리
| 기능    | 엔드포인트                                         | SOAP                  | 설명           |
| ----- | --------------------------------------------- | --------------------- | ------------ |
| 좌석 마킹 | `POST /adjustFlightInventory/markSeat`        | AdjustFlightInventory | 좌석 홀딩        |
| 좌석 해제 | `POST /adjustFlightInventory/unmarkSeat`      | AdjustFlightInventory | 좌석 홀딩 해제     |
| 세션 체크 | `POST /adjustFlightInventory/sessionValidChk` | AdjustFlightInventory | 좌석 홀딩 세션 유효성 |

### 2-6. 웹 체크인
| 기능      | 엔드포인트                                        | SOAP              | 설명                     |
| ------- | -------------------------------------------- | ----------------- | ---------------------- |
| 승객 조회   | `POST /checkin/searchGuest`                  | SearchGuest       | 체크인 가능 승객 (CHKIN_PORT) |
| APIS 수집 | `POST /checkin/collectApis`                  | CollectApis       | 여권/문서 정보               |
| 좌석 배치도  | `POST /checkin/showSeatMap`                  | ShowSeatMap       | 좌석맵 조회                 |
| 좌석 배정   | `POST /checkin/assignSeatsForMultipleGuests` | AssignSeats       | 다수 승객 좌석 배정            |
| 체크인     | `POST /checkin/checkinGuest`                 | CheckinGuest      | 체크인 실행                 |
| 체크인 취소  | `POST /checkin/uncheckGuest`                 | UncheckGuest      | 체크인 취소                 |
| 보딩패스    | `POST /checkin/printBoardingPass`            | PrintBoardingPass | 보딩패스 출력                |
| 항공편 조회  | `POST /checkin/flightInfoDisplay`            | FlightInfoDisplay | 체크인용 항공편 정보            |

### 2-7. 기타
| 기능     | 엔드포인트                                          | 설명                          |
| ------ | ---------------------------------------------- | --------------------------- |
| 스케줄 조회 | `POST /schedule/flightSchedule`                | 배치용 스케줄 (IBES SOAP)         |
| 승객 명부  | `POST /passengerManifest/getPassengerManifest` | 승객 명부 조회 (배치/알림용)           |
| 프로필 생성 | `POST /profile/create`                         | 회원가입 시 IBS loyaltyNumber 발급 |

### 2-8. 내부 DB 관리
- 주문 기본 정보 (ORD_BAS) 관리
- 예약 상세 (ORD_BOOK_BAS) CRUD
- 승객 정보 (ORD_PSNGR) 관리
- 결제 트랜잭션 (ORD_PYM) 기록
- SSR 요금 (ORD_SSR_CHRG) 기록
- 에러 로그 (ORD_BOOK_ERROR_TXN) 기록

---

## 3. sumair-be-pay (결제 서비스, :8200)

### 3-1. Toss Payments 결제
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 액세스 토큰 | `GET/POST /booking/payment/getAccessToken` | BrandPay 토큰 발급 |
| 결제 승인 | `POST /booking/payment/confirm` | 사전조회 → INSERT → Toss confirm → UPDATE |
| 결제 취소 | `POST /booking/payment/cancel` | Toss cancel + 환불 기록 |
| 결제 조회 | `POST /booking/payment/search` | 결제 내역 조회 |
| 정산 조회 | `POST /booking/payment/getSettlements` | 정산 상세 |
| 결제 삭제 | `POST /booking/payment/remove` | 결제 기록 삭제 |

### 3-2. 포인트
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 적립 | `POST /point/add` | 단건 포인트 적립 (PM/SA/AC) |
| 일괄 적립 | `POST /point/multiAdd` | 배치용 다건 적립 |
| 사용 | `POST /point/use` | FIFO 차감 (만료일 오름차순) |
| 취소 | `POST /point/cancel` | 사용 포인트 복원 |
| 롤백 | `POST /point/rollback` | 예상 적립 취소 (SA→SC) |
| 소멸 | `POST /point/expiry` | 만료 포인트 처리 (EX) |
| 조회 | `POST /point/search` | 잔액 + 상세 이력 |
| 변경 | `POST /point/change` | 포인트 변경 |
| 예상 적립 | `GET /point/expect` | 적립율 기반 예상치 계산 |

### 3-3. 쿠폰
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 조회 | `POST /coupon/search` | 쿠폰 코드로 상세 조회 |
| 등록 | `POST /coupon/register` | 유효성 검증 → PT 쿠폰은 포인트 자동 적립 |

---

## 4. sumair-be-batch (배치 서비스, :8300)

### 4-1. 배치 작업
| Job | 스케줄 | 외부 연동 | 설명 |
|-----|--------|----------|------|
| HolidayJob | 매월 1일 05:00 | data.go.kr REST | 공휴일 데이터 갱신 (올해+내년) |
| WeatherJob | 주기적 | data.go.kr REST | 공항별 동네예보 수집 (3시간 간격) |
| ScheduleJob | 매일 05:00 | IBE → IBES SOAP | 전체 노선 운항 스케줄 (12개월) |
| PointExtinctJob | 매일 05:00 | PAY REST | 만료 포인트 소멸 처리 |
| PointSaveJob | 주기적 | IBS SFTP + PAY REST | Revenue Report Excel → 포인트 확정 적립 (SA→AC) |
| RevenueReportJob | 매일 | IBS SFTP | Revenue Report → PRD_REVENUE_REPORT_TXN |
| BoardingReminderJob | 주기적 | IBE + MESSAGE | 출발 3시간 전 승객 알림 |

### 4-2. Quartz 관리
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| Job 목록 | `GET /batch/admin/jobs` | 등록된 Quartz Job 조회 |
| Trigger 관리 | `POST/PUT/DELETE /batch/admin/trigger` | Cron 추가/변경/삭제/일시정지/재개 |

---

## 5. sumair-be-message (메시징 서비스, :8400)

### 5-1. SMS
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 예약 확인 | `/sms/reservation/*` | 예약 생성/변경/취소 SMS |
| 웹체크인 | `/sms/webcheckin/*` | 체크인 완료 SMS |
| 스케줄 발송 | `/sms/schedule/*` | 운항 변경 안내 SMS |
| 회원 알림 | `/sms/members/*` | 회원 관련 SMS |
| 고객 서비스 | `/sms/cstmrservice/*` | CS 대응 SMS |

### 5-2. Email
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| 예약 확인 | `/email/reservation/*` | 이티켓 이메일 (IRES SendITR 콜백) |
| 웹체크인 | `/email/webcheckin/*` | 체크인 확인 이메일 |
| 스케줄 발송 | `/email/schedule/*` | 운항 변경 이메일 |
| 회원 알림 | `/email/members/*` | 휴면 안내, 환영 이메일 |
| 고객 서비스 | `/email/cstmrservice/*` | CS 대응 이메일 |

### 5-3. IRES 콜백
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| SendITR 수신 | SOAP `/ws/ires` | IBS에서 이티켓 발권 시 콜백 → 이메일+SMS 자동 발송 |

### 5-4. 템플릿 관리
| 기능 | 엔드포인트 | 설명 |
|------|-----------|------|
| Infobip 템플릿 | `/infobip/template/*` | 메시지 템플릿 관리 |

### 5-5. 이중 게이트웨이
- **Infobip (주채널)**: REST API, SMS + Email
- **IRES (레거시)**: SOAP, IBS 연동 메시지
- SMS/Email 병렬 발송 (동일 이벤트에 대해 SMS + Email 동시)
