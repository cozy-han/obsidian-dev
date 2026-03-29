# K3s 설치 가이드

> 최종 수정: 2026-03-01
> 환경: Ubuntu Server (24.04 LTS 권장), 32GB RAM, Intel i5

---

## 1. 사전 준비: Ubuntu Server 초기 설정

K3s를 설치하기 전, 서버의 기본 보안과 네트워크를 설정한다.

### 1.1 시스템 업데이트

```bash
# 패키지 목록 업데이트 및 업그레이드
sudo apt update && sudo apt upgrade -y
```

> **`apt update`**: 패키지 저장소에서 최신 목록을 가져온다 (설치는 안 함).
> **`apt upgrade`**: 실제로 설치된 패키지를 최신 버전으로 업그레이드한다.
> **`-y`**: 모든 확인 프롬프트에 자동으로 "yes"를 선택한다.

### 1.2 기본 패키지 설치

```bash
sudo apt install -y curl wget git vim htop net-tools
```

| 패키지 | 용도 |
|--------|------|
| `curl` | URL에서 데이터 다운로드 (K3s 설치 스크립트에 필요) |
| `wget` | 파일 다운로드 (curl과 비슷하지만 파일 저장에 특화) |
| `git` | 버전 관리 (ArgoCD 연동 테스트 시 필요) |
| `vim` | 텍스트 편집기 |
| `htop` | 시스템 리소스 모니터링 (CPU/메모리 사용량 확인) |
| `net-tools` | `ifconfig` 등 네트워크 유틸리티 |

### 1.3 방화벽 설정

```bash
# UFW (Uncomplicated Firewall) 설치 및 활성화
sudo apt install -y ufw

# SSH 접속 허용 (반드시 먼저!)
sudo ufw allow ssh

# K3s API 서버 포트 (kubectl 원격 접속용)
sudo ufw allow 6443/tcp

# HTTP/HTTPS (웹 접근용)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# NodePort 범위 (서비스 외부 노출용)
sudo ufw allow 30000:32767/tcp

# 방화벽 활성화
sudo ufw enable

# 상태 확인
sudo ufw status verbose
```

> **주의**: `sudo ufw allow ssh`를 먼저 실행하지 않으면 방화벽 활성화 후 SSH 접속이 끊긴다!

| 포트 | 용도 |
|------|------|
| 22 | SSH 원격 접속 |
| 6443 | K3s API 서버 (kubectl 명령어 수신) |
| 80 | HTTP 트래픽 (Traefik Ingress) |
| 443 | HTTPS 트래픽 (Traefik Ingress) |
| 30000-32767 | NodePort 서비스 (ArgoCD UI 등) |

### 1.4 서버 IP 확인

```bash
# 내부 IP 확인
ip addr show | grep "inet " | grep -v 127.0.0.1

# 또는
hostname -I
```

이 IP를 기억해두자. 이후 kubectl 원격 접속, ArgoCD UI 접속 등에 사용한다.

---

## 2. K3s 설치

### 2.1 K3s란?

K3s는 Rancher Labs가 만든 **경량 Kubernetes**이다.
- 단일 바이너리 (~100MB)로 전체 K8s 기능 제공
- CNCF 공식 인증 배포판 (표준 K8s와 100% API 호환)
- IoT, Edge, 개발환경, 소규모 프로덕션에 적합

**K3s가 자동으로 포함하는 것:**

```
K3s 바이너리 하나에 포함:
├── kube-apiserver      ← K8s API 서버
├── kube-controller     ← 리소스 상태 관리
├── kube-scheduler      ← Pod 배치 결정
├── kubelet             ← Pod 실행 관리
├── kube-proxy          ← 네트워크 프록시
├── containerd          ← 컨테이너 런타임 (Docker 대신)
├── SQLite              ← 기본 데이터스토어 (etcd 대신)
├── Traefik             ← Ingress Controller
├── CoreDNS             ← 클러스터 DNS
├── ServiceLB           ← LoadBalancer 구현
├── local-path-provisioner ← 로컬 스토리지
└── kubectl             ← CLI 도구
```

### 2.2 설치 실행

```bash
# K3s 설치 (한 줄이면 끝!)
curl -sfL https://get.k3s.io | sh -
```

