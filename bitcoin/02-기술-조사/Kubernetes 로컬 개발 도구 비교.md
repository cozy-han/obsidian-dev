# Kubernetes 로컬 개발 도구 비교

> Windows 환경에서 비트코인 자동매매 시스템 운영을 위한 K8s 도구 선정 가이드

---

## Kubernetes란?

컨테이너화된 애플리케이션의 **배포, 스케일링, 관리를 자동화**하는 오픈소스 컨테이너 오케스트레이션 플랫폼이다. Google이 개발하고 현재 CNCF(Cloud Native Computing Foundation)가 관리한다.

### 핵심 개념

| 개념                     | 설명                                                                |
| ---------------------- | ----------------------------------------------------------------- |
| **Pod**                | 배포 가능한 최소 단위. 하나 이상의 컨테이너를 포함하며 같은 네트워크/스토리지 공유                   |
| **Service**            | Pod 집합에 대한 안정적인 네트워크 엔드포인트 제공 (ClusterIP, NodePort, LoadBalancer) |
| **Deployment**         | Stateless 앱 관리. 롤링 업데이트, 롤백, 스케일링 처리                              |
| **StatefulSet**        | Stateful 앱(DB 등) 관리. 고유 ID와 안정적 스토리지 보장                           |
| **ReplicaSet**         | 지정된 수의 Pod 복제본이 항상 실행되도록 보장                                       |
| **Namespace**          | 클러스터 내 리소스를 논리적으로 분리                                              |
| **ConfigMap / Secret** | 설정 데이터와 민감 정보를 Pod와 분리하여 관리                                       |
| **Ingress**            | 외부에서 내부 Service로 HTTP/HTTPS 라우팅                                   |
| **PV / PVC**           | 스토리지 추상화. PostgreSQL 같은 DB에 필수                                    |
| **Helm**               | K8s 패키지 매니저. Chart로 복잡한 배포를 템플릿화                                  |

### 아키텍처 흐름

```
외부 요청 → Ingress → Service → Deployment → Pod(컨테이너)
                                                  ↕
                                          PVC → PersistentVolume
```

---

## K8s vs K3s vs K3d — 관계와 차이

이 세 가지는 별개의 제품이 아니라 **계층 관계**에 있다.

```
K8s (Kubernetes)        ← 원본. 풀스펙 오케스트레이션 플랫폼
  └─ K3s                ← K8s의 경량 배포판. 단일 바이너리로 축소
      └─ K3d            ← K3s를 Docker 컨테이너 안에서 실행하는 래퍼
```

### K8s (Kubernetes)

| 항목 | 내용 |
|------|------|
| **정체** | 컨테이너 오케스트레이션의 표준. CNCF 관리 오픈소스 |
| **구성 요소** | etcd, kube-apiserver, kube-scheduler, kube-controller-manager, kubelet, kube-proxy 등 |
| **바이너리 크기** | ~500MB+ (전체 컴포넌트) |
| **최소 RAM** | 2GB+ (마스터 노드만) |
| **용도** | 프로덕션 클라우드 환경 (AWS EKS, GCP GKE, Azure AKS 등) |

- 모든 K8s API와 기능을 **완전하게** 포함
- 프로덕션에서는 보통 관리형 서비스(EKS, GKE)를 사용
- 로컬에서 직접 설치하면 복잡하고 무거움 → 그래서 Minikube, K3s 같은 도구가 등장

### K3s

| 항목          | 내용                                    |
| ----------- | ------------------------------------- |
| **정체**      | Rancher Labs(SUSE)가 만든 **경량 K8s 배포판** |
| **관계**      | K8s의 **서브셋** — CNCF 인증 Kubernetes     |
| **바이너리 크기** | ~100MB 미만 (단일 바이너리)                   |
| **최소 RAM**  | 512MB                                 |
| **용도**      | Edge, IoT, CI/CD, 개발 환경, 소규모 프로덕션     |

**K8s에서 뭘 빼고 뭘 바꿨나?**

