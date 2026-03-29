# ArgoCD 설치 및 GitOps 가이드

> 최종 수정: 2026-03-09
> 목적: 매니페스트 레포(liooos-k8s-manifests) 감시 → K8s 자동 배포 (GitOps CD)
> 역할 분담: Woodpecker CI = 빌드 + 매니페스트 태그 업데이트, ArgoCD = 배포

---

## 1. GitOps란?

**Git을 인프라의 진실의 원천(Single Source of Truth)으로 사용**하는 운영 방식이다.

```
전통적 배포:
개발자 → SSH 접속 → 서버에서 직접 명령어 실행
문제: 누가, 언제, 무엇을 변경했는지 추적 어려움

GitOps 배포:
개발자 → 코드 push → Woodpecker CI(빌드) → 매니페스트 레포에 태그 커밋
→ ArgoCD가 매니페스트 레포 감시 → 변경 감지 → 클러스터에 자동 적용
장점: 변경 이력 = Git 히스토리 (누가, 언제, 무엇을 100% 추적)
```

**GitOps 원칙:**
1. 선언적: 원하는 상태를 YAML로 선언
2. 버전 관리: 모든 변경은 Git 커밋
3. 자동 적용: Git 변경 → 자동으로 클러스터에 반영
4. 자가 치유: 클러스터 상태가 Git과 다르면 자동으로 보정

## 2. ArgoCD란?

ArgoCD는 K8s를 위한 **GitOps CD 도구**이다.

```
┌──────────────────────┐                     ┌─────────────────────────┐
│  매니페스트 레포       │                     │  K3s 클러스터            │
│  (liooos-k8s-manifests)│    3분마다 폴링     │                         │
│  bitcoin/             │ ←─────────────── │  ArgoCD                 │
│  ├── base/            │                     │  ├── Application CRD    │
│  ├── overlays/prod/   │    변경 감지         │  ├── 동기화 엔진         │
│  └── (SealedSecret)   │ ──────────────→ │  └── 웹 UI              │
└──────────────────────┘                     └─────────────────────────┘
         ↑
         │ newTag 업데이트 커밋
         │
┌──────────────────────┐
│  Woodpecker CI       │
│  (소스 레포 빌드 후)   │
└──────────────────────┘
```

**주요 기능:**
- Git 저장소(매니페스트 레포) 감시 → 변경 시 자동 배포
- 웹 UI: 배포 상태 시각화 (Pod, Service 등의 관계도)
- 자동 동기화: Git = 원하는 상태, 클러스터 = 실제 상태 → 차이 자동 보정
- 롤백: 이전 Git 커밋으로 쉽게 롤백

## 3. 왜 매니페스트를 별도 레포로 분리하는가?

### 3.1 기존 방식의 문제점

소스 레포와 매니페스트가 같은 레포에 있을 때:

```
소스 레포 (bitcoin-auto-trader)
├── src/                   # 소스 코드
├── k8s/                   # K8s 매니페스트
│   └── overlays/prod/
│       └── kustomization.yaml   ← CI가 newTag를 여기에 업데이트
└── ...

문제:
① CI가 이미지 태그를 업데이트하려면 소스 레포에 커밋해야 함
② 소스 레포 Git 히스토리에 "ci: update image tag to abc1234" 같은 커밋이 쌓임
③ 코드 변경 이력과 배포 이력이 뒤섞여 추적이 어려움
④ CI 커밋이 다시 CI를 트리거하는 무한 루프 위험 (분기 조건 필요)
```

### 3.2 매니페스트 레포 분리 방식

```
소스 레포 (bitcoin-auto-trader)        매니페스트 레포 (liooos-k8s-manifests)
├── src/                               ├── bitcoin/
├── docker/                            │   ├── base/
│   └── Dockerfile.ci                  │   │   ├── app-deployment.yaml
└── .woodpecker.yml                    │   │   ├── app-service.yaml
     (빌드 → 이미지 push →              │   │   ├── app-sealed-secret.yaml
      매니페스트 레포 태그 업데이트)       │   │   └── ...
                                       │   └── overlays/prod/
                                       │       └── kustomization.yaml  ← newTag 여기에 업데이트
                                       └── argocd/
                                           └── bitcoin-app.yaml

장점:
① 소스 레포 Git 히스토리가 깨끗하게 유지됨 (순수 코드 변경만)
② 매니페스트 레포 커밋은 배포 기록이므로 오히려 쌓이는 것이 정상
③ CI 무한 루프 위험 없음 (다른 레포이므로)
④ 모노레포 구조로 여러 프로젝트 매니페스트를 한 곳에서 관리 가능
```

