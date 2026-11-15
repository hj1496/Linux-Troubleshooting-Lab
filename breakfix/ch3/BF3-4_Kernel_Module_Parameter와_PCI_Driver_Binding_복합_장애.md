# 3-4. Kernel Module Parameter와 PCI Driver Binding 복합 장애

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / VirtualBox |
| 대상 | `dummy` 모듈, PCI `0000:00:08.0`, `e1000` 드라이버 |
| 증상 | `dummy1`과 `enp0s8`이 동시에 누락됨 |
| 원인 | `numdummies=1` 설정과 PCI 드라이버 바인딩 해제 |
| 진단 전략 | 가상 인터페이스 생성 수와 물리 장치 드라이버 연결 상태를 별도로 확인 |

## 2. 문제 현상

```bash
ip -br link
```

- `dummy0`은 존재했지만 `dummy1`이 보이지 않았다.
- 기존 PCI 네트워크 인터페이스인 `enp0s8`도 보이지 않았다.

서로 다른 유형의 인터페이스가 동시에 사라져 원인을 분리해 조사했다.

## 3. 원인 분석

**① `dummy1` 누락 — 모듈 파라미터 확인**

```bash
lsmod | grep '^dummy'
modprobe -c | grep -E '^options[[:space:]]+dummy'
```

- `dummy` 모듈은 로드되어 있었다.
- 설정 조사에서 `numdummies=0`과 `numdummies=1` 항목을 확인했다.
- `/etc/modprobe.d/zz-rh342-dummy.conf`의 설정값이 `numdummies=1`이었고, 실제로 `dummy0`만 생성되었다.
- `/usr/lib/modprobe.d/systemd.conf`에도 `numdummies=0` 설정이 존재해 설정 출처를 함께 확인했다.

**② `enp0s8` 누락 — PCI 바인딩 확인**

```bash
lspci -k
lsmod | grep '^e1000'
ls /sys/bus/pci/drivers/e1000/
```

- PCI `00:08.0` 장치는 존재했다.
- `e1000` 모듈 역시 로드되어 있었다.
- 그러나 `lspci -k`에 해당 장치의 `Kernel driver in use`가 표시되지 않았고, `e1000` 드라이버 디렉터리에도 `0000:00:08.0`이 없었다.

**원인:** `dummy1`은 모듈 파라미터 설정 문제, `enp0s8`은 PCI 드라이버 바인딩 문제였다.

## 4. 해결 과정

**① `dummy` 인터페이스 생성 수 복구**

`/etc/modprobe.d/zz-rh342-dummy.conf`의 설정을 수정했다.

```ini
options dummy numdummies=2
```

변경 내용을 적용하기 위해 `dummy` 모듈을 다시 로드했다.

```bash
modprobe -r dummy
modprobe dummy
```

**② PCI 드라이버 바인딩 복구**

```bash
echo '0000:00:08.0' > /sys/bus/pci/drivers/e1000/bind
```

두 인터페이스가 복구된 뒤 재부팅하여 상태가 유지되는지 확인했다.

## 5. 결과

- 재부팅 후 `dummy0`, `dummy1` 두 인터페이스가 존재함을 확인했다.
- `enp0s8`이 다시 생성되었고, PCI `00:08.0`의 `Kernel driver in use: e1000`을 확인했다.
