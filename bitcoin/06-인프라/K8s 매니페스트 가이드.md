# K8s 매니페스트 가이드

> 최종 수정: 2026-03-08
> 도구: Kustomize (kubectl 내장)

---

## 1. K8s 매니페스트란?

K8s 매니페스트는 "어떤 상태로 클러스터를 유지해달라"는 **선언적 설정 파일**(YAML)이다.

```yaml
# "Pod 2개를 실행하고, 각각 이 이미지를 사용해라"
apiVersion: apps/v1
kind: Deployment          ← 리소스 종류
metadata:
  name: my-app            ← 리소스 이름
spec:
  replicas: 2             ← 원하는 상태 (Pod 2개)
  ...
```

K8s는 이 매니페스트를 받으면:
1. 현재 상태와 원하는 상태를 비교
2. 차이가 있으면 자동으로 조정 (Pod 생성/삭제/업데이트)

이것이 **선언적(Declarative)** 방식의 핵심이다. "이렇게 해라"가 아니라 "이 상태가 되어라"고 선언한다.

## 2. Kustomize란?

Kustomize는 K8s 매니페스트를 **환경별로 커스터마이징**하는 도구이다.

```
k8s/
├── base/                  ← 공통 매니페스트 (모든 환경에서 공유)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── app-deployment.yaml
│   ├── app-service.yaml
│   └── postgres-*.yaml
└── overlays/
    └── prod/              ← 프로덕션 환경 오버라이드
        ├── kustomization.yaml
        └── patches/       ← base를 변경하는 패치
```

**왜 Kustomize인가?**

| 비교 | Helm | Kustomize |
|------|------|-----------|
| 방식 | 템플릿 엔진 (Go template) | 패치 기반 (원본 유지) |
| 학습 곡선 | 높음 (Chart 구조, values.yaml) | 낮음 (YAML만 알면 됨) |
| 설치 | 별도 설치 | kubectl 내장 (추가 설치 불필요) |
| 적합한 상황 | 복잡한 앱, 공개 배포 | 내부 프로젝트, 환경별 설정 |

이 프로젝트는 내부용이고 환경이 1~2개이므로 Kustomize가 적합하다.

## 3. Namespace

```yaml
# k8s/base/namespace.yaml
```

Namespace는 클러스터 내에서 리소스를 **논리적으로 분리**하는 단위이다.

```
클러스터
├── kube-system (네임스페이스)     ← K3s 시스템 컴포넌트
│   ├── coredns
│   ├── traefik
│   └── ...
├── argocd (네임스페이스)          ← ArgoCD
│   ├── argocd-server
│   └── ...
├── database (네임스페이스)        ← 공용 PostgreSQL
│   └── postgres (StatefulSet)
├── bitcoin (네임스페이스)         ← 우리 앱!
│   └── bitcoin-trader-app
├── woodpecker (네임스페이스)      ← CI 인프라
│   ├── woodpecker-server/agent
│   └── registry
└── default (네임스페이스)         ← 기본 (사용하지 않는 것이 좋음)
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bitcoin
  labels:
    app.kubernetes.io/part-of: bitcoin-auto-trader
```

> **label이란?** 리소스에 붙이는 key-value 태그이다.
> 나중에 `kubectl get all -l app.kubernetes.io/part-of=bitcoin-auto-trader`로 관련 리소스를 한 번에 조회할 수 있다.

## 4. PostgreSQL 매니페스트

### 4.1 왜 StatefulSet인가?

| 비교 | Deployment | StatefulSet |
|------|-----------|-------------|
| Pod 이름 | 랜덤 (app-a7x3k) | 순서 (postgres-0) |
| 스토리지 | Pod 삭제 시 소멸 | PVC로 영속 유지 |
| 네트워크 | 불안정 | 안정적 DNS |
| 용도 | Stateless 앱 (웹서버) | Stateful 앱 (DB, 캐시) |

