# Sumair BE - 알려진 이슈 & 기술 부채

## 1. 인프라 & DevOps

### 1.1 CI/CD 파이프라인 미구성

**현황**: CI/CD 자동화 파이프라인이 구성되어 있지 않음. Jenkinsfile, GitHub Actions, CodePipeline 등의 설정 파일이 없음.

**영향**:
- Docker 이미지 빌드 및 ECR 푸시가 수동으로 이루어짐
- 배포 시 휴먼 에러 가능성 존재
- 코드 리뷰 없이 배포 가능
- 테스트 자동 실행 없음

**권장 조치**:
- GitHub Actions 또는 AWS CodePipeline을 활용한 CI/CD 구축
- 파이프라인 단계: lint → build → test → Docker build → ECR push → deploy
- 환경별 (dev/prod) 분리된 파이프라인 구성
- PR 기반 배포 승인 워크플로우 도입

### 1.2 독립 Git 저장소 관리 복잡성

**현황**: 6개 모듈이 각각 독립된 `.git` 저장소를 보유하고 있으나, 모노레포처럼 하나의 디렉토리 하위에 배치됨.

**영향**:
- Common 모듈 변경 시 의존 모듈과의 버전 동기화가 수동
- 통합 빌드/테스트가 어려움
- 이력 추적 시 여러 저장소를 교차 확인해야 함

**권장 조치**:
- Git Submodule 또는 모노레포 전환 검토
- 최소한 Common 모듈 버전 태깅 및 변경 공지 프로세스 수립
- 통합 빌드 스크립트 작성

### 1.3 Docker 이미지 태깅 전략 부재

**현황**: 모든 이미지가 `latest` 태그만 사용 (`dev-sumair-{module}:latest`, `prod-sumair-{module}:latest`).

**영향**:
- 특정 버전으로의 롤백이 어려움
- 어떤 코드가 배포되어 있는지 추적 불가
- 이미지 캐시 무효화 문제

**권장 조치**:
- 시맨틱 버저닝 또는 Git commit SHA 기반 태깅 도입
- 예: `prod-sumair-home:v1.2.3` 또는 `prod-sumair-home:abc1234`
- `latest` 태그는 병행 유지하되, 배포 시 명시적 태그 사용

## 2. 데이터베이스

### 2.1 DB 마이그레이션 도구 미사용

**현황**: Flyway, Liquibase 등 DB 마이그레이션 도구를 사용하지 않음. 스키마 변경이 수동 SQL 실행으로 이루어짐.

**영향**:
- 스키마 변경 이력 추적 불가
- 환경 간 스키마 불일치 가능성
- 배포 시 DB 변경과 애플리케이션 배포의 순서/호환성 관리 어려움
- 롤백 시 DB 변경 되돌리기가 수동

**권장 조치**:
- Flyway 도입 (Spring Boot 내장 지원, 설정 간단)
- 기존 스키마를 baseline으로 설정하고 이후 변경분부터 마이그레이션 파일 관리
- 각 모듈의 마이그레이션 파일을 `src/main/resources/db/migration/`에 관리

### 2.2 커넥션 풀 사이즈 매우 보수적

**현황**: HikariCP max-pool-size가 메인 DB 5, 메시지 DB 3으로 설정됨.

**영향**:
- 트래픽 급증 시 커넥션 풀 고갈로 요청 실패 가능
- 특히 SOAP 통신 지연 시 커넥션 점유 시간 증가로 병목 발생 우려

**권장 조치**:
- 서비스별 트래픽 패턴 분석 후 적정 풀 사이즈 산정
- 일반적으로 `max-pool-size = (core_count * 2) + disk_spindles` 공식 참고
- 최소 10~20 수준으로 상향 검토

### 2.3 Prod 비밀번호 평문 노출

**현황**: `application-databases.yml`의 Prod 프로파일에서 DB 비밀번호가 Jasypt 암호화 없이 평문으로 존재.

**영향**:
- Git 저장소 접근 가능한 누구나 Prod DB 비밀번호 확인 가능
- 보안 감사 시 지적 대상

