# 4-1. Mount Failure — UUID 및 파일시스템 유형 불일치

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 (`node1`) |
| 대상 | `/dev/sdb1` (XFS), 마운트 지점 `/data` |
| 증상 | `/etc/fstab` 설정 오류로 `/data` 마운트 실패 |
| 진단 | `blkid`의 실제 UUID·파일시스템 유형과 `/etc/fstab` 비교 |
| 구성 | BF 4-1(UUID 오류) + BF 4-2(파일시스템 유형 오류) 통합 |

## 2. 문제 현상

### 사례 A — UUID 오기

`/etc/fstab`의 UUID 끝에 `9`가 추가되어 실제 장치를 찾지 못했다.

```text
실제 UUID : fdcde736-7aac-4c38-91fa-1371b316b3e4
설정 UUID : fdcde736-7aac-4c38-91fa-1371b316b3e49
```

`mount -a` 실행 시 다음 오류가 발생했다.

```text
mount: /data: can't find UUID=fdcde736-7aac-4c38-91fa-1371b316b3e49
```

### 사례 B — 파일시스템 유형 불일치

실제 `/dev/sdb1`은 XFS였지만 `/etc/fstab`에는 `ext4`로 지정되어 있었다.

```text
실제 파일시스템 : xfs
/etc/fstab 설정  : ext4
```

## 3. 원인 분석

① `blkid /dev/sdb1`으로 장치의 UUID와 실제 파일시스템 유형을 확인했다.

- `UUID`: `fdcde736-7aac-4c38-91fa-1371b316b3e4`
- `TYPE`: `xfs`

② `/etc/fstab`과 비교하여 서로 다른 설정 오류를 구분했다.

| 사례 | 확인된 원인 | 영향 |
|---|---|---|
| A | 존재하지 않는 UUID 참조 | 마운트할 장치를 찾지 못함 |
| B | XFS 장치를 `ext4`로 지정 | 실제 파일시스템과 마운트 설정이 불일치 |

## 4. 해결 과정

- 사례 A: `/etc/fstab`의 UUID를 `fdcde736-7aac-4c38-91fa-1371b316b3e4`로 수정했다.
- 사례 B: `/etc/fstab`의 파일시스템 유형을 `ext4`에서 `xfs`로 수정했다.
- 수정 후 다음 명령으로 설정을 반영하고 마운트를 확인했다.

```bash
systemctl daemon-reload
mount -a
findmnt /data
```

```text
TARGET  SOURCE     FSTYPE
/data   /dev/sdb1  xfs
```

## 5. 결과

- `findmnt /data`에서 `/dev/sdb1`이 `xfs`로 마운트된 상태를 확인했다.
- `/data/text.txt`가 유지된 것을 확인했다.
- 수정 후 재부팅 검증을 완료했다.
