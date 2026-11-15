# BF7-7 — Loopback Bind로 인한 TCP 9090 접속 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `bf7-server` namespace |
| 목적지 | `10.100.0.20:9090` |
| 원인 | HTTP 서비스가 `127.0.0.1`에만 Bind |
| 진단 | `ss -lntp`로 Listening IP 확인 |
```
[node1]

┌──────────────────────┐
│ External Client      │
└──────────┬───────────┘
           │
           │ TCP 10.100.0.20:9090
           ▼
┌──────────────────────────────┐
│ bf7-server                   │
│ 10.100.0.20                  │
│                              │
│ eth/veth Interface           │
│          │                   │
│          X Connection Failed │
│                              │
│ HTTP Server                  │
│ 127.0.0.1:9090 [LISTEN]      │
└──────────────────────────────┘

[장애]
Loopback 주소에만 Bind

[정상]
10.100.0.20:9090 [LISTEN]
HTTP 응답 : RH342 BF7 PORT 9090 OK
```
## 2. 문제 현상

TCP 9090에서 HTTP 서비스가 실행되고 있었지만, 외부 Client가 `10.100.0.20:9090`에 접속하지 못했다.

## 3. 원인 분석

```bash
ip netns exec bf7-server ss -lntp
```

- 서비스 Listening 주소는 `127.0.0.1:9090`이었다.
- Loopback 전용 Bind 때문에 Server의 `10.100.0.20:9090` 주소로 접속할 수 없었다.

## 4. 해결 과정

기존 서비스를 종료한 뒤 HTTP 서비스를 Server IP에 명시적으로 Bind해 실행했다.

```bash
python3 -m http.server 9090 --bind 10.100.0.20 --directory /tmp/bf7-www-alt
```

실제 변경 후 `ss -lntp`의 핵심 출력:

```text
10.100.0.20:9090 LISTEN
```

Client의 HTTP 요청 응답:

```text
RH342 BF7 PORT 9090 OK
```

## 5. 결과

- `10.100.0.20:9090` Listening 확인.
- Client HTTP 요청 정상 응답: `RH342 BF7 PORT 9090 OK`.
