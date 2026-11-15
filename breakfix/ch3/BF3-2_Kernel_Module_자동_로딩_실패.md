# 3-2. Kernel Module 자동 로딩 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / VirtualBox |
| 대상 | `dummy` 커널 모듈 |
| 증상 | 수동 로딩 시 인터페이스가 생성되지만, 재부팅 후 모듈이 로드되지 않음 |
| 원인 | 부팅 시 자동 로딩 설정 누락 |
| 진단 전략 | 모듈 설치 여부 → 수동 로딩 → 부팅 설정 순서로 확인 |

## 2. 문제 현상

재부팅 후 `dummy` 모듈과 `dummy0`, `dummy1` 인터페이스가 보이지 않았다.

```bash
lsmod | grep '^dummy'
ip -br link | grep dummy
```

두 명령에서 해당 모듈 및 인터페이스를 확인할 수 없었다.

## 3. 원인 분석

**① 모듈 파일 존재 여부 확인**

```bash
modinfo dummy
```

- 커널 모듈 파일(`dummy.ko.xz`)이 존재했다.
- 따라서 모듈 파일 누락보다는 로딩 과정의 문제로 범위를 좁혔다.

**② 수동 로딩으로 동작 확인**

```bash
modprobe dummy
lsmod | grep '^dummy'
ip -br link | grep dummy
```

- 수동 로딩에 성공했고 `dummy0`, `dummy1`이 생성되었다.
- 모듈 자체는 정상 동작하므로 부팅 시 로딩 설정을 조사했다.

**③ 자동 로딩 설정 확인**

`/etc/modules-load.d/`에 `dummy`를 자동 로딩하도록 지정한 설정 파일이 없었다.

**원인:** 모듈은 사용 가능하지만 부팅 시 자동으로 로드하도록 등록되지 않았다.

## 4. 해결 과정

`/etc/modules-load.d/rh342-dummy.conf` 파일을 생성하고 모듈 이름을 등록했다.

```ini
dummy
```

이 설정은 부팅 과정에서 `dummy` 모듈을 로딩하기 위한 것이다. 모듈의 `numdummies` 파라미터 설정과는 역할이 다르다.

## 5. 결과

- 재부팅 후 `dummy` 모듈이 자동 로드됨을 확인했다.
- `dummy0`, `dummy1` 두 인터페이스가 생성된 상태를 확인했다.
