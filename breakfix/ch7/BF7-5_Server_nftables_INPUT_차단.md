# BF7-5 — Server nftables INPUT 차단

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `bf5-client` → `bf5-server` |
| 목적지 | `10.80.0.20:8080` |
| 증상 | 서비스는 LISTEN 중이지만 Client HTTP 접속 실패 |
| 진단 | `tcpdump`로 SYN 도착 확인 후 Server nftables INPUT 규칙 조사 |
```
[node1]

┌──────────────────┐
│ bf5-client       │
└────────┬─────────┘
         │
         │ TCP SYN :8080
         ▼
┌─────────────────────────────┐
│ bf5-server                  │
│ 10.80.0.20                  │
│                             │
│ Network Interface           │
│         │                   │
│         ▼                   │
│ nftables INPUT              │
│ tcp dport 8080 DROP  [X]    │
│         │                   │
│         ▼                   │
│ HTTP Server :8080           │
└─────────────────────────────┘

[장애]
SYN 도착 확인
SYN-ACK 없음

[정상]
INPUT DROP 규칙 제거
HTTP 응답 : RH342 BF5 OK
```
## 2. 문제 현상

Server의 TCP 8080 서비스는 Listening 중이었지만 Client 요청에 응답하지 않았다.

패킷 관찰 결과, Client SYN은 Server 인터페이스까지 도착했지만 SYN-ACK가 보이지 않았다.

## 3. 원인 분석

Server nftables에서 확인된 차단 규칙의 핵심:

```text
table inet bf5filter
chain input
    tcp dport 8080 drop
```

- 요청 패킷이 Server에 도착했으므로 앞단 라우팅 단절로 보기 어려웠다.
- INPUT 체인의 TCP 8080 DROP 규칙으로 인해 로컬 서비스로 패킷이 전달되지 않았다.

## 4. 해결 과정

`bf5filter`의 차단 규칙을 제거한 뒤 실제로 다음을 실행해 확인했다.

```bash
ip netns exec bf5-server nft -a list chain inet bf5filter input
ip netns exec bf5-client curl --connect-timeout 3 http://10.80.0.20:8080
```

- 첫 번째 명령 출력에서는 기존 DROP 규칙이 더 이상 존재하지 않았다.
- 두 번째 명령의 HTTP 응답 본문:

```text
RH342 BF5 OK
```

## 5. 결과

- INPUT 체인의 TCP 8080 DROP 규칙 제거 확인.
- Client HTTP 요청 정상 응답: `RH342 BF5 OK`.
