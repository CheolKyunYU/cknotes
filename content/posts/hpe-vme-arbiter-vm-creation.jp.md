---
title: "HPE VME Managerで手動でArbiter VMを作成する"
description: "HPE VME Manager 環境で Arbiter VM を手動で作成する virt-install スクリプト、VNC 5901 接続、GRUB コンソール設定、および dpkg パッケージのインストール手順ガイド。"
date: 2026-01-07T10:00:00+09:00
draft: false
categories: ["Tech"]
tags: ["HPE", "VME", "Arbiter", "Linux", "Ubuntu", "KVM", "virsh", "Troubleshooting"]
---


HPE VME Manager 環境で手動で Arbiter VM を作成および設定する手順およびコマンドガイドです.

---

## 1. Ubuntu ISO イメージのコピー

ISOイメージをlibvirtイメージパスにコピーし、正常にコピーされたことを確認します。

```bash
# ISO 이미지 복사
cp ubuntu-22.04.5-live-server-amd64.iso /var/lib/libvirt/images/

# 복사 확인
ls /var/lib/libvirt/images/
# 출력: ubuntu-22.04.5-live-server-amd64.iso
```

---

## 2. Arbiter VM生成スクリプトの作成と実行

VMディスクを保存するディレクトリを作成し、 `virt-install`スクリプトを作成して展開します。

```bash
# 저장소 디렉토리 생성
mkdir -p /var/morpheus/kvm/vms/arbiter

# VM 생성 스크립트 작성 (arbiter.sh)
vi arbiter.sh
```

** `arbiter.sh`スクリプトの内容**：
```bash
sudo virt-install --name arbiter --ram 4096 --vcpus 2 --cpu host-passthrough \
--disk path=/var/morpheus/kvm/vms/arbiter/arbiter.qcow2,size=40,format=qcow2,bus=virtio \
--cdrom /var/lib/libvirt/images/ubuntu-22.04.5-live-server-amd64.iso \
--network network=Management,model=virtio \
--graphics vnc,listen=0.0.0.0,password=P@ssw0rd \
--boot uefi \
--noautoconsole --autostart
```

スクリプトの実行：
```bash
sh -x arbiter.sh
```

**出力例**：
```text
Starting install...
Allocating 'arbiter.qcow2' ...
Creating domain...
Domain is still running. Installation may be in progress.
```

---

## 3. VNC ClientでOSをインストールする

VNCディスプレイポートを確認し、VNCビューア（MobaXtermなど）を介してOSのインストールを進めます。

```bash
# VNC 포트 확인
virsh vncdisplay arbiter
# 출력 예시: :1 -> VME Manager 서버 IP:5901 로 접속 (기본 포트 5900 + 1)
```

---

## 4. OSインストール完了後のVMの状態の確認と起動

インストールが完了してVMがシャットオフ状態になると、自動起動設定後にドライブします。

```bash
# VM 리스트 확인
virsh list --all
```

**出力例**：
```text
root@vmemgr:/home/vmeadmin# virsh list --all
 Id   Name      State
--------------------------
 1    vmemgr    running
 -    arbiter   shut off
```

`arbiter`ステータスが`shut off`と表示された場合の自動起動の設定と起動：
```bash
virsh autostart arbiter
virsh start arbiter
```

---

## 5. Virsh Console 接続を有効にする

`virsh console`コマンドで直接VMターミナルにアクセスできるようにGRUB設定を変更します。

```bash
# GRUB 설정 수정
vi /etc/default/grub

# 아래 항목 추가/수정
GRUB_CMDLINE_LINUX="console=ttyS0"

# GRUB 업데이트
update-grub
```

設定後は以下のコマンドで直接接続可能です。
```bash
virsh console arbiter
```

---

## 6. Arbiter パッケージのインストール

Arbiterインストールパッケージ（ `svtarb`）をインストールします。

```bash
dpkg -i ./svtarb_6.0.0.39_amd64.deb
```

**出力と応答の例**：
```text
Do you accept the End User License Agreement (y/n) y
Certificate request self-signature ok
subject=CN = arbiter
Created symlink /etc/systemd/system/multi-user.target.wants/svtarb.service -> /lib/systemd/system/svtarb.service.
```

---

## 💡コアポイントとヒント

1. **ISOパス**： `/var/lib/libvirt/images/`にあります
2. **VM ディスクパス**: `/var/morpheus/kvm/vms/arbiter/` に作成
3. **VNC接続**: MobaXtermなどのVNC Clientを利用して5901ポートで接続
4. **コンソール接続**: `GRUB_CMDLINE_LINUX="console=ttyS0"` 設定で `virsh console` 便利に管理
5. **自動登録**：Arbiterパッケージ（ `svtarb`）をインストールすると、 `systemd`サービスとして自動的に登録されます
