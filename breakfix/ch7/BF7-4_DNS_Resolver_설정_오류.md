# BF7-4 — DNS Resolver 설정 오류

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `bf4-client` namespace |
| 증상 | `app.rh342.test` 이름 해석 실패 |
| 원인 | Client의 잘못된 Resolver 설정 |
| 진단 | DNS 서버 직접 질의와 시스템 이름 해석 결과 비교 |
```
[node1]

┌─────────────────────┐
│ bf4-client          │
│                     │
│ /etc/netns/         │
│ bf4-client/         │
│ resolv.conf         │
└──────────┬──────────┘
           │
           │ DNS Query : UDP/TCP 53
           ▼
┌─────────────────────┐
│ DNS Server          │
│ 10.70.0.20          │
│                     │
│ app.rh342.test      │
│       ↓             │
│ 10.70.0.20          │
└─────────────────────┘

[장애]
Client Resolver 설정 오류

[정상]
nameserver 10.70.0.20
```
## 2. 문제 현상

`bf4-client`에서 `app.rh342.test`를 이름으로 조회하지 못했다. DNS 서버 자체 문제인지 Client의 Resolver 설정 문제인지 구분할 필요가 있었다.

## 3. 원인 분석

```bash
dig @10.70.0.20 app.rh342.test A
getent hosts app.rh342.test
```

- 지정 서버 `10.70.0.20`에 직접 질의해 DNS 응답 경로를 확인했다.
- `getent hosts`로 시스템이 사용하는 이름 해석 경로를 확인했다.
- 비교 결과, 올바른 DNS 서버를 선택하지 못하는 Client Resolver 설정이 원인이었다.

## 4. 해결 과정

Network namespace의 Resolver 설정 파일을 수정했다.

```text
/etc/netns/bf4-client/resolv.conf
```

```text
nameserver 10.70.0.20
```

수정 후 `dig`와 `getent hosts`로 다시 조회했다.

```text
status: NOERROR
ANSWER: 1
SERVER: 10.70.0.20#53
app.rh342.test.  IN  A  10.70.0.20
```

```text
10.70.0.20  app.rh342.test
```

## 5. 결과

- DNS 질의 `NOERROR`, Answer 1개 확인.
- `getent hosts`에서 `app.rh342.test → 10.70.0.20` 정상 조회.
