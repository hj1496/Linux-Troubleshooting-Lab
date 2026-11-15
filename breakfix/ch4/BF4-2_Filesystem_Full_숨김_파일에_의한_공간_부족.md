# 4-2. Filesystem Full — 숨김 파일에 의한 공간 부족

## 1. 개요

| 항목 | 내용 |
|---|---|
| 환경 | Rocky Linux 8.10 (`node1`) |
| 대상 | `/dev/sdb1` → `/data` |
| 증상 | `/data` 사용률 100% |
| 진단 | `df`로 사용률 확인 → `du`와 숨김 파일 조사 |
| 원인 | `/data/.bf4-fill`이 약 2.0GB 점유 |

## 2. 문제 현상

`df -h /data`에서 파일시스템 여유 공간이 거의 없는 상태를 확인했다.

```text
Filesystem  Size  Used  Avail  Use%  Mounted on
/dev/sdb1   2.0G  2.0G   456K  100%  /data
```

## 3. 원인 분석

① `/data`의 디렉터리 사용량을 조사했다.

```bash
du -h /data
```

```text
2.0G  /data
```

② 숨김 파일까지 확인하여 공간을 점유하는 파일을 식별했다.

```text
파일: /data/.bf4-fill
크기: 2,087,780,352 bytes (약 2.0GB)
```

일반 파일 목록에서 놓치기 쉬운 `.`으로 시작하는 파일이 공간 부족의 직접 원인이었다.

## 4. 해결 과정

원인 파일을 제거한 뒤 사용률을 다시 확인했다.

```bash
rm -f /data/.bf4-fill
df -h /data
du -sh /data
```

삭제 후 `df -h /data`에서 `47M` 사용, `2.0G` 가용, `3%` 사용률을 확인했다.

실제 파일 쓰기가 가능한지도 확인했다.

```bash
echo "write test" > /data/write-test.txt
cat /data/write-test.txt
```

## 5. 결과

- `/data` 사용률이 **100% → 3%**로 감소했다 (`47M` 사용, `2.0G` 가용).
- 파일 삭제 후 `du -sh /data` 출력은 `4.0K`였다.
- `cat /data/write-test.txt`에서 `write test`를 확인했다.
