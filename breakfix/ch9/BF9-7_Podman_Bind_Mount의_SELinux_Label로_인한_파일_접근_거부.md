# BF9-7. Podman Bind Mount의 SELinux Label로 인한 파일 접근 거부

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / Podman / SELinux Enforcing |
| 증상 | Apache 컨테이너가 바인드 마운트된 `index.html`을 읽지 못함 |
| 원인 | 마운트된 콘텐츠에 `var_t`가 적용되어 컨테이너 접근 거부 |
| 해결 | 볼륨에 전용 SELinux 재레이블링 옵션 `:Z` 적용 |

## 2. 문제 현상

호스트의 `/srv/rh342-bf97`을 컨테이너 `/var/www/html`에 마운트했지만 웹 콘텐츠 접근이 실패했다.

Apache 로그에는 다음 메시지가 나타났다.

```text
(13) Permission denied
AH00035: access to /index.html denied ...
because search permissions are missing on a component of the path
```

## 3. 원인 분석

확인한 권한 및 Context는 다음과 같았다.

| 대상 | 확인 결과 |
|---|---|
| 호스트 디렉터리 | `755` |
| 호스트 파일 | `644` |
| 컨테이너 프로세스 사용자 | `uid=1001(default)` |
| 마운트된 콘텐츠의 SELinux Context | `unconfined_u:object_r:var_t:s0` |

일반 파일 권한만으로는 거부 원인을 설명하기 어려웠다. 컨테이너가 접근하는 경로에 `var_t`가 적용되어 있어 SELinux Type 불일치로 판단했다.

## 4. 해결 과정

기존 컨테이너를 제거한 뒤 볼륨을 `:Z`로 재마운트했다.

```bash
podman rm -f rh342-bf97
```

```text
컨테이너 생성 시 사용한 핵심 옵션
-p 127.0.0.1:18097:8080
-v /srv/rh342-bf97:/var/www/html:ro,Z
```

`ro`는 컨테이너에서 읽기 전용으로 마운트하고, `Z`는 해당 컨테이너가 접근할 수 있는 전용 SELinux Label로 재레이블링한다.

## 5. 결과

- 볼륨 Label이 `system_u:object_r:container_file_t:s0:c662,c675`로 변경된 것 확인.
- `127.0.0.1:18097` 접근 시 HTTP `200 OK` 확인.
