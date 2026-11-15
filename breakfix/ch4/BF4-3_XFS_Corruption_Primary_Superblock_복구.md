# 4-3. XFS Corruption — Primary Superblock 복구

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 (`node1`) |
| 대상 | `/dev/sdb1` (XFS), `/data` |
| 증상 | XFS 식별 정보가 보이지 않고 마운트 실패 |
| 진단 | `xfs_repair -n`으로 Primary/Secondary Superblock 확인 |
| 복구 | `xfs_repair`로 Primary Superblock 재구성 |

## 2. 문제 현상

복구 실습에서 `/data`의 `recovery-marker.txt`를 준비한 뒤 장치 앞부분 4바이트를 덮어써 손상을 발생시켰다. 당시 첫 `umount /data` 시도는 `target is busy`로 실패했고, 장치가 마운트된 상태에서 덮어쓰기가 이루어졌다.

이후 언마운트한 상태에서 `lsblk -f`와 `blkid`로 확인했을 때 `/dev/sdb1`의 파일시스템 유형·UUID가 표시되지 않았다. 재마운트도 실패했다.

```text
wrong fs type, bad option, bad superblock ...
```

## 3. 원인 분석

① 변경 없는 점검을 위해 `xfs_repair -n`을 실행했다.

```bash
xfs_repair -n /dev/sdb1
```

결정적인 진단 메시지는 다음과 같았다.

```text
bad primary superblock - bad magic number !!!
```

② 도구가 **Secondary Superblock을 탐색·검증**했으며, Primary Superblock을 수정해야 하는 상태임을 확인했다. `-n`은 실제 수정 없이 종료되었다.

따라서 마운트 설정이나 UUID 오기가 아니라 XFS 메타데이터 손상으로 판단했다.

## 4. 해결 과정

파일시스템이 마운트되지 않은 상태에서 복구를 수행했다.

```bash
xfs_repair /dev/sdb1
```

- 복구가 Phase 1~7까지 진행되었다.
- 수정된 Primary Superblock을 기록했다.
- 이후 `/dev/sdb1`을 `/data`에 마운트했다.

## 5. 결과

- `blkid` 및 `lsblk -f`에서 XFS와 기존 UUID `fdcde736-7aac-4c38-91fa-1371b316b3e4`가 다시 확인되었다.
- `/dev/sdb1`을 `/data`에 마운트했다.
- `/data/recovery-marker.txt` 파일이 존재하는 것을 확인했다.