DB는 데이터가 영속적이어야 하므로 StatefulSet을 사용한다.

### 4.2 ConfigMap (DB 초기화 설정)

```yaml
# k8s/base/postgres-configmap.yaml
```

ConfigMap은 설정 데이터를 저장하는 리소스이다. 환경변수나 설정 파일로 Pod에 주입할 수 있다.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: database
data:
  POSTGRES_DB: bitcoin_trader
  POSTGRES_USER: bitcoin
```

> **Secret과 ConfigMap의 차이:**
> - ConfigMap: DB 이름, 포트 등 민감하지 않은 설정
> - Secret: 패스워드, API 키 등 민감한 데이터 (base64 인코딩)

### 4.3 StatefulSet + Service

```yaml
# k8s/base/postgres-service.yaml
```

```yaml
# Headless Service: StatefulSet의 각 Pod에 안정적인 DNS를 제공
# 일반 Service는 로드밸런싱을 하지만,
# Headless Service (clusterIP: None)는 개별 Pod DNS를 생성한다.
# → postgres-0.postgres.bitcoin.svc.cluster.local
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: database
  labels:
    app: postgres
spec:
  clusterIP: None        # Headless Service
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: postgres         # app=postgres 라벨을 가진 Pod에 연결
```

```yaml
# k8s/base/postgres-statefulset.yaml
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: database
spec:
  serviceName: postgres            # 위의 Headless Service 이름과 일치해야 함
  replicas: 1                      # 싱글 노드이므로 1개
  selector:
    matchLabels:
      app: postgres                # 이 라벨을 가진 Pod을 관리
  template:                        # Pod 템플릿
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - configMapRef:
                name: postgres-config       # ConfigMap의 모든 data를 환경변수로
            - secretRef:
                name: postgres-secret       # Secret의 모든 data를 환경변수로
          resources:                         # 리소스 요청/상한
            requests:                        # 최소 보장 리소스
              cpu: 250m                      # 0.25 코어
              memory: 256Mi                  # 256MB
            limits:                          # 최대 사용 가능 리소스
              cpu: "1"                       # 1 코어
              memory: 2Gi                    # 2GB
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data  # PostgreSQL 데이터 디렉토리
          livenessProbe:                     # 컨테이너가 살아있는지 확인
            exec:
              command: ["pg_isready", "-U", "bitcoin"]
            initialDelaySeconds: 30          # 시작 후 30초 뒤부터 체크
            periodSeconds: 10                # 10초마다 체크
          readinessProbe:                    # 트래픽을 받을 준비가 됐는지 확인
            exec:
              command: ["pg_isready", "-U", "bitcoin"]
            initialDelaySeconds: 5
            periodSeconds: 5
  volumeClaimTemplates:                      # PVC 자동 생성 템플릿
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]       # 단일 노드에서만 읽기/쓰기
        storageClassName: local-path          # K3s 내장 스토리지
        resources:
          requests:
            storage: 10Gi                     # 10GB 요청
```

**주요 개념 설명:**

| 필드 | 설명 |
|------|------|
| `resources.requests` | "최소한 이만큼은 보장해주세요" → 스케줄러가 노드 배치 시 참고 |
| `resources.limits` | "이것 이상은 사용 불가" → 초과 시 OOM Kill |
| `livenessProbe` | 실패하면 컨테이너를 **재시작** (죽은 프로세스 복구) |
| `readinessProbe` | 실패하면 Service에서 **제외** (트래픽 안 보냄) |
| `volumeClaimTemplates` | StatefulSet이 자동으로 PVC를 생성. Pod이 삭제돼도 데이터 유지 |
| `storageClassName: local-path` | K3s 내장 provisioner. 노드의 로컬 디스크에 볼륨 생성 |

## 5. Application 매니페스트

### 5.1 ConfigMap (앱 설정)

```yaml
# k8s/base/app-configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: bitcoin
data:
  SERVER_PORT: "9000"
  # DB 접속 정보 (database 네임스페이스의 PostgreSQL에 cross-namespace 접근)
  DB_HOST: postgres.database.svc.cluster.local
  DB_PORT: "5432"
  DB_NAME: bitcoin_trader
  DB_USERNAME: bitcoin
  # 프로파일
  SPRING_PROFILES_ACTIVE: prod
  # 매매 설정
  TRADING_MODE: PAPER
  # 마켓 선정
  MARKET_SELECTION_ENABLED: "true"
  MARKET_SELECTION_TOP_N: "20"
