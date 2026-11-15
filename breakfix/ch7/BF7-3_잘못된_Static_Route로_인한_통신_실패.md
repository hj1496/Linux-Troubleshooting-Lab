# BF7-3 — 잘못된 Static Route로 인한 통신 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `bf3-client`–Router–Server namespace |
| 증상 | Router의 Client 측 주소는 도달하지만 다른 서브넷 접속 실패 |
| 원인 | 정상 Default Route보다 우선하는 잘못된 `/24` Static Route |
| 진단 | `ip route`, `ip route get`으로 선택 경로 확인 |

```

[node1 / Rocky Linux 8.10]

┌────────────────────┐
│ bf3-client         │
│ 10.60.1.10         │
│ veth3-cr-c         │
└─────────┬──────────┘
          │
          │ 10.60.1.0/24
          │
┌─────────▼──────────┐
│ Router             │
│ 10.60.1.1          │
└─────────┬──────────┘
          │
          │ 10.60.2.0/24
          │
┌─────────▼──────────┐
│ Server             │
│ 10.60.2.20         │
└────────────────────┘

[장애]
Client Static Route
10.60.2.0/24 via 10.60.1.254  [잘못된 Next Hop]

[정상]
10.60.2.20 via 10.60.1.1

```


## 2. 문제 현상

Client에서 Router의 `10.60.1.1`은 도달했지만 반대편 네트워크 `10.60.2.0/24`로 통신할 수 없었다.

## 3. 원인 분석

`bf3-client` 라우팅 테이블의 핵심 출력:

```text
default via 10.60.1.1
10.60.2.0/24 via 10.60.1.254 dev veth3-cr-c
```

- 목적지 `10.60.2.20`에는 Default Route보다 Prefix가 긴 `/24` 경로가 우선 적용된다.
- 따라서 정상 Next Hop `10.60.1.1` 대신 잘못된 `10.60.1.254`가 선택되었다.

## 4. 해결 과정

잘못된 정적 경로를 제거했다.

```bash
ip -n bf3-client route del 10.60.2.0/24 via 10.60.1.254
```

수정 후 경로 조회:

```bash
ip netns exec bf3-client ip route get 10.60.2.20
```

```text
10.60.2.20 via 10.60.1.1 dev veth3-cr-c src 10.60.1.10 uid 0
    cache
```

## 5. 결과

- `10.60.2.20`의 Next Hop이 정상 게이트웨이 `10.60.1.1`로 선택되는 것을 확인했다.