> **명령어 해석:**
> - `curl -sfL https://get.k3s.io`: K3s 설치 스크립트를 다운로드
>   - `-s`: silent (진행 표시 숨김)
>   - `-f`: 서버 에러 시 실패 처리
>   - `-L`: 리다이렉트 따라가기
> - `| sh -`: 다운로드한 스크립트를 셸에서 실행

설치가 완료되면 K3s가 **systemd 서비스로 자동 등록**되어 부팅 시 자동 시작된다.

### 2.3 설치 확인

```bash
# K3s 서비스 상태 확인
sudo systemctl status k3s

# 출력 예시:
# ● k3s.service - Lightweight Kubernetes
#      Loaded: loaded (/etc/systemd/system/k3s.service; enabled; ...)
#      Active: active (running) since ...
```

```bash
# kubectl로 노드 확인
sudo kubectl get nodes

# 출력 예시:
# NAME        STATUS   ROLES                  AGE   VERSION
# mini-pc     Ready    control-plane,master   1m    v1.31.x+k3s1
```

`STATUS`가 `Ready`이면 K3s가 정상적으로 동작하는 것이다.

```bash
# 시스템 Pod 확인 (K3s 내장 컴포넌트들)
sudo kubectl get pods -A

# 출력 예시:
# NAMESPACE     NAME                                     READY   STATUS
# kube-system   coredns-xxx                              1/1     Running
# kube-system   local-path-provisioner-xxx               1/1     Running
# kube-system   metrics-server-xxx                       1/1     Running
# kube-system   svclb-traefik-xxx                        1/1     Running
# kube-system   traefik-xxx                              1/1     Running
```

모든 Pod이 `Running` 상태인지 확인한다.

### 2.4 kubectl 권한 설정

K3s의 kubeconfig 파일은 `/etc/rancher/k3s/k3s.yaml`에 있으며, 기본적으로 root 권한이 필요하다.
매번 `sudo`를 붙이지 않으려면:

```bash
# 현재 사용자가 kubectl을 sudo 없이 사용할 수 있도록 설정
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# 환경변수 설정 (.bashrc 또는 .zshrc에 추가)
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

> **kubeconfig란?**
> kubectl이 K8s 클러스터에 접속하기 위한 설정 파일이다.
> 클러스터 주소, 인증 정보, 컨텍스트 등이 포함되어 있다.

이제 `sudo` 없이 kubectl을 사용할 수 있다:

```bash
kubectl get nodes       # sudo 불필요
kubectl get pods -A     # sudo 불필요
```

---

## 3. kubectl 기본 사용법

kubectl은 K8s 클러스터를 제어하는 CLI 도구이다.

### 3.1 기본 명령어 패턴

```bash
kubectl <동사> <리소스 타입> [이름] [옵션]
```

### 3.2 자주 사용하는 명령어

```bash
# === 조회 (get) ===
kubectl get nodes                    # 노드 목록
kubectl get pods                     # 현재 네임스페이스의 Pod 목록
kubectl get pods -A                  # 모든 네임스페이스의 Pod 목록
kubectl get pods -n bitcoin          # 특정 네임스페이스의 Pod 목록
kubectl get services                 # Service 목록
kubectl get deployments              # Deployment 목록
kubectl get all                      # 주요 리소스 모두 조회

# === 상세 정보 (describe) ===
kubectl describe pod <pod-이름>      # Pod 상세 정보 (이벤트, 상태 등)
kubectl describe node <node-이름>    # 노드 상세 정보 (리소스 사용량 등)

# === 로그 (logs) ===
kubectl logs <pod-이름>              # Pod 로그 출력
kubectl logs -f <pod-이름>           # 로그 실시간 스트리밍 (follow)
kubectl logs <pod-이름> --tail=100   # 최근 100줄만

# === 실행 (exec) ===
kubectl exec -it <pod-이름> -- bash  # Pod 내부에 접속 (SSH처럼)
kubectl exec -it <pod-이름> -- sh    # bash가 없으면 sh 사용

# === 적용/삭제 (apply/delete) ===
kubectl apply -f manifest.yaml       # 매니페스트 적용 (생성 또는 업데이트)
kubectl delete -f manifest.yaml      # 매니페스트에 정의된 리소스 삭제