**권장 조치**:
- 모든 Prod 비밀번호에 Jasypt 암호화 적용 (`ENC(...)` 래핑)
- 장기적으로 AWS Secrets Manager로 전환

## 3. 보안

### 3.1 민감 정보 설정 파일 내 하드코딩

**현황**: 다수의 API 키, 시크릿, 인증서 비밀번호가 `application.yml`에 직접 기록됨.

**영향을 받는 정보**:
- Toss Payments API 키 (test/live)
- Infobip API 키
- Slack Webhook URL
- OAuth2 Client Secret (Google, Kakao, Naver)
- Apple/Samsung Wallet 인증서 비밀번호
- IBES/IBEP API 키 및 인증 정보
- Samsung Wallet API 토큰

**권장 조치**:
- **단기**: 모든 민감 값에 Jasypt 암호화 적용
- **중기**: AWS Secrets Manager 또는 Systems Manager Parameter Store로 외부화
- **장기**: Vault 같은 시크릿 관리 시스템 도입 검토

### 3.2 Jasypt 마스터 키 하드코딩

**현황**: `SUMAIR-JASYPT-KEY`가 코드 내 상수 또는 설정 파일에 직접 포함됨.

**영향**:
- 마스터 키 노출 시 모든 암호화된 값이 복호화 가능
- Jasypt 암호화의 보안 가치가 크게 감소

**권장 조치**:
- Jasypt 키를 환경 변수로 주입: `JASYPT_ENCRYPTOR_PASSWORD` 환경변수 사용
- 또는 Docker ENTRYPOINT에서 `-Djasypt.encryptor.password=${SECRET}` 방식으로 전달

## 4. 아키텍처

### 4.1 서비스 간 동기 호출 의존성

**현황**: 모든 서비스 간 통신이 WebClient를 통한 동기적 HTTP 호출로 이루어짐. 메시지 큐나 이벤트 버스 미사용.

**영향**:
- 하위 서비스 장애 시 상위 서비스까지 연쇄 장애 가능 (cascading failure)
- 예약 → 결제 → 알림 플로우에서 한 단계 실패 시 전체 트랜잭션 처리 복잡

**권장 조치**:
- 알림 발송 등 비동기 처리 가능한 영역에 메시지 큐 도입 (SQS, SNS 등)
- Circuit Breaker 패턴 도입 (Resilience4j)
- 서비스 간 타임아웃/재시도 정책 표준화

### 4.2 공유 DB 구조

**현황**: home, ibe, pay, batch 모듈이 동일한 Aurora 클러스터의 `sumair` 데이터베이스를 공유.

**영향**:
- 서비스 간 데이터 경계가 명확하지 않아 마이크로서비스 독립 배포의 이점 감소
- 한 서비스의 대량 쿼리가 다른 서비스에 영향
- DB 스키마 변경 시 다수 서비스에 영향

**권장 조치**:
- 당장은 현실적으로 유지하되, 서비스별 테이블 소유권 명확화
- 타 서비스 테이블 직접 접근 금지 원칙 수립 (API를 통한 데이터 접근)
- 장기적으로 서비스별 스키마/DB 분리 검토

### 4.3 Common 모듈 비대화 위험

**현황**: Common 모듈에 보안, REST/WebClient, 도메인 객체, 설정 등 광범위한 코드가 포함됨.

**영향**:
- Common 변경 시 모든 서비스 재빌드/재배포 필요
- 모듈 간 결합도 증가
- 불필요한 의존성까지 각 서비스에 포함됨

**권장 조치**:
- Common 모듈을 기능별 하위 모듈로 분리 검토 (common-security, common-web, common-domain 등)
- 특정 서비스에서만 사용하는 코드는 해당 서비스로 이동

## 5. 기술 스택

### 5.1 Java 8 / Spring Boot 2.7 EOL

**현황**: Java 1.8과 Spring Boot 2.7.10을 사용 중. Spring Boot 2.7.x는 2023년 11월 OSS 지원 종료됨.

**영향**:
- 보안 패치 미제공
- 최신 라이브러리/프레임워크와의 호환성 문제 증가
- 신규 개발자 채용/온보딩 시 구버전 스택 장벽

