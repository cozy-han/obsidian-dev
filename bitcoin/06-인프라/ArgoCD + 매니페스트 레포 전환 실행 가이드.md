# ArgoCD + 매니페스트 레포 전환 실행 가이드

> 최종 수정: 2026-03-04
> 목적: Woodpecker CI(빌드+배포) → Woodpecker CI(빌드만) + ArgoCD(배포) 전환 시 수동 실행할 작업 정리
> 사전 조건: 로컬 파일 변경 완료 (매니페스트 레포 생성, .woodpecker.yml 수정, 소스 레포 정리)

---

## 전체 흐름

```
[로컬 Mac] ──────────────────────────────────────────────────────────
  1. GitHub에 매니페스트 레포 생성 + push
  2. 소스 레포 변경사항 커밋 + push

[미니 PC (SSH)] ─────────────────────────────────────────────────────
  3. ArgoCD 설치
  4. ArgoCD CLI 설치 + 로그인
  5. 매니페스트 레포 연결 (SSH 또는 HTTPS)
  5.5 CI 인프라 수동 배포 (Registry + Secret)
  6. ArgoCD Application 적용 (CLI 또는 kubectl)
  7. Woodpecker Secrets 등록 (웹 UI)
  8. .woodpecker.yml GITHUB_USER 치환
  9. 기존 Woodpecker 배포 RBAC 정리
  10. E2E 검증
```

---

## 1. GitHub에 매니페스트 레포 생성 + push

> 실행 위치: **로컬 Mac**

### 1.1 GitHub에서 레포 생성

GitHub 웹 → New Repository:

| 항목              | 값                                         |
| --------------- | ----------------------------------------- |
| Repository name | `liooos-k8s-manifests`                    |
| Visibility      | **Private**                               |
| Initialize      | 체크 해제 (README, .gitignore, license 모두 없이) |

### 1.2 GITHUB_USER 치환 + push

```bash
cd /Users/han/works/workspace/personal/liooos-k8s-manifests

# 모든 파일에서 GITHUB_USER를 실제 사용자명으로 치환
sed -i '' 's/GITHUB_USER/실제사용자명/g' argocd/bitcoin-app.yaml

# 치환 확인
grep -r 'GITHUB_USER' . || echo "치환 완료 - GITHUB_USER 없음"

git add -A
git commit -m "init: bitcoin K8s 매니페스트 및 ArgoCD Application"
git remote add origin https://github.com/GITHUB_USER/liooos-k8s-manifests.git
git push -u origin main
```

### 1.3 확인

```bash
# GitHub 웹에서 파일 구조 확인
# bitcoin/base/, bitcoin/overlays/prod/, argocd/ 가 있어야 함
```

---

## 2. 소스 레포 변경사항 커밋 + push

> 실행 위치: **로컬 Mac**

```bash
cd /Users/han/works/workspace/personal/bitcoin

git add -A
git status
# 확인할 변경사항:
#   수정: .woodpecker.yml (deploy → update-manifest)
#   삭제: k8s/base/*, k8s/overlays/*, k8s/ci/deploy-rbac.yaml, k8s/sealed-secrets/

git commit -m "refactor: Woodpecker 빌드 전용 전환, 매니페스트 레포 분리"
git push origin dev
```

> **주의**: 아직 ArgoCD가 설치되지 않았으므로, 이 push로 Woodpecker 파이프라인의 update-manifest Step이 실패할 수 있다. ArgoCD 설치 후 다시 실행하면 된다.

---

## 3. ArgoCD 설치

> 실행 위치: **미니 PC (SSH)**

```bash
ssh minipc
```

### 3.1 네임스페이스 생성 및 설치

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3.2 Pod 정상 기동 대기

```bash
# 모든 Pod이 Running이 될 때까지 대기 (약 2-3분)
kubectl get pods -n argocd -w

# 출력 예시 (모두 Running이면 Ctrl+C):
# argocd-application-controller-0       1/1     Running
# argocd-dex-server-xxx                 1/1     Running
# argocd-notifications-controller-xxx   1/1     Running
# argocd-redis-xxx                      1/1     Running
# argocd-repo-server-xxx                1/1     Running
# argocd-server-xxx                     1/1     Running
```

### 3.3 NodePort로 외부 노출

```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "targetPort": 8080, "nodePort": 30443}]}}'
```

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 8080, "nodePort": 30080}]}}'
```

### 3.4 초기 비밀번호 확인

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

> 출력된 비밀번호를 메모한다. ID는 `admin`.

### 3.5 접속 확인

브라우저에서 `https://미니PC_IP:30443` 접속 → `admin` / 위 비밀번호로 로그인

---