```

> **중요**: PostgreSQL은 `database` 네임스페이스에 있으므로 FQDN으로 접근한다.
> CoreDNS가 `postgres.database.svc.cluster.local` → Pod IP로 해석해준다.
> 같은 네임스페이스라면 Service 이름만으로 충분하지만, cross-namespace 접근 시 전체 DNS 필요.

### 5.2 Deployment

```yaml
# k8s/base/app-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bitcoin-trader
  namespace: bitcoin
  labels:
    app: bitcoin-trader
spec:
  replicas: 1
  # Rolling Update 전략: 새 Pod이 Ready된 후 이전 Pod 종료
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # 업데이트 중 최대 1개 추가 Pod 허용
      maxUnavailable: 0     # 업데이트 중 사용 불가 Pod 0개 (무중단)
  selector:
    matchLabels:
      app: bitcoin-trader
  template:
    metadata:
      labels:
        app: bitcoin-trader
    spec:
      containers:
        - name: bitcoin-trader
          # 이미지는 Woodpecker CI가 빌드 → ArgoCD가 태그 업데이트
          image: registry:5000/bitcoin-trader:latest
          ports:
            - containerPort: 9000
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secret       # Sealed Secrets로 관리
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "2"
              memory: 1Gi
          # Spring Boot Actuator 헬스체크 사용
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 9000
            initialDelaySeconds: 60     # Spring Boot 시작 시간 고려
            periodSeconds: 15
            failureThreshold: 3         # 3번 연속 실패 시 재시작
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 9000
            initialDelaySeconds: 30
            periodSeconds: 10
          # 종료 시 Graceful Shutdown
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 5"]  # 트래픽 드레인 대기
      terminationGracePeriodSeconds: 30  # 최대 30초 동안 우아한 종료 대기
```

**Rolling Update 전략 시각화:**

```
현재: [Pod v1] ← 트래픽 수신 중

업데이트 시작:
1. [Pod v1] [Pod v2 시작중...]     ← maxSurge=1로 새 Pod 추가
2. [Pod v1] [Pod v2 Ready ✓]      ← readinessProbe 통과
3. [Pod v1 종료중] [Pod v2] ←     ← 트래픽이 v2로 전환
4. [Pod v2]                        ← 업데이트 완료 (무중단!)
```

### 5.3 Service

```yaml
# k8s/base/app-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: bitcoin-trader
  namespace: bitcoin
  labels:
    app: bitcoin-trader
spec:
  type: ClusterIP              # 클러스터 내부에서만 접근 가능
  ports:
    - port: 9000               # Service 포트
      targetPort: 9000         # 실제 Pod의 포트
      protocol: TCP
  selector:
    app: bitcoin-trader        # 이 라벨을 가진 Pod에 트래픽 전달
```

> **Service 타입 비교:**
>
> | 타입 | 접근 범위 | 용도 |
> |------|----------|------|
> | ClusterIP | 클러스터 내부만 | 내부 서비스 간 통신 (기본값) |
> | NodePort | 노드IP:포트 | 외부에서 직접 접근 |
> | LoadBalancer | 외부 LB | 클라우드 환경에서 외부 노출 |
>
> Ingress를 사용하므로 ClusterIP면 충분하다.

### 5.4 Ingress (Traefik)

```yaml
# k8s/base/app-ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: bitcoin-trader
  namespace: bitcoin
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: bitcoin-strip-coin@kubernetescrd
spec:
  rules:
    - host: cozyhan.liooos.synology.me
      http:
        paths:
          - path: /coin
            pathType: Prefix
            backend:
              service:
                name: bitcoin-trader   # 위의 Service 이름
                port:
                  number: 9000         # Service 포트
