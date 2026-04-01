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

## 배치 Job 상세

### CtrlEndBatchService (관제 종료)
```
1. 처리 상태 'R' (Running)인 관제 데이터 조회
2. 마지막 이력 생성 시간으로부터 경과 시간 계산
3. 경과 > GP_END_TIME (기본 1분) 이면:
   - 관제 상태 → END
   - 총 비행시간 계산 (시작~종료 시간 차이)
   - 총 비행거리 계산 (Haversine 공식, 5000건 청크)
   - Redis 비정상 상황 데이터 삭제
4. 에러 시 처리 상태 → 'F'
```

### CtrlRgnBatchService (관제 지역)
```
1. 처리 상태 'R'인 관제 지역 데이터 조회
2. 좌표(위도/경도) → 행정구역 변환 (VWorld API)
   - 도/광역시, 시/군/구, 읍/면/동
   - 우편번호, 도로명 주소
3. 처리 상태 → 'S' (성공) 또는 'F' (실패)
```

### StatsAbnSitBatchService (이상상황 통계)
```
- Chunk 기반 배치 (ChunkSize: 1000)
- 스케줄: 매일 자정 (cron: "0 0 0 * * ?")
- Reader → Processor → Writer 패턴
```

### 사용자 관리 배치
- **del**: 삭제 대상 사용자 처리
- **drmc**: 장기 미접속 사용자 휴면 처리
- **whdwl**: 탈퇴 요청 사용자 처리

### Quartz API (`/api/v1/batch/quartz`)

| HTTP | URL | 설명 |
|------|-----|------|
| POST | `/job/run/{jobNm}` | Job 즉시 실행 |
| PUT | `/trigger/cron` | Cron 스케줄 변경 |
| GET | `/job/details` | Job/Trigger 상세 조회 |
| GET | `/job/names` | 모든 Job Name |
| GET | `/trigger/names` | 모든 Trigger Name |

---

## 관련 문서
- [[04_앱간 연관관계]] - 앱 간 통신 상세
- [[app-mngr]] - 관리자 포털 (배치 관리 UI)
- [[module-stats]] - 통계 도메인
- [[module-psty]] - 관제 도메인 (비행 종료 관련)
