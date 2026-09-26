---
title: "[Linux 실무] 재부팅 없이 온라인으로 디스크 용량 증설하기: LVM 확장 및 XFS/EXT4 파일시스템 완벽 가이드"
description: "운영 중인 리눅스 서버에서 서비스 중단 없이 가상 디스크를 증설하고 SCSI 버스 재스캔, LVM 확장(pvresize, lvextend), XFS/EXT4 파일시스템 적용까지의 실무 절차를 정리합니다."
date: 2026-09-22T14:00:00+09:00
draft: false
tags: ["Linux", "LVM", "Storage", "Troubleshooting", "서버운영", "인프라"]
categories:
  - Linux
---

> **글쓴이**: 16년 차 IT 시스템 엔지니어 (CK notes)  
> **대상 OS**: RHEL / Rocky Linux / CentOS 7~9, Ubuntu 20.04~24.04 LTS  
> **주요 파일시스템**: XFS, EXT4 (LVM 기반)

---

엔터프라이즈 환경에서 24시간 가동되는 데이터베이스(DB)나 서비스 서버의 디스크 사용량이 90%를 초과해 긴급 알람이 울리면, 가장 먼저 떠오르는 과제는 **"서비스를 내리지 않고 온라인 상태에서 안전하게 디스크를 늘릴 수 있는가?"**입니다.

과거에는 야간 점검 시간을 잡아 서버를 끄고 파티션을 조정했지만, 현대 가상화(VMware, KVM, Nutanix, SimpliVity 등)와 클라우드 환경에서는 **서버 재부팅 없이 100% 무중단(Online)으로 디스크를 확장**하는 것이 엔지니어의 기본 역량입니다.

16년간 데이터센터 현장에서 수천 번 수행했던 **SCSI 버스 재스캔부터 LVM 볼륨 확장, 파일시스템(XFS vs EXT4) 반영까지의 검증된 표준 작업 절차**를 깔끔하게 정리해 드립니다.

---

## 1. 전체 작업 흐름 (4단계 프로세스)

디스크 확장은 아래의 4단계를 순서대로 거치며, 중간에 마운트를 해제(umount)할 필요가 전혀 없습니다.

```mermaid
flowchart LR
    A["1. 스토리지/VM 디스크 증설"] --> B["2. 리눅스 OS 레벨 SCSI 버스 재스캔"]
    B --> C["3. LVM 물리 볼륨 & 논리 볼륨 확장"]
    C --> D["4. 파일시스템 온라인 확장 (xfs_growfs / resize2fs)"]
```

---

## 2. Step 1: 리눅스 커널에 변경된 디스크 용량 인식시키기 (SCSI Rescan)

하이퍼바이저나 스토리지에서 디스크 크기를 늘렸더라도(예: 100GB -> 200GB), 리눅스 커널은 이를 즉시 인지하지 못합니다. 이때 서버를 재부팅하지 않고 커널에 SCSI 버스를 다시 읽도록 신호를 줍니다.

### 증설 대상 디스크 확인
```bash
# 디스크 및 LVM 구조 확인
lsblk
```

예를 들어 대상 디스크가 `/dev/sdb`라면 아래 명령어를 실행합니다:

```bash
# /dev/sdb 디스크의 SCSI 재스캔 명령
echo 1 > /sys/class/block/sdb/device/rescan
```

*여러 디스크를 한 번에 스캔하고 싶을 때:*
```bash
# 전체 SCSI 호스트 버스 재스캔
for host in /sys/class/scsi_host/host*/scan; do echo "- - -" > ${host}; done
```

### 인식 여부 확인
```bash
# 커널 dmesg 메시지 확인 (용량 변경 로그 출력 확인)
dmesg | tail -n 15

# 용량이 200GB로 늘어났는지 확인
lsblk /dev/sdb
```

---

## 3. Step 2: 파티션 테이블 조정 (필요한 경우)

디스크를 파티션(`/dev/sdb1`) 없이 디스크 통째로 PV로 사용(`pvcreate /dev/sdb`)했다면 이 단계는 건너뛰고 바로 Step 3으로 가시면 됩니다.

만약 파티션(`/dev/sdb1`)으로 나뉘어 있다면 `growpart` 유틸리티를 사용해 파티션을 안전하게 늘립니다.