| K8s 원본                   | K3s 변경                        |
| ------------------------ | ----------------------------- |
| etcd (분산 저장소)            | **SQLite** 기본 (etcd도 선택 가능)   |
| cloud-controller-manager | 제거 (클라우드 프로바이더 비의존)           |
| 별도 설치 필요한 CNI            | **Flannel** 내장                |
| 별도 설치 필요한 Ingress        | **Traefik** 내장                |
| 별도 설치 필요한 DNS            | **CoreDNS** 내장                |
| 별도 설치 필요한 스토리지           | **local-path-provisioner** 내장 |
| 여러 바이너리                  | **단일 바이너리** (`k3s`)           |

> K3s는 "5 less than K8s"라는 의미. K8s에서 5를 빼면 K3s (8-5=3)

**핵심 포인트**: K3s는 K8s API와 **99% 호환**된다. `kubectl`을 그대로 사용할 수 있고, Helm Chart도 대부분 동작한다. CNCF 공식 인증 Kubernetes 배포판이다.

### K3d

| 항목        | 내용                                              |
| --------- | ----------------------------------------------- |
| **정체**    | K3s를 **Docker 컨테이너 안에서** 실행하는 래퍼 도구             |
| **관계**    | K3s의 **실행 도구** — K3s 자체를 포함하지 않고 Docker 이미지로 실행 |
| **설치 크기** | CLI 도구만 (~수 MB) + K3s Docker 이미지                |
| **용도**    | 로컬 개발, CI/CD 파이프라인, 일회용 클러스터                    |

**K3s와 K3d의 차이**

| 항목 | K3s | K3d |
|------|-----|-----|
| **실행 방식** | 호스트 OS에 직접 설치/실행 | Docker 컨테이너 안에서 K3s 실행 |
| **설치 대상** | Linux (네이티브) | Docker가 있는 모든 OS |
| **Windows 지원** | 미지원 (WSL2 내에서만) | Docker만 있으면 가능 |
| **클러스터 생성** | `k3s server` (호스트 직접) | `k3d cluster create` (컨테이너) |
| **멀티노드** | 여러 머신 또는 VM 필요 | **단일 머신에서 컨테이너로 시뮬레이션** |
| **격리** | 호스트 OS에 영향 | Docker 컨테이너로 격리 |
| **정리** | 언인스톨 필요 | `k3d cluster delete`로 깔끔 제거 |

### 실제 사용 시나리오 비교

```
┌─────────────────────────────────────────────────────┐
│ 프로덕션 (AWS/GCP/Azure)                              │
│   → K8s 관리형 서비스 (EKS, GKE, AKS) 사용             │
├─────────────────────────────────────────────────────┤
│ Edge/IoT/소규모 서버                                   │
│   → K3s 직접 설치 (Linux 서버에 단일 바이너리)             │
├─────────────────────────────────────────────────────┤
│ 로컬 개발/테스트 (Windows/Mac)                          │
│   → K3d (Docker 안에서 K3s 실행)                       │
│   → 또는 Rancher Desktop (내부적으로 K3s 사용)           │
│   → 또는 Minikube / Docker Desktop K8s               │
├─────────────────────────────────────────────────────┤
│ CI/CD 파이프라인 (GitHub Actions 등)                    │
│   → K3d (빠른 생성/삭제, 컨테이너 기반)                    │
└─────────────────────────────────────────────────────┘
```

### 한 줄 요약

| 도구 | 한 줄 정리 |
|------|-----------|
| **K8s** | 컨테이너 오케스트레이션의 **표준 사양** |
| **K3s** | K8s의 **경량 구현체** (단일 바이너리, 내장 컴포넌트) |
| **K3d** | K3s를 **Docker에서 실행하는 도구** (로컬 개발 특화) |

---

## 도구별 상세 비교

### 1. Minikube

> K8s 공식 프로젝트. 가장 풍부한 문서와 호환성

- **최신 버전**: v1.38.0 (K8s v1.35.0)
- **라이선스**: Apache 2.0 (완전 무료)
- **드라이버**: Docker, Hyper-V, VirtualBox, QEMU 등

#### 장점
- K8s 공식 프로젝트, 가장 완전한 K8s API 호환
- Addon 시스템 풍부 (dashboard, ingress, metrics-server)
- 다양한 드라이버/런타임 선택 가능
- 학습 자료가 가장 많음

#### 단점
- 시작 속도 느림 (30초~1분+)
- 리소스 소비량이 상대적으로 큼
- 멀티노드 설정 복잡
- 초기 설정 옵션이 많아 혼란 가능

