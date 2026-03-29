# 06. 개발 환경 셋업 가이드

> 필수 도구, 로컬 환경 구성, docker-compose, 모듈 실행 방법, IDE 설정

---

## 목차

1. [필수 도구](#1-필수-도구)
2. [프로젝트 구조 & Git](#2-프로젝트-구조--git)
3. [로컬 인프라 (Docker)](#3-로컬-인프라-docker)
4. [Common 모듈 빌드](#4-common-모듈-빌드)
5. [각 모듈 실행](#5-각-모듈-실행)
6. [IDE 설정 (IntelliJ)](#6-ide-설정-intellij)
7. [환경 변수 & 프로파일](#7-환경-변수--프로파일)
8. [Docker 빌드 (배포)](#8-docker-빌드-배포)
9. [트러블슈팅](#9-트러블슈팅)

---

## 1. 필수 도구

| 도구 | 버전 | 비고 |
|------|------|------|
| **Java (JDK)** | 1.8 (Zulu 8 권장) | `.java-version` 파일 참조 |
| **Maven** | 3.8.7 | 각 모듈에 mvnw 포함 |
| **Docker** | 최신 | Valkey 로컬 실행용 |
| **IntelliJ IDEA** | 최신 | `.idea/` 설정 포함 |
| **Git** | 최신 | 모듈별 독립 저장소 |
| **MySQL Client** | 8.0+ | DB 접근용 (선택) |

---

## 2. 프로젝트 구조 & Git

```
/Users/han/works/workspace/palnet/sumair/be/
├── docker-compose.yml          # Valkey 로컬 환경
├── sumair-be-common/  (.git)   # 공유 라이브러리
├── sumair-be-home/    (.git)   # 메인 포탈 (:8000)
├── sumair-be-ibe/     (.git)   # 예약 엔진 (:8100)
├── sumair-be-pay/     (.git)   # 결제 (:8200)
├── sumair-be-batch/   (.git)   # 배치 (:8300)
└── sumair-be-message/ (.git)   # 메시지 (:8400)
```

**각 모듈이 독립 Git 저장소**입니다. 루트 디렉토리에는 `.git`이 없습니다.
- 커밋, 브랜치, 태그가 모듈별로 독립 관리
- `common` 모듈 변경 시 다른 모듈에도 반영 필요 (mvn install)

---

## 3. 로컬 인프라 (Docker)

### Valkey (Redis 호환) 실행

```bash
cd /Users/han/works/workspace/palnet/sumair/be
docker-compose up -d
```

**docker-compose.yml:**
```yaml
services:
  valkey:
    image: valkey/valkey:latest
    ports:
      - "7379:6379"        # 로컬 7379 → 컨테이너 6379
    command: valkey-server --appendonly yes --save ""
    volumes:
      - /data/valkey-data/:/data
```

### DB 접속

로컬에서는 AWS Aurora Dev 환경에 직접 연결합니다.
- **Host**: `sumair-dev-aurora.cluster-chh3noeqbmvd.ap-northeast-2.rds.amazonaws.com`
- **Port**: 3306
- **Schema**: sumair
- **Username**: admin
- **Password**: Jasypt 암호화 (별도 전달 필요)

> **주의**: 로컬 MySQL 없음. Dev Aurora에 직접 연결하므로 VPN 또는 네트워크 설정 필요할 수 있음.

---

## 4. Common 모듈 빌드

**모든 모듈이 common에 의존**하므로 반드시 먼저 빌드합니다.

```bash
cd /Users/han/works/workspace/palnet/sumair/be/sumair-be-common
./mvnw clean install -DskipTests
```

이 명령은 `com.sumair:sumair-common:0.0.1`을 로컬 `.m2` 저장소에 설치합니다.

**Common 변경 시 반드시 다시 `install` 후 의존 모듈 재시작해야 합니다.**

---

## 5. 각 모듈 실행

### 빌드 & 실행

```bash
# 예: Home 모듈
cd /Users/han/works/workspace/palnet/sumair/be/sumair-be-home
./mvnw clean package -DskipTests
java -jar target/sumair-home-0.0.1.jar --spring.profiles.active=local
```

### 모듈별 포트

| 모듈 | 포트 | 프로파일 | JAR명 |
|------|------|---------|------|
| home | 8000 | local | sumair-home-0.0.1.jar |
| ibe | 8100 | local | sumair-ibe-0.0.1.jar |
| pay | 8200 | local | sumair-pay-0.0.1.jar |
| batch | 8300 | local | sumair-batch-0.0.1.jar |
| message | 8400 | local | sumair-message-0.0.1.jar |

### 최소 실행 구성

전체 기능 테스트가 아닌 경우:
- **필수**: `home` (인증, 회원, 기본 API)
- **예약 테스트**: `home` + `ibe`
- **결제 테스트**: `home` + `ibe` + `pay`
- **메시지 테스트**: `message`
- **배치 테스트**: `batch`

---

## 6. IDE 설정 (IntelliJ)

### 기존 설정

프로젝트에 `.idea/` 디렉토리가 포함되어 있어 IntelliJ에서 바로 열 수 있습니다.

- **JDK**: `zulu-1.8` (Preferences → Project Structure → SDK)
- **인코딩**: UTF-8
- **모듈**: 6개 모듈 모두 등록됨
- **DB 연결**: `.idea/dataSources/` 에 설정 포함

### Lombok 설정

별도 `lombok.config` 없으므로 기본 설정 사용:
1. IntelliJ Lombok 플러그인 설치
2. Settings → Build → Compiler → Annotation Processors → Enable annotation processing 체크

### 실행 구성 (Run Configuration)

각 모듈의 `Application` 클래스를 IntelliJ에서 직접 실행:

```
Main class: com.sumair.home.SumairHomeApplication
Active profiles: local
VM options: -Duser.timezone=Asia/Seoul
```

### Swagger UI (로컬)

| 모듈 | Swagger URL |
|------|------------|
| home | http://localhost:8000/swagger-ui/index.html |
| ibe | http://localhost:8100/swagger-ui/index.html |
| pay | http://localhost:8200/swagger-ui/index.html |
| batch | http://localhost:8300/swagger-ui/index.html |
| message | http://localhost:8400/swagger-ui/index.html |

---

## 7. 환경 변수 & 프로파일

### Spring 프로파일

| 프로파일 | 용도 | DB | Redis |
|---------|------|-----|-------|
| `local` | 로컬 개발 | Aurora Dev | localhost:7379 |
| `dev` | 개발 서버 | Aurora Dev | AWS Valkey |
| `prod` | 운영 서버 | Aurora Prod | AWS Valkey |

### 필수 환경 변수

| 변수 | 용도 | 기본값 |
|------|------|--------|
| `JASYPT_ENCRYPTOR_PASSWORD` | Jasypt 복호화 키 | `SUMAIR-JASYPT-KEY` (yml에 하드코딩) |

> 현재 Jasypt 키가 `application-common.yml`에 평문으로 포함되어 있어 별도 환경변수 설정 없이 실행 가능합니다. 프로덕션에서는 환경변수로 분리 권장.

### 설정 파일 로드 순서

```
application-init.yml
  → application-databases.yml  (DB 설정)
  → application-common.yml     (JWT, 보안, Redis)
  → application-ibe.yml        (IBES/IBEP SOAP)
  → application-quartz.yml     (배치 - batch 모듈만)
  → application.yml            (모듈별 개별 설정)
```

---

## 8. Docker 빌드 (배포)

### Multi-stage 빌드 패턴

```
[Stage 1: Builder]
  ├─ Base: ECR에서 common 빌드 이미지
  ├─ Common .m2 캐시 복사
  ├─ mvnw clean package -DskipTests
  └─ JAR 생성

[Stage 2: Runtime]
  ├─ Base: eclipse-temurin-8
  ├─ JAR 복사
  ├─ JVM 옵션:
  │   -Duser.timezone=Asia/Seoul
  │   -XX:+UseContainerSupport
  │   -XX:MaxRAMPercentage=75.0
  │   -XX:+UseG1GC
  └─ Spring Profile: dev,swagger 또는 prod,swagger
```

### 빌드 명령 예시

```bash
# Dev 환경
cd sumair-be-home
docker build -f Dockerfile.dev -t sumair-home:dev .

# Prod 환경
docker build -f Dockerfile.prod -t sumair-home:prod .
```

### ECR 이미지

```
072062626370.dkr.ecr.ap-northeast-2.amazonaws.com/
```

---

## 9. 트러블슈팅

### Common 모듈 의존성 에러

```
Could not find artifact com.sumair:sumair-common:jar:0.0.1
```
**해결:** `sumair-be-common`에서 `./mvnw clean install -DskipTests` 실행

### Valkey 연결 실패

```
Unable to connect to Redis; nested exception is io.lettuce.core.RedisConnectionException
```
**해결:** `docker-compose up -d`로 Valkey 실행 확인, 포트 7379 확인

### DB 연결 실패

```
Communications link failure
```
**해결:** VPN 연결 확인, Aurora Dev 클러스터 상태 확인

### Jasypt 복호화 실패

```
ENC(xxx) - EncryptionOperationNotPossibleException
```
**해결:** `JASYPT_ENCRYPTOR_PASSWORD` 환경변수 또는 `application-common.yml`의 jasypt.key 확인

### SOAP 관련 빌드 에러 (IBE/Message)

CXF WSDL 코드 생성 관련 에러 시:
- `sumair-be-common/src/main/generated/` 경로에 생성된 코드 존재 확인
- Maven `generate-sources` 페이즈 확인
