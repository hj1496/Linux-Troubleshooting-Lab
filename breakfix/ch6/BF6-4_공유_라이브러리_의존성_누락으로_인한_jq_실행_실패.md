# BF 6-4. 공유 라이브러리 의존성 누락으로 인한 jq 실행 실패

## 1. 개요

| 항목    | 내용                                                           |
| ----- | ------------------------------------------------------------ |
| 환경    | Rocky Linux 8.10                                             |
| 대상    | `jq` / `libonig.so.5`                                        |
| 문제    | 실행에 필요한 공유 라이브러리 제공 패키지 누락                                   |
| 진단·복구 | `ldd`, `rpm --whatprovides`, `dnf provides` → `oniguruma` 설치 |

## 2. 문제 현상

`jq --version` 실행 시 `libonig.so.5`를 찾지 못해 실행되지 않았다.

```bash
ldd /usr/bin/jq
```

```text
libonig.so.5 => not found
```

## 3. 원인 분석

**① 설치된 제공 패키지 확인**

```bash
rpm -q --whatprovides 'libonig.so.5()(64bit)'
```

설치된 패키지 중 해당 64비트 라이브러리의 제공자가 없었다.

**② 저장소에서 제공 패키지 검색**

```bash
dnf provides '*/libonig.so.5'
```

검색 결과 `oniguruma`가 해당 라이브러리를 제공하며, `x86_64` 패키지는 `/usr/lib64/libonig.so.5`를 포함하는 것을 확인했다. `i686` 결과와 구별하여 64비트 패키지를 선택했다.

## 4. 해결 과정

```bash
dnf install oniguruma
```

누락된 공유 라이브러리를 제공하는 패키지를 설치했다.

## 5. 결과

- `rpm -q --whatprovides 'libonig.so.5()(64bit)'` → `oniguruma-6.8.2-3.el8.x86_64`.
- `jq --version` → `jq-1.6`: 실행 정상 확인.
- `dnf check` 무출력 확인.