#### 리소스
| 항목 | 최소 | 권장 |
|------|------|------|
| CPU | 2코어 | 4코어 |
| RAM | 2GB | 8GB |
| 디스크 | 20GB | 40GB+ |

#### Windows 설치
```powershell
# Chocolatey
choco install minikube

# 또는 winget
winget install minikube

# 시작 (Hyper-V)
minikube start --driver=hyperv

# 시작 (Docker)
minikube start --driver=docker
```

---

### 2. K3s / K3d

> K3s = 경량 K8s 배포판 (Rancher Labs/SUSE)
> K3d = K3s를 Docker 컨테이너 안에서 실행하는 래퍼

- **K3s 최신 버전**: v1.35.0
- **K3d 최신 버전**: v5.8.3+
- **라이선스**: K3s Apache 2.0 / K3d MIT (모두 무료)
- **특징**: 단일 바이너리 ~100MB, etcd 대신 SQLite, 내장 Traefik

#### 장점
- **가장 빠른 시작 속도** (5초 미만)
- 메모리 사용량이 가장 적음 (512MB~)
- 멀티노드 클러스터 쉽게 생성
- CI/CD 파이프라인에 이상적
- 일회용 클러스터 생성/삭제가 빠름

#### 단점
- 완전한 K8s가 아닌 경량 배포판 (일부 기능 제거)
- **K3s는 Windows 네이티브 미지원** (WSL2/Docker 의존)
- GUI 없음 (CLI만 제공)
- 일부 에코시스템 도구 호환성 이슈 가능

#### 리소스
| 항목 | 최소 | 권장 |
|------|------|------|
| CPU | 1코어 | 2코어 |
| RAM | 512MB | 2~4GB |
| 디스크 | 최소한 | Docker 의존 |

#### Windows 설치 (K3d)
```powershell
# 전제: Docker Desktop 또는 Rancher Desktop 필요
choco install k3d

# 클러스터 생성
k3d cluster create my-cluster

# 멀티노드 클러스터
k3d cluster create my-cluster --servers 1 --agents 3

# 이미지 로드
k3d image import my-app:latest -c my-cluster
```

---

### 3. Rancher Desktop

> Docker Desktop의 완전 무료 대안. GUI 제공

- **최신 버전**: v1.21.0
- **라이선스**: Apache 2.0 (완전 무료, 기업 규모 무관)
- **내부 엔진**: K3s 기반
- **런타임 선택**: containerd 또는 dockerd(moby)

#### 장점
- **완전 무료** (기업 규모 무관 — Docker Desktop과 가장 큰 차이)
- 직관적인 GUI
- Docker Desktop과 유사한 사용 경험
- K8s 버전을 GUI에서 쉽게 전환
- dockerd 모드로 기존 Docker 워크플로우 유지 가능

#### 단점
- Docker Desktop 대비 UI 완성도 약간 낮음
- 커뮤니티 규모가 Docker Desktop보다 작음
- Windows에서 간헐적 메모리 누수 보고
- Docker Compose 호환성 간혹 불완전

#### 리소스
| 항목 | 기본값 | 설명 |
|------|--------|------|
| CPU | 호스트의 50% | GUI에서 조정 가능 |
| RAM | 호스트의 50% | GUI에서 조정 가능 |

#### Windows 설치
```powershell
# 전제: WSL2 설치 필요
winget install suse.RancherDesktop

# 설치 후 GUI에서:
# 1. Container Runtime: dockerd(moby) 선택
# 2. Kubernetes 버전 선택
# 3. Enable Kubernetes 체크
```

---

### 4. Docker Desktop Kubernetes

> 가장 간편한 설정. 체크박스 하나로 K8s 활성화

- **최신 버전**: Docker Desktop 4.38+
- **K8s 엔진**: kubeadm 기반 + kind 멀티노드 (2026~)
- **라이선스**: 조건부 무료

#### 라이선스 정책

| 조건 | 가격 |
|------|------|
| 개인 / 교육 / 오픈소스 | **무료** |
| 소규모 기업 (250명 미만 AND 매출 $10M 미만) | **무료** |
| Docker Pro | $9/월 |
| Docker Team | $15/유저/월 |
| Docker Business (대기업) | $21/유저/월 |