## 4. ArgoCD CLI 설치 + 로그인

> 실행 위치: **미니 PC (SSH)**

### 4.1 CLI 설치

```bash
# Ubuntu (미니 PC)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# 설치 확인
argocd version --client
```

> Mac에서도 CLI를 설치하려면: `brew install argocd`

### 4.2 로그인

```bash
argocd login 미니PC_IP:30443 --insecure
# Username: admin
# Password: 3단계에서 확인한 비밀번호
```

### 4.3 비밀번호 변경 (권장)

```bash
argocd account update-password
```

---

## 5. 매니페스트 레포 연결

> 실행 위치: **미니 PC (SSH)**

두 가지 인증 방식 중 하나를 선택한다.

### 방식 A: SSH (추천 — 토큰 만료 없음)

#### 5.1a SSH 키 생성 (ArgoCD 전용)

```bash
ssh-keygen -t ed25519 -C "argocd@k3s" -f ~/argocd-repo-key -N ""
```

> `-C`는 단순 코멘트(주석)이며 인증과 무관하다. GitHub 이메일이 아니어도 된다.

#### 5.2a GitHub에 Deploy Key 등록

1. GitHub 웹 → `liooos-k8s-manifests` 레포 → Settings → Deploy keys → Add deploy key
2. 공개키 내용 붙여넣기:

```bash
cat ~/argocd-repo-key.pub
```

3. **Allow write access**는 체크 해제 (ArgoCD는 읽기만 함)

#### 5.3a ArgoCD에 SSH 레포 등록

```bash
argocd repo add git@github.com:GITHUB_USER/liooos-k8s-manifests.git \
  --ssh-private-key-path ~/argocd-repo-key
```

### 방식 B: HTTPS + PAT

#### 5.1b GitHub PAT 생성

GitHub 웹 → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token:

| 항목 | 값 |
|------|---|
| Note | `ArgoCD + Woodpecker manifest repo` |
| Expiration | 90 days (또는 원하는 기간) |
| Scopes | **`repo`** (Full control of private repositories) 체크 |

> 생성된 토큰을 메모한다. 이 토큰은 ArgoCD(5번)와 Woodpecker(7번) 모두에서 사용한다.

#### 5.2b ArgoCD에 HTTPS 레포 등록

```bash
argocd repo add https://github.com/GITHUB_USER/liooos-k8s-manifests.git \
  --username GITHUB_USER \
  --password GITHUB_PAT_TOKEN
```

### 두 방식 비교

| 항목 | SSH | HTTPS + PAT |
|------|-----|-------------|
| 토큰 만료 | **없음** | 있음 (갱신 필요) |
| 보안 | Deploy Key (레포 단위 제한) | PAT (계정 전체 접근 가능) |
| 설정 난이도 | 약간 복잡 | 쉬움 |
| 방화벽 | 22 포트 | 443 포트 |

> SSH 방식을 선택해도 Woodpecker의 매니페스트 레포 push에는 별도의 PAT가 필요하다 (7번 참조).

### 5.4 연결 확인

```bash
argocd repo list

# 출력 예시:
# TYPE  NAME  REPO                                                        STATUS
# git         git@github.com:GITHUB_USER/liooos-k8s-manifests.git         Successful
```

> `Successful`이 아니면 SSH 키 또는 PAT 권한을 확인한다.

---

## 5.5 CI 인프라 수동 배포 (Registry + Secret)

> 실행 위치: **미니 PC (SSH)**
> ArgoCD는 매니페스트 레포(`liooos-k8s-manifests`)만 감시한다.
> CI 인프라(Registry)와 Secret은 소스 레포(`bitcoin/k8s/ci/`)에 있으므로 ArgoCD가 관리하지 않는다.
> 한번 설치하면 거의 변경이 없으므로 수동으로 배포한다.

### 5.5.1 Local Registry 배포

```bash
# woodpecker 네임스페이스 생성
kubectl create namespace woodpecker

# Registry (PVC + Deployment + NodePort Service) 배포
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-data
  namespace: woodpecker
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: woodpecker
  labels:
    app: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: registry:2
          ports:
            - containerPort: 5000
          volumeMounts:
            - name: data
              mountPath: /var/lib/registry
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 128Mi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: registry-data
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: woodpecker
spec:
  type: NodePort
  selector:
    app: registry
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 31500
EOF
```

Registry 확인:

```bash
kubectl get pods -n woodpecker
# registry-xxx   1/1   Running 이면 성공
```

### 5.5.2 Secret 생성

매니페스트에서 참조하는 Secret 2개를 생성한다.

