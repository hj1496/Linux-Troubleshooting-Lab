# BF9-2. 파일 이동 후 SELinux Context 불일치로 발생한 웹 접근 장애

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / Apache / SELinux Enforcing |
| 증상 | 웹 문서 경로로 옮긴 파일에 HTTP 접근 실패 |
| 원인 | `mv` 후 기존 SELinux Type `admin_home_t` 유지 |
| 해결 | 대상 경로의 기본 Context를 `restorecon`으로 복구 |

## 2. 문제 현상

콘텐츠를 `/var/www/html/bf92/`로 이동했지만 Apache가 파일을 정상적으로 제공하지 못했다.

파일의 SELinux Type은 웹 콘텐츠에 기대되는 유형이 아니라 다음과 같았다.

```text
admin_home_t
```

## 3. 원인 분석

- 같은 파일시스템에서 `mv`로 이동한 파일은 기존 SELinux Label을 유지할 수 있다.
- 웹 문서 경로에 배치했더라도 `admin_home_t`가 남아 있어 Apache의 접근 정책과 충돌했다.
- 파일 자체의 위치보다 **실제 SELinux Type과 경로별 기본 Type의 불일치**가 핵심이었다.

## 4. 해결 과정

```bash
restorecon -Rv /var/www/html/bf92/
```

이동된 파일에 웹 문서 경로의 기본 SELinux Context를 다시 적용했다.

## 5. 결과

- 해당 경로의 웹 콘텐츠에 대해 HTTP `200 OK` 응답 확인.
