# BF 1-2. systemd 서비스 반복 실패 — 설정 파일 누락

## 1. 개요

| 항목    | 내용                                        |
| ----- | ----------------------------------------- |
| 환경    | Rocky Linux 8.10 (`node1`)                |
| 증상    | rh342-monitor.service가 실행과 실패를 반복         |
| 진단 전략 | 서비스 상태 → journal 로그 → `ExecStart` 스크립트 확인 |

## 2. 문제 현상

`rh342-monitor.service`가 `active`와 `failed` 상태를 반복했고, 재시작 횟수가 258회까지 증가했다.

```text
ExecStart=/run/rh342-monitor
status=1
Error: cannot load required configuration
```

## 3. 원인 분석

### ① 서비스 실패 원인 확인

```bash
systemctl status rh342-monitor.service
journalctl -u rh342-monitor.service
```

- 실행 파일 `/run/rh342-monitor`가 종료 코드 `1`을 반환했다.
- journal에 필수 설정을 불러올 수 없다는 오류를 발견했다.
- 서비스의 재시작 간격은 `3s`였으며, 실패 후 재시작을 반복했다.

### ② 실행 스크립트 확인

```bash
cat /run/rh342-monitor
```

스크립트는 `/run/rh342/monitor.conf` 의 읽기 가능 여부를 검사했고, 읽을 수 없으면 exit 1로 종료하도록 작성되어 있었다.

확인된 원인: `/run/rh342/monitor.conf` 파일이 없어 서비스 실행이 실패했다.

## 4. 해결 과정

```bash
mkdir -p /run/rh342
touch /run/rh342/monitor.conf
```

스크립트가 요구하는 경로와 설정 파일을 생성했다. 

## 5. 결과

- 누락된 `/run/rh342/monitor.conf`를 생성했다.
- `rh342-monitor.service`의 반복 실패 및 자동 재시작이 중단된 것을 확인했다.




