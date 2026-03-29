# 네트워크 아키텍처 및 Ingress 가이드

> 최종 수정: 2026-03-08

---

## 1. 물리 네트워크 구성

```
인터넷
  │
  ▼
┌──────────────────────────────────┐
│  iptime 공유기                    │
│  포트포워딩: 80/443 → Synology    │
└──────────┬───────────────────────┘
           │ (LAN)
     ┌─────┴──────────────────────────────────┐
     │                                         │
     ▼                                         ▼
┌─────────────────────┐         ┌──────────────────────────────┐
│  Synology NAS        │         │  han-nuc (미니 PC)             │
│  192.168.x.x         │         │  192.168.2.101 (K3s)          │
│                      │         │                               │
│  - DSM (80/443)      │  ──→   │  - Traefik (LB: 80/443)       │
│  - DDNS              │         │    NodePort: 32688(HTTP)      │
│  - 리버스 프록시       │         │              31863(HTTPS)     │
│  - Let's Encrypt     │         │  - 각종 K8s 서비스              │
└──────────────────────┘         └──────────────────────────────┘
```

## 2. 도메인 체계

### 서비스용 (Path 분기)

| URL | 대상 | 네임스페이스 |
|-----|------|------------|
| `cozyhan.liooos.synology.me/` | 소개 페이지 | (미정) |
| `cozyhan.liooos.synology.me/coin` | bitcoin-trader | bitcoin |
| `cozyhan.liooos.synology.me/blog` | 블로그 | (미정) |

### DevOps용 (서브도메인 분기)

| URL | 대상 | 네임스페이스 |
|-----|------|------------|
| `dev-ci.liooos.synology.me` | Woodpecker CI | woodpecker |
| `dev-cd.liooos.synology.me` | ArgoCD | argocd |
| `dev-db.liooos.synology.me` (포트 기반) | PostgreSQL | database |

### 실제 도메인 구매 시

```
DNS (와일드카드 A 레코드)
  *.aaa.co.kr  →  집 공인 IP
  aaa.co.kr    →  집 공인 IP

서비스: cozyhan.aaa.co.kr/coin, /blog, ...
DevOps: dev-ci.aaa.co.kr, dev-cd.aaa.co.kr
```

> DNS는 IP 안내만 담당. 실제 라우팅은 Ingress(또는 Synology 리버스 프록시)에서 처리.

## 3. 트래픽 라우팅 구조 (방법 2: 이중 프록시)

DSM이 80/443을 사용 중이므로, Synology 리버스 프록시를 유지하면서 Ingress를 추가하는 방식.

```
외부 HTTPS 요청
  │
  ▼
iptime (80/443 → Synology)
  │
  ▼
Synology 리버스 프록시
  │
  ├─ DSM 관련 도메인 → Synology 자체 처리
  │
  └─ K8s 관련 도메인 → han-nuc:80 (Traefik LB IP: 192.168.2.101)
     │
     ▼
  Traefik Ingress Controller (K3s 내장)
     │
     ├─ Host: cozyhan.xxx
     │    ├─ /coin  → bitcoin-trader Service (bitcoin NS)
     │    ├─ /blog  → blog Service (blog NS)
     │    └─ /      → intro-page Service
     │
     ├─ Host: dev-ci.xxx → woodpecker-server Service (woodpecker NS)
     │
     └─ Host: dev-cd.xxx → argocd-server Service (argocd NS)
```

### Synology 리버스 프록시 설정

모든 K8s 도메인은 **같은 대상**으로 설정. Traefik이 Host 헤더로 분기.

| 소스 도메인 | 대상 | 비고 |
|------------|------|------|
| `cozyhan.liooos.synology.me` | `http://192.168.2.101:80` | Traefik LB |
| `dev-ci.liooos.synology.me` | `http://192.168.2.101:80` | Traefik LB |
| `dev-cd.liooos.synology.me` | `http://192.168.2.101:80` | Traefik LB |

> 사용자 지정 헤더에 `Host: $host` 설정 필수 (원본 도메인을 Traefik에 전달).

## 4. Ingress vs 기존 방식 비교

| 기준 | Synology만 사용 (이전) | Synology + Ingress (현재) |
|------|---------------------|-------------------------|
| K8s 서비스 추가 시 | Synology에서 수동 설정 | Ingress 매니페스트만 추가 |
| GitOps 관리 | 불가 | 가능 (매니페스트 레포) |
| DSM 영향 | 없음 | 없음 |
| 복잡도 | 낮음 | 중간 (프록시 2단) |

