# Module: STATS (통계)

#kac-utm #backend #module

| 항목 | 값 |
|------|---|
| **경로** | `module/stats` |
| **Gradle** | `:module:module-stats` |
| **패키지** | `com.kac.utm.stats` |
| **클래스 수** | 36개 |

---

## 역할

**통계** 도메인이다. 비행 통계, 이상상황 통계, 성능 통계 등 관제 데이터를 기반으로 일별/누적 통계를 생성하고 조회하는 기능을 제공한다.

> **언제 이 코드를 보게 되나?**
> 대시보드 통계 화면, 통계 보고서 기능을 개발할 때. 배치 모듈([[app-batch]])에서 통계를 생성하고, 관리자 포털([[app-mngr]])에서 조회한다.

---

## 주요 엔티티

### StatsFlyngBscEntity (비행 통계)

```
PK: StatsFlyngBscId (복합키: 날짜 + 식별번호)

주요 필드:
- flyngTm: 비행 시간 (분 단위)
- flyngDstnc: 비행 거리 (미터)
- frstCtrlYn: 첫 관제 여부
- dclrNo: 신고 번호
```

### 기타 엔티티

| 엔티티 | 설명 |
|--------|------|
| `StatsFlyngBscId` | 비행 통계 복합키 |
| `StatsFlyngPrfmncBscEntity` | 비행 성능 통계 |
| `StatsAbnSitBscEntity` | 이상상황 통계 |

---

## 모델 클래스

| 모델 | 설명 |
|------|------|
| StatsFlyngBscModel | 비행 통계 조회 모델 |
| StatsAbnSitBscModel | 이상상황 통계 조회 모델 |
| StatsBaseRq | 통계 기본 요청 (기간 조건) |
| StatsAbnSitRq | 이상상황 통계 요청 |
| DateTimeParts | 날짜/시간 분해 유틸 |

---

## 통계 생성 흐름

```
관제 데이터 (module-psty)
       │
       ▼
배치 작업 (app-batch) ──→ 통계 테이블 (module-stats)
       │                          │
       │                          ▼
       │                   관리자 포털 (app-mngr) 조회
       │                   USS 포털 (app-uss-pub) 조회
       │                   솔루션 포털 (app-sol) 조회
```

---

## 관련 문서
- [[app-batch]] - 통계 생성 배치
- [[app-mngr]] - 통계 조회 화면
- [[module-psty]] - 관제 데이터 (통계 원천)
