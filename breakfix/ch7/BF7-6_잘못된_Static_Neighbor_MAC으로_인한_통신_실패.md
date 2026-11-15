# BF7-6 — 잘못된 Static Neighbor MAC으로 인한 통신 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `bf6-client`, `bf6-server` |
| 목적지 | `10.90.0.20` |
| 원인 | 실제 Server MAC과 다른 정적 Neighbor 매핑 |
| 진단 | `ip neigh`와 `tcpdump -nn -e`로 L2 주소 비교 |
```
[node1]

┌────────────────────────┐
│ bf6-client             │
│ bf6-veth-c             │
│                        │
│ Neighbor Table         │
│ 10.90.0.20             │
│       ↓                │
│ 02:00:00:00:00:66 [X]  │
└───────────┬────────────┘
            │
            │ Ethernet Frame
            │ Wrong Destination MAC
            ▼
┌────────────────────────┐
│ bf6-server             │
│                        │
│ IP  : 10.90.0.20       │
│ MAC : 02:00:00:00:00:20│
└────────────────────────┘

[장애]
잘못된 정적 Neighbor 엔트리 (PERMANENT)

[해결]
ip neigh del → 정상 MAC
```
## 2. 문제 현상

Client에서 Server IP로 통신할 때, 목적지 IP에 연결된 MAC 주소가 실제 Server MAC과 달랐다.

## 3. 원인 분석

Client의 Neighbor 엔트리:

```text
10.90.0.20 lladdr 02:00:00:00:00:66 PERMANENT
```

실제 Server MAC:

```text
02:00:00:00:00:20
```

`tcpdump -nn -e`에서 목적지 MAC이 잘못된 `02:00:00:00:00:66`으로 지정된 Ethernet 프레임을 확인했다.

`PERMANENT` 정적 항목이 실제 MAC과 일치하지 않아 패킷이 정상 목적지로 전달되지 않았다.

## 4. 해결 과정

당시 확인한 정적 Neighbor 엔트리 제거 명령:

```bash
ip -n bf6-client neigh del 10.90.0.20 dev bf6-veth-c
```

잘못 고정된 IP–MAC 매핑을 삭제해 정상 MAC을 다시 학습하도록 하는 조치다. 위 명령은 실습 대화에서 제시된 해결 명령이며, 실제 삭제 후 터미널 출력은 인용하지 않았다.

## 5. 결과

- 장애 원인을 `10.90.0.20 → 02:00:00:00:00:66`의 잘못된 정적 Neighbor 매핑으로 특정하고 제거 조치로 실습을 마무리했다.
