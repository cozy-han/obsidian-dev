# Module: PSTY (관제)

#kac-utm #backend #module

| 항목 | 값 |
|------|---|
| **경로** | `module/psty` |
| **Gradle** | `:module:module-psty` |
| **패키지** | `com.kac.utm.psty` |
| **클래스 수** | 99개 |

---

## 역할

**관제(Posture/Safety)** 도메인을 관리한다. 실시간 비행 감시, 관제 이력, 사고 보고, 이상 상황 로그 등 드론이 비행 중일 때의 모든 관제 데이터를 처리한다.

> **언제 이 코드를 보게 되나?**
> 관제 화면 기능, 비행 중 이상 상황 처리, 사고 보고서 기능을 개발할 때. 실시간 비행 데이터와 관련된 모든 엔티티가 여기 있다.

---

## 주요 엔티티

### PstyCtrlBscEntity (관제 기본)

```
PK: ctrlNo (관제 번호)

주요 필드:
- idntfNo: 드론 식별 번호
- fbctnNo: 제작 번호
- acrTypeCd: 기체 유형 코드
- ctrlTypeCd: 관제 유형 (일반/불법/기타)
- ctrlSttsCd: 관제 상태
- prcsSttsCd: 처리 상태
- flyngBgngDt / flyngEndDt: 비행 시작/종료
- ctrlBgngDt / ctrlEndDt: 관제 시작/종료
- tfltTm: 총 비행 시간
- tfltDstnc: 총 비행 거리
- avgSpd: 평균 속도
- mainCtrlNo: 메인 관제 번호 (참조)
```

### 관련 엔티티

| 엔티티 | 설명 |
|--------|------|
| `PstyCtrlFplRelEntity` | 관제-비행계획 관계 (어떤 비행계획의 관제인지) |
| `PstyCtrlRgnBscEntity` | 관제 지역 (비행 위치 데이터) |
| `PstyCtrlHstryEntity` | 관제 이력 (상태 변경 기록) |
| `PstyAcdntRptBscEntity` | 사고 보고서 기본 |
| `PstyAcdntRptDtlEntity` | 사고 보고서 상세 |
| `PstyAcdntRptDtlHstryEntity` | 사고 보고서 이력 |
| `PstyAbnSitLogEntity` | 이상 상황 로그 |
| `PstyAbnSitRelEntity` | 이상 상황 관계 |
| `PstySeqBscEntity` | 시퀀스 관리 |

---

## 관제 유형

| 코드 | 설명 |
|------|------|
| 일반 | 정상적인 비행계획 기반 관제 |
| 불법 | 미승인 드론 탐지 시 관제 |
| 기타 | 기타 상황 |

---

## 사용하는 App 모듈

| App | 용도 |
|-----|------|
| [[app-engine]] | 관제 데이터 처리, 이상상황 감지 |
| [[app-uss-pub]] | USS 관제 기능 |
| [[app-batch]] | 비행 종료 배치, 통계 생성 |
| [[app-ws]] / [[app-uss-ws]] | 실시간 관제 데이터 스트리밍 |

---

## 관련 문서
- [[module-fpl]] - 비행계획 (관제 대상)
- [[module-stats]] - 통계 (관제 데이터 기반 통계)
- [[app-psty]] - 메시지 엔진 (이름이 같지만 역할 다름)
