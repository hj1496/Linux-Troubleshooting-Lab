# BF 5-1 — LVM LV 비활성화로 인한 마운트 실패

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / VirtualBox / `/dev/sdc` (5 GiB) |
| 정상 구성 | `/dev/sdc` → `myVG` → `dataLV` (3 GiB, XFS) → `/lvdata` |
| 증상 | VG와 LV는 조회되지만 LV 장치가 비활성 상태 |
| 진단 | `pvs`, `vgs`, `lvs`, `lsblk`로 LVM 인식 여부와 활성화 상태를 분리 |

## 2. 문제 현상

`/lvdata`를 언마운트하고 VG를 비활성화한 상태에서 LV 매퍼 장치가 나타나지 않았다.

```text
LV Path      /dev/myVG/dataLV
LV Status    NOT available
LV Attr      -wi-------
```

`blkid /dev/sdc`는 여전히 `LVM2_member`를 인식했다.

## 3. 원인 분석

- `pvs`/`vgs`에서 PV `/dev/sdc`와 VG `myVG`가 조회됨 → PV/VG 메타데이터 인식 실패는 아님.
- `lvs`의 `-wi-------`와 `lvdisplay`의 `NOT available` → LV가 **비활성 상태**.
- `/dev/mapper/myVG-dataLV`가 없음 → 파일시스템을 마운트할 블록 매퍼 장치가 생성되지 않음.

## 4. 해결 과정

```bash
lvchange -ay /dev/myVG/dataLV
```

활성화 후 `lvs`는 `-wi-a-----`로 변경되었고 `/dev/mapper/myVG-dataLV -> ../dm-2`가 확인됐다. 이후 기존 XFS 파일시스템을 `/lvdata`에 다시 마운트했다. 파일시스템을 재생성하지 않았다.

## 5. 결과

- 재부팅 후 `lvs`에서 `dataLV` **3.00 GiB / `-wi-ao----`** 확인.
- `findmnt /lvdata` → `/dev/mapper/myVG-dataLV` / `xfs`.
- `df` → 3.0G 중 54M 사용, 사용률 2%.