### 3.3 매니페스트 모노레포 구조

`liooos-k8s-manifests` 레포는 여러 프로젝트의 K8s 매니페스트를 관리하는 모노레포이다:

```
liooos-k8s-manifests/
├── bitcoin/                # bitcoin-auto-trader 프로젝트
│   ├── base/               # 공통 매니페스트 (SealedSecret 포함)
│   └── overlays/prod/      # 프로덕션 오버레이
├── argocd/                 # ArgoCD Application 정의
│   └── bitcoin-app.yaml
└── (향후 다른 프로젝트 추가 가능)
```

## 4. ArgoCD 설치

미니 PC(K3s)에서 실행한다.

### 4.1 네임스페이스 생성 및 설치

```bash
# ArgoCD 네임스페이스 생성
kubectl create namespace argocd

# ArgoCD 설치 (공식 매니페스트, Server-Side Apply 필수)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side=true --force-conflicts
```

> **왜 `--server-side=true`인가?**
>
> ArgoCD의 CRD(특히 `ApplicationSet`)는 매니페스트가 매우 크다. 일반 `kubectl apply`(Client-Side Apply)는 이전 상태를 `metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"]`에 JSON으로 저장하는데, **annotation은 262KB 제한**이 있어 대용량 CRD에서 에러가 발생한다.
>
> `--server-side=true`는 이 작업을 API 서버가 직접 처리하며, annotation 대신 `managedFields`로 추적하므로 크기 제한이 없다.
>
> | | Client-Side Apply (기본) | Server-Side Apply |
> |---|---|---|
> | 처리 위치 | kubectl (클라이언트) | API 서버 |
> | 상태 추적 | `last-applied-configuration` annotation | `managedFields` |
> | 크기 제한 | annotation 262KB 제한 | 없음 |
> | 충돌 처리 | 덮어씀 | 필드 소유권 기반 충돌 감지 |
>
> `--force-conflicts`는 다른 관리자가 소유한 필드도 강제로 가져온다. 초기 설치/업그레이드 시 안전하게 사용할 수 있다.

### 4.2 설치 확인

```bash
# ArgoCD Pod 확인 (모두 Running이 될 때까지 대기)
kubectl get pods -n argocd -w

# 출력 예시:
# NAME                                  READY   STATUS    AGE
# argocd-application-controller-0       1/1     Running   2m
# argocd-dex-server-xxx                 1/1     Running   2m
# argocd-notifications-controller-xxx   1/1     Running   2m
# argocd-redis-xxx                      1/1     Running   2m
# argocd-repo-server-xxx                1/1     Running   2m
# argocd-server-xxx                     1/1     Running   2m
```

| Pod | 역할 |
|-----|------|
| application-controller | Git과 클러스터 상태 비교, 동기화 실행 |
| server | API 서버 + 웹 UI 제공 |
| repo-server | Git 저장소에서 매니페스트 가져오기 |
| redis | 캐시 (세션, 상태 정보) |
| dex-server | SSO 인증 (사용하지 않으면 무시 가능) |
| notifications-controller | 알림 (Slack, Discord 등) |

### 4.3 ArgoCD CLI 설치

```bash
# Mac
brew install argocd

# Ubuntu (미니 PC)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

## 5. ArgoCD UI 접속

### 5.1 NodePort로 외부 노출

```bash
# argocd-server Service를 NodePort로 변경
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "targetPort": 8080, "nodePort": 30443}]}}'
```

이제 `https://미니PC_IP:30443`으로 접속할 수 있다.

> **NodePort란?**
> 클러스터의 모든 노드에서 특정 포트(30000~32767)를 열어 외부 접근을 허용하는 Service 타입이다.
> `노드IP:NodePort` → Service → Pod으로 트래픽이 전달된다.

### 5.2 초기 비밀번호 확인