## 5. K8s Ingress 개념

### Ingress란?

외부 HTTP(S) 트래픽을 K8s 내부 Service로 라우팅하는 **규칙 선언**.
실제 처리는 Ingress Controller(K3s에서는 Traefik)가 담당.

### 라우팅 방식

#### 서브도메인 분기 (Host 기반)

```yaml
spec:
  rules:
  - host: dev-ci.xxx      # Host 헤더로 분기
    http:
      paths:
      - path: /
        backend:
          service:
            name: woodpecker-server
```

#### Path 분기

```yaml
spec:
  rules:
  - host: cozyhan.xxx
    http:
      paths:
      - path: /coin        # URL 경로로 분기
        pathType: Prefix
        backend:
          service:
            name: bitcoin-trader
      - path: /blog
        pathType: Prefix
        backend:
          service:
            name: blog-service
```

### pathType 종류

| pathType | 동작 | 예시 |
|----------|------|------|
| `Prefix` | 해당 경로로 시작하는 모든 요청 매칭 | `/coin` → `/coin`, `/coin/api/v1` 모두 매칭 |
| `Exact` | 정확히 일치하는 요청만 매칭 | `/coin`만 매칭 |

### 네임스페이스와 Ingress

Ingress는 **같은 네임스페이스의 Service만 참조** 가능.
다른 네임스페이스의 서비스를 같은 Host로 라우팅하려면 **네임스페이스별로 Ingress를 분리** 생성.

```
bitcoin NS:    Ingress (host: cozyhan.xxx, path: /coin → bitcoin-trader)
blog NS:       Ingress (host: cozyhan.xxx, path: /blog → blog-service)
```

> Traefik이 같은 Host의 여러 Ingress를 **자동 병합**하여 하나의 라우팅 테이블로 처리.

### stripPrefix 미들웨어

앱이 `/coin` prefix를 인식하지 못하면 Traefik Middleware로 제거:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: strip-coin
  namespace: bitcoin
spec:
  stripPrefix:
    prefixes:
      - /coin
```

`/coin/api/health` 요청 → bitcoin-trader에는 `/api/health`로 전달.

## 6. PostgreSQL 외부 접근 (TCP)

DB는 HTTP가 아닌 **TCP 프로토콜**이므로 Ingress(L7)로 처리 불가.
**NodePort + 포트포워딩**으로 노출.

```
외부 psql 클라이언트
  │
  ▼
iptime (5432 → Synology 또는 han-nuc 직접)
  │
  ▼
han-nuc:30432 (NodePort)
  │
  ▼
PostgreSQL Pod (database NS)
```

### NodePort Service 생성

```yaml
# postgres-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-external
  namespace: database
spec:
  type: NodePort
  selector:
    app: postgres          # 기존 Pod label에 맞춰야 함
  ports:
  - port: 5432
    targetPort: 5432
    nodePort: 30432
```

### 보안 주의사항

- `pg_hba.conf`에서 접속 IP 제한
- 강력한 비밀번호 필수
- VPN(WireGuard 등) 경유 접속 권장
- 불필요 시 NodePort 삭제하여 접근 차단

## 7. 실제 도메인 전환 시 체크리스트

도메인 구매 후 전환 순서:

1. **DNS 설정**: 와일드카드 A 레코드 (`*.aaa.co.kr → 공인 IP`)
2. **Ingress Host 변경**: `liooos.synology.me` → `aaa.co.kr`
3. **Synology 리버스 프록시 업데이트**: 소스 도메인 변경
4. **Woodpecker WOODPECKER_HOST 변경**: Helm values 업데이트
5. **TLS 인증서**: Synology Let's Encrypt 또는 cert-manager 설정

### 향후 Ingress 단독 전환 (방법 1)

서비스가 충분히 늘어나면 Synology를 제거하고 Ingress를 단일 진입점으로 전환 가능:

1. Synology DSM 포트를 5000/5001로 변경
2. iptime 80/443 포트포워딩을 han-nuc으로 변경
3. cert-manager + Let's Encrypt로 TLS 처리
4. Synology 리버스 프록시 비활성화

## 8. 관련 문서

- [[06-인프라/인프라 아키텍처 개요|인프라 아키텍처 개요]] — 전체 인프라 구성
- [[06-인프라/K3s 설치 가이드|K3s 설치 가이드]] — Traefik 포함
- [[06-인프라/Woodpecker CI 설치 및 설정 가이드|Woodpecker CI 가이드]] — Synology 리버스 프록시 설정 포함
