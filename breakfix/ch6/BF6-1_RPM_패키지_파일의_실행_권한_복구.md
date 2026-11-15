# BF 6-1. RPM 패키지 파일의 실행 권한 복구

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 |
| 대상 | `bind-utils` / `/usr/bin/host` |
| 문제 | 패키지에 포함된 실행 파일의 권한이 RPM 기록과 불일치 |
| 진단·복구 | `rpm -V` → `rpm --setperms` |

## 2. 문제 현상

```bash
rpm -V bind-utils
```

```text
.M.......    /usr/bin/host
```

`/usr/bin/host`의 실제 권한은 `-rw-r--r--`로, 실행 비트가 없었다.

## 3. 원인 분석

- `rpm -V`의 `M`은 파일의 mode(권한/유형) 차이를 나타낸다.
- 파일 자체가 사라진 것이 아니라, 설치된 패키지의 기록과 파일 권한이 달라진 상태였다.

## 4. 해결 과정

```bash
rpm --setperms bind-utils
```

RPM 데이터베이스에 기록된 파일 권한을 기준으로 패키지 소속 파일의 권한을 복구했다.

## 5. 결과

- `rpm -V bind-utils` 무출력: 패키지 파일 검증 통과.
- `/usr/bin/host` 권한이 `-rwxr-xr-x`로 복원됨.
- `host example.com` 실행 정상 확인.