```bash
# 초기 admin 비밀번호 확인
argocd admin initial-password -n argocd

# 또는 kubectl로 직접 확인
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

- ID: `admin`
- PW: 위 명령어로 확인한 비밀번호

### 5.3 비밀번호 변경 (권장)

```bash
# ArgoCD 서버에 로그인
argocd login 미니PC_IP:30443 --insecure

# 비밀번호 변경
argocd account update-password
```

## 6. 매니페스트 레포 연결

### 6.1 Private Repository 연결 (CLI)

GitHub Private Repository인 `liooos-k8s-manifests`를 ArgoCD에 등록한다:

```bash
# GitHub PAT(Personal Access Token) 필요
# GitHub → Settings → Developer settings → Personal access tokens
# 권한: repo (Full control of private repositories)

argocd repo add https://github.com/GITHUB_USER/liooos-k8s-manifests.git \
  --username GITHUB_USER \
  --password GITHUB_PAT_TOKEN
```

### 6.2 UI에서 연결

ArgoCD UI에서도 가능하다:
Settings → Repositories → Connect Repo → VIA HTTPS

| 필드 | 값 |
|------|-----|
| Repository URL | `https://github.com/GITHUB_USER/liooos-k8s-manifests.git` |
| Username | GitHub 사용자명 |
| Password | GitHub PAT |

### 6.3 연결 확인

```bash
argocd repo list

# 출력 예시:
# TYPE  NAME  REPO                                                      STATUS
# git         https://github.com/GITHUB_USER/liooos-k8s-manifests.git   Successful
```

## 7. Application 설정

ArgoCD Application은 "어떤 Git 저장소의 어떤 경로를 어떤 클러스터에 배포할지" 정의한다.

### 7.1 Application YAML 작성

매니페스트 레포의 `argocd/bitcoin-app.yaml`:

```yaml
# argocd/bitcoin-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bitcoin-trader              # ArgoCD에서 보이는 앱 이름
  namespace: argocd                 # Application 리소스는 argocd NS에 생성
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # Application 삭제 시 리소스도 정리
spec:
  project: default                  # ArgoCD 프로젝트 (기본값 사용)

  # === 소스 (매니페스트 레포) ===
  source:
    repoURL: https://github.com/GITHUB_USER/liooos-k8s-manifests.git
    targetRevision: main             # 감시할 브랜치
    path: bitcoin/overlays/prod      # Kustomize overlay 경로

  # === 대상 (어디에 배포할 것인가) ===
  destination:
    server: https://kubernetes.default.svc  # 현재 클러스터
    namespace: bitcoin                       # 배포할 네임스페이스

  # === 동기화 정책 ===
  syncPolicy:
    automated:                        # 자동 동기화 활성화
      prune: true                     # Git에서 삭제된 리소스 → 클러스터에서도 삭제
      selfHeal: true                  # 수동 변경 → 자동으로 Git 상태로 복원
    syncOptions:
      - CreateNamespace=true          # 네임스페이스가 없으면 자동 생성
      - ApplyOutOfSyncOnly=true       # 변경된 리소스만 적용 (성능 최적화)
    retry:
      limit: 5                        # 동기화 실패 시 최대 5회 재시도
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**핵심 설정 설명:**

| 설정 | 설명 |
|------|------|
| `repoURL` | **매니페스트 레포** (소스 레포가 아님에 주의) |
| `targetRevision: main` | main 브랜치 감시 (Woodpecker CI가 newTag 업데이트하면 감지) |
| `path: bitcoin/overlays/prod` | 매니페스트 레포 내 Kustomize overlay 경로 |
| `automated.prune` | Git에서 리소스를 제거하면 클러스터에서도 자동 삭제 |
| `automated.selfHeal` | 누군가 kubectl로 직접 변경해도 Git 상태로 자동 복원 |
| `ApplyOutOfSyncOnly` | 변경된 리소스만 적용 (불필요한 재적용 방지) |
| `finalizers` | Application 삭제 시 배포된 K8s 리소스도 함께 정리 |

### 7.2 Application 배포

```bash
kubectl apply -f argocd/bitcoin-app.yaml
```

또는 ArgoCD CLI:

```bash
argocd app create bitcoin-trader \
  --repo https://github.com/GITHUB_USER/liooos-k8s-manifests.git \
  --path bitcoin/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace bitcoin \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

## 8. 전체 배포 사이클

