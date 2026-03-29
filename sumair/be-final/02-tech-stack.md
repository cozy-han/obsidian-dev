# 기술 스택

## 핵심 프레임워크

| 항목 | 기술 | 버전 |
|------|------|------|
| Framework | Spring Boot | 2.7.10 |
| Language | Java | 1.8 |
| Build | Maven | 3.x (mvnw 포함) |
| ORM | MyBatis | 3.5.0 |
| API Docs | SpringFox (Swagger) | 3.0.0 |
| Container | Docker Compose | - |

## 보안 / 인증

| 항목 | 기술 | 상세 |
|------|------|------|
| 인증 | JWT | jjwt 0.9.1, HS256, Auth(30분) + Refresh(30일) |
| 소셜 로그인 | OAuth2 Client | Naver, Kakao, Google |
| 암호화 | AES-256 | 개인정보 암/복호화 |
| 설정 암호화 | Jasypt | 3.0.5, application.yml 민감정보 |
| 비밀번호 | BCrypt | jbcrypt |
| 데이터 마스킹 | 커스텀 AOP | PII 마스킹 처리 |

## 데이터베이스 / 캐시

| 항목 | 기술 | 상세 |
|------|------|------|
| RDBMS | MySQL → Aurora MySQL | AWS RDS, Read/Write 분리 |
| ORM | MyBatis | XML Mapper 기반, Base* 공통 매퍼 101개 |
| 캐시 | Redis (Valkey) | AWS ElastiCache, 항공편 검색 캐시 (TTL 180초) |
| 커넥션 풀 | HikariCP | Spring Boot 기본 |
| Quartz DB | c3p0 | Batch 모듈 Quartz 전용 |

## SOAP / 외부 연동

| 항목 | 기술 | 상세 |
|------|------|------|
| SOAP Client | Apache CXF | 3.5.5, IBES/IBEP 연동 |
| SOAP Endpoint | CXF Server | MESSAGE 모듈 /ws/ires (IRES 콜백 수신) |
| HTTP Client | Spring WebClient | 모듈 간 내부 통신, 외부 REST API |
| PG | Toss Payments | BrandPay + 일반결제, Idempotency-Key |
| 메시징 | Infobip | SMS(8218774325) + Email(sumair@mail.sumair.kr) |
| 공공데이터 | data.go.kr | 공휴일 API, 동네예보 API (XML) |
| FTP | JSch (SFTP) | IBS Revenue Report, Flight Schedule |
| 알림 | Slack API Client | 에러 알림 Webhook |

## 배치 / 스케줄링

| 항목 | 기술 | 상세 |
|------|------|------|
| 배치 프레임워크 | Spring Batch | 배치 처리 기반 |
| 스케줄러 | Quartz | 클러스터 모드, DB 기반 트리거 관리 |
| 주요 배치 | HolidayJob, WeatherJob, ScheduleJob, PointExtinctJob, PointSaveJob, RevenueReportJob | |

## 유틸리티 / 기타

| 항목 | 기술 | 용도 |
|------|------|------|
| commons-lang3 | Apache Commons | 문자열/객체 유틸 |
| ModelMapper | - | DTO ↔ Entity 변환 |
| iText | - | PDF 생성 |
| Apache POI | - | Excel 파싱 (Revenue Report) |
| ZXing | - | QR 코드 생성 (보딩패스) |
| XStream | - | XML 직렬화 |
| PageHelper | MyBatis 페이징 | 리스트 페이징 처리 |
| log4jdbc | SQL 로깅 | 개발 환경 SQL 로그 (Aurora 마이그레이션 시 제거 예정) |

## 모듈별 고유 의존성

### sumair-be-home
- `spring-boot-starter-security` — Spring Security
- `spring-boot-starter-data-redis` — Redis 세션/캐시
- `spring-boot-starter-oauth2-client` — OAuth2 소셜 로그인
- `jjwt 0.9.1` — JWT 토큰 생성/검증
- `pagehelper` — MyBatis 페이징

### sumair-be-ibe
- `cxf-rt-frontend-jaxws` — CXF SOAP Client
- `cxf-rt-transports-http` — CXF HTTP 전송

### sumair-be-pay
- Toss Payments REST 연동 (별도 SDK 없이 WebClient 직접 호출)

### sumair-be-batch
- `spring-boot-starter-batch` — Spring Batch
- `spring-boot-starter-quartz` — Quartz 스케줄러
- `c3p0` — Quartz 전용 커넥션 풀

### sumair-be-message
- `cxf-rt-frontend-jaxws` — CXF SOAP Server (IRES 콜백)
- `cxf-rt-transports-http` — CXF HTTP 전송
- Infobip REST 연동

## 빌드 / 배포

```bash
# 빌드 (테스트 스킵)
./mvnw clean install -Dmaven.test.skip=true

# 로컬 실행 (Docker Compose - Valkey)
docker-compose up -d   # valkey:7379 → 6379

# 환경 프로파일
-Dspring.profiles.active=local|dev|prod
```

| 환경 | 설명 | 특이사항 |
|------|------|---------|
| local | 로컬 개발 | Docker Valkey, 로컬 MySQL |
| dev | 개발 서버 | AWS 환경, Toss 테스트 키 |
| prod | 운영 서버 | AWS CodeBuild/CodeDeploy, Toss 라이브 키 |
