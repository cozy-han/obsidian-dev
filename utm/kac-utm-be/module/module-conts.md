# Module: CONTS (콘텐츠)

#kac-utm #backend #module

| 항목 | 값 |
|------|---|
| **경로** | `module/conts` |
| **Gradle** | `:module:module-conts` |
| **패키지** | `com.kac.utm.conts` |
| **클래스 수** | 49개 |

---

## 역할

**콘텐츠 관리** 도메인이다. 게시판, FAQ, Q&A 등 일반적인 CMS(Content Management System) 기능을 제공한다.

> **언제 이 코드를 보게 되나?**
> 공지사항, FAQ, Q&A 게시판 기능을 개발하거나 수정할 때.

---

## 주요 엔티티

| 엔티티 | 설명 |
|--------|------|
| `ContsBbsBscEntity` | 게시판 기본 |
| `ContsFaqBscEntity` | FAQ |
| `ContsPstBscEntity` | 포스트(공지사항) |
| `ContsQnaBscEntity` | Q&A 질문 |
| `ContsQnaDtlEntity` | Q&A 답변 |

---

## 상수/Enum

| Enum | 설명 |
|------|------|
| `FaqSeCd` | FAQ 구분 코드 (카테고리) |
| `QnaPrcsSttsCd` | Q&A 처리 상태 (접수, 답변완료 등) |

---

## 관련 문서
- [[app-web]] - 사용자 포털 (콘텐츠 표시)
- [[app-mngr]] - 관리자 포털 (콘텐츠 관리)
- [[app-sol]] - 솔루션 포털 (콘텐츠 표시)