```bash
# PostgreSQL 비밀번호
kubectl create secret generic postgres-secret -n bitcoin \
  --from-literal=POSTGRES_PASSWORD=비밀번호

# 앱 시크릿 (DB 비밀번호, Upbit API Key, Discord Webhook 등)
kubectl create secret generic app-secret -n bitcoin \
  --from-literal=DB_PASSWORD=비밀번호 \
  --from-literal=UPBIT_ACCESS_KEY=키 \
  --from-literal=UPBIT_SECRET_KEY=키 \
  --from-literal=DISCORD_WEBHOOK_URL=URL
```

> 나중에 Sealed Secrets로 전환하면 Git에서 암호화된 상태로 관리할 수 있다.

### 5.5.3 확인

```bash
kubectl get secret -n bitcoin
# postgres-secret, app-secret 이 있어야 함

kubectl get pods -n bitcoin
# postgres, bitcoin-trader가 자동 복구되는지 확인
```

---

## 6. ArgoCD Application 적용

> 실행 위치: **미니 PC (SSH)**
> 사전 조건: 1번 단계에서 `argocd/bitcoin-app.yaml`의 `GITHUB_USER`를 실제 사용자명으로 치환 후 push한 상태여야 한다.

두 가지 방식 중 하나를 선택한다.

### 방식 A: ArgoCD CLI (추천 — clone 불필요)

ArgoCD CLI로 직접 Application을 생성한다. clone이 필요 없어 가장 간단하다.

```bash
argocd app create bitcoin-trader \
  --repo git@github.com:GITHUB_USER/liooos-k8s-manifests.git \
  --path bitcoin/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace bitcoin \
  --sync-policy automated \
  --self-heal
```

### 방식 B: kubectl apply (매니페스트 파일 기반)

`bitcoin-app.yaml`을 직접 적용한다. Application YAML에 세부 설정이 포함되어 있어 선언적 관리가 가능하다.

```bash
# 매니페스트 레포 clone (SSH)
git clone git@github.com:GITHUB_USER/liooos-k8s-manifests.git /tmp/liooos-k8s-manifests

# Application 적용
kubectl apply -f /tmp/liooos-k8s-manifests/argocd/bitcoin-app.yaml

# 임시 디렉토리 정리
rm -rf /tmp/liooos-k8s-manifests
```

### 두 방식 비교

| 항목 | 방식 A (CLI) | 방식 B (kubectl) |
|------|-------------|-----------------|
| clone 필요 | 불필요 | 필요 |
| 설정 관리 | CLI 인자로 지정 | YAML 파일로 선언적 관리 |
| Git 추적 | 안됨 | Application YAML이 Git에 남음 |
| 재현성 | 명령어 기록 필요 | YAML 파일만 있으면 재현 가능 |

> 운영 환경에서는 **방식 B**가 Git 기반 관리에 유리하지만, 초기 구축 시에는 **방식 A**가 빠르다.

### 확인

```bash
# ArgoCD에서 Application 상태 확인
argocd app get bitcoin-trader

# 또는 웹 UI에서 확인 (https://미니PC_IP:30443)
# bitcoin-trader가 Synced + Healthy 상태여야 함
```

---

## 7. Woodpecker Secrets 등록

> 실행 위치: **브라우저 (Woodpecker 웹 UI)**

1. Woodpecker 웹 UI 접속
2. `bitcoin` 레포 → Settings → Secrets
3. Secret 추가:

| Name | Value | 용도 |
|------|-------|------|
| `manifest_repo_token` | 5번에서 생성한 GitHub PAT | 매니페스트 레포 push |

> Woodpecker에서 `${MANIFEST_REPO_TOKEN}`으로 참조된다.

---

## 8. `.woodpecker.yml`의 GITHUB_USER 치환

> 실행 위치: **로컬 Mac**

```bash
cd /Users/han/works/workspace/personal/bitcoin

# .woodpecker.yml에서 GITHUB_USER를 실제 GitHub 사용자명으로 변경
sed -i '' 's/GITHUB_USER/실제사용자명/g' .woodpecker.yml

# 변경 확인
cat .woodpecker.yml | grep github.com

# 커밋 + push
git add .woodpecker.yml
git commit -m "chore: set actual GitHub username in woodpecker pipeline"
git push origin dev
```

---

## 9. 기존 Woodpecker 배포 RBAC 정리

> 실행 위치: **미니 PC (SSH)**

ArgoCD가 배포를 담당하므로, Woodpecker의 직접 배포용 RBAC을 제거한다.

```bash
# 존재 여부 먼저 확인
kubectl get serviceaccount woodpecker-deploy -n woodpecker 2>/dev/null
kubectl get role deployer -n bitcoin 2>/dev/null
kubectl get rolebinding woodpecker-deployer -n bitcoin 2>/dev/null

# 존재하면 삭제
kubectl delete serviceaccount woodpecker-deploy -n woodpecker
kubectl delete role deployer -n bitcoin
kubectl delete rolebinding woodpecker-deployer -n bitcoin
```

