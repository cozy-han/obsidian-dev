# Upbit API IP 화이트리스트 설정 가이드

## 왜 필요한가?

Upbit Open API는 IP 화이트리스트 설정 시:
- **API Key 도용 방지** — 등록된 IP에서만 API 호출 가능
- **보안 강화** — Key가 유출되어도 등록되지 않은 IP에서는 사용 불가
- **필수 권한** — 출금 API 사용 시 IP 화이트리스트 필수 (현재 프로젝트에서는 미사용)

> Upbit에서 IP 제한 없이 API Key를 발급하면 경고가 표시됨

---

## 설정 방법

### 1. 외부 IP 확인

#### K3s Pod에서 확인
```bash
# Pod 내부에서 외부 IP 확인
kubectl exec -it deploy/bitcoin-trader -n bitcoin -- curl -s ifconfig.me

# 또는 호스트에서 직접 확인
curl -s ifconfig.me
```

#### Synology NAS에서 확인
```bash
# SSH 접속 후
curl -s ifconfig.me

# 또는 제어판 > 외부 액세스 > DDNS 에서 확인
```

> K3s Pod의 외부 트래픽은 호스트 노드의 IP로 NAT되므로, **호스트의 외부 IP = Pod의 외부 IP**

### 2. Upbit 개발자 센터에서 등록

1. [Upbit 마이페이지 > Open API 관리](https://upbit.com/mypage/open_api_management) 접속
2. 사용 중인 API Key 선택
3. **허용 IP 주소** 항목에 확인한 외부 IP 입력
4. 저장

### 3. 설정 후 검증

```bash
# API 호출 테스트 (계좌 조회)
curl -H "Authorization: Bearer <JWT_TOKEN>" \
  https://api.upbit.com/v1/accounts

# 정상 응답: 200 + 계좌 정보 JSON
# IP 미등록: 401 또는 403 에러
```

애플리케이션에서 검증:
```bash
# K3s Pod 로그 확인
kubectl logs deploy/bitcoin-trader -n bitcoin | grep "Upbit API"
```

---

## 동적 IP 환경 대응

### 문제
- 가정용 인터넷은 ISP가 주기적으로 IP를 변경할 수 있음
- IP 변경 시 Upbit API 호출이 차단됨

### 대응 방안

#### 방법 1: 고정 IP 신청 (권장)
- ISP에 고정 IP 요청 (월 추가 비용 발생)
- 가장 안정적인 방법

#### 방법 2: IP 변경 모니터링
```bash
# crontab에 IP 변경 감지 스크립트 등록
*/5 * * * * /path/to/check-ip.sh
```

```bash
#!/bin/bash
# check-ip.sh
CURRENT_IP=$(curl -s ifconfig.me)
SAVED_IP=$(cat /tmp/last_known_ip 2>/dev/null)

if [ "$CURRENT_IP" != "$SAVED_IP" ]; then
    echo "$CURRENT_IP" > /tmp/last_known_ip
    # Discord 웹훅으로 알림
    curl -H "Content-Type: application/json" \
         -d "{\"content\":\"외부 IP 변경 감지: $SAVED_IP → $CURRENT_IP. Upbit API 화이트리스트 업데이트 필요!\"}" \
         "$DISCORD_WEBHOOK_URL"
fi
```

#### 방법 3: DDNS 주의사항
- DDNS는 도메인 → IP 매핑이지, Upbit 화이트리스트는 **IP 주소만 허용**
- DDNS 사용 시에도 실제 IP가 변경되면 화이트리스트 수동 업데이트 필요

---

## 트러블슈팅

### 403 Forbidden / 401 Unauthorized 발생 시

1. **외부 IP 확인**
   ```bash
   kubectl exec -it deploy/bitcoin-trader -n bitcoin -- curl -s ifconfig.me
   ```

2. **Upbit 화이트리스트 확인**
   - 마이페이지 > Open API 관리에서 등록된 IP 확인
   - 확인한 IP와 일치하는지 비교

3. **API Key 상태 확인**
   - Key가 만료되지 않았는지 확인
   - 필요한 권한(자산조회, 주문 등)이 활성화되어 있는지 확인

4. **네트워크 경로 확인**
   - VPN 사용 시 VPN 서버 IP가 등록되어야 함
   - 프록시/NAT 환경에서는 최종 출구 IP 확인

### 흔한 실수
- IPv4/IPv6 혼동 — Upbit는 IPv4만 지원
- 공유기 재부팅 후 IP 변경 — ISP에 따라 재부팅 시 IP 변경 가능
- 여러 IP 등록 누락 — 개발 환경과 운영 환경 IP 모두 등록 필요