```

> `/coin` 경로로 접근하면 `strip-coin` Middleware가 prefix를 제거하고 앱에 전달한다.
> Middleware 정의는 `strip-prefix-middleware.yaml` 참고.

> **Ingress 동작 흐름:**
> ```
> 외부 요청 → Traefik (Ingress Controller) → Middleware (strip /coin) → Service → Pod
> https://cozyhan.liooos.synology.me/coin → traefik:80 → strip /coin → bitcoin-trader:9000 → Pod:9000
> ```

## 6. Kustomize 설정

### 6.1 Base Kustomization

```yaml
# k8s/base/kustomization.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 이 Kustomization에 포함할 리소스 목록
# PostgreSQL은 별도 database 네임스페이스 (postgres/ 디렉토리)에서 관리
resources:
  - namespace.yaml
  - app-configmap.yaml
  - app-deployment.yaml
  - app-service.yaml
  - app-ingress.yaml
  - strip-prefix-middleware.yaml
```

### 6.2 Production Overlay

```yaml
# k8s/overlays/prod/kustomization.yaml
```

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# base를 기반으로 함
resources:
  - ../../base

# 프로덕션용 이미지 태그 지정
# Woodpecker CI가 빌드 후 newTag를 전체 commit SHA로 자동 업데이트
images:
  - name: registry:5000/bitcoin-trader
    newTag: latest      # ← Woodpecker CI가 자동 업데이트 (전체 commit SHA)
```

### 6.3 Kustomize 사용법

```bash
# 매니페스트 미리보기 (실제 적용하지 않음)
kubectl kustomize k8s/overlays/prod

# 매니페스트 적용
kubectl apply -k k8s/overlays/prod

# 매니페스트 삭제
kubectl delete -k k8s/overlays/prod
```

> **`kubectl apply -f` vs `kubectl apply -k`:**
> - `-f`: 단일 YAML 파일 적용
> - `-k`: Kustomize 디렉토리 적용 (kustomization.yaml 기반으로 합침)

## 7. 핵심 YAML 문법 정리

```yaml
# 문자열
name: "hello"
name: hello          # 따옴표 없어도 됨 (숫자로 시작하면 필요)

# 숫자
replicas: 3
port: 9000

# 리스트 (배열)
ports:
  - 80
  - 443

# 맵 (객체)
resources:
  requests:
    cpu: 250m
    memory: 512Mi

# 멀티라인 문자열
description: |       # | : 줄바꿈 유지
  첫 번째 줄
  두 번째 줄

# K8s 리소스 단위
# CPU: 1 = 1코어, 500m = 0.5코어, 250m = 0.25코어
# Memory: 1Gi = 1024Mi = 1048576Ki
```

## 8. 배포 순서

```bash
# 1. 네임스페이스 먼저 생성
kubectl apply -f k8s/base/namespace.yaml

# 2. Secret 적용 (ArgoCD가 자동 관리, 수동 시 아래 참고)
# SealedSecret은 각 base/ 디렉토리에 포함 → Kustomize에서 함께 적용됨

# 3. 전체 매니페스트 적용
kubectl apply -k k8s/overlays/prod

# 4. 배포 상태 확인
kubectl get all -n bitcoin

# 5. Pod 로그 확인
kubectl logs -f deployment/bitcoin-trader -n bitcoin
```

## 9. 다음 단계

K8s 매니페스트 작성이 완료되었으면, 다음 단계로 진행한다:

→ [[06-인프라/Sealed Secrets 가이드|Sealed Secrets 가이드]] — API 키를 안전하게 Git에 저장하는 방법
