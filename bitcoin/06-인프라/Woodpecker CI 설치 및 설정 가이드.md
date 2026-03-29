# Woodpecker CI 설치 및 설정 가이드

> 최종 수정: 2026-03-08
> 목적: GitHub push (dev) → Woodpecker CI 파이프라인 (K8s 백엔드) → 매니페스트 레포 태그 업데이트
> 대체: GitHub Actions CI 가이드, ArgoCD 설치 및 GitOps 가이드, 로컬 CI-CD 파이프라인 가이드

---

## 1. Woodpecker CI란?

Go로 작성된 **경량 CI/CD 엔진**이다. Drone CI(0.8)의 오픈소스 포크.

```
Woodpecker CI가 하는 일:

GitHub push (dev 브랜치)
    ↓ Webhook
Woodpecker Server (K3s Pod, 웹 UI + Webhook 수신)
    ↓ 파이프라인 트리거
Woodpecker Agent (K3s Pod, K8s 백엔드 → Step을 Pod으로 실행)
    ├── Step 1 Pod: kaniko → 멀티스테이지 Dockerfile로 빌드+패키징 → Local Registry push
    └── Step 2 Pod: alpine/git → 매니페스트 레포 태그 업데이트 (SSH)
    ↓
ArgoCD가 매니페스트 레포 변경 감지 → K8s 배포 (Rolling Update)
    ↓
웹 UI에서 빌드 과정 모니터링 + 성공/실패 히스토리
```

**핵심 특징:**
- **Server**: 웹 UI, Webhook 수신, 파이프라인 스케줄링 (K3s Pod, Helm 설치)
- **Agent**: K8s 백엔드로 파이프라인 Step마다 Pod 생성 (K3s Pod, Helm 설치)
- **호스트에 K3s만 설치** — Docker, JDK 등 추가 도구 불필요
- 별도 webhook listener 불필요 — **Woodpecker가 직접 Webhook 수신**
- Woodpecker는 **빌드까지만 담당** — 매니페스트 레포에 태그를 커밋하면 ArgoCD가 실제 배포를 수행

## 2. 사전 준비

미니 PC에 **K3s만** 설치되어 있으면 된다:

```bash
# 확인
k3s --version         # K3s
kubectl version       # kubectl (K3s에 포함)
helm version          # Helm (별도 설치 필요)
```

> Docker, JDK는 **호스트에 설치 불필요**. 빌드 도구가 모두 K8s Pod으로 실행된다.

### 2.1 Helm 설치 (아직 없다면)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## 3. GitHub OAuth App 생성

Woodpecker가 GitHub에 접근하려면 OAuth App이 필요하다.

### 3.1 OAuth App 등록

GitHub → Settings → Developer settings → OAuth Apps → New OAuth App

| 항목 | 값 |
|------|---|
| Application name | `Woodpecker CI` |
| Homepage URL | `https://yourname.synology.me/woodpecker` |
| Authorization callback URL | `https://yourname.synology.me/woodpecker/authorize` |

> 등록 후 **Client ID**와 **Client Secret**을 메모한다.

## 4. CI 인프라 배포 (Registry)

Woodpecker 설치 전에 Local Registry를 먼저 배포한다.

### 4.1 매니페스트 적용

```bash
# woodpecker 네임스페이스 + Registry 한 번에 배포
kubectl apply -f k8s/ci/
```

### 4.2 배포 확인

```bash
# Registry Pod 정상 기동 확인
kubectl get pods -n woodpecker

# Registry Service 확인 (NodePort 31500)
kubectl get svc -n woodpecker
```

### 4.3 배포된 리소스 설명

| 파일 | 리소스 | 역할 |
|------|--------|------|
| `namespace.yaml` | Namespace `woodpecker` | CI 관련 리소스 격리 |
| `registry.yaml` | Deployment + PVC + NodePort Service | 컨테이너 이미지 저장소 (registry:2) |

