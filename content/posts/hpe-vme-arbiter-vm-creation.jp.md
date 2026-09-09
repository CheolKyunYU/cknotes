---
title: "HPE VME Manager에 수동으로 Arbiter VM 생성"
description: "HPE VME Manager 환경에서 수동으로 Arbiter VM을 생성하는 virt-install 스크립트, VNC 5901 접속, GRUB 콘솔 설정 및 dpkg 패키지 설치 절차 가이드입니다."
date: 2026-01-07T10:00:00+09:00
draft: false
categories: ["Tech"]
tags: ["HPE", "VME", "Arbiter", "Linux", "Ubuntu", "KVM", "virsh", "Troubleshooting"]
---

HPE VME Manager 환경에서 수동으로 Arbiter VM을 생성하고 설정하는 절차 및 명령어 가이드입니다.

---

## 1. Ubuntu ISO 이미지 복사

ISO 이미지를 libvirt 이미지 경로에 복사하고 정상적으로 복사되었는지 확인합니다.

```bash
# ISO 이미지 복사
cp ubuntu-22.04.5-live-server-amd64.iso /var/lib/libvirt/images/

# 복사 확인
ls /var/lib/libvirt/images/
# 출력: ubuntu-22.04.5-live-server-amd64.iso
```

---

## 2. Arbiter VM 생성 스크립트 작성 및 실행

VM 디스크를 저장할 디렉토리를 생성하고 `virt-install` 스크립트를 작성하여 배포합니다.

```bash
# 저장소 디렉토리 생성
mkdir -p /var/morpheus/kvm/vms/arbiter

# VM 생성 스크립트 작성 (arbiter.sh)
vi arbiter.sh
```

**`arbiter.sh` 스크립트 내용**:
```bash
sudo virt-install --name arbiter --ram 4096 --vcpus 2 --cpu host-passthrough \
--disk path=/var/morpheus/kvm/vms/arbiter/arbiter.qcow2,size=40,format=qcow2,bus=virtio \
--cdrom /var/lib/libvirt/images/ubuntu-22.04.5-live-server-amd64.iso \
--network network=Management,model=virtio \
--graphics vnc,listen=0.0.0.0,password=P@ssw0rd \
--boot uefi \
--noautoconsole --autostart
```

스크립트 실행:
```bash
sh -x arbiter.sh
```

**출력 예시**:
```text
Starting install...
Allocating 'arbiter.qcow2' ...
Creating domain...
Domain is still running. Installation may be in progress.
```

---

## 3. VNC Client로 OS 설치

VNC 디스플레이 포트를 확인하고 VNC 뷰어(예: MobaXterm)를 통해 OS 설치 과정을 진행합니다.

```bash
# VNC 포트 확인
virsh vncdisplay arbiter
# 출력 예시: :1 -> VME Manager 서버 IP:5901 로 접속 (기본 포트 5900 + 1)
```

---

## 4. OS 설치 완료 후 VM 상태 확인 및 시작

설치가 완료되어 VM이 종료(shut off) 상태가 되면 자동 시작 설정 후 구동합니다.

```bash
# VM 리스트 확인
virsh list --all
```

**출력 예시**:
```text
root@vmemgr:/home/vmeadmin# virsh list --all
 Id   Name      State
--------------------------
 1    vmemgr    running
 -    arbiter   shut off
```

`arbiter` 상태가 `shut off`로 표시되면 자동 시작 설정 및 기동:
```bash
virsh autostart arbiter
virsh start arbiter
```

---

## 5. Virsh Console 접속 사용 설정

`virsh console` 명령어로 직접 VM 터미널에 접속할 수 있도록 GRUB 설정을 변경합니다.

```bash
# GRUB 설정 수정
vi /etc/default/grub

# 아래 항목 추가/수정
GRUB_CMDLINE_LINUX="console=ttyS0"

# GRUB 업데이트
update-grub
```

설정 후에는 아래 명령어로 직접 접속 가능합니다:
```bash
virsh console arbiter
```

---

## 6. Arbiter 패키지 설치

Arbiter 설치 패키지(`svtarb`)를 설치합니다.

```bash
dpkg -i ./svtarb_6.0.0.39_amd64.deb
```

**출력 및 응답 예시**:
```text
Do you accept the End User License Agreement (y/n) y
Certificate request self-signature ok
subject=CN = arbiter
Created symlink /etc/systemd/system/multi-user.target.wants/svtarb.service -> /lib/systemd/system/svtarb.service.
```

---

## 💡 핵심 포인트 및 팁

1. **ISO 경로**: `/var/lib/libvirt/images/` 에 위치
2. **VM 디스크 경로**: `/var/morpheus/kvm/vms/arbiter/` 에 생성
3. **VNC 접속**: MobaXterm 등의 VNC Client를 활용하여 5901 포트로 접속
4. **Console 접속**: `GRUB_CMDLINE_LINUX="console=ttyS0"` 설정으로 `virsh console` 유용하게 관리
5. **자동 등록**: Arbiter 패키지(`svtarb`) 설치 시 `systemd` 서비스로 자동 등록됨
