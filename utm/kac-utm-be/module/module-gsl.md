# Module: GSL (공간정보)

#kac-utm #backend #module

| 항목 | 값 |
|------|---|
| **경로** | `module/gsl` |
| **Gradle** | `:module:module-gsl` |
| **패키지** | `com.kac.utm.gsl` |
| **클래스 수** | 64개 |

---

## 역할

**공간정보(Geospatial)** 도메인을 관리한다. 행정구역, 공역(비행금지/제한 구역), 위험지역, 관할기관, 관측소 등 UTM 시스템의 지리적 기반 데이터를 제공한다.

> **언제 이 코드를 보게 되나?**
> 공역 데이터 추가/수정, 비행 가능 구역 판단 로직, 지도 관련 기능을 개발할 때. Hibernate Spatial과 GeoTools를 직접 다루게 된다.

---

## 주요 엔티티

| 엔티티 | 설명 | 핵심 특징 |
|--------|------|----------|
| `GslAdmdstBscEntity` | 행정구역 | 계층 구조 (시도→시군구→읍면동) |
| `GslAspBscEntity` | 공역(Airspace) | Geometry 칼럼 (비행금지/제한 구역) |
| `GslCmptncInstBscEntity` | 관할기관 | 공역별 담당 기관 |
| `GslCmptncInstDtlEntity` | 관할기관 상세 | 담당자 정보 |
| `GslCmptncInstAdmdstRelEntity` | 관할기관-행정구역 관계 | 다대다 관계 |
| `GslDrpBscEntity` | 위험지역(Drop Zone) | 낙하산 강하 구역 등 |
| `GslObsBscEntity` | 관측소 | 기상 관측소 위치 |
| `GslObsLmtSrfcBscEntity` | 관측 제한 표면 | 관측 장비 보호 구역 |

---

## 공간 데이터 특징

이 모듈은 **Hibernate Spatial**을 사용하여 MySQL의 공간 데이터 타입을 직접 다룬다.

```java
// 엔티티 예시
@Column(columnDefinition = "geometry")
private Geometry aspGeom;  // 공역의 기하학적 형상 (폴리곤 등)
```

사용되는 Dialect: `MySQL8SpatialDialect` (application-gsl.yml에서 설정)

### 주요 공간 연산
- **포함(Contains)**: 특정 좌표가 공역 내에 있는지 판단
- **교차(Intersects)**: 비행 경로가 공역과 겹치는지 판단
- **버퍼(Buffer)**: 공역 주변 완충 구역 생성
- **거리(Distance)**: 두 지점 간 거리 계산

---

## 초기 데이터

`_doc/db/gsl/` 폴더에 공간정보 초기 데이터가 있다:
- `gsl_admdst_bsc.sql` - 행정구역
- `gsl_asp_bsc.sql` - 공역
- `gsl_cmptnc_inst_bsc.sql` - 관할기관
- `gsl_cmptnc_inst_admdst_rel.sql` - 관할기관-행정구역 관계

---

## 관련 문서
- [[module-fpl]] - 비행계획 (공역 검증에 GSL 사용)
- [[app-engine]] - 공간 연산 처리
- [[common-모듈정리]] - GeoTools 유틸리티
