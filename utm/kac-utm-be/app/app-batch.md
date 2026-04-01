# App: Batch (배치 처리)

#kac-utm #backend #app

| 항목 | 값 |
|------|---|
| **포트** | 8800 |
| **JAR명** | utm-batch |
| **패키지** | `com.kac.utm.batch` |
| **메인 클래스** | `BatchApplication.java` |
| **프로필** | db, connector, redis, gsl |
| **클래스 수** | 67개 |

---

## 역할

**정기적인 배치 작업을 처리**하는 모듈이다. Quartz 스케줄러를 사용하여 DB 기반으로 Job을 관리하며, 관리자 포털([[app-mngr]])에서 스케줄을 동적으로 변경할 수 있다.

> **언제 이 코드를 보게 되나?**
> 정기 통계가 안 나오거나, 사용자 계정 자동 처리(비활성화, 탈퇴)가 제대로 안 될 때. 새로운 배치 작업을 추가해야 할 때.

---

## 패키지 구조

```
com.kac.utm.batch/
├── config/              ← Quartz 설정
├── job/                 ← 배치 작업 정의
│   ├── aerl/
│   │   └── snrsns/      ← 센서 데이터 배치
│   ├── com/
│   │   └── user/        ← 사용자 계정 관리
│   │       ├── del/     ← 삭제 처리
│   │       ├── drmc/    ← 비활성화(휴면) 처리
│   │       └── whdwl/   ← 탈퇴 처리
│   ├── ctrl/
│   │   ├── end/         ← 비행 종료 처리
│   │   └── rgn/         ← 지역 처리
│   └── stats/
│       ├── abn/         ← 이상상황 통계 생성
│       ├── flyng/       ← 비행 통계 생성
│       └── prfmnc/      ← 성능 통계 생성
├── module/
│   ├── ctrl/
│   │   └── service/     ← Control Batch Service
│   └── stats/
│       ├── model/
│       ├── repository/
│       └── service/     ← Stats Batch Service
├── web/
│   ├── api/
│   │   ├── CommonController   ← 헬스체크
│   │   └── QuartzController   ← Quartz 스케줄 관리 API
│   └── service/
│       └── QuartzService      ← Quartz 서비스
└── BatchApplication.java
```

---

## 주요 배치 작업

| 작업 | 설명 |
|------|------|
| **센서 데이터 정제** | SNR/SNS 센서 데이터 수집 및 저장 |
| **사용자 삭제** | 조건에 해당하는 사용자 계정 삭제 |
| **사용자 비활성화** | 장기 미접속 사용자 휴면 처리 |
| **사용자 탈퇴** | 탈퇴 요청 처리 |
| **비행 종료** | 미종료 비행 자동 종료 처리 |
| **이상상황 통계** | 일별 이상상황 통계 생성 |
| **비행 통계** | 일별 비행 통계 생성 |
| **성능 통계** | 성능 지표 통계 생성 |

---

## Quartz 설정

| 항목 | 값 |
|------|---|
| 저장 방식 | JDBC (DB에 Job/Trigger 저장) |
| 스레드 풀 | 20개 |
| 클러스터 | 지원 (여러 인스턴스 동시 실행 시 중복 방지) |
| 테이블 프리픽스 | `QRTZ_` |
| 스케줄 관리 | QuartzController를 통해 동적 변경 가능 |

---

## 관련 문서
- [[app-mngr]] - 관리자 포털 (배치 관리 UI)
- [[module-stats]] - 통계 도메인
- [[module-psty]] - 관제 도메인 (비행 종료 관련)