```
① 개발자가 소스 레포(bitcoin-auto-trader)에 dev 브랜치로 push

② GitHub Webhook → Synology 리버스 프록시 → Woodpecker CI Server

③ Woodpecker Agent가 Step Pod 생성:
   Step 1: eclipse-temurin:21-jdk Pod → Gradle 빌드 + 테스트
   Step 2: kaniko Pod → 이미지 빌드 → registry:5000/bitcoin-trader:abc1234 push
   Step 3: alpine/git Pod → 매니페스트 레포 clone → newTag 업데이트 → git push

④ 매니페스트 레포(liooos-k8s-manifests) main 브랜치에 커밋 발생
   변경 내용: bitcoin/overlays/prod/kustomization.yaml의 newTag: abc1234

⑤ ArgoCD가 매니페스트 레포 변경 감지 (3분마다 폴링)

⑥ kustomize build → 렌더링된 매니페스트와 클러스터 현재 상태 비교

⑦ 차이 발견 → 자동 Sync → K8s Rolling Update 실행
   - 새 Pod 생성 (maxSurge: 1)
   - Readiness Probe 통과 대기
   - 이전 Pod 종료 (maxUnavailable: 0)

⑧ ArgoCD UI에서 확인: Healthy + Synced 상태
```

**역할 분담 다이어그램:**

```
소스 레포 (dev push)
    │
    ▼
Woodpecker CI (빌드 전담)         매니페스트 레포 (liooos-k8s-manifests)
    ├── Gradle 빌드+테스트              │
    ├── kaniko 이미지 빌드              │
    └── newTag 업데이트 ───────→ git push (main)
                                        │
                                        ▼
                                 ArgoCD (배포 전담)
                                    ├── 변경 감지 (3분 폴링)
                                    ├── kustomize build
                                    └── K8s Sync (Rolling Update)
                                        │
                                        ▼
                                 K3s 클러스터 (무중단 배포 완료)
```

> Woodpecker CI의 `.woodpecker.yml` 파이프라인 상세는 [[06-인프라/Woodpecker CI 설치 및 설정 가이드|Woodpecker CI 설치 및 설정 가이드]]를 참조한다.

## 9. 롤백 방법

### 9.1 ArgoCD UI에서 롤백

1. Application 클릭
2. History and Rollback 탭
3. 이전 버전 선택 → Rollback 클릭

> 주의: ArgoCD UI 롤백은 일시적이다. `selfHeal: true` 설정이면 다음 폴링 주기에 다시 최신 Git 상태로 동기화된다.
> 영구 롤백이 필요하면 Git 기반 롤백을 사용해야 한다.

### 9.2 Git 기반 롤백 (권장)

매니페스트 레포에서 이전 커밋을 되돌린다:

```bash
# 매니페스트 레포에서 작업
cd liooos-k8s-manifests

# 최신 커밋 되돌리기
git revert HEAD
git push

# ArgoCD가 자동 감지 → 이전 이미지 태그로 롤백 배포
```

GitOps 방식에서는 **Git revert가 가장 깔끔한 롤백 방법**이다. 롤백 이력도 Git에 남는다.

### 9.3 kubectl rollout undo (비권장)

```bash
kubectl rollout undo deployment/bitcoin-trader -n bitcoin
```

> 주의: `kubectl rollout undo`로 롤백해도 ArgoCD의 `selfHeal` 설정이 클러스터 상태를 Git 상태와 다시 일치시키므로, 곧 원래대로 돌아간다. kubectl 직접 롤백은 ArgoCD 환경에서는 사실상 무효하다.

## 10. ArgoCD 웹 UI 사용법

`https://미니PC_IP:30443` 접속

### 10.1 Application 상태

```
┌─────────────────────────────────────────────┐
│  bitcoin-trader                              │
│  Status: Healthy, Synced                     │
│                                              │
│  ┌──────────┐   ┌──────────┐   ┌─────────┐ │
│  │Deployment│──→│ Service  │──→│ Ingress │ │
│  │ 1/1      │   │          │   │         │ │
│  └────┬─────┘   └──────────┘   └─────────┘ │
│       │                                      │
│  ┌────▼─────┐                                │
│  │  Pod     │                                │
│  │ Running  │                                │
│  └──────────┘                                │
└─────────────────────────────────────────────┘
```

