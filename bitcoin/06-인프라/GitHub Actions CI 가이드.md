# GitHub Actions CI 가이드

> 최종 수정: 2026-03-01
> 목적: 코드 push → 빌드 → 테스트 → Docker 이미지 → ghcr.io 푸시 자동화

---

## 1. GitHub Actions란?

GitHub에서 제공하는 **CI/CD 자동화 플랫폼**이다. 코드가 push되면 GitHub 서버에서 지정한 작업을 자동으로 실행한다.

```
개발자 git push → GitHub Actions 트리거 → GitHub 서버에서 실행
                                          ├── Gradle 빌드
                                          ├── 테스트 실행
                                          ├── Docker 이미지 빌드
                                          └── ghcr.io에 이미지 push
```

**핵심 포인트:**
- GitHub 서버에서 실행 → 미니 PC에 부하 없음
- 무료 사용량: Public repo 무제한, Private repo 월 2,000분
- 워크플로우 파일(YAML)로 정의

## 2. GitHub Actions 핵심 개념

```yaml
# .github/workflows/ci.yml

name: CI                          # 워크플로우 이름

on:                               # 트리거 조건
  push:
    branches: [main]

jobs:                             # 작업 목록
  build:                          # 작업 이름
    runs-on: ubuntu-latest        # 실행 환경
    steps:                        # 단계 목록
      - name: Checkout            # 단계 이름
        uses: actions/checkout@v4 # 사전 정의된 액션 사용
```

| 개념 | 설명 |
|------|------|
| **Workflow** | 자동화 프로세스 전체. `.github/workflows/*.yml`에 정의 |
| **Trigger (on)** | 워크플로우를 실행하는 이벤트 (push, PR, schedule 등) |
| **Job** | 워크플로우 내의 독립 작업 단위. 별도 VM에서 실행 |
| **Step** | Job 내의 개별 명령어 또는 액션 |
| **Action** | 재사용 가능한 미리 만들어진 단계 (마켓플레이스) |
| **Runner** | 워크플로우를 실행하는 서버 (GitHub 호스팅 또는 Self-hosted) |

## 3. CI 워크플로우 작성

### 3.1 전체 흐름

```
┌─────────────────────────────────────────────────────────────────┐
│                        CI 워크플로우                               │
│                                                                 │
│  ① Checkout    ② JDK 설치   ③ Gradle 빌드   ④ 테스트            │
│  소스 체크아웃   Java 21       컴파일          JUnit 실행          │
│                                                                 │
│  ⑤ Docker 빌드             ⑥ ghcr.io 푸시                       │
│  멀티스테이지 이미지 생성     컨테이너 레지스트리에 업로드           │
│                                                                 │
│  ⑦ K8s 매니페스트 이미지 태그 업데이트                              │
│  → ArgoCD가 변경 감지 → 자동 배포                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 워크플로우 YAML

```yaml
# .github/workflows/ci.yml

name: CI

# === 트리거 조건 ===
on:
  push:
    branches: [main]           # main 브랜치에 push 시 실행
    paths-ignore:              # 이 경로 변경은 무시 (불필요한 빌드 방지)
      - '*.md'
      - 'docs/**'
      - 'k8s/**'              # K8s 매니페스트 변경은 ArgoCD가 처리
  pull_request:
    branches: [main]           # main으로 PR 생성 시 실행

# === 환경변수 ===
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}   # owner/repo-name 형태

# === 작업 ===
jobs:
  build-and-push:
    name: Build, Test & Push
    runs-on: ubuntu-latest

    # ghcr.io에 이미지 push하기 위한 권한
    permissions:
      contents: write          # 코드 체크아웃 + 매니페스트 업데이트
      packages: write          # ghcr.io에 이미지 push

    steps:
      # --- 1. 소스코드 체크아웃 ---
      - name: Checkout repository
        uses: actions/checkout@v4

      # --- 2. JDK 21 설치 ---
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'       # Eclipse Temurin (구 AdoptOpenJDK)

      # --- 3. Gradle 캐시 설정 ---
      # 의존성을 캐시하여 빌드 시간 단축
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4

      # --- 4. 빌드 및 테스트 ---
      - name: Build and Test
        run: ./gradlew build --no-daemon
        # build = compileJava + test + jar
        # 테스트 실패 시 워크플로우가 여기서 중단됨

      # --- 5. Docker 빌드 환경 설정 (QEMU + Buildx) ---
      # Buildx: Docker의 확장 빌드 도구 (멀티 아키텍처 지원)
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # --- 6. ghcr.io 로그인 ---
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}            # GitHub 사용자명
          password: ${{ secrets.GITHUB_TOKEN }}     # 자동 제공되는 토큰

      # --- 7. 이미지 태그 생성 ---
      # 짧은 커밋 SHA를 태그로 사용 (예: ghcr.io/han/bitcoin:a7ee694)
      - name: Generate image tag
        id: tag
        run: echo "sha_short=$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT

      # --- 8. Docker 이미지 빌드 및 푸시 ---
      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .                               # 빌드 컨텍스트 (프로젝트 루트)
          file: ./docker/Dockerfile                # Dockerfile 위치
          push: ${{ github.event_name == 'push' }} # push 이벤트일 때만 실제 push
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.tag.outputs.sha_short }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha                     # GitHub Actions 캐시 사용
          cache-to: type=gha,mode=max

      # --- 9. K8s 매니페스트 이미지 태그 업데이트 ---
      # ArgoCD가 이 변경을 감지하여 자동 배포
      - name: Update K8s manifest image tag
        if: github.event_name == 'push'            # push 이벤트일 때만
        run: |
          cd k8s/overlays/prod
          # kustomization.yaml의 newTag를 새 커밋 SHA로 변경
          sed -i "s|newTag:.*|newTag: ${{ steps.tag.outputs.sha_short }}|" kustomization.yaml

      # --- 10. 변경된 매니페스트를 Git에 커밋 ---
      - name: Commit manifest update
        if: github.event_name == 'push'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add k8s/overlays/prod/kustomization.yaml
          git diff --staged --quiet || git commit -m "ci: update image tag to ${{ steps.tag.outputs.sha_short }}"
          git push
