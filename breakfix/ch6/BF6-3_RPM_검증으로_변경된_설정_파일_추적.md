# BF 6-3. RPM 검증으로 변경된 설정 파일 추적

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 |
| 대상 | `grep` 패키지의 `/etc/GREP_COLORS` |
| 문제 | 패키지 설치 후 설정 파일 내용이 변경됨 |
| 진단·복구 | `rpm -V`, `diff -u` → 백업 파일 복원 |

## 2. 문제 현상

```bash
rpm -V grep
```

```text
S.5....T.  c /etc/GREP_COLORS
```

RPM 검증에서 설정 파일의 크기, 내용, 수정 시각이 패키지 기록과 다른 것으로 나타났다.

## 3. 원인 분석

```bash
diff -u /root/GREP_COLORS.before-rh342 /etc/GREP_COLORS
```

비교 결과 `/etc/GREP_COLORS`에 다음 주석 한 줄이 추가된 것을 확인했다.

```text
# Accidental local change
```

검증 코드의 의미:

| 표시 | 의미 |
|---|---|
| `S` | 파일 크기 불일치 |
| `5` | 파일 다이제스트 불일치 |
| `T` | 수정 시각 불일치 |
| `c` | 설정 파일로 분류됨 |

## 4. 해결 과정

변경 전 백업 파일로 복원했다.

```bash
cp -a /root/GREP_COLORS.before-rh342 /etc/GREP_COLORS
rpm -V grep
```

## 5. 결과

- 복원 후 `rpm -V grep` 무출력 확인.
- `/etc/GREP_COLORS`의 추가 주석이 제거된 것 확인.