```bash
# cloud-utils-growpart 패키지가 필요합니다
# RHEL/Rocky: yum install -y cloud-utils-growpart
# Ubuntu: apt install -y cloud-guest-utils

# /dev/sdb의 1번 파티션을 남은 전체 용량으로 확장
growpart /dev/sdb 1
```

---

## 4. Step 3: LVM 물리 볼륨(PV) 및 논리 볼륨(LV) 확장

이제 LVM 계층에서 늘어난 물리 용량을 흡수하고, 용량이 부족한 마운트 포인트(LV)에 할당합니다.

### 1) PV (Physical Volume) 크기 갱신
```bash
# 물리 볼륨 용량 리사이즈
pvresize /dev/sdb
# (파티션인 경우: pvresize /dev/sdb1)

# PFree(여유 공간)가 정상적으로 늘어났는지 확인
pvs
vgs
```

### 2) LV (Logical Volume) 확장
볼륨 그룹(VG)에 확보된 여유 공간을 특정 논리 볼륨(예: `/dev/mapper/vg_data-lv_data`)에 배정합니다.

```bash
# 50GB를 추가로 더 늘릴 때
lvextend -L +50G /dev/mapper/vg_data-lv_data

# 또는 남은 여유 공간(Free Space)을 100% 전부 몰아줄 때 (가장 많이 씀)
lvextend -l +100%FREE /dev/mapper/vg_data-lv_data
```

---

## 5. Step 4: 파일시스템 온라인 확장 (XFS vs EXT4)

LVM 블록 디바이스 크기가 커졌어도, 운영체제의 파일시스템이 이를 인식해야 최종적으로 사용 가능한 공간이 됩니다. 

여기서 **파일시스템 종류에 따라 명령어가 완전히 다르므로** 반드시 현재 파일시스템을 먼저 확인해야 합니다.

```bash
# 파일시스템 유형 확인 (Type 열 확인: xfs 또는 ext4)
df -Th /data
```

### Case A: XFS 파일시스템인 경우 (RHEL 7/8/9, Rocky, CentOS 기본)
XFS는 마운트 포인트 디렉터리 경로를 인자로 넘겨줍니다:

```bash
# 마운트된 디렉터리 경로를 지정합니다
xfs_growfs /data
```

> ⚠️ **엔지니어 필수 상식: XFS 축소 불가**  
> XFS 파일시스템은 온라인 확장은 완벽하게 지원하지만, **축소(Shrink)는 아예 불가능**하게 설계되어 있습니다. 용량을 과도하게 한 번에 늘렸다가 나중에 줄일 수 없으므로 필요한 만큼 계획적으로 증설하세요.

### Case B: EXT4 파일시스템인 경우 (Ubuntu 기본)
EXT4는 논리 볼륨 디바이스 경로를 인자로 넘겨줍니다:

```bash
# LV 디바이스 경로를 지정합니다
resize2fs /dev/mapper/vg_data-lv_data
```

---

## 6. 최종 검증

모든 작업이 완료된 후 실제 여유 공간이 반영되었는지 확인합니다:

```bash
df -h /data
```

`Available` 용량이 즉시 증가되어 있고, 서비스 중인 애플리케이션 로그에 아무런 I/O 에러가 발생하지 않았다면 100점짜리 무중단 작업이 완료된 것입니다.

---

## 현장 트러블슈팅 팁 & 체크리스트

1. **`echo 1 > .../rescan` 명령 후에도 용량이 안 변할 때**:  
   가상화 환경(ESXi, Nutanix 등)에서 디스크 용량을 수정한 뒤 "저장" 버튼이 정상 처리되었는지 하이퍼바이저 이벤트를 먼저 확인하세요.
2. **`xfs_growfs` 실행 시 "is not a mounted XFS filesystem" 에러가 날 때**:  
   장치명(`/dev/mapper/...`)이 아닌 **마운트 포인트(예: `/data`)**를 인자로 주어야 합니다.
3. **루트 파티션(`/`) 증설 시 팁**:  
   루트 파티션도 위와 동일한 방법으로 무중단 확장이 100% 가능합니다. 단, 작업 전 `vgs`, `lvs` 스냅샷을 확인하고 오타에 각별히 유의하세요.