---

## 10. E2E 검증

### 10.1 소스 레포에서 dev push

```bash
# 로컬 Mac에서
cd /Users/han/works/workspace/personal/bitcoin

# 아무 변경이라도 추가 (또는 8번에서 이미 push한 상태)
git push origin dev
```

### 10.2 Woodpecker 파이프라인 확인

Woodpecker 웹 UI에서 파이프라인 3단계 성공 확인:

| Step | 상태 | 확인 사항 |
|------|------|----------|
| build-and-test | 성공 | Gradle 빌드 + 테스트 통과 |
| build-image | 성공 | kaniko 이미지 빌드 → Registry push |
| update-manifest | 성공 | 매니페스트 레포 newTag 업데이트 → git push |

### 10.3 매니페스트 레포 커밋 확인

```bash
# 미니 PC 또는 로컬에서
cd liooos-k8s-manifests
git pull
git log --oneline -3

# "chore(bitcoin): update image tag to abc1234" 같은 커밋이 있어야 함
```

### 10.4 ArgoCD 자동 Sync 확인

- ArgoCD 웹 UI(`https://미니PC_IP:30443`)에서 bitcoin-trader 앱 확인
- 상태: **Healthy + Synced**
- 이미지 태그가 최신 커밋 SHA와 일치하는지 확인

### 10.5 앱 헬스체크

```bash
curl http://미니PC_IP:NODE_PORT/actuator/health

# 또는 kubectl로 확인
kubectl get pods -n bitcoin
kubectl logs -f deployment/bitcoin-trader -n bitcoin --tail=20
```

### 10.6 자가 치유(Self-Heal) 테스트 (선택)

```bash
# kubectl로 replicas 수동 변경
kubectl scale deployment bitcoin-trader -n bitcoin --replicas=3

# ArgoCD가 자동으로 Git 상태(replicas: 1)로 복원하는지 확인
kubectl get pods -n bitcoin -w
# 약간의 시간 후 다시 1개로 돌아가면 성공
```

---

## 트러블슈팅

### ArgoCD CLI 세션 만료 (Unauthenticated)

```
rpc error: code = Unauthenticated desc = invalid session: account password has changed since token issued
```

비밀번호 변경 후 기존 토큰이 무효화된 것. 재로그인으로 해결:

```bash
argocd login 미니PC_IP:30443 --insecure
```

### update-manifest Step 실패

```
fatal: Authentication failed
```

→ Woodpecker Secrets에 `manifest_repo_token` 확인. GitHub PAT가 `repo` 권한을 가지고 있는지 확인.

### ArgoCD OutOfSync 상태 지속

```bash
# 수동 Sync 실행
argocd app sync bitcoin-trader

# 상세 로그
kubectl logs -n argocd deployment/argocd-application-controller --tail=50
```

### ArgoCD에서 매니페스트 레포 접근 실패

```bash
# 레포 상태 확인
argocd repo list

# PAT 만료 시 재등록
argocd repo add https://github.com/GITHUB_USER/liooos-k8s-manifests.git \
  --username GITHUB_USER --password NEW_PAT
```

### 이미지 Pull 실패 (ImagePullBackOff)

```bash
# Registry에 이미지가 있는지 확인
curl http://localhost:31500/v2/bitcoin-trader/tags/list

# K3s registries.yaml 설정 확인
cat /etc/rancher/k3s/registries.yaml
```

---

## 완료 후 상태

```
소스 레포 (bitcoin)
  .woodpecker.yml → build-and-test → build-image → update-manifest
  k8s/ci/ → CI 인프라만 (namespace, registry, woodpecker-values)

매니페스트 레포 (liooos-k8s-manifests)
  bitcoin/base/ → K8s 공통 매니페스트
  bitcoin/overlays/prod/ → newTag (Woodpecker가 업데이트)
  argocd/ → Application 정의

K3s 클러스터
  argocd NS → ArgoCD (매니페스트 레포 감시 → 자동 배포)
  woodpecker NS → Woodpecker CI (빌드만) + Registry
  bitcoin NS → App + PostgreSQL
```

> 관련 문서: [[06-인프라/인프라 아키텍처 개요|인프라 아키텍처 개요]], [[06-인프라/Woodpecker CI 설치 및 설정 가이드|Woodpecker CI 가이드]], [[06-인프라/ArgoCD 설치 및 GitOps 가이드|ArgoCD 설치 및 GitOps 가이드]]
