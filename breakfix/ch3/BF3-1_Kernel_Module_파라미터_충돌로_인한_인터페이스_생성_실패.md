# BF 3-1. Kernel Module 파라미터 충돌로 인한 인터페이스 생성 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 |
| 대상 | `dummy` Kernel Module |
| 증상 | 모듈은 로드되지만 `dummy0`, `dummy1` 인터페이스가 생성되지 않음 |
| 원인 | 서로 다른 `modprobe.d` 설정 파일에서 `numdummies` 중복 지정 |
| 목표 | 인터페이스 2개 생성 및 설정 충돌 해결 |

## 2. 문제 현상

`dummy` 모듈을 다시 로드했지만 네트워크 인터페이스가 생성되지 않았다.

```bash
modprobe -r dummy
modprobe dummy
lsmod | grep dummy
ip -br link show type dummy
```

```text
dummy                  16384  0
# dummy 인터페이스 출력 없음
```

## 3. 원인 분석

### ① 모듈 설정 중복 확인

`/etc/modprobe.d/rh342.conf`에 `numdummies=2`를 설정했지만 인터페이스는 생성되지 않았다.

```bash
modprobe -c | grep '^options dummy'
```

```text
options dummy numdummies=2
options dummy numdummies=0
```

설정 출처는 다음과 같았다.

| 파일 | 설정 |
|---|---|
| `/etc/modprobe.d/rh342.conf` | `options dummy numdummies=2` |
| `/usr/lib/modprobe.d/systemd.conf` | `options dummy numdummies=0` |

### ② 실제 모듈 로드 옵션 확인

```bash
modprobe -r dummy
modprobe -n -v dummy
```

```text
insmod /lib/modules/4.18.0-553.134.1.el8_10.x86_64/kernel/drivers/net/dummy.ko.xz numdummies=2 numdummies=0
```

서로 다른 파일명의 설정이 파일명 순서대로 처리되면서 `numdummies=0`이 마지막에 전달되는 것을 확인했다. 이 때문에 모듈은 로드되어도 인터페이스가 생성되지 않았다.

## 4. 해결 과정

패키지 관리 파일인 `/usr/lib/modprobe.d/systemd.conf`는 변경하지 않고, 사용자 설정 파일명을 정렬 순서상 뒤로 이동했다.

```bash
mv /etc/modprobe.d/rh342.conf \
   /etc/modprobe.d/zz-rh342-dummy.conf

modprobe -n -v dummy
```

```text
insmod /lib/modules/4.18.0-553.134.1.el8_10.x86_64/kernel/drivers/net/dummy.ko.xz numdummies=0 numdummies=2
```

`numdummies=2`가 마지막에 전달되는 것을 확인하고 모듈을 로드했다.

```bash
modprobe dummy
ip -br link show type dummy
```

## 5. 결과

- 모듈 로드 옵션 순서가 `numdummies=0 numdummies=2`로 변경됨.
- `dummy0`, `dummy1` 인터페이스 생성 확인.

```text
dummy0           DOWN           ea:81:67:f1:32:ff <BROADCAST,NOARP>
dummy1           DOWN           ae:4b:89:a9:e1:aa <BROADCAST,NOARP>
```
