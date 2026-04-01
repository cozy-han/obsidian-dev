# Module: AERL (항공정보)

#kac-utm #backend #module

| 항목 | 값 |
|------|---|
| **경로** | `module/aerl` |
| **Gradle** | `:module:module-aerl` |
| **패키지** | `com.kac.utm.aerl` |
| **클래스 수** | 23개 |

---

## 역할

**항공정보(Aeronautical)** 도메인을 관리한다. NOTAM(항공 공지사항), 기상정보(METAR/TAF/SIGMET/AIRMET), 센서 데이터를 처리한다.

> **언제 이 코드를 보게 되나?**
> NOTAM 표시 기능, 기상 정보 조회 화면, 센서 데이터 관련 기능을 개발할 때.

---

## 주요 엔티티

### AerlNotamBscEntity (NOTAM)

```
PK: notamSn (NOTAM 일련번호)

주요 필드:
- notamId: NOTAM ID
- flyngInfoZoneCd: 비행정보구역 코드
- qCd: NOTAM Q 코드 (유형 분류)
- vldBgngDt / vldEndDt: 유효 기간
- telgmCn: NOTAM 텔레그램 내용 (원문)
- notamGeom: 지리공간 정보 (Geometry)
- notamJson: JSON 형식 데이터
- useYn: 사용 여부
```

### AerlSnrSnsBscEntity (센서 SNS)
- 슈어 SNS(센서) 서비스 데이터

---

## 모델 클래스

| 모델 | 설명 |
|------|------|
| NotamModel | NOTAM 조회 결과 |
| NotamSerachModel | NOTAM 검색 조건 |
| WthrMetarRs | METAR(공항기상보고) 응답 |
| WthrTafRs | TAF(터미널 예보) 응답 |
| WthrSigmetRs | SIGMET(중요기상정보) 응답 |
| WthrAirmetRs | AIRMET(비행장 기상정보) 응답 |
| CtrlWthrRq/Rs | 제어기상 요청/응답 |
| SnrSnsRq/Rs | 센서 SNS 요청/응답 |

---

## 기상정보 용어

| 약어 | 정식 명칭 | 설명 |
|------|----------|------|
| **METAR** | METeorological Aerodrome Report | 공항 기상 관측 보고 |
| **TAF** | Terminal Aerodrome Forecast | 공항 기상 예보 |
| **SIGMET** | Significant Meteorological Information | 중요 기상 정보 (위험 기상) |
| **AIRMET** | Airmen's Meteorological Information | 비행사 기상 정보 |
| **NOTAM** | Notice to Airmen | 항공 종사자 공지사항 |

---

## 관련 문서
- [[common-모듈정리]] - common-connector (외부 기상 API 클라이언트)
- [[app-engine]] - 기상 데이터 처리
- [[app-web]] - 기상 정보 표시
