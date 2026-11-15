# BF 8-2. systemd 203/EXEC와 CRLF 스크립트

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10, `rh342-app8-2.service` |
| 증상 | 서비스 시작 실패, `status=203/EXEC` |
| 원인 | 실행 스크립트 shebang의 CRLF 줄 끝 문자 |
| 진단 | `journalctl`, `file`, `head`, `cat -A` |

## 2. 문제 현상

서비스 시작 시 아래 오류가 확인되었다.

```text
status=203/EXEC
Failed to execute command: No such file or directory
Failed at step EXEC spawning /opt/rh342/ch08-2/bin/rh342-agent
```

## 3. 원인 분석

### ① 파일 존재 및 유형 검사

```bash
ls -l /opt/rh342/ch08-2/bin/rh342-agent
file /opt/rh342/ch08-2/bin/rh342-agent
ldd /opt/rh342/ch08-2/bin/rh342-agent
```

- 실행 파일은 실제 존재했고 실행 권한도 있었다.
- `ldd` 결과는 `not a dynamic executable`로, 동적 ELF 실행 파일이 아닌 스크립트였다.
- `file` 결과에 `Bourne-Again shell script, ASCII text executable, with CRLF line terminators`가 표시되었다.

### ② shebang의 숨은 문자 확인

```bash
head -n 1 /opt/rh342/ch08-2/bin/rh342-agent | cat -A
```

```text
#!/bin/bash^M$
```

`^M`은 캐리지 리턴(`\r`)을 의미한다. Linux 커널이 shebang을 해석할 때 `/bin/bash\r`을 인터프리터 경로로 취급할 수 있어, 파일 자체가 존재해도 `No such file or directory`와 `203/EXEC`가 발생한다.

**확인된 원인:** 실행 파일 누락이 아니라 CRLF 형식 때문에 잘못 해석된 shebang.

## 4. 해결 과정

`vi`로 스크립트를 수정하고 첫 줄을 다시 검사했다.

```bash
head -n 1 /opt/rh342/ch08-2/bin/rh342-agent | cat -A
```

```text
#!/bin/bash$
```

첫 줄의 `^M`이 제거된 것을 확인했다.

## 5. 결과

- shebang이 `#!/bin/bash^M$`에서 `#!/bin/bash$`로 변경된 것을 확인.
