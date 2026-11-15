# BF 5-2 — VG 이름 변경으로 인한 LV 경로 불일치

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 / `/dev/sdc` / VG `myVG` / LV `dataLV` |
| 증상 | 기존 `/dev/myVG/dataLV` 경로로 LV를 찾을 수 없음 |
| 원인 | VG 이름이 `myVG`에서 `dataVG`로 변경됨 |
| 진단 | VG UUID·이름 및 `/etc/fstab`이 참조하는 장치 경로 비교 |

## 2. 문제 현상

기존 경로를 지정한 LV 활성화 명령이 실패했다.

```text
VG myVG not found
```

`vgs`/`vgdisplay`에서는 VG가 `dataVG`라는 이름으로 조회되었다.

## 3. 원인 분석

- VG UUID는 동일하나 VG 이름만 `myVG` → `dataVG`로 변경된 상태.
- 기존 마운트 구성은 `/dev/mapper/myVG-dataLV`를 참조.
- 즉, 용량 부족이나 PV 손상이 아니라 **VG 이름과 참조 경로의 불일치**가 문제.

## 4. 해결 과정

기존 명명 규칙과 마운트 구성을 유지하기 위해 VG 이름을 원복했다.

```bash
vgrename dataVG myVG
```

이후 `vgs`, `lvs`, 매퍼 장치 및 `/lvdata` 마운트 상태를 확인했다.

## 5. 결과

- `vgs`에서 `myVG`, `lvs`에서 `dataLV myVG` 확인.
- 재부팅 후 `dataLV`는 **3.00 GiB / `-wi-ao----`** 상태.
- `findmnt /lvdata` → `/dev/mapper/myVG-dataLV` / `xfs`.