## 5. K3s registries.yaml 설정

K3s containerd가 Local Registry에서 이미지를 pull할 수 있도록 미러 설정이 필요하다.

```bash
# /etc/rancher/k3s/registries.yaml 생성
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/registries.yaml << 'EOF'
mirrors:
  "registry:5000":
    endpoint:
      - "http://localhost:31500"
EOF

# K3s 재시작 (설정 반영)
sudo systemctl restart k3s
```

**설명:**
- kaniko가 `registry:5000`으로 push (ClusterIP, Pod 내부 통신)
- K3s containerd가 `registry:5000` 이미지를 pull할 때 → `localhost:31500`으로 리다이렉트 (NodePort)
- 이렇게 하면 Deployment의 `image: registry:5000/bitcoin-trader:TAG`가 정상 동작

## 6. Woodpecker CI 설치 (Helm)

### 6.1 Helm 레포 추가

```bash
helm repo add woodpecker https://woodpecker-ci.org/
helm repo update
```

### 6.2 values 파일 수정

`k8s/ci/woodpecker-values.yaml`에서 플레이스홀더를 실제 값으로 변경:

```yaml
# 변경할 항목
WOODPECKER_GITHUB_CLIENT: "<실제_GITHUB_CLIENT_ID>"
WOODPECKER_GITHUB_SECRET: "<실제_GITHUB_CLIENT_SECRET>"
WOODPECKER_HOST: "https://실제도메인/woodpecker"
WOODPECKER_ADMIN: "실제-github-username"
```

### 6.3 설치

```bash
helm install woodpecker woodpecker/woodpecker \
  -n woodpecker \
  -f k8s/ci/woodpecker-values.yaml
```

### 6.4 설치 확인

```bash
kubectl get pods -n woodpecker
# NAME                                    READY   STATUS
# registry-xxx                            1/1     Running
# woodpecker-server-xxx                   1/1     Running
# woodpecker-agent-xxx                    1/1     Running
```

### 6.5 Woodpecker Server 외부 접근

Woodpecker Server는 ClusterIP Service이므로, Synology 리버스 프록시로 외부 접근을 설정한다.
NodePort로 노출 후 Synology에서 프록시:

```bash
# NodePort로 변경 (또는 Ingress 사용)
kubectl patch svc woodpecker-server -n woodpecker -p '{"spec":{"type":"NodePort"}}'
kubectl get svc woodpecker-server -n woodpecker
# 할당된 NodePort 확인 후 Synology 리버스 프록시에 등록
```

## 7. Synology 리버스 프록시 설정

Synology DSM → 제어판 → 로그인 포털 → 고급 → 역방향 프록시

### 7.1 Woodpecker UI + Webhook 규칙 추가

| 항목 | 값 |
|------|---|
| 설명 | Woodpecker CI |
| 소스 프로토콜 | HTTPS |
| 소스 호스트명 | `yourname.synology.me` |
| 소스 포트 | 443 |
| 경로 | `/woodpecker` |
| 대상 프로토콜 | HTTP |
| 대상 호스트명 | `미니-PC-내부-IP` (예: 192.168.1.100) |
| 대상 포트 | Woodpecker NodePort (kubectl get svc에서 확인) |

> GitHub Webhook URL이 `https://yourname.synology.me/woodpecker/hook` 형태가 된다.

### 7.2 WebSocket 지원 활성화

Woodpecker UI는 실시간 로그 스트리밍에 WebSocket을 사용한다.
역방향 프록시 설정 → 사용자 지정 헤더:

| 헤더 이름 | 값 |
|----------|---|
| Upgrade | `$http_upgrade` |
| Connection | `$connection_upgrade` |

## 8. 파이프라인 설정 (.woodpecker.yml)

### 8.1 K8s 백엔드 파이프라인

