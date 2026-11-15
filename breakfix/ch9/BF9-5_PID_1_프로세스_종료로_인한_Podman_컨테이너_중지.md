# BF9-5. PID 1 프로세스 종료로 인한 Podman 컨테이너 중지

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / Podman |
| 증상 | 컨테이너 실행 직후 `Exited` 상태로 전환 |
| 원인 | 컨테이너의 주 프로세스 `/bin/bash` 종료 |
| 해결 | 주 프로세스를 지속 실행되는 `sleep infinity`로 변경 |

## 2. 문제 현상

실행한 컨테이너가 계속 동작하지 않고 종료됐다. `podman inspect`로 확인한 핵심 상태는 다음과 같다.

```text
Path=/bin/bash
Args=["/bin/bash"]
Status=exited
Running=false
ExitCode=0
Error=""
OOMKilled=false
Pid=0
```

## 3. 원인 분석

- `ExitCode=0`은 주 프로세스가 오류 없이 종료했음을 보여준다.
- `OOMKilled=false`이므로 메모리 부족으로 강제 종료된 상황과 구분된다.
- 비대화형으로 실행된 `/bin/bash`가 종료되면서 컨테이너의 생명주기도 끝났다.

## 4. 해결 과정

기존 컨테이너를 제거하고 지속 실행 프로세스를 주 명령으로 지정했다.

```bash
podman rm rh342-bf95
podman run -d --name rh342-bf95 \
  registry.access.redhat.com/ubi8/ubi-minimal:latest sleep infinity
```

셸이 필요하면 실행 중인 컨테이너에 별도로 접근한다.

```bash
podman exec -it rh342-bf95 /bin/bash
```

## 5. 결과

- `podman ps`에서 `sleep infinity` 프로세스 및 `Up` 상태 확인.
- `podman inspect`에서 `Status=running`, `Running=true`, `ExitCode=0` 확인.