#### 장점
- **가장 간편한 설정** (Settings > K8s 체크박스)
- 빌드 이미지를 바로 K8s에서 사용 가능 (push 불필요)
- 가장 큰 커뮤니티와 문서
- Docker Compose와 완벽 통합
- Resource Saver 모드 (유휴 시 리소스 절약)
- 2026년부터 kind 기반 멀티노드 지원

#### 단점
- 대기업에서 유료 (가격 인상 추세)
- **리소스 소비량이 가장 큼**
- WSL2 관련 메모리 이슈 간헐적 보고

#### 리소스
| 항목 | 기본값 | 권장 |
|------|--------|------|
| CPU | 호스트의 50% | 4코어+ |
| RAM | 호스트의 50% | 12~16GB |
| K8s 추가 | +1~2GB | - |
| 디스크 | 가변 | 50GB+ |

#### Windows 설치
```powershell
winget install Docker.DockerDesktop

# 설치 후 GUI에서:
# Settings > Kubernetes > Enable Kubernetes 체크
```

---

## 종합 비교표

| 항목             | Minikube     | K3d         | Rancher Desktop | Docker Desktop K8s |
| -------------- | ------------ | ----------- | --------------- | ------------------ |
| **라이선스**       | 무료           | 무료          | 무료              | 조건부 무료             |
| **GUI**        | 제한적          | 없음          | 있음              | 있음 (최상)            |
| **시작 속도**      | 30초~1분+      | **5초 미만**   | 20~40초          | 30초~1분             |
| **최소 RAM**     | 2GB          | **512MB**   | 호스트 50%         | 호스트 50%            |
| **권장 RAM**     | 8GB          | **2~4GB**   | 4~8GB           | 12~16GB            |
| **멀티노드**       | 복잡           | **쉬움**      | 미지원             | kind 가능            |
| **K8s 호환성**    | **완전**       | 경량(대부분 호환)  | 경량(K3s)         | 완전                 |
| **이미지 통합**     | docker-env   | import 명령   | dockerd 모드      | **네이티브**           |
| **Windows 지원** | Hyper-V/WSL2 | WSL2+Docker | WSL2            | WSL2/Hyper-V       |
| **학습 곡선**      | 중간           | 높음          | **낮음**          | **낮음**             |

---

## Windows 환경 추천

### 1순위: Rancher Desktop ★★★★★

- **완전 무료**이면서 Docker Desktop과 유사한 GUI 경험
- dockerd 모드로 기존 Docker 워크플로우 유지 가능
- K8s 버전 GUI 전환 지원
- 오픈소스 프로젝트에 라이선스 부담 없음
- Spring Boot 이미지 빌드 후 바로 K8s 배포 가능

### 2순위: Docker Desktop Kubernetes ★★★★☆

- **가장 매끄러운 개발 경험**
- 개인 사용이므로 무료
- 이미지 빌드-배포 워크플로우가 가장 자연스러움
- 2026년 멀티노드 지원으로 추후 확장성 좋음
- 단, 라이선스 정책 변경 리스크 존재

### 3순위: K3d ★★★★☆ (CLI 숙련자)

- 리소스가 제한된 환경에서 최선
- 가장 빠른 클러스터 생성/삭제
- CLI에 익숙하다면 가장 빠른 개발 사이클
- 멀티노드 테스트에 유리

### 4순위: Minikube ★★★☆☆

- K8s 학습 목적이라면 최적
- 완전한 K8s 호환이 필요한 경우
- 실제 운영에는 다소 무거움

---

## 자동매매 시스템 기준 최종 권장

> **Rancher Desktop** + 필요 시 K3d 병행

| 이유 | 설명 |
|------|------|
| 무료 | 소스코드 오픈 시 다른 사용자도 라이선스 부담 없음 |
| GUI | K8s 버전 전환, 리소스 모니터링 편리 |
| dockerd 호환 | Docker Compose로 개발 → K8s로 전환 용이 |
| K3s 기반 | 경량이면서도 프로덕션 사용 사례 풍부 |
| CI/CD 연동 | GitHub Actions에서 K3d로 동일 환경 테스트 가능 |
