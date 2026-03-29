# Sealed Secrets 가이드

> 최종 수정: 2026-03-09
> 목적: API 키, DB 패스워드 등 민감 데이터를 Git에 안전하게 저장

---

## 1. 문제: Secret을 Git에 어떻게 저장하는가?

GitOps에서는 **모든 K8s 매니페스트를 Git에 저장**한다. 하지만 Secret은 민감 데이터를 포함한다.

```yaml
# K8s Secret (base64 인코딩 = 암호화가 아님!)
apiVersion: v1
kind: Secret
data:
  UPBIT_ACCESS_KEY: bXktYWNjZXNzLWtleQ==     # echo -n 'my-access-key' | base64
  UPBIT_SECRET_KEY: bXktc2VjcmV0LWtleQ==     # 누구나 디코딩 가능!
```

이것을 Git에 커밋하면 **API 키가 노출**된다.

**해결 방법 비교:**

| 방법 | 동작 | 장점 | 단점 |
|------|------|------|------|
| Sealed Secrets | 암호화된 Secret을 Git에 저장 | 간단, GitOps 친화적 | 클러스터 종속적 |
| External Secrets | 외부 저장소(Vault, AWS SM)에서 동기화 | 중앙 관리 | 외부 서비스 필요 |
| SOPS | 파일 단위 암호화 | 범용적 | ArgoCD 연동 복잡 |

**Sealed Secrets**가 가장 간단하고 GitOps와 잘 맞으므로 이것을 사용한다.

## 2. Sealed Secrets 동작 원리

```
개발자 PC                        K3s 클러스터
┌────────────────┐               ┌─────────────────────────────┐
│                │               │ Sealed Secrets Controller   │
│ 1. Secret YAML │               │    (비대칭 키 쌍 보유)        │
│    작성        │               │    - 공개키: 암호화용          │
│                │               │    - 개인키: 복호화용 (클러스터 │
│ 2. kubeseal    │               │              내부에만 존재)   │
│    로 암호화   │               │                             │
│    (공개키 사용)│               │                             │
│                │               └─────────────────────────────┘
│ 3. SealedSecret│                         │
│    YAML 생성   │                         │
└───────┬────────┘                         │
        │                                  │
        ▼                                  │
  ┌──────────┐    ArgoCD/kubectl apply     │
  │  GitHub   │ ─────────────────────────→ │
  │  (안전!)  │                            │
  │ Sealed    │                            ▼
  │ Secret    │               ┌─────────────────────────┐
  │ (암호화됨) │               │ Controller가 복호화     │
  └──────────┘               │ SealedSecret → Secret    │
                              │ (일반 Secret 자동 생성)  │
                              └─────────────────────────┘
```

**핵심:**
- **공개키**(Public Key)로 암호화 → 누구나 할 수 있음 (kubeseal)
- **개인키**(Private Key)로 복호화 → 클러스터 내 Controller만 가능
- Git에는 공개키로 암호화된 SealedSecret만 저장 → 안전!

## 3. Sealed Secrets Controller 설치

미니 PC(K3s)에서 실행한다.

### 3.1 Helm으로 설치

```bash
# Helm이 없으면 설치
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

> **Helm이란?** K8s의 패키지 매니저이다.
> apt(Ubuntu), brew(Mac)처럼 K8s 앱을 쉽게 설치/관리할 수 있다.

```bash
# Sealed Secrets Helm 저장소 추가
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update

# sealed-secrets 네임스페이스에 설치
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --set fullnameOverride=sealed-secrets-controller
```

### 3.2 설치 확인

```bash
# Controller Pod 확인
kubectl get pods -n kube-system | grep sealed-secrets

# 출력 예시:
# sealed-secrets-controller-xxx   1/1   Running   0   1m

