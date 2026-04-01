# Module: Common (공통 도메인)

#kac-utm #backend #module

| 항목 | 값 |
|------|---|
| **경로** | `module/common` |
| **Gradle** | `:module:module-common` |
| **패키지** | `com.kac.utm.common` |
| **클래스 수** | 290개 (모듈 중 최대) |

---

## 역할

시스템 전반에서 공유하는 **기본 데이터 도메인**을 관리한다. 사용자, 권한, 메뉴, 공통 코드, 파일, 기업 정보 등 거의 모든 app 모듈이 의존하는 핵심 모듈이다.

> **언제 이 코드를 보게 되나?**
> 사용자 관리, 권한 체계, 공통 코드, 메뉴 구조, 파일 업로드 등을 수정할 때. 가장 자주 보게 되는 모듈.

---

## 주요 엔티티

### 사용자/인증

| 엔티티 | 설명 | 주요 필드 |
|--------|------|----------|
| `ComUserBscEntity` | 사용자 기본 | userId, userNm, userPswd, userTypeCd, acntSttsCd |
| `ComAuthrtBscEntity` | 권한 | authrtId, authrtNm |
| `ComLgnHstryEntity` | 로그인 이력 | lgnDt, lgnIp, lgnSttsCd |
| `ComSnsBscEntity` | SNS 연동 | snsTypeCd, snsId |

### 기업

| 엔티티 | 설명 |
|--------|------|
| `ComEntBscEntity` | 기업 기본 정보 |
| `ComEntUserRelEntity` | 기업-사용자 관계 |

### 시스템

| 엔티티 | 설명 |
|--------|------|
| `ComCdBscEntity` | 공통 코드 |
| `ComMenuBscEntity` | 메뉴 |
| `ComFileBscEntity` | 파일 |
| `ComFwkBscEntity` | 프레임워크 설정 |
| `ComSiteBscEntity` | 사이트 |
| `ComTrmsBscEntity` | 약관 |
| `ComTmpltBscEntity` | 템플릿 |
| `ComApiHstryEntity` | API 호출 이력 |
| `ComUseHstryEntity` | 사용 이력 |

---

## ComUserBscEntity 상세

```
PK: userNo (사용자 번호)

주요 관계:
├── ManyToOne → ComAuthrtBscEntity (권한)
├── ManyToOne → ComEntBscEntity (기업)
└── OneToMany → ComSnsBscEntity (SNS)

주요 필드:
- userId: 로그인 ID
- userPswd: 비밀번호 (BCrypt)
- userNm: 사용자명
- userTypeCd: 사용자 유형 코드
- entUserTypeCd: 기업회원 유형 (MGR/EMP)
- entAprvSttsCd: 기업회원 승인 상태
- acntSttsCd: 계정 상태
- lgnLckYn: 로그인 잠금 여부
- joinDt / whdwlDt: 가입일 / 탈퇴일
- emlRcptnAgreYn: 이메일 수신 동의
- smsRcptnAgreYn: SMS 수신 동의
```

---

## 주요 서비스

| 서비스 | 기능 |
|--------|------|
| WebUserService | 사용자 CRUD, 검색 |
| CodeService | 공통 코드 조회/관리 |
| AuthrtService | 권한 관리 |
| MenuService | 메뉴 관리 |
| ComFileBscService | 파일 업로드/다운로드 |
| ComSeqService | 시퀀스 번호 생성 |
| ComSiteService | 사이트 관리 |
| EntService | 기업 관리 |
| TrmsService | 약관 관리 |
| PopupService | 팝업 관리 |
| AlrtService | 알림 관리 |
| FwkService | 프레임워크 설정 |

---

## Repository 패턴

각 엔티티마다 3개의 Repository가 존재:
```
ComUserBscRepository         ← JpaRepository 상속 (기본 CRUD)
ComUserBscBaseRepository     ← 커스텀 쿼리 인터페이스
ComUserBscBaseRepositoryImpl ← QueryDSL로 구현
```

---

## 관련 문서
- [[app-web]] - 사용자 포털 (이 모듈 사용)
- [[app-mngr]] - 관리자 포털 (이 모듈 사용)
- [[common-모듈정리]] - 공통 라이브러리