# === 네임스페이스 (namespace) ===
kubectl create namespace bitcoin     # 네임스페이스 생성
kubectl get namespaces               # 네임스페이스 목록
```

### 3.3 유용한 별칭 (alias)

```bash
# ~/.bashrc 또는 ~/.zshrc에 추가
echo '
alias k="kubectl"
alias kgp="kubectl get pods"
alias kgpa="kubectl get pods -A"
alias kgs="kubectl get services"
alias kgd="kubectl get deployments"
alias kga="kubectl get all"
alias kd="kubectl describe"
alias kl="kubectl logs"
alias klf="kubectl logs -f"
alias ka="kubectl apply -f"
' >> ~/.bashrc
source ~/.bashrc
```

이제 `k get pods` 처럼 짧게 사용할 수 있다.

---

## 4. 원격에서 kubectl 접속 (워크스테이션에서)

개발 PC에서 미니 PC의 K3s 클러스터를 원격으로 제어할 수 있다.

### 4.1 미니 PC에서 kubeconfig 복사

```bash
# 미니 PC에서 실행
sudo cat /etc/rancher/k3s/k3s.yaml
```

출력된 내용을 복사한다.

### 4.2 개발 PC에서 설정

```bash
# 개발 PC (Mac/Windows)에서 실행
mkdir -p ~/.kube

# 복사한 내용을 config 파일로 저장
vim ~/.kube/config   # 또는 nano, code 등 편집기 사용
```

**중요**: `server` 항목의 `127.0.0.1`을 **미니 PC의 실제 IP**로 변경해야 한다.

```yaml
# 변경 전
server: https://127.0.0.1:6443

# 변경 후 (미니 PC IP가 192.168.1.100인 경우)
server: https://192.168.1.100:6443
```

### 4.3 접속 테스트

```bash
# 개발 PC에서 실행
kubectl get nodes

# 미니 PC의 노드가 보이면 성공!
```

> **주의**: kubectl이 개발 PC에 설치되어 있어야 한다.
> Mac: `brew install kubectl`
> Windows: `choco install kubernetes-cli`

---

## 5. K3s 관리 명령어

### 5.1 서비스 제어

```bash
# K3s 서비스 상태 확인
sudo systemctl status k3s

# K3s 재시작
sudo systemctl restart k3s

# K3s 중지
sudo systemctl stop k3s

# K3s 자동 시작 활성화/비활성화
sudo systemctl enable k3s     # 부팅 시 자동 시작
sudo systemctl disable k3s    # 자동 시작 끄기
```

### 5.2 K3s 완전 제거 (필요시)

```bash
# K3s 및 모든 데이터 완전 삭제
/usr/local/bin/k3s-uninstall.sh
```

> **주의**: 이 명령은 K3s와 함께 모든 Pod, 볼륨 데이터를 삭제한다.

### 5.3 K3s 업그레이드

```bash
# 재설치로 업그레이드 (데이터 유지됨)
curl -sfL https://get.k3s.io | sh -
```

---

## 6. 트러블슈팅

### 6.1 K3s 시작 실패 시

```bash
# 로그 확인
sudo journalctl -u k3s -f

# 일반적인 원인:
# 1. 포트 충돌: 6443 포트를 다른 프로세스가 사용 중
# 2. 메모리 부족: 최소 512MB 필요
# 3. 방화벽: 6443 포트가 차단됨
```

### 6.2 Pod이 Pending 상태일 때

```bash
# Pod 상세 정보 확인
kubectl describe pod <pod-이름>

# Events 섹션을 확인:
# - Insufficient memory → 리소스 부족
# - FailedScheduling → 노드에 배치 불가
# - ImagePullBackOff → 이미지 다운로드 실패
```

### 6.3 이미지 Pull 실패 시

```bash
# Private registry 인증 확인
kubectl get secrets

# 이미지 이름/태그 확인
kubectl describe pod <pod-이름> | grep Image
```

---

## 7. 다음 단계

K3s 설치가 완료되었으면, 다음 단계로 진행한다:

→ [[06-인프라/Dockerfile 작성 가이드|Dockerfile 작성 가이드]] — 애플리케이션을 컨테이너로 패키징