```

## 4. 주요 개념 상세 설명

### 4.1 `secrets.GITHUB_TOKEN`

GitHub Actions가 **자동으로 제공**하는 토큰이다. 별도 설정 불필요.

| 용도 | 권한 |
|------|------|
| ghcr.io 이미지 push | `packages: write` |
| Git 커밋/푸시 | `contents: write` |
| PR 코멘트 | `pull-requests: write` |

### 4.2 이미지 태그 전략

```
ghcr.io/han/bitcoin-auto-trader:a7ee694    ← 커밋 SHA (고유, 불변)
ghcr.io/han/bitcoin-auto-trader:latest     ← 최신 버전 (편의용)
```

커밋 SHA를 태그로 사용하는 이유:
1. **고유성**: 같은 태그가 다른 이미지를 가리키는 일이 없음
2. **추적성**: 이미지 → 커밋 → 코드 변경 추적 가능
3. **롤백 용이**: 이전 SHA 태그로 쉽게 롤백

### 4.3 빌드 캐시

```yaml
cache-from: type=gha    # GitHub Actions 캐시에서 레이어 로드
cache-to: type=gha      # 빌드 레이어를 GitHub Actions 캐시에 저장
```

첫 빌드: ~5분, 캐시 적중 시: ~1~2분

### 4.4 `paths-ignore`

```yaml
paths-ignore:
  - '*.md'              # README 변경에 빌드 불필요
  - 'docs/**'           # 문서 변경에 빌드 불필요
  - 'k8s/**'            # K8s 매니페스트는 ArgoCD가 처리
```

불필요한 CI 실행을 방지하여 **GitHub Actions 사용량을 절약**한다.

## 5. GitHub Repository 설정

### 5.1 ghcr.io 패키지 가시성 설정

처음 이미지를 push한 후, GitHub에서 패키지 가시성을 설정해야 한다.

1. GitHub Repository → Packages 탭
2. 패키지 클릭 → Package settings
3. Visibility: Private 또는 Public 선택

### 5.2 K3s에서 Private 이미지 Pull 설정

Private 이미지를 K3s에서 pull하려면 인증이 필요하다.

```bash
# GitHub Personal Access Token (PAT) 생성
# GitHub → Settings → Developer settings → Personal access tokens
# 권한: read:packages

# K3s에 레지스트리 인증 설정
sudo mkdir -p /etc/rancher/k3s
sudo cat > /etc/rancher/k3s/registries.yaml << 'EOF'
mirrors:
  ghcr.io:
    endpoint:
      - "https://ghcr.io"
configs:
  ghcr.io:
    auth:
      username: GITHUB_USERNAME
      password: GITHUB_PAT_TOKEN
EOF

# K3s 재시작
sudo systemctl restart k3s
```

또는 K8s Secret으로 관리:

```bash
# imagePullSecret 생성
kubectl create secret docker-registry ghcr-secret \
  --namespace bitcoin \
  --docker-server=ghcr.io \
  --docker-username=GITHUB_USERNAME \
  --docker-password=GITHUB_PAT_TOKEN
```

그리고 Deployment에 추가:

```yaml
spec:
  template:
    spec:
      imagePullSecrets:
        - name: ghcr-secret
```

## 6. 워크플로우 실행 확인

```
GitHub Repository → Actions 탭
├── 워크플로우 실행 목록
│   ├── ✅ ci: update image tag to a7ee694 (성공)
│   ├── ❌ Build and Test (실패 - 테스트 에러)
│   └── 🔄 Build, Test & Push (실행 중)
└── 각 실행 클릭 → 단계별 로그 확인 가능
```

### 실패 시 확인 포인트

| 실패 단계 | 원인 | 해결 |
|-----------|------|------|
| Build and Test | 컴파일/테스트 오류 | 로컬에서 `./gradlew build` 실행하여 확인 |
| Login to ghcr.io | 토큰 권한 부족 | permissions에 `packages: write` 확인 |
| Build and push Docker image | Dockerfile 오류 | 로컬에서 `docker build` 테스트 |
| Commit manifest update | push 권한 없음 | permissions에 `contents: write` 확인 |

## 7. 로컬에서 워크플로우 테스트 (act)

GitHub에 push하지 않고 로컬에서 워크플로우를 테스트할 수 있다.

```bash
# act 설치 (Mac)
brew install act

# 워크플로우 실행
act push

# 특정 job만 실행
act push -j build-and-push
```

> 참고: Docker가 로컬에 설치되어 있어야 한다.

## 8. 다음 단계

GitHub Actions CI가 완료되었으면, 다음 단계로 진행한다:

→ [[06-인프라/ArgoCD 설치 및 GitOps 가이드|ArgoCD 설치 및 GitOps 가이드]] — CD 파이프라인으로 K3s에 자동 배포
