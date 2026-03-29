# API 서비스

서비스 레이어는 Zustand + React Query를 조합하여 API 호출 관리. 모든 서비스는 `src/service/` 에 위치.

## API Routes (BFF)
Next.js API Routes로 서버사이드 처리가 필요한 기능 구현:

| 파일 | 설명 |
|------|------|
| `api/payConfirm.ts` | Toss Payments 결제 승인 (NORMAL/BRANDPAY) |
| `api/callback-auth.ts` | Toss BrandPay 인증 콜백 |
| `api/snsRedirect.ts` | SNS OAuth 콜백 (JOIN/LOGIN/DUPLICATE/REL/휴면/잠금 처리) |
| `api/hello.ts` | 샘플 엔드포인트 |

## booking.ts - 항공편 검색/예약

| 함수 | 설명 |
|------|------|
| `getAvailFn()` | 항공편 가용편 조회 (승객수, 프로모 코드) |
| `useGetEnhancedAvail()` | 동적 필터링 가용편 |
| `getConfirmPriceFn()` | 최종 가격 확인 |
| `getStatusDiscountFn()` | 할인 조회 (학생, 경로 등) |
| `getRecentListFn()` | 최근 탑승자 목록 |
| `getFavoriteFn()` | 즐겨찾기 탑승자 |
| `setFavoriteFn()` | 즐겨찾기 추가/삭제 |
| `getAncillaryFn()` | 부가서비스 목록 (수하물, 기내식 등) |
| `getSeatMapFn()` | 좌석 배치도 조회 |
| `setCreateBookingFn()` | 예약 생성 (120초 타임아웃) |
| `getExpectPointFn()` | 예상 적립 포인트 계산 |

## customer.ts - 인증/회원관리 (40+개)

### 인증
| 함수 | 설명 |
|------|------|
| `postJoinFn()` | 회원가입 |
| `postLoginFn()` | 로그인 |
| `postGuestLoginFn()` | 비회원 로그인 |
| `getSnsUserFn()` | SNS 프로필 조회 |
| `postDuplicateIdFn()` | ID 중복 확인 |
| `postDuplicateCheckFn()` | 회원 존재 확인 |
| `postDuplicateConfirmFn()` | 본인 확인 |
| `postResetPasswordFn()` | 비밀번호 재설정 |
| `postAccountActiveFn()` | 잠긴 계정 활성화 |

### 이메일/전화
| 함수 | 설명 |
|------|------|
| `postDuplicateEmailFn()` | 이메일 중복 확인 |
| `putUpdateEmailFn()` | 이메일 변경 |
| `postDuplicatePhoneFn()` | 전화번호 중복 확인 |
| `putUpdatePhoneFn()` | 전화번호 변경 |

### 비밀번호/보안
| 함수 | 설명 |
|------|------|
| `postPasswordCheckFn()` | 현재 비밀번호 확인 |
| `putUpdatePasswordFn()` | 비밀번호 변경 |

### 포인트/상품권
| 함수 | 설명 |
|------|------|
| `postPointSearchFn()` | 포인트 잔액/이력 조회 |
| `postInfinitePointFn()` | 포인트 무한스크롤 |
| `postGiftCardRegistrationFn()` | 상품권 등록 |
| `postCheckGiftCardVerificationFn()` | 상품권 유효성 확인 |

### 계정 정보
| 함수 | 설명 |
|------|------|
| `getFindInfoFn()` | 프로필 조회 |
| `getTermsListFn()` | 약관 동의 이력 |
| `putUpdateTermsFn()` | 마케팅 동의 수정 |

### 탈퇴/법인
| 함수 | 설명 |
|------|------|
| `postWithDrawalFn()` | 회원 탈퇴 |
| `deleteDisConnectSnsFn()` | SNS 연동 해제 |
| `postCorpRegistFn()` | 법인 등록 |
| `getCorpDuplicateFn()` | 사업자번호 중복 확인 |
| `getValidCorpEmailFn()` | 법인 이메일 검증 |
| `getCorpFindFn()` | 법인 코드 조회 |
| `putUpdateCorpFn()` | 법인 코드 적용 |

