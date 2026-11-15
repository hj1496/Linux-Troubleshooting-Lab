# BF7-9 — tcpdump로 Router FORWARD 차단 지점 특정

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `bf9-client` → `bf9-router` → `bf9-server` |
| 목적지 | `10.120.2.20:8080` |
| 원인 | Router nftables FORWARD 체인의 TCP 8080 DROP |
| 진단 | Router ingress·egress 패킷 캡처 비교 |
```
[node1]

┌───────────────────────────┐
│ bf9-client                │
│ 10.120.1.10               │
└────────────┬──────────────┘
             │
             │ TCP SYN :8080
             ▼
┌───────────────────────────┐
│ bf9-router                │
│                           │
│ bf9-r1 : 10.120.1.1       │
│    │                      │
│    ▼                      │
│ nftables FORWARD          │
│ tcp dport 8080 DROP [X]   │
│    │                      │
│    ▼                      │
│ bf9-r2 : 10.120.2.1       │
└────────────┬──────────────┘
             │
             │ TCP SYN 미도착
             ▼
┌───────────────────────────┐
│ bf9-server                │
│ 10.120.2.20:8080          │
└───────────────────────────┘

[패킷 캡처]
bf9-r1 : SYN 확인
bf9-r2 : SYN 미확인

[원인]
Router FORWARD 체인의 TCP 8080 DROP
```
## 2. 문제 현상

Client의 TCP 8080 연결이 Server에 도달하지 못했다. Client가 보낸 SYN은 Router의 Client 측 인터페이스에서 보이지만 Server 측에서는 보이지 않았다.

## 3. 원인 분석

통신 경로:

```text
bf9-client 10.120.1.10
    → bf9-router: bf9-r1(10.120.1.1), bf9-r2(10.120.2.1)
    → bf9-server 10.120.2.20:8080
```

| 패킷 캡처 지점 | TCP SYN |
|---|---|
| Router ingress `bf9-r1` | 관찰됨 |
| Router egress `bf9-r2` | 관찰되지 않음 |

Router에서 `ip_forward=1`을 확인했다. 이어 nftables에서 다음 규칙을 확인했다.

```text
table inet bf9filter
chain forward
    policy accept
    tcp dport 8080 drop
```

FORWARD 체인의 명시적 DROP이 `policy accept`보다 먼저 패킷에 적용된다.

**원인:** Router의 FORWARD 단계에서 TCP 8080 패킷이 폐기됨.

## 4. 해결 과정

당시 제시된 조치 절차는 FORWARD 체인에서 실제 DROP 규칙의 handle을 조회하고, 그 규칙만 삭제하는 방식이었다.

```bash
ip netns exec bf9-router nft -a list chain inet bf9filter forward
```

```bash
# 위 출력에서 확인한 실제 숫자 handle을 사용
ip netns exec bf9-router nft delete rule inet bf9filter forward handle <HANDLE>
```

해당 삭제 명령은 원인에 맞춰 제시된 조치다. `<HANDLE>`은 실제 값이 아닌 자리표시자이며, 기록에 없는 실행 번호나 HTTP 결과를 만들어 넣지 않았다.

## 5. 결과

- Router ingress·egress 캡처로 패킷 소실 구간을 좁히고, `FORWARD`의 `tcp dport 8080 drop` 규칙을 원인으로 특정해 조치 대상으로 확인했다.