```yaml
# .woodpecker.yml

when:
  - branch: dev
    event: push

clone:
  - name: clone
    image: woodpeckerci/plugin-git
    settings:
      depth: 1

steps:
  # === Step 1: 이미지 빌드 (멀티스테이지 Dockerfile로 빌드+패키징) ===
  - name: build-image
    image: woodpeckerci/plugin-kaniko
    settings:
      registry: registry:5000
      repo: bitcoin-trader
      tags: "${CI_COMMIT_SHA}"
      dockerfile: docker/Dockerfile
      insecure: true

  # === Step 2: 매니페스트 레포 이미지 태그 업데이트 (SSH) ===
  - name: update-manifest
    image: alpine/git
    environment:
      SSH_KEY:
        from_secret: manifest_repo_ssh_key
    commands:
      - IMAGE_TAG="${CI_COMMIT_SHA}"
      - mkdir -p ~/.ssh
      - echo "$SSH_KEY" > ~/.ssh/id_ed25519
      - chmod 600 ~/.ssh/id_ed25519
      - ssh-keyscan -t ed25519 github.com >> ~/.ssh/known_hosts
      - git clone git@github.com:cozy-han/liooos-k8s-manifests.git
      - cd liooos-k8s-manifests/bitcoin/overlays/prod
      - 'sed -i "s/newTag: .*/newTag: $IMAGE_TAG/" kustomization.yaml'
      - git config user.email "woodpecker@ci.local"
      - git config user.name "Woodpecker CI"
      - git add .
      - 'git diff --cached --quiet && echo "No changes" && exit 0'
      - 'git commit -m "ci: update image tag to $IMAGE_TAG"'
      - git push
```

### 8.2 파이프라인 설명

| Step | Pod 이미지 | 역할 | 실패 시 |
|------|-----------|------|--------|
| build-image | woodpeckerci/plugin-kaniko | 멀티스테이지 Dockerfile로 Gradle 빌드 + 이미지 패키징 → Registry push | 파이프라인 중단 |
| update-manifest | alpine/git | SSH로 매니페스트 레포의 kustomization.yaml 태그 업데이트 → ArgoCD가 배포 | 파이프라인 중단, 수동 태그 업데이트 필요 |

> **왜 2단계인가?**
> 이전에는 Gradle 빌드(Step 1) + kaniko(Step 2, Dockerfile.ci) + git push(Step 3)로 3단계였다.
> 그러나 kaniko가 멀티스테이지 `docker/Dockerfile`을 직접 빌드하면 Gradle 빌드와 이미지 패키징을 한 번에 처리할 수 있어 **2단계로 단순화**했다.
> `docker/Dockerfile.ci`(슬림 버전)는 레거시로 남아있다.

### 8.3 멀티스테이지 Dockerfile (CI에서 사용)

kaniko가 `docker/Dockerfile`(멀티스테이지)을 직접 빌드한다. Gradle 빌드와 이미지 패키징이 하나의 Step에서 수행된다.

```dockerfile
# docker/Dockerfile (멀티스테이지 — CI + 로컬 개발 공용)
# Stage 1: Gradle 빌드
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY gradlew gradle/ build.gradle.kts settings.gradle.kts ./
COPY */build.gradle.kts ./         # 각 모듈 빌드 파일
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon
COPY */src ./                       # 소스 코드
RUN ./gradlew :app:bootJar -x test --no-daemon && rm -rf /tmp/hsperfdata_*

# Stage 2: JRE 런타임
FROM eclipse-temurin:21-jre-alpine AS runtime
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder /app/app/build/libs/app-*.jar app.jar
USER appuser
EXPOSE 9000
ENTRYPOINT ["java", "-XX:+UseZGC", "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", "-jar", "app.jar"]
```

> `rm -rf /tmp/hsperfdata_*`는 kaniko 스냅샷 시 JVM 임시 파일 오류를 방지한다.

### 8.4 `when` 조건 상세

```yaml
when:
  branch: dev       # dev 브랜치만 트리거
  event: push       # push 이벤트만 (PR 이벤트 제외)
```

