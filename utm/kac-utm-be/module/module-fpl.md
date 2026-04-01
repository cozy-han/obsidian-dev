# Module: FPL (비행계획)

#kac-utm #backend #module

| 항목 | 값 |
|------|---|
| **경로** | `module/fpl` |
| **Gradle** | `:module:module-fpl` |
| **패키지** | `com.kac.utm.fpl` |
| **클래스 수** | 133개 |

---

## 역할

**비행계획(Flight Plan)** 도메인을 관리한다. 드론 비행 신청부터 승인, 항공기 등록, 조종사 관리, 비행 구간/경로 관리까지 비행계획의 전체 라이프사이클을 담당한다.

> **언제 이 코드를 보게 되나?**
> 비행계획 신청/승인 프로세스를 수정하거나, 비행 구간/항공기/조종사 관련 기능을 개발할 때. UTM 시스템의 핵심 도메인.

---

## 주요 엔티티

### FplBscEntity (비행계획 기본)

```
PK: fplNo (비행계획 번호)

주요 필드:
- custNo: 고객 번호
- aplyNo: 신청 번호
- aplyYmd: 신청 일자
- fplBgngDt / fplEndDt: 비행계획 시작/종료 일시
- fplPrpsCd: 비행 목적 코드
- fplMthCd: 비행 방식 코드
- fplTypeCd: 비행 유형 코드
- fplPrgrsSttsCd: 진행 상태 (신청중 → 검토중 → 검토완료)
- fplSrcCd: 출처 (KAC, DOS 등)
- fplVer: 버전
- refFplNo: 참조 비행계획 번호

관계:
├── OneToMany → FplRlmBscEntity (비행 구간)
├── OneToMany → FplPltBscEntity (조종사)
└── OneToMany → FplAcrBscEntity (항공기)
```

### 관련 엔티티

| 엔티티 | 설명 |
|--------|------|
| `FplAcrBscEntity` | 항공기 정보 |
| `FplPltBscEntity` | 조종사 정보 |
| `FplRlmBscEntity` | 비행 구간 |
| `FplRlmCmptncBscEntity` | 구간별 관할기관 |
| `FplPrmsnBscEntity` | 비행 허가 |
| `FplArvlBscEntity` | 도착 정보 |
| `FplDptreBscEntity` | 출발 정보 |
| `FplFlyngItrpBscEntity` | 비행 경로 |
| `FplSeqBscEntity` | 시퀀스 관리 |
| `FplCustAcrBscEntity` | 고객 항공기 (자주 사용하는 항공기 등록) |
| `FplCustRlmBscEntity` | 고객 구간 (자주 사용하는 구간 등록) |
| `FplCustPltBscEntity` | 고객 조종사 (자주 사용하는 조종사 등록) |

---

## 비행계획 상태 흐름

```
신청(APPLY) → 검토중(REVIEW) → 검토완료(REVIEWED)
                                  ├── 승인(APPROVED)
                                  └── 반려(REJECTED)
```

---

## 주요 서비스

| 서비스 | 기능 |
|--------|------|
| BscService | 비행계획 기본 CRUD |
| AcrService | 항공기 등록/조회 |
| PltService | 조종사 등록/조회 |
| RlmService | 비행 구간 관리 |
| ItrpService | 비행 경로 관리 |
| PrmsnService | 비행 허가 처리 |
| SeqService | 시퀀스 번호 생성 |

---

## 사용하는 App 모듈

| App | 용도 |
|-----|------|
| [[app-web]] | 사용자 비행계획 조회 |
| [[app-mngr]] | 비행계획 검토/승인 |
| [[app-sol]] | 솔루션 업체 비행계획 조회 |
| [[app-uss-pub]] | USS 비행계획 전체 관리 |
| [[app-engine]] | 비행계획 검증/처리 |

---

## 관련 문서
- [[module-gsl]] - 공역 정보 (비행 구간 검증 시 사용)
- [[module-psty]] - 관제 (비행 중 감시)
- [[module-dos]] - DOS 비행계획 (유사 구조)
