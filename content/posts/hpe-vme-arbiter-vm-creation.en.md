---
title: "Creating Arbiter VMs manually in HPE VME Manager"
description: "This is a guide to the virt-install script, VNC 5901 connection, GRUB console setup, and dpkg package installation procedures to manually create an Arbiter VM in the HPE VME Manager environment."
date: 2026-01-07T10:00:00+09:00
draft: false
categories: ["Tech"]
tags: ["HPE", "VME", "Arbiter", "Linux", "Ubuntu", "KVM", "virsh", "Troubleshooting"]
---


This is a procedure and command guide for manually creating and setting up an Arbiter VM in the HPE VME Manager environment.

---

## 1. Copy Ubuntu ISO image

Copy the ISO image to the libvirt image path and check whether it was copied properly.

```bash
# ISO 이미지 복사
cp ubuntu-22.04.5-live-server-amd64.iso /var/lib/libvirt/images/

# 복사 확인
ls /var/lib/libvirt/images/
# 출력: ubuntu-22.04.5-live-server-amd64.iso
```

---

## 2. Write and run the Arbiter VM creation script

Create a directory to store the VM disk and write a `virt-install` script to deploy it.

```bash
# 저장소 디렉토리 생성
mkdir -p /var/morpheus/kvm/vms/arbiter

# VM 생성 스크립트 작성 (arbiter.sh)
vi arbiter.sh
```

**`arbiter.sh` script content**:
```bash
sudo virt-install --name arbiter --ram 4096 --vcpus 2 --cpu host-passthrough \
--disk path=/var/morpheus/kvm/vms/arbiter/arbiter.qcow2,size=40,format=qcow2,bus=virtio \
--cdrom /var/lib/libvirt/images/ubuntu-22.04.5-live-server-amd64.iso \
--network network=Management,model=virtio \
--graphics vnc,listen=0.0.0.0,password=P@ssw0rd \
--boot uefi \
--noautoconsole --autostart
```

Run script:
```bash
sh -x arbiter.sh
```

**Example output**:
```text
Starting install...
Allocating 'arbiter.qcow2' ...
Creating domain...
Domain is still running. Installation may be in progress.
```

---

## 3. Install OS using VNC Client

Check the VNC display port and proceed with the OS installation process through a VNC viewer (e.g. MobaXterm).

```bash
# VNC 포트 확인
virsh vncdisplay arbiter
# 출력 예시: :1 -> VME Manager 서버 IP:5901 로 접속 (기본 포트 5900 + 1)
```

---

## 4. Check VM status and start after completing OS installation

When the installation is complete and the VM is in the shut off state, it is set to auto-start and then runs.

```bash
# VM 리스트 확인
virsh list --all
```

**Example output**:
```text
root@vmemgr:/home/vmeadmin# virsh list --all
 Id   Name      State
--------------------------
 1    vmemgr    running
 -    arbiter   shut off
```

Set up and start autostart when `arbiter` status shows `shut off`:
```bash
virsh autostart arbiter
virsh start arbiter
```

---

## 5. Setting up Virsh Console connection

Change GRUB settings so that you can directly access the VM terminal with the `virsh console` command.

```bash
# GRUB 설정 수정
vi /etc/default/grub

# 아래 항목 추가/수정
GRUB_CMDLINE_LINUX="console=ttyS0"

# GRUB 업데이트
update-grub
```

After setup, you can access directly with the following command:
```bash
virsh console arbiter
```

---

## 6. Install Arbiter package

Install the Arbiter installation package (`svtarb`).

```bash
dpkg -i ./svtarb_6.0.0.39_amd64.deb
```

**Example output and response**:
```text
Do you accept the End User License Agreement (y/n) y
Certificate request self-signature ok
subject=CN = arbiter
Created symlink /etc/systemd/system/multi-user.target.wants/svtarb.service -> /lib/systemd/system/svtarb.service.
```

---

## 💡 Key points and tips

1. **ISO Path**: Located in `/var/lib/libvirt/images/`
2. **VM disk path**: Created in `/var/morpheus/kvm/vms/arbiter/`
3. **VNC connection**: Connect to port 5901 using a VNC client such as MobaXterm.
4. **Console connection**: Conveniently manage `virsh console` by setting `GRUB_CMDLINE_LINUX="console=ttyS0"`
5. **Auto-registration**: Automatically registered as a `systemd` service when installing the Arbiter package (`svtarb`)