| 상태 | 의미 |
|------|------|
| Synced | Git과 클러스터 상태 일치 |
| OutOfSync | Git과 클러스터 상태 불일치 (동기화 대기) |
| Healthy | 모든 리소스가 정상 동작 |
| Degraded | 일부 리소스에 문제 |
| Progressing | 배포 진행 중 |

### 10.2 유용한 기능

| 기능 | 설명 |
|------|------|
| Sync | 수동으로 즉시 동기화 (3분 폴링 대기 없이) |
| Rollback | 이전 배포 버전으로 롤백 |
| Diff | Git 상태와 클러스터 상태의 차이 확인 |
| Logs | Pod 로그 실시간 확인 |
| Events | K8s 이벤트 확인 |

## 11. 리소스 사용량

ArgoCD는 6개의 Pod으로 구성되며, 싱글 노드 환경에서 약 **350~500MB RAM**을 사용한다.

### 11.1 리소스 최적화 (선택)

싱글 노드 환경에서 리소스를 절약하려면:

```bash
# Application Controller 리소스 제한
kubectl patch statefulset argocd-application-controller -n argocd \
  --type='json' -p='[
    {"op": "replace", "path": "/spec/template/spec/containers/0/resources",
     "value": {"requests": {"cpu": "100m", "memory": "256Mi"},
               "limits": {"cpu": "500m", "memory": "512Mi"}}}
  ]'
```

> 미니 PC(32GB RAM)에서는 기본 설정 그대로 사용해도 충분하다. K3s + PostgreSQL + App + Woodpecker + Registry + ArgoCD + Sealed Secrets 전체가 약 5GB 이내이다.

## 12. 트러블슈팅

### 12.1 ApplicationSet CRD 설치 실패 (annotation 크기 초과)

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid:
metadata.annotations: Too long: may not be more than 262144 bytes
```

**원인**: `kubectl apply`(Client-Side Apply)가 `last-applied-configuration` annotation에 전체 CRD를 저장하려 하는데, ApplicationSet CRD가 262KB를 초과한다.

**해결**: Server-Side Apply로 재적용한다.

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side=true --force-conflicts

# CRD 등록 확인
kubectl get crd applicationsets.argoproj.io

# applicationset-controller Pod 재시작
kubectl rollout restart deployment argocd-applicationset-controller -n argocd
```

> ArgoCD 업그레이드 시에도 동일하게 `--server-side=true --force-conflicts`를 사용해야 한다.

### 12.2 이미지 Pull 실패

```
ImagePullBackOff: registry:5000/bitcoin-trader:abc1234
```

원인과 해결:
- K3s `registries.yaml`에 Local Registry 미러 설정 확인
- 이미지 태그가 실제로 Registry에 존재하는지 확인: `curl http://미니PC_IP:31500/v2/bitcoin-trader/tags/list`
- Deployment의 `imagePullPolicy` 확인 (`IfNotPresent` 권장)

### 12.3 동기화 실패

```bash
# ArgoCD 앱 상태 확인
argocd app get bitcoin-trader

# 동기화 수동 실행 (디버깅)
argocd app sync bitcoin-trader --prune

# 상세 로그 확인
kubectl logs -n argocd deployment/argocd-application-controller
```

### 12.4 Git 접근 실패 (매니페스트 레포)

```bash
# 저장소 연결 상태 확인
argocd repo list

# 저장소 재등록 (PAT 만료 시)
argocd repo add https://github.com/GITHUB_USER/liooos-k8s-manifests.git \
  --username GITHUB_USER --password NEW_GITHUB_PAT
```

GitHub PAT가 만료되면 ArgoCD가 매니페스트 레포를 읽지 못한다. PAT 유효기간을 확인하고, 만료 시 갱신 후 재등록해야 한다.

### 12.5 ArgoCD와 Woodpecker 동시 배포 충돌

Woodpecker CI 파이프라인이 매니페스트 레포에 push한 직후, ArgoCD가 이를 감지하여 배포한다. 이 과정에서 충돌이 발생할 수 있는 경우:

| 상황 | 원인 | 해결 |
|------|------|------|
| OutOfSync 상태 지속 | ArgoCD 폴링 주기(3분) 내에 여러 번 push | 잠시 대기, 자동으로 최종 상태에 동기화됨 |
| 이전 배포와 새 배포가 교차 | Rolling Update 중 새 이미지 태그 push | maxSurge/maxUnavailable 설정으로 자연스럽게 처리됨 |