- `main` 브랜치 push → 파이프라인 실행 안 함
- `dev` 브랜치 push → 파이프라인 실행
- PR 생성/업데이트 → 파이프라인 실행 안 함 (필요시 `event: [push, pull_request]`로 변경)

## 9. Woodpecker 레포지토리 설정

### 9.1 레포 활성화

1. Woodpecker 웹 UI 접속 → GitHub 로그인
2. 레포 목록에서 `bitcoin` 레포 활성화 (Toggle ON)
3. Settings → General:
   - Pipeline path: `.woodpecker.yml`
   - Trusted: **ON** (볼륨 마운트 허용)

### 9.2 Secrets 설정

Woodpecker UI → Repository Settings → Secrets

필요한 시크릿을 여기서 등록한다:

| Name | Value | 용도 |
|------|-------|------|
| `manifest_repo_ssh_key` | SSH 개인키 (ed25519) | 매니페스트 레포 push (SSH) |
| `discord_webhook_url` | `https://discord.com/api/webhooks/...` | 배포 알림 |

> `manifest_repo_ssh_key`는 매니페스트 레포에 push 권한이 있는 SSH 개인키(ed25519)이다.
> GitHub → Repository → Settings → Deploy keys에 공개키를 등록하고, Write access를 활성화한다.
> Upbit API 키 등은 Sealed Secrets로 K8s에서 관리하므로 Woodpecker에는 등록 불필요.

## 10. GitHub Webhook 확인

Woodpecker가 레포를 활성화하면 **GitHub Webhook을 자동으로 등록**한다.

확인: GitHub → Repository → Settings → Webhooks

| 항목 | 값 |
|------|---|
| Payload URL | `https://yourname.synology.me/woodpecker/hook` (자동 설정) |
| Content type | `application/json` |
| Events | Push, Pull Request (자동 설정) |

> 자동 등록된 Webhook이 올바른지 확인한다.
> Synology 리버스 프록시 경로와 일치해야 한다.

## 11. 수동 배포 및 롤백

### 11.1 Woodpecker UI에서 수동 실행

웹 UI → Repository → Pipelines → "Run pipeline" 버튼 → dev 브랜치 선택 → 실행

### 11.2 롤백

ArgoCD + 매니페스트 레포 기반 롤백:

```bash
# 방법 1: 매니페스트 레포에서 이전 커밋으로 revert
cd liooos-k8s-manifests
git log --oneline  # 되돌릴 커밋 확인
git revert HEAD    # 직전 태그 업데이트 커밋 되돌리기
git push           # ArgoCD가 변경 감지 → 이전 이미지로 재배포

# 방법 2: ArgoCD UI/CLI에서 직접 롤백
argocd app history bitcoin
argocd app rollback bitcoin <REVISION>

# 방법 3: kubectl 직접 롤백 (긴급 시)
# 주의: ArgoCD가 매니페스트 레포 상태로 다시 덮어쓸 수 있음
kubectl rollout undo deployment/bitcoin-trader -n bitcoin
```

> ArgoCD를 사용하므로, `kubectl rollout undo`는 긴급 상황에서만 사용한다.
> ArgoCD가 주기적으로 매니페스트 레포와 동기화하므로, 영구적인 롤백은 매니페스트 레포의 커밋을 revert해야 한다.

### 11.3 SSH로 수동 배포

Woodpecker를 거치지 않고 직접 배포해야 할 때:

```bash
ssh minipc
cd /home/han/bitcoin
git pull origin dev

# Gradle 빌드
./gradlew build --no-daemon

# kaniko 대신 호스트에서 이미지 빌드하려면 nerdctl 사용 (K3s containerd 도구)
# 또는 Registry에 직접 push
sudo k3s ctr images import <이미지>.tar

# 또는 kubectl로 직접 배포 (Registry에 이미지가 있는 경우)
kubectl set image deployment/bitcoin-trader bitcoin-trader=registry:5000/bitcoin-trader:manual -n bitcoin
kubectl rollout status deployment/bitcoin-trader -n bitcoin
```

