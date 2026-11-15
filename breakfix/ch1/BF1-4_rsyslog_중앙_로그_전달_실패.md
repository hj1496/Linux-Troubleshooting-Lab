# BF 1-4. rsyslog 중앙 로그 전달 실패

## 1. 개요

| 항목  | 내용                                                     |
| --- | ------------------------------------------------------ |
| 환경  | Rocky Linux 8.10 (`node1`), rsyslog, 로컬 Python TCP 수신기 |
| 증상  | 중앙 로그 전달 실패 및 rsyslog forwarding 오류                    |
| 원인  | 송신·수신 포트 불일치, SELinux TCP 연결 차단                        |
| 진단  | rsyslog 설정, 수신 포트, journal 및 audit 로그 교차 확인            |

## 2. 문제 현상

rsyslog의 전달 대상과 Python 수신기의 실제 대기 포트가 달랐다.

```conf
# /etc/rsyslog.d/rh342-forward.conf (기존 설정)
*.* @@127.0.0.1:1514
```

- rsyslog 전달 대상: `127.0.0.1:1514` (TCP)
- Python 수신기(PID 1739): `127.0.0.1:1515`에서 LISTEN
- rsyslog 로그에 `cannot connect`, `Permission denied`, `omfwd suspended` 메시지 발생

## 3. 원인 분석

### ① 송신·수신 포트 불일치

rsyslog 설정은 1514/tcp로 로그를 보내지만 수신기는 1515/tcp에서 대기하고 있었다. 따라서 전달 경로의 포트 설정을 맞춰야 했다.

### ② SELinux 접근 거부

전달 대상 포트를 수정한 뒤에도 연결 오류가 이어져 audit 로그를 조사했다.

```text
AVC denied { name_connect }
대상 포트 유형: unreserved_port_t
```

SELinux가 rsyslog의 TCP 연결을 거부하고 있음을 확인했다. rsyslog에서 사용하는 포트를 `syslogd_port_t`로 지정해야 했다.

## 4. 해결 과정

① rsyslog의 전달 대상을 수신기의 실제 포트인 1515로 변경했다.

```conf
# /etc/rsyslog.d/rh342-forward.conf (수정 후)
*.* @@127.0.0.1:1515
```

② SELinux에 1515/tcp를 syslog 전송용 포트로 등록하고 rsyslog를 재시작했다.

```bash
semanage port -a -t syslogd_port_t -p tcp 1515
systemctl restart rsyslog
```

③ 테스트 메시지를 보내고 수신 파일을 확인했다.

```bash
logger -t RH342TEST "forward-test-001"
sleep 1
tail /tmp/rh342-received.log
```

실습용 Python 수신기가 메시지를 받은 뒤 연결을 닫으면서 `remote server ... closed connection` 메시지가 추가로 발생했다. 최초의 포트 불일치·SELinux 차단과는 다른 현상이다.

## 5. 결과

- `/tmp/rh342-received.log`에 전송한 테스트 메시지가 기록되어 로그 전달에 성공했다.
- 1515/tcp의 SELinux 접근 거부를 해소한 뒤 수신 로그를 확인했다.

```text
<13>Sep 10 22:22:05 node1 RH342TEST[2854]: forward-test-001
```