# Controller 로그 확인
kubectl logs -n kube-system -l app.kubernetes.io/name=sealed-secrets
```

## 4. kubeseal CLI 설치

개발 PC(워크스테이션)에서 Secret을 암호화하기 위해 kubeseal을 설치한다.

### Mac

```bash
brew install kubeseal
```

### Ubuntu (미니 PC에서 직접 할 경우)

```bash
KUBESEAL_VERSION=$(curl -s https://api.github.com/repos/bitnami-labs/sealed-secrets/releases/latest | grep tag_name | cut -d '"' -f4 | cut -d 'v' -f2)
curl -OL "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"
tar -xvzf kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

## 5. Secret 암호화하기

### 5.1 평문 Secret YAML 작성 (Git에 커밋하지 않는다!)

PostgreSQL과 App의 Secret은 **각각 다른 네임스페이스**에 생성한다.

```yaml
# /tmp/postgres-secret.yaml (임시 파일, 작업 후 삭제!)
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: database              # database 네임스페이스
type: Opaque
stringData:
  POSTGRES_PASSWORD: "관리자_비밀번호"
  BITCOIN_DB_PASSWORD: "bitcoin_사용자_비밀번호"
```

```yaml
# /tmp/app-secret.yaml (임시 파일, 작업 후 삭제!)
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: bitcoin               # bitcoin 네임스페이스
type: Opaque
stringData:
  DB_PASSWORD: "bitcoin_사용자_비밀번호"
```

> **`data` vs `stringData`:**
> - `data`: base64 인코딩된 값 (`echo -n 'value' | base64`)
> - `stringData`: 평문 (K8s가 자동으로 base64 인코딩)

### 5.2 kubeseal로 암호화

```bash
# PostgreSQL Secret → SealedSecret으로 암호화
kubeseal --format yaml < /tmp/postgres-secret.yaml > postgres/base/postgres-sealed-secret.yaml

# App Secret → SealedSecret으로 암호화
kubeseal --format yaml < /tmp/app-secret.yaml > bitcoin/base/app-sealed-secret.yaml

# 원본 평문 Secret 삭제 (중요!)
rm /tmp/postgres-secret.yaml /tmp/app-secret.yaml
```

> **파일 위치**: Kustomize는 `base/` 디렉토리 외부의 파일을 참조할 수 없으므로,
> SealedSecret 파일은 각 프로젝트의 `base/` 디렉토리 안에 위치한다.

생성된 SealedSecret은 이렇게 생겼다:

```yaml
# bitcoin/base/app-sealed-secret.yaml (Git에 커밋 안전!)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secret
  namespace: bitcoin
spec:
  encryptedData:
    DB_PASSWORD: AgDk8sP1...암호화된_긴_문자열...
```

이 파일은 **개인키 없이는 복호화가 불가능**하므로 Git에 안전하게 커밋할 수 있다.

### 5.3 SealedSecret 적용

SealedSecret은 각 프로젝트의 `kustomization.yaml`에 resource로 포함되어 있으므로,
**ArgoCD가 자동으로 적용**한다. 수동 적용이 필요한 경우:

```bash
# ArgoCD를 통해 적용 (권장)
argocd app sync bitcoin-trader
argocd app sync postgres

# 생성된 일반 Secret 확인
kubectl get secrets -n bitcoin
# NAME         TYPE     DATA   AGE
# app-secret   Opaque   1      10s

kubectl get secrets -n database
# NAME              TYPE     DATA   AGE
# postgres-secret   Opaque   2      10s
```

## 6. Secret 업데이트 방법

API 키가 변경되었을 때:

```bash
# 1. 새 평문 Secret 작성
cat > /tmp/updated-secret.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: bitcoin
type: Opaque
stringData:
  UPBIT_ACCESS_KEY: "새로운_access_key"
  UPBIT_SECRET_KEY: "새로운_secret_key"
  DB_PASSWORD: "비밀번호"
  DISCORD_ALERTS_WEBHOOK_URL: "webhook_url"
  DISCORD_ERRORS_WEBHOOK_URL: "webhook_url"
EOF

# 2. 다시 암호화 (매니페스트 레포에서 실행)
kubeseal --format yaml < /tmp/updated-secret.yaml > bitcoin/base/app-sealed-secret.yaml

# 3. 원본 삭제
rm /tmp/updated-secret.yaml

# 4. Git 커밋 & 푸시 → ArgoCD가 자동 적용
git add bitcoin/base/app-sealed-secret.yaml
git commit -m "chore: update sealed secrets"
git push
```

## 7. 백업과 복구

### 7.1 개인키 백업 (매우 중요!)

```bash
# Controller의 개인키를 백업
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > /안전한/위치/sealed-secrets-key-backup.yaml
```

> **주의**: 이 백업 파일은 **절대 Git에 커밋하지 않는다!**
> USB, 별도 비밀 관리 도구 등에 안전하게 보관한다.
> 이 키를 잃으면 기존 SealedSecret을 복호화할 수 없다.

### 7.2 K3s 재설치 시 복구

```bash
# 백업한 키를 먼저 적용
kubectl apply -f /안전한/위치/sealed-secrets-key-backup.yaml

# Sealed Secrets Controller 재설치
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# ArgoCD가 매니페스트 레포의 SealedSecret을 자동 적용
argocd app sync bitcoin-trader
argocd app sync postgres
```

## 8. 주의사항

1. **평문 Secret YAML은 절대 Git에 커밋하지 않는다**
2. **SealedSecret은 특정 클러스터에 종속적** — 다른 클러스터에서는 복호화 불가
3. **Controller 개인키를 반드시 백업** — 키 분실 시 복구 불가
4. **namespace가 일치해야 한다** — SealedSecret의 namespace와 실제 배포 namespace

## 9. 다음 단계

Sealed Secrets 설정이 완료되었으면, 다음 단계로 진행한다:

→ [[06-인프라/Woodpecker CI 설치 및 설정 가이드|Woodpecker CI 가이드]] — CI 파이프라인으로 자동 빌드 및 이미지 푸시
