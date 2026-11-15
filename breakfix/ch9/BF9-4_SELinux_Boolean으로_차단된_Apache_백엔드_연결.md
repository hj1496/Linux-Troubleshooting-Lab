# BF9-4. SELinux Boolean으로 차단된 Apache 백엔드 연결

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / Apache Reverse Proxy / SELinux Enforcing |
| 증상 | Reverse Proxy가 백엔드에 연결하지 못하고 `Permission denied` 발생 |
| 원인 | `httpd_can_network_connect=off` |
| 해결 | Apache 네트워크 연결 관련 Boolean 활성화 |

## 2. 문제 현상

Apache를 경유한 백엔드 접근이 실패했다. 파일을 제공하는 문제가 아니라, Apache 프로세스가 백엔드로 **네트워크 연결을 생성하는 단계**에서 거부가 발생했다.

## 3. 원인 분석

```bash
getsebool httpd_can_network_connect
```

```text
httpd_can_network_connect --> off
```

SELinux가 Apache의 네트워크 연결을 제한하고 있었다. 파일 Context와는 다른 정책 제어 문제이므로 해당 Boolean을 수정 대상으로 판단했다.

## 4. 해결 과정

```bash
setsebool httpd_can_network_connect on
getsebool httpd_can_network_connect
```

```text
httpd_can_network_connect --> on
```

`-P` 없이 **현재 실행 환경의 Boolean 값**을 변경했다. 재부팅 후 유지 여부까지 보장하는 영구 설정과는 구분한다.

## 5. 결과

- Boolean 값이 `off → on`으로 변경된 것 확인.
- Reverse Proxy 경유 요청에서 HTTP `200 OK` 확인.
