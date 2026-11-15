# BF 5-4 — crypttab UUID 불일치로 인한 LUKS 접근 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `/dev/sdd` (LUKS2) |
| 정상 구성 | `/dev/sdd` → `/dev/mapper/cryptdata` → XFS → `/cryptdata` |
| 증상 | 암호화 매핑이 활성화되지 않아 마운트 실패; 부팅 중 emergency mode 진입도 관찰 |
| 진단 | 실제 LUKS UUID와 `/etc/crypttab`의 UUID 대조 |

## 2. 문제 현상

```text
cryptsetup status cryptdata
# cryptdata is inactive
```

`/etc/crypttab`에는 실제 디스크와 다른 UUID가 설정되어 있었다.

```text
crypttab UUID: 11111111-2222-3333-4444-555555555555
actual UUID:   fead8cd5-407e-425a-bbee-5685e3dd0679
```

## 3. 원인 분석

- `blkid /dev/sdd`에서 실제 LUKS2 UUID `fead8cd5-407e-425a-bbee-5685e3dd0679` 확인.
- `/etc/crypttab`은 존재하지 않는 UUID를 참조.
- 따라서 LUKS 헤더나 XFS 자체 손상이 아닌 **암호화 장치 식별 설정 오류**로 판단.

## 4. 해결 과정

`/etc/crypttab`의 장치 UUID와 형식을 실제 `/dev/sdd`에 맞게 수정했다. 이후 systemd의 cryptsetup 매핑을 다시 활성화하고 기존 XFS를 `/cryptdata`에 마운트했다.

`luksFormat`이나 `mkfs.xfs`를 다시 수행하지 않았다.

## 5. 결과

- `cryptsetup status cryptdata` → `/dev/mapper/cryptdata is active and is in use`.
- `findmnt /cryptdata` → `/dev/mapper/cryptdata` / `xfs`.
- `cat /cryptdata/test.txt` → `RH342-LUKS-TEST`.
