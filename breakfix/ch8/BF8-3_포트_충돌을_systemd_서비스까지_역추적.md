# BF 8-3. 포트 충돌을 systemd 서비스까지 역추적

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10, `rh342-app8-3.service` |
| 증상 | 애플리케이션 바인딩 실패 (`Errno 98`) |
| 원인 | 다른 systemd 서비스가 `127.0.0.1:18733` 선점 |
| 진단 | `ss` → `lsof` → `/proc/PID/cgroup` |

## 2. 문제 현상

애플리케이션이 TCP 포트에 바인딩하지 못했다.

```text
OSError: [Errno 98] Address already in use
```

## 3. 원인 분석

### ① 포트 점유 확인

```bash
ss -lntp | grep 18733
lsof -iTCP:18733 -sTCP:LISTEN -nP
```

```text
127.0.0.1:18733  LISTEN
python3  PID 12796
```

PID `12796`이 포트를 점유하고 있었다.

### ② 점유 프로세스 확인

```bash
ps -fp 12796
cat /proc/12796/cgroup
```

프로세스 실행 명령:

```text
/bin/python3 -u -m http.server 18733 --bind 127.0.0.1
```

`/proc/12796/cgroup`에서 프로세스가 `rh342-archive8-3.service`에 속해 있음을 확인했다.

### ③ 서비스 구성 비교

```bash
systemctl status rh342-archive8-3
systemctl cat rh342-archive8-3
```

기존 서비스가 `active/enabled`였고 동일한 포트를 사용하고 있었다.

**확인된 원인:** `rh342-archive8-3.service`가 목표 서비스보다 먼저 포트를 점유함.

## 4. 해결 과정

포트를 선점한 기존 서비스를 중지하고 자동 시작을 해제했다.

```bash
systemctl stop rh342-archive8-3
systemctl disable rh342-archive8-3
```

이후 목표 애플리케이션의 서비스 상태와 HTTP 응답을 검증했다.

```bash
systemctl status rh342-app8-3
curl -i http://127.0.0.1:18733
```

## 5. 결과

- `rh342-app8-3.service`: `active (running)`.
- HTTP 응답: `HTTP/1.0 200 OK`.
- 응답 본문: `RH342 app8-3 READY`.
