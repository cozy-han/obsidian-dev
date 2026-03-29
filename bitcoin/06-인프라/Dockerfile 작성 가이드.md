# Dockerfile 작성 가이드

> 최종 수정: 2026-03-08
> 프로젝트: bitcoin-auto-trader (Gradle 멀티모듈, Kotlin, Java 21)

---

## 1. Docker 이미지란?

Docker 이미지는 애플리케이션을 실행하는 데 필요한 **모든 것**(코드, 런타임, 라이브러리, 설정)을 패키징한 읽기 전용 템플릿이다.

```
Docker 이미지 = 앱 코드 + JDK + OS 기본 라이브러리 + 설정
    ↓ 실행
Docker 컨테이너 = 이미지의 실행 인스턴스 (프로세스)
```

Kubernetes는 이 이미지를 가져와서(pull) 컨테이너로 실행한다.

## 2. 멀티 스테이지 빌드란?

Dockerfile에서 **여러 단계(stage)** 를 나누어 빌드하는 기법이다.

```
일반 빌드 (하나의 스테이지):
├── JDK 21 (~300MB)
├── Gradle 캐시 (~500MB)
├── 소스 코드 (~50MB)
├── 빌드 도구
└── 최종 JAR (~80MB)
→ 이미지 크기: ~1GB+  ← 불필요한 것들이 포함됨

멀티 스테이지 빌드 (두 스테이지):
Stage 1 "builder" (빌드 전용, 최종 이미지에 포함 안 됨):
├── JDK 21 Full
├── Gradle
├── 소스 코드
└── → JAR 파일 생성

Stage 2 "runtime" (최종 이미지):
├── JRE 21 (~100MB)  ← JDK보다 훨씬 작음
└── JAR 파일만 복사
→ 이미지 크기: ~200MB  ← 훨씬 작고 안전함
```

**장점:**
1. **이미지 크기 감소**: 빌드 도구가 최종 이미지에 포함되지 않음
2. **보안 향상**: 소스코드, 빌드 도구가 프로덕션 이미지에 없음
3. **빌드 캐시**: Docker 레이어 캐싱으로 빌드 속도 향상

## 3. Dockerfile 작성

### 3.1 파일 위치

```
bitcoin/
├── docker/
│   ├── Dockerfile           ← 멀티스테이지 (CI + 로컬 개발 공용)
│   ├── Dockerfile.ci        ← 슬림 버전 (레거시, 현재 미사용)
│   └── docker-compose.yml   ← 기존 로컬 개발용
```

> CI(Woodpecker) 파이프라인에서 kaniko가 이 `Dockerfile`을 직접 빌드한다.
> 별도 Gradle 빌드 Step 없이 멀티스테이지로 빌드+패키징을 한 번에 처리한다.

### 3.2 Dockerfile 전문

