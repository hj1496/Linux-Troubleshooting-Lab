# BF 8-1. Shared Library 로딩 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10, `/opt/rh342/ch08-1` 격리 실습 |
| 증상 | 실행 파일 시작 실패 |
| 원인 | `LD_LIBRARY_PATH`가 우선 참조한 비정상 `libpcre2-8.so.0` |
| 진단 | 실행 로그, `ldd`, 런타임 라이브러리 경로 확인 |

## 2. 문제 현상

애플리케이션 실행 스크립트가 사설 런타임 라이브러리를 우선 사용하도록 설정되어 있었다.

```bash
/opt/rh342/ch08-1/run /etc/hostname
```

실행 실패 시 오류는 `/opt/rh342/ch08-1/log/application.log`에 기록되도록 구성되어 있었다.

## 3. 원인 분석

### ① 라이브러리 의존성 확인

```bash
ldd /opt/rh342/ch08-1/bin/rh342-inspect
```

애플리케이션의 의존성 및 사설 런타임 디렉터리를 조사했다.

```bash
/opt/rh342/ch08-1/runtime/libpcre2-8.so.0
```

해당 경로에 정상 ELF 공유 라이브러리가 아닌 손상된 파일이 존재했다. 실행 스크립트의 `LD_LIBRARY_PATH` 지정으로 이 파일이 시스템 라이브러리보다 먼저 참조되었다.

**확인된 원인:** 시스템 전체 라이브러리 장애가 아닌, 애플리케이션 전용 런타임 디렉터리의 라이브러리 파일 손상.

## 4. 해결 과정

문제가 된 사설 라이브러리 경로를 정상 시스템 라이브러리를 가리키는 심볼릭 링크로 복구했다.

```bash
ln -s /usr/lib64/libpcre2-8.so.0.7.1 \
  /opt/rh342/ch08-1/runtime/libpcre2-8.so.0

readlink -f /opt/rh342/ch08-1/runtime/libpcre2-8.so.0
```

`readlink -f`로 실제 대상 경로가 올바르게 해석되는지 확인했다.

## 5. 결과

- `/opt/rh342/ch08-1/run /etc/hostname` 실행 시 `/etc/hostname` 출력.
- 실행 직후 `echo $?` 결과 `0`.
