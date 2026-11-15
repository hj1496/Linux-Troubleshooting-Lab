# BF 6-2. RPM 패키지 실행 파일 누락 복구

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 |
| 대상 | bind-utils / /usr/bin/nslookup |
| 문제 | 패키지가 설치된 상태에서 실행 파일 누락 |
| 진단 | RPM 파일 검증 및 패키지 소유권 확인 |
| 해결 | bind-utils 재설치 |

## 2. 문제 현상

bind-utils 패키지에 속한 nslookup 실행 파일이 누락됐다.

```bash
rpm -V bind-utils
```

```text
missing     /usr/bin/nslookup
```

## 3. 원인 분석

패키지 검증 결과에서 누락된 파일을 특정하고, 파일의 소속 패키지를 조사했다.

```bash
rpm -qf /usr/bin/nslookup
```

- RPM 검증 결과의 `missing`은 패키지 데이터베이스에 등록된 파일이 실제 파일시스템에 없음을 뜻한다.
- 해당 파일은 bind-utils 패키지에 속한다.
- 따라서 패키지 자체가 설치되지 않은 문제가 아니라, 설치된 패키지의 구성 파일이 누락된 문제로 판단했다.

## 4. 해결 과정

누락된 실행 파일을 패키지 원본에서 복원하기 위해 bind-utils를 재설치했다.

```bash
dnf reinstall bind-utils
```

## 5. 결과

- 재설치 후 `rpm -V bind-utils`에서 출력이 없음을 확인했다.
- `nslookup` 명령이 정상 실행됨을 확인했다.
