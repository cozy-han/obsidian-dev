# Module: DOS (드론운영사 비행계획)

#kac-utm #backend #module

| 항목 | 값 |
|------|---|
| **경로** | `module/dos` |
| **Gradle** | `:module:module-dos` |
| **패키지** | `com.kac.utm.dos` |
| **클래스 수** | 17개 |

---

## 역할

**DOS(드론운영사)** 에서 제출하는 비행계획을 관리하는 도메인이다. [[module-fpl]]과 유사한 구조이지만, DOS 전용 비행계획 데이터를 별도로 저장한다.

> **언제 이 코드를 보게 되나?**
> 외부 DOS 시스템에서 연동되는 비행계획 데이터를 처리할 때.

---

## 주요 엔티티

| 엔티티 | 설명 |
|--------|------|
| `DosFplBscEntity` | DOS 비행계획 기본 |
| `DosFplDrnBscEntity` | DOS 비행 드론 정보 |
| `DosFplPltBscEntity` | DOS 비행 조종사 |
| `DosFplRlmBscEntity` | DOS 비행 구간 |
| `DosFplRlmCmptncBscEntity` | DOS 구간별 관할기관 |
| `DosFplRlmCmptncBscId` | 복합키 (구간+관할기관) |

---

## FPL vs DOS

| 항목 | [[module-fpl]] | module-dos |
|------|------------|------------|
| 출처 | KAC 시스템 내부 | 외부 DOS 시스템 |
| 엔티티 프리픽스 | Fpl | DosFpl |
| 클래스 수 | 133개 | 17개 |
| 구조 | 동일 패턴 | FPL 간소화 버전 |

---

## 관련 문서
- [[module-fpl]] - 비행계획 (메인 도메인)
- [[app-engine]] - DOS 비행계획 처리
