# BF 8-4. File Descriptor 제한으로 인한 서비스 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10, `rh342-app8-4.service` |
| 증상 | 서비스 실행 실패, HTTP 연결 거부 |
| 원인 | systemd의 `LimitNOFILE=32`가 애플리케이션 요구량보다 작음 |
| 진단 | `journalctl`, `systemctl cat/show` |

## 2. 문제 현상

Python 애플리케이션이 `/dev/null` 파일을 48개 여는 과정에서 실패했다.

```text
OSError: [Errno 24] Too many open files
```

```bash
curl -i http://127.0.0.1:18734
```

연결 거부가 발생했고 서비스는 `failed` 상태였다.

## 3. 원인 분석

### ① 오류와 실행 조건 확인

```bash
systemctl status rh342-app8-4
journalctl -u rh342-app8-4
systemctl cat rh342-app8-4
```

서비스 설정에서 다음 제한을 확인했다.

```ini
[Service]
User=nobody
LimitNOFILE=32
```

### ② 적용된 제한 확인

```bash
systemctl show rh342-app8-4 -p LimitNOFILE
```

```text
LimitNOFILE=32
```

48개의 파일 외에도 표준 입출력 등 다른 FD가 필요하다. 따라서 한도 32로는 정상 실행이 불가능했다.

**확인된 원인:** 셸 환경이 아니라 systemd 서비스에 설정된 FD 제한.

## 4. 해결 과정

서비스의 `LimitNOFILE` 한도를 상향 조정했다. 실습 중 제시된 override 구성은 다음과 같다.

```ini
[Service]
LimitNOFILE=128
```

수정 후 서비스가 다시 실행되는지 확인했다.

```bash
systemctl status rh342-app8-4
```

## 5. 결과

- `rh342-app8-4.service`가 `active (running)` 상태로 전환.
- 확인된 메인 PID: `1781`.