> 수동 배포 시에는 호스트에 빌드 도구가 없으므로 (K3s만 설치),
> Woodpecker UI에서 수동 파이프라인 실행을 사용하는 것을 권장한다.

## 12. 모니터링 및 디버깅

### 12.1 Woodpecker UI

- 파이프라인 실행 목록 (성공/실패/진행 중)
- 각 Step 클릭 → 실시간 로그 스트리밍
- 빌드 소요 시간 추적

### 12.2 K8s 상태 확인

```bash
# Pod 상태
kubectl get pods -n bitcoin -w

# 배포 이벤트
kubectl describe deployment bitcoin-trader -n bitcoin

# Pod 로그
kubectl logs -f deployment/bitcoin-trader -n bitcoin
```

### 12.3 Woodpecker / CI 인프라 상태

```bash
# Woodpecker Server 로그
kubectl logs -f deployment/woodpecker-server -n woodpecker

# Woodpecker Agent 로그
kubectl logs -f deployment/woodpecker-agent -n woodpecker

# Registry 상태
kubectl logs -f deployment/registry -n woodpecker

# 파이프라인 Step Pod 확인 (실행 중일 때)
kubectl get pods -n woodpecker -w
```

### 12.4 Registry 이미지 확인

```bash
# Registry에 저장된 이미지 목록
curl http://localhost:31500/v2/_catalog

# 특정 이미지의 태그 목록
curl http://localhost:31500/v2/bitcoin-trader/tags/list
```

## 13. 보안 체크리스트

- [ ] GitHub OAuth Client Secret 보호 (Helm values에서 Secret으로 관리)
- [ ] Woodpecker Agent Secret 충분히 긴 랜덤 값 사용
- [ ] Synology 리버스 프록시 HTTPS 활성화
- [ ] Woodpecker Admin 사용자 본인 GitHub 계정만 등록
- [ ] Sealed Secrets로 K8s 내 API 키 암호화
- [ ] Woodpecker Secrets에 민감 정보 등록 시 "이미지에 노출" 옵션 비활성화
- [ ] `manifest_repo_ssh_key`는 매니페스트 레포 전용 Deploy Key 사용 (Write access)

## 14. 구축 순서 요약

```
1. GitHub OAuth App 생성 (Client ID + Secret)
2. CI 인프라 배포: kubectl apply -f k8s/ci/
3. K3s registries.yaml 설정 + K3s 재시작
4. Helm으로 Woodpecker 설치 (woodpecker-values.yaml)
5. Woodpecker Server NodePort 노출 + Synology 리버스 프록시 설정 (WebSocket 포함)
6. Woodpecker 웹 UI 접속 → GitHub 로그인 → 레포 활성화
7. Woodpecker Secrets에 manifest_repo_ssh_key 등록 (SSH 개인키)
8. .woodpecker.yml + docker/Dockerfile 확인
9. 테스트: dev 브랜치 push → 파이프라인 실행 → Step Pod 생성 → 매니페스트 레포 태그 업데이트 확인
```

## 15. 추후 확장

| 확장 | 방법 |
|------|------|
| K8s 시각화 추가 | Kubernetes Dashboard 또는 Portainer 설치 |
| 모니터링 | Prometheus + Grafana 스택 추가 |
| 도메인 구매 시 | Synology DDNS → 커스텀 도메인으로 변경 |
| 노드 추가 시 | K3s 워커 노드 추가 (Registry 변경 불필요) |
| PR 빌드 | `.woodpecker.yml`에 `event: [push, pull_request]` 추가 |
| 알림 | Woodpecker의 Discord 플러그인으로 빌드/배포 결과 알림 |
| Registry 정리 | CronJob으로 오래된 이미지 태그 자동 삭제 |