> 정상적인 상황에서는 충돌이 거의 발생하지 않는다. Woodpecker가 매니페스트 레포에 push → ArgoCD가 감지 → 배포로 이어지는 단방향 흐름이기 때문이다.

## 13. K8s에서 정리할 리소스

ArgoCD를 도입하면 Woodpecker CI가 더 이상 직접 `kubectl set image`를 실행하지 않으므로, 기존 Woodpecker 배포용 RBAC 리소스를 제거한다:

```bash
# Woodpecker 배포용 ServiceAccount, Role, RoleBinding 삭제
kubectl delete serviceaccount woodpecker-deploy -n woodpecker
kubectl delete role deployer -n bitcoin
kubectl delete rolebinding woodpecker-deployer -n bitcoin
```

> 이 리소스들은 Woodpecker가 `kubectl set image`로 직접 배포할 때 필요했던 것이다. ArgoCD가 배포를 담당하므로 더 이상 필요하지 않다.

## 14. 검증 방법

설치 완료 후 다음 순서로 검증한다:

### 14.1 매니페스트 레포 직접 수정 테스트

```bash
# 1. 매니페스트 레포에서 newTag를 수동으로 변경
cd liooos-k8s-manifests
# bitcoin/overlays/prod/kustomization.yaml의 newTag 수정 후 push

# 2. ArgoCD UI에서 Synced 상태 확인 (최대 3분 대기)

# 3. Pod이 새 이미지로 교체되었는지 확인
kubectl get pods -n bitcoin -w
```

### 14.2 전체 파이프라인 테스트 (E2E)

```bash
# 1. 소스 레포(bitcoin-auto-trader)에서 dev 브랜치에 push

# 2. Woodpecker CI 파이프라인 성공 확인 (웹 UI)

# 3. 매니페스트 레포에 newTag 커밋이 생겼는지 확인
cd liooos-k8s-manifests && git pull && git log --oneline -3

# 4. ArgoCD UI에서 자동 Sync 확인 → Rolling Update 진행

# 5. 애플리케이션 헬스체크
curl http://미니PC_IP:NODE_PORT/actuator/health
```

### 14.3 자가 치유(Self-Heal) 테스트

```bash
# 1. kubectl로 replicas 수동 변경
kubectl scale deployment bitcoin-trader -n bitcoin --replicas=3

# 2. ArgoCD가 자동으로 Git 상태(replicas: 1)로 복원하는지 확인
kubectl get pods -n bitcoin -w
```

## 15. 구축 순서 요약

| 순서 | 작업 | 명령어/위치 |
|------|------|-------------|
| 1 | 매니페스트 레포 생성 | GitHub에 `liooos-k8s-manifests` 생성 |
| 2 | 매니페스트 파일 이동 | 소스 레포 `k8s/` → 매니페스트 레포 `bitcoin/` |
| 3 | ArgoCD 네임스페이스 생성 | `kubectl create namespace argocd` |
| 4 | ArgoCD 설치 | `kubectl apply -n argocd -f install.yaml` |
| 5 | NodePort 설정 | `kubectl patch svc argocd-server ...` |
| 6 | CLI 설치 + 로그인 | `brew install argocd` + `argocd login` |
| 7 | 매니페스트 레포 등록 | `argocd repo add ...liooos-k8s-manifests.git` |
| 8 | Application 생성 | `kubectl apply -f argocd/bitcoin-app.yaml` |
| 9 | Woodpecker 파이프라인 수정 | deploy Step을 update-manifest Step으로 변경 |
| 10 | 기존 RBAC 정리 | `kubectl delete serviceaccount/role/rolebinding` |
| 11 | E2E 검증 | dev push → Woodpecker → 매니페스트 커밋 → ArgoCD Sync |

> K8s 매니페스트 구조는 [[06-인프라/K8s 매니페스트 가이드|K8s 매니페스트 가이드]]를, Sealed Secrets는 [[06-인프라/Sealed Secrets 가이드|Sealed Secrets 가이드]]를, Woodpecker CI 파이프라인은 [[06-인프라/Woodpecker CI 설치 및 설정 가이드|Woodpecker CI 설치 및 설정 가이드]]를 참조한다.
