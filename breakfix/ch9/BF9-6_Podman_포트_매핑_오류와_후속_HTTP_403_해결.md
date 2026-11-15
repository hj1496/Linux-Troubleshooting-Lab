# BF9-6. Podman 포트 매핑 오류와 후속 HTTP 403 해결

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / Podman / UBI8 HTTPD |
| 증상 | HTTP 연결 거부 → 포트 수정 후 `403 Forbidden` |
| 원인 | 내부 서비스 포트 불일치 및 웹 문서 루트의 기본 페이지 부재 |
| 해결 | `18096:8080` 재매핑 후 `index.html` 생성 |

## 2. 문제 현상

호스트의 `127.0.0.1:18096`으로 접근하면 연결이 거부됐다.

```bash
curl -i http://127.0.0.1:18096/
podman port rh342-bf96
```

```text
8090/tcp -> 127.0.0.1:18096
```

매핑을 수정하자 TCP 연결 거부는 사라졌지만, 이번에는 HTTP `403 Forbidden`이 발생했다.

## 3. 원인 분석

### ① 실제 수신 포트와 포트 매핑 불일치

Apache 로그에는 정상 기동 메시지가 있었다.

```text
configured -- resuming normal operations
```

그러나 Apache가 컨테이너 내부 `8080`에서 수신하는 동안 포트 매핑은 `18096:8090`이었다. 호스트 요청이 실제 HTTP 서비스에 전달되지 않았다.

### ② 포트 수정 후 HTTP 403

`18096:8080`으로 매핑을 바꾸자 HTTP 응답이 반환되었다. 따라서 네트워크 연결과 HTTP 응답을 구분해 추가 진단했다.

Apache의 `/var/www/html`에 기본 `index.html`이 없어 디렉터리 요청이 `403 Forbidden`으로 처리됐다.

## 4. 해결 과정

컨테이너를 실제 Apache 포트에 맞춰 재생성했다.

```bash
podman rm -f rh342-bf96
podman run -d --name rh342-bf96 \
  -p 127.0.0.1:18096:8080 \
  registry.access.redhat.com/ubi8/httpd-24:latest
```

다음으로 컨테이너 내부 DocumentRoot인 `/var/www/html`에 `index.html`을 생성했다. 웹 콘텐츠 부재로 발생한 403을 해결하기 위한 조치다.

## 5. 결과

- 호스트 포트 `18096`에서 컨테이너 내부 `8080`으로의 연결 정상화 확인.
- 기본 페이지 생성 후 HTTP `200 OK` 확인.