```dockerfile
# ============================================================
# Stage 1: Build (빌드 스테이지)
# ============================================================
# Gradle 빌드에 JDK가 필요하므로 JDK 이미지 사용
FROM eclipse-temurin:21-jdk-alpine AS builder

# 작업 디렉토리 설정
WORKDIR /app

# --- Gradle Wrapper 및 설정 파일만 먼저 복사 (의존성 캐시용) ---
# Docker는 레이어 단위로 캐시한다.
# 이 파일들이 변경되지 않으면 의존성 다운로드를 건너뛴다.
COPY gradlew .
COPY gradle/ gradle/
COPY build.gradle.kts .
COPY settings.gradle.kts .
COPY gradle/libs.versions.toml gradle/

# 각 모듈의 빌드 파일 복사 (의존성 해석에 필요)
COPY core/build.gradle.kts core/
COPY api-client/build.gradle.kts api-client/
COPY strategy/build.gradle.kts strategy/
COPY execution/build.gradle.kts execution/
COPY monitoring/build.gradle.kts monitoring/
COPY app/build.gradle.kts app/

# Gradle Wrapper 실행 권한 부여
RUN chmod +x gradlew

# 의존성 미리 다운로드 (캐시 레이어)
# --no-daemon: 컨테이너 환경에서는 데몬 불필요
# dependencies: 의존성만 해석하고 다운로드
RUN ./gradlew dependencies --no-daemon

# --- 소스 코드 복사 및 빌드 ---
COPY core/src core/src
COPY api-client/src api-client/src
COPY strategy/src strategy/src
COPY execution/src execution/src
COPY monitoring/src monitoring/src
COPY app/src app/src

# bootJar 실행 (실행 가능한 Fat JAR 생성)
# -x test: 테스트 건너뛰기 (CI에서 이미 실행했으므로)
# hsperfdata 정리: kaniko 스냅샷 시 JVM 임시 파일 오류 방지
RUN ./gradlew :app:bootJar -x test --no-daemon && rm -rf /tmp/hsperfdata_*

# ============================================================
# Stage 2: Runtime (실행 스테이지)
# ============================================================
# JRE만 포함된 경량 이미지 사용 (JDK의 컴파일러 등 불필요)
FROM eclipse-temurin:21-jre-alpine AS runtime

# 보안: root가 아닌 전용 사용자로 실행
# 컨테이너가 해킹당해도 root 권한을 얻지 못하게 방어
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# builder 스테이지에서 생성된 JAR 파일만 복사
# 소스코드, Gradle, JDK는 이 이미지에 포함되지 않음
COPY --from=builder /app/app/build/libs/app-*.jar app.jar

# 전용 사용자로 전환
USER appuser

# 컨테이너가 사용하는 포트 문서화
# 실제 포트 오픈은 K8s Service에서 처리
EXPOSE 9000

# JVM 옵션:
# -XX:+UseZGC: Z Garbage Collector (저지연 GC, Virtual Thread와 궁합)
# -XX:MaxRAMPercentage=75.0: 컨테이너 메모리 제한의 75%를 힙에 할당
# -Djava.security.egd: 빠른 난수 생성 (컨테이너 환경에서 중요)
# spring.profiles.active: 활성 프로파일 (환경변수로 오버라이드 가능)
ENTRYPOINT ["java", \
  "-XX:+UseZGC", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

### 3.3 각 명령어 상세 설명

| 명령어 | 역할 |
|--------|------|
| `FROM ... AS builder` | 빌드 스테이지 시작. 이름을 붙여 나중에 참조 가능 |
| `WORKDIR /app` | 이후 명령어의 작업 디렉토리 설정 (cd와 비슷) |
| `COPY` | 호스트의 파일을 이미지 내부로 복사 |
| `RUN` | 이미지 빌드 시 실행할 명령어 (레이어 생성) |
| `FROM ... AS runtime` | 새로운 스테이지 시작 (이전 스테이지는 버려짐) |
| `COPY --from=builder` | builder 스테이지에서 파일을 가져옴 |
| `USER appuser` | 이후 명령어를 지정 사용자로 실행 |
| `EXPOSE` | 컨테이너가 사용하는 포트 문서화 (실제 오픈 아님) |
| `ENTRYPOINT` | 컨테이너 시작 시 실행할 명령어 |

### 3.4 .dockerignore 파일

빌드 컨텍스트에서 불필요한 파일을 제외한다.

```
# bitcoin/.dockerignore
.git
.github
.gradle
.idea
*.md
docker/docker-compose.yml
k8s/
**/build/
**/out/
**/.gradle/
```

> **빌드 컨텍스트란?**
> `docker build` 시 Docker 데몬에게 전송하는 파일들의 집합이다.
> `.dockerignore`에 지정된 파일은 전송하지 않아 빌드 속도가 빨라진다.

## 4. Docker 레이어 캐싱 전략

Dockerfile의 각 명령어는 **레이어**를 생성한다. Docker는 변경되지 않은 레이어를 캐시한다.

```
레이어 1: FROM eclipse-temurin:21-jdk-alpine     ← 거의 변경 안 됨 (캐시 히트)
레이어 2: COPY gradlew, gradle/, *.kts           ← 빌드 설정 변경 시만 재실행
레이어 3: RUN ./gradlew dependencies              ← 의존성 변경 시만 재실행 (오래 걸림)
레이어 4: COPY */src                              ← 코드 변경 시 재실행 (자주 변경)
레이어 5: RUN ./gradlew :app:bootJar              ← 코드 변경 시 재실행
```

**핵심 원칙**: 자주 변경되는 것은 아래에, 변경이 적은 것은 위에 배치한다.
→ 코드만 수정한 경우 레이어 1~3은 캐시를 사용하므로 빌드가 빠르다.

## 5. JVM 옵션 설명

### 5.1 ZGC (Z Garbage Collector)

```
-XX:+UseZGC
```

| GC 종류 | 특징 | 적합한 상황 |
|---------|------|-----------|
| G1GC | 기본 GC, 범용적 | 일반적인 서버 앱 |
| ZGC | 초저지연 (<1ms 정지), 대용량 힙 | Virtual Thread, 실시간성 |
| Shenandoah | ZGC와 유사한 저지연 | Red Hat 환경 |

이 프로젝트는 **Virtual Thread + 실시간 시세 수신**을 사용하므로 ZGC가 적합하다.

### 5.2 메모리 설정

```
-XX:MaxRAMPercentage=75.0
```

K8s에서 컨테이너 메모리 제한을 설정하면, JVM이 이를 인식하여 힙 크기를 자동 조절한다.
- 컨테이너 메모리 제한: 1GB → JVM 힙: ~750MB
- 나머지 25%: 메타스페이스, 스레드 스택, 네이티브 메모리 등에 사용

> **주의**: `-Xmx`(고정 힙 크기) 대신 `MaxRAMPercentage`를 사용하는 이유는
> K8s에서 메모리 제한을 변경할 때 Dockerfile을 수정하지 않아도 되기 때문이다.

## 6. 로컬에서 이미지 빌드/테스트

```bash
# 프로젝트 루트에서 실행
# -f: Dockerfile 위치 지정
# -t: 이미지 이름:태그
# . : 빌드 컨텍스트 (현재 디렉토리)
docker build -f docker/Dockerfile -t bitcoin-trader:local .

# 빌드된 이미지 확인
docker images | grep bitcoin-trader

# 로컬에서 실행 테스트 (DB가 필요하므로 docker-compose와 함께)
docker run --rm \
  --network host \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  -e DB_NAME=bitcoin_trader \
  -e DB_USERNAME=bitcoin \
  -e DB_PASSWORD=bitcoin \
  -e SPRING_PROFILES_ACTIVE=prod \
  bitcoin-trader:local
```

## 7. 이미지 크기 최적화 팁

| 기법 | 절감 효과 |
|------|----------|
| Alpine 기반 이미지 | ~200MB 절감 (vs Ubuntu 기반) |
| 멀티 스테이지 빌드 | ~500MB 절감 (빌드 도구 제거) |
| .dockerignore | 빌드 시간 단축 |
| JRE 사용 (JDK 대신) | ~100MB 절감 |

최종 이미지 크기 예상: **약 200~250MB**

## 8. 다음 단계

Dockerfile 작성이 완료되었으면, 다음 단계로 진행한다:

→ [[06-인프라/K8s 매니페스트 가이드|K8s 매니페스트 가이드]] — Kubernetes에 배포하기 위한 매니페스트 작성
