# BF 8-6. RSS 증가와 애플리케이션 반복 재시작

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10, `rh342-app8-6.service` |
| 증상 | RSS 증가 후 프로세스 종료·systemd 재시작 반복 |
| 원인 | 버퍼를 계속 보관하도록 설정된 `retained` 정책 |
| 진단 | 메모리 사용량, journal, 재시작 횟수, 환경 설정 |

## 2. 문제 현상

프로세스 메모리 사용량이 증가하다가 RSS가 약 88 MiB를 넘으면 종료 코드 `1`로 종료되었다.

- systemd가 종료된 프로세스를 재시작했다.
- 관찰된 PID 변화: `2115 → 2120 → 2124 → 2126`.
- `MemoryCurrent` 증가와 재시작 횟수 `NRestarts=8`을 확인했다.

## 3. 원인 분석

### ① 서비스 재시작 설정 확인

서비스 상태 및 설정에서 다음 값을 확인했다.

```ini
Restart=on-failure
RestartSec=3
MemoryMax=infinity
```

메모리 사용량이 늘었다는 사실만으로 systemd의 메모리 제한에 의한 강제 종료라고 판단할 수 없다. 이 사례에서는 애플리케이션이 자체 조건에 따라 `exit(1)`했고, `Restart=on-failure`에 의해 재시작되었다.

### ② 메모리 누적 정책 확인

```bash
cat /etc/sysconfig/rh342-app8-6
```

```ini
RH342_BUFFER_POLICY=retained
```

실습 애플리케이션은 약 1.5초마다 2 MiB 버퍼를 추가한다. `retained`는 생성한 버퍼를 계속 보관하는 정책이다.

**확인된 원인:** 버퍼가 무제한 누적되는 애플리케이션 정책 때문에 RSS가 증가했으며, 자체 종료 조건에 도달해 반복 재시작됨.

## 4. 해결 과정

분석 후 버퍼 보관 정책을 다음과 같이 변경하는 방안이 제시되었다.

```ini
RH342_BUFFER_POLICY=bounded
```

`bounded`는 최근 4개의 버퍼만 보관하는 정책이다. 해당 설정의 수정 후 실행 결과는 현재 확보된 실습 기록에 포함되지 않았다.

## 5. 결과

- `retained` 정책, RSS 88 MiB 초과 후 `exit(1)`, systemd의 반복 재시작을 확인.
- 서비스 재시작 횟수 `NRestarts=8` 확인.