**권장 조치**:
- **1단계**: Spring Boot 3.x + Java 17 마이그레이션 계획 수립
- **2단계**: 의존성 호환성 점검 (CXF, MyBatis, AWS JDBC Wrapper 등)
- **3단계**: 모듈별 순차 마이그레이션 (common → 나머지)
- 특히 `javax.*` → `jakarta.*` 네임스페이스 변경 주의

### 5.2 Swagger (Springfox) 지원 중단

**현황**: Springfox 3.0.0 사용 중. Springfox는 2020년 이후 유지보수 중단.

**권장 조치**:
- SpringDoc (OpenAPI 3.0) 전환 검토
- Spring Boot 3.x 마이그레이션 시 함께 전환

### 5.3 일부 라이브러리 구버전

| 라이브러리      | 현재 버전 | 비고                       |
| ---------- | ----- | ------------------------ |
| Apache POI | 3.15  | 5.x 사용 권장, 보안 취약점 존재 가능  |
| XStream    | 1.4.9 | CVE 다수 보고, 1.4.20+ 사용 권장 |
| jjwt       | 0.9.1 | 0.12.x 사용 권장             |

## 6. 테스트

### 6.1 테스트 코드 부재/부족

**현황**: `mvn package -DskipTests`로 빌드하고 있어, 테스트 코드 존재 여부 및 실행 가능 여부 불명확.

**영향**:
- 코드 변경 시 사이드 이펙트 탐지 불가
- 리팩토링 안전망 부재
- CI/CD 도입 시 자동 테스트 게이트가 무의미

**권장 조치**:
- 주요 비즈니스 로직 (예약, 결제)에 대한 단위 테스트 작성
- 서비스 간 통합 테스트 작성
- 빌드 시 `-DskipTests` 제거 목표 설정

## 7. 운영

### 7.1 모니터링/APM 미구성

**현황**: Spring Boot Actuator 최소 설정, 별도 APM(Application Performance Monitoring) 미사용.

**권장 조치**:
- Actuator 엔드포인트 확장 (metrics, prometheus, health details)
- CloudWatch Container Insights 또는 Datadog/New Relic APM 도입
- 분산 추적(Distributed Tracing) 도입 검토 (Sleuth + Zipkin 또는 AWS X-Ray)

### 7.2 헬스체크 엔드포인트 표준화 필요

**현황**: 서비스별 헬스체크 엔드포인트 구성이 불명확.

**권장 조치**:
- 모든 서비스에 `/actuator/health` 표준 엔드포인트 노출
- DB, Cache, 외부 API 연결 상태를 포함한 상세 헬스체크 구성
- 컨테이너 오케스트레이션의 liveness/readiness probe와 연동

---

## 우선순위 요약

| 우선순위 | 항목 | 카테고리 |
|---------|------|---------|
| **P1 (긴급)** | Prod 비밀번호 평문 노출 | 보안 |
| **P1 (긴급)** | Jasypt 마스터 키 하드코딩 | 보안 |
| **P1 (긴급)** | 민감 정보 설정 파일 하드코딩 | 보안 |
| **P2 (높음)** | CI/CD 파이프라인 구축 | 인프라 |
| **P2 (높음)** | Docker 이미지 태깅 전략 | 인프라 |
| **P2 (높음)** | DB 마이그레이션 도구 도입 | DB |
| **P3 (중간)** | XStream, POI 등 구버전 라이브러리 업데이트 | 보안/기술 |
| **P3 (중간)** | 모니터링/APM 도입 | 운영 |
| **P3 (중간)** | 테스트 코드 작성 | 품질 |
| **P4 (장기)** | Java 17 + Spring Boot 3.x 마이그레이션 | 기술 |
| **P4 (장기)** | 서비스 간 비동기 통신 도입 | 아키텍처 |
| **P4 (장기)** | 서비스별 DB 분리 | 아키텍처 |

---

> **문서 버전**: 2026-03-22 | **작성**: 시스템 아키텍트
> **참고**: 이 문서는 아키텍트 관점에서 작성되었으며, 프론트엔드/백엔드 팀의 의견 반영이 필요합니다.