## itinerary.ts - 여정/예약 관리 (30+개)

### 예약 조회
| 함수 | 설명 |
|------|------|
| `itineraryFn()` | 예약 목록 (예정/과거/취소) |
| `postPassengerFn()` | 예약 탑승자 목록 |
| `postGuestBookingSearchFn()` | 비회원 예약 존재 확인 |
| `postGuestBookingListFn()` | 비회원 예약 상세 |
| `postGuestItineraryFn()` | 비회원 예약 조회 |

### 웹 체크인
| 함수 | 설명 |
|------|------|
| `checkInUserListFn()` | 체크인 대상 조회 |
| `postCheckInCancelFn()` | 체크인 취소 |
| `postBoardingPassFn()` | 보딩패스 생성 |

### 부가서비스
| 함수 | 설명 |
|------|------|
| `postUserServiceFn()` | 예약 부가서비스 조회 |
| `postAncillaryListFn()` | 추가 구매 가능 서비스 |
| `postModifySsrFn()` | SSR 수정 |
| `postSaveSsrFn()` | SSR 저장 |
| `postAncillaryCancelCheckFn()` | 서비스 취소 확인 |
| `postAncillaryCancelFn()` | 서비스 취소 |

### 좌석/취소
| 함수 | 설명 |
|------|------|
| `postAncillarySeatFn()` | 사전좌석 구매 |
| `postAncillarySeatApprovalFn()` | 좌석 결제 승인 |
| `postBookingCancelCheckFn()` | 예약 취소 자격 확인 |
| `postBookingCancelFn()` | 예약 취소 |
| `postBoardConfirmFn()` | 탑승 확인 |

### 결제/영수증
| 함수 | 설명 |
|------|------|
| `postPaymentDetailFn()` | 결제 내역 |
| `postPaymentApprovalFn()` | 결제 승인 내역 |
| `getSendReceiptFn()` | 영수증 이메일 전송 |
| `getSendBoardingPassFn()` | 보딩패스 링크 전송 |
| `postTicketFn()` | 타 채널 티켓 조회 |

## schedule.ts - 운항 스케줄
| 함수 | 설명 |
|------|------|
| `postFlightFn()` | 노선/날짜별 스케줄 |
| `getConfirmFn()` | 운항/결항 확인 |
| `getDepArrFn()` | 출발/도착 스케줄 |

## main.ts - 메인페이지 콘텐츠
| 함수 | 설명 |
|------|------|
| `getSpecialOffersFn()` | 특가 항공편 |
| `getJourneyFn()` | 추천 여행 노선 |
| `getHashtagFn()` | 콘텐츠 해시태그 |
| `getContentsFn()` | 에디토리얼 콘텐츠 |
| `putContentsCountFn()` | 콘텐츠 조회수 |
| `getAboutSumaiFn()` | SUM Air 소개 |
| `getNoticeFn()` | 하단 공지 |
| `getTopNoticeFn()` | 상단 고정 공지 |
| `getPopupFn()` | 모달 팝업 |
| `getSliderFn()` | 메인 캐러셀 이미지 |
| `getWeatherFn()` | 공항 날씨 |
| `postBannerFn()` | 개인화 배너 |
| `getGnbFn()` | GNB 메뉴 |
| `getFooterFn()` | 푸터 콘텐츠 |

## webcheckin.ts - 웹 체크인
| 함수 | 설명 |
|------|------|
| `postseatFn()` | 체크인 좌석배치도 |
| `getAgreeFn()` | 체크인 약관 |
| `postBoardingPassFn()` | 보딩패스 생성 |
| `postAppleWalletPassFn()` | Apple Wallet 패스 (PKPass) |
| `postSamsungWalletPassFn()` | Samsung Wallet 패스 |
| `getSendBoardingPassFn()` | 보딩패스 공유 URL |
