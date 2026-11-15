# BF 5-5 — iSCSI Initiator IQN과 Target ACL 불일치

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | `node1` Initiator `192.168.100.11` / `node2` Target `192.168.100.12` |
| Target | `iqn.2026-09.rh342.lab:storage` / TCP 3260 / `/dev/sdb` 5 GiB |
| 증상 | Target에 대한 Initiator 로그인 불가 |
| 진단 | Network → Discovery → Session → ACL/IQN → Block Device 순서로 점검 |

## 2. 문제 현상

정상 상태에서 Target Discovery와 로그인이 가능했고, Initiator에 `/dev/sde`가 연결되어 XFS `/iscsi-data`를 사용할 수 있었다. 이후 Target ACL을 다른 IQN으로 변경한 장애를 실습했다.

```text
node1 InitiatorName: iqn.1994-05.com.redhat:f0d04c88d060
node2 ACL:          iqn.1994-05.com.redhat:deadbeef
```

## 3. 원인 분석

- Target IP는 `192.168.100.12`, 포트는 `3260`.
- Target에 설정된 ACL의 Initiator IQN이 node1의 실제 IQN과 다름.
- Discovery 또는 TCP 도달성과 별개로, **Target ACL의 접근 허가가 맞지 않으면 로그인 단계에서 실패**함.

## 4. 해결 과정

`node2`의 `targetcli`에서 잘못된 ACL을 삭제하고 실제 Initiator IQN을 등록했다.

```text
/iscsi/iqn.2026-09.rh342.lab:storage/tpg1/acls delete iqn.1994-05.com.redhat:deadbeef
/iscsi/iqn.2026-09.rh342.lab:storage/tpg1/acls create iqn.1994-05.com.redhat:f0d04c88d060
saveconfig
```

이후 `node1`에서 iSCSI 로그인·세션·블록 장치·마운트를 확인했다. 기존 원격 디스크의 파일시스템을 재생성하지 않았다.

## 5. 결과

- `iscsiadm -m session` → `192.168.100.12:3260,1 iqn.2026-09.rh342.lab:storage` 세션 확인.
- `findmnt /iscsi-data` → `/dev/sde` / `xfs`.
- `cat /iscsi-data/test.txt` → `RH342-ISCSI-TEST`.
