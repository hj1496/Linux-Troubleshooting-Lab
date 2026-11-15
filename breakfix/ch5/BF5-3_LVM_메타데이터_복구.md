# BF 5-3 — LVM 메타데이터 복구

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `/dev/sdc` (5 GiB) |
| 정상 구성 | PV → `myVG` → `dataLV` (3 GiB) → XFS `/lvdata` |
| 증상 | 기존 PV/VG 연결 정보를 복원해야 하는 상태 |
| 진단 | LVM 백업의 PV UUID와 VG 구성 정보를 확인하고 단계별 복원 |

## 2. 문제 현상

기존 구성의 PV UUID와 VG 메타데이터를 기준으로 복구를 수행했다. 확인된 백업은 `/etc/lvm/backup/myVG`이며, 해당 PV UUID는 다음과 같다.

```text
/dev/sdc
PV UUID: Jkkk8B-x7RP-leTz-JTVt-BfG5-uok0-1PZufC
```

장애를 유발한 원본 명령 자체는 현재 회수한 기록에 명확하지 않으므로 추정하지 않는다.

## 3. 원인 분석

- 백업에 VG `myVG`, LV `dataLV`, PV `/dev/sdc`의 연결 정보가 남아 있음.
- PV UUID를 복원한 직후 `pvs`/`pvdisplay`에서 `/dev/sdc`는 **NEW PV**로 조회되고 VG에 소속되지 않았음.
- 따라서 PV 헤더 복원만으로 끝나지 않고 **VG 메타데이터 복원**이 별도로 필요했음.

## 4. 해결 과정

기존 PV UUID와 백업 파일을 지정하여 PV 정보를 복구했다.

```bash
pvcreate --uuid Jkkk8B-x7RP-leTz-JTVt-BfG5-uok0-1PZufC \
  --restorefile /etc/lvm/backup/myVG /dev/sdc
```

이어서 VG 메타데이터를 복원하고 LV를 활성화했다.

```bash
vgcfgrestore -f /etc/lvm/backup/myVG myVG
lvchange -ay /dev/myVG/dataLV
```

기존 XFS UUID가 유지되는지 확인한 다음 `/lvdata` 마운트를 검증했다. `mkfs`로 파일시스템을 재생성하지 않았다.

## 5. 결과

- `vgdisplay`/`lvs`에서 **`myVG`와 `dataLV` 3.00 GiB (768 PE)** 복구 확인.
- 기존 XFS UUID `e8b398e9-0df1-492b-b901-d15eefa4007b` 유지.
- 재부팅 후 `findmnt /lvdata` → `/dev/mapper/myVG-dataLV` / `xfs`.
