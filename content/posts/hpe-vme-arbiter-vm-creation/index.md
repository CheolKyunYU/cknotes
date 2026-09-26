---
title: "[HPE SimpliVity & VME] 라이선스 비용 없이 KVM CLI로 Arbiter VM 수동 구축하기 (16년 차 엔지니어의 현장 우회 노하우)"
description: "SimpliVity 2노드 클러스터 구축 전 필수인 Arbiter를 VME Manager 라이선스 코어 카운트 차감 없이, KVM virt-install과 VNC로 무과금 수동 생성하는 실무 우회 가이드입니다."
date: 2026-01-07T10:00:00+09:00
draft: false
tags: ["HPE", "SimpliVity", "VME", "Arbiter", "Linux", "Ubuntu", "KVM", "virsh", "라이선스", "트러블슈팅"]
aliases:
  - /posts/hpe-vme-arbiter-vm-creation/
categories:
  - SimpliVityVME
---

> **글쓴이**: 16년 차 IT 시스템 엔지니어 (CK notes)  
> **대상 환경**: HPE SimpliVity 6.2.0, HPE VM Essentials (VME), KVM / libvirt 기반 관리 호스트  
> **아비터 OS**: Ubuntu 22.04.5 LTS (Server)

---

## 1. 배경: 현장에서 마주하는 '닭과 달걀의 딜레마'와 '라이선스 함정'

HPE SimpliVity with VME (VM Essentials) 2노드 클러스터를 납품하러 현장에 나가면, 십중팔구 고객사 전산 담당자에게서 똑같은 요구를 받게 됩니다:

> *"이번에 새로 들어온 고성능 SimpliVity 서버 2대에 아비터(Arbiter) VM도 같이 올려서 다 알아서 돌려주세요."*

하지만 인프라를 직접 구축해 본 실무 엔지니어라면 이 요구가 마주하는 **치명적인 딜레마 2가지**를 즉시 떠올릴 수밖에 없습니다.

### 딜레마 1: SimpliVity 스토리지 생성 전 아비터가 먼저 살아있어야 한다
2노드 SimpliVity 클러스터는 **배포 위저드가 돌아가기 전에 아비터(Arbiter) IP와 통신이 되어야만** 클러스터 생성이 진행됩니다. 그런데 아직 노드 초기화도 안 끝났고 스토리지 풀도 생성되지 않은 상태에서 SimpliVity 위에 아비터 VM을 어떻게 올릴 수 있을까요? 완벽한 **닭과 달걀의 모순**입니다. 

게다가 아비터는 2개 노드가 장애로 통신이 끊겼을 때 스플릿 브레인(Split-Brain)을 판정해 주는 '재판관'이므로, **절대로 본인이 판정해야 할 SimpliVity 내부 스토리지에 올라가면 안 됩니다.**

### 딜레마 2: VME GUI에서 만들면 유료 라이선스 비용이 청구된다
그렇다면 아비터는 어디에 올려야 할까요? 정답은 **VME Manager가 구동되고 있는 외부 독립 관리 호스트(KVM 기반)**입니다.

그런데 여기서 큰 문제가 생깁니다. 많은 엔지니어들이 편하게 작업하려고 VME Manager 웹 GUI 콘솔에서 관리 호스트를 정식 등록하고 아비터 VM을 생성하려 합니다. 하지만 VME Manager 정책상, **콘솔에 호스트를 등록하는 순간 해당 물리 서버의 모든 CPU 코어가 VME 유료 라이선스 카운트에 포함**됩니다! 아비터 하나 띄우자고 수백~수천만 원 상당의 소프트웨어 라이선스를 추가 구매할 수는 없는 노릇입니다.

---

## 2. 16년 차 엔지니어의 실무 우회 해법: KVM CLI + VNC 조합

해법은 의외로 간단하고 강력합니다.  
VME Manager 호스트의 밑바탕 OS(BaseOS)는 **표준 리눅스 KVM / libvirt 기반**입니다.

즉, VME Manager GUI를 통하지 않고, **호스트 CLI 레벨에서 직접 `virt-install` 명령어로 KVM 가상머신을 생성**하면 VME 라이선스 관리 영역에 전혀 잡히지 않습니다. 그리고 설치 GUI 화면은 `virsh vncdisplay`와 VNC Viewer(MobaXterm)를 이용해 띄우면 10분 만에 **라이선스 소모 0원의 완벽한 독립 Arbiter VM**이 완성됩니다.

---

## 3. 환경 및 사전 조건

| 항목 | 요구 스펙 / 설정값 | 실무 설정 이유 |
| --- | --- | --- |
| **관리 호스트 OS** | Ubuntu 22.04 LTS / KVM BaseOS | VME Manager 구동 기본 하이퍼바이저 환경 |
| **Arbiter VM OS** | Ubuntu 22.04.5 LTS (Server) | SimpliVity 6.2.0 공식 호환 OS |
| **vCPU / RAM** | 2 vCPU / 4 GB RAM | Arbiter 데몬 쿼럼 판정용 초경량 자원 요구사항 |
| **Disk 공간** | 40 GB (qcow2 format) | Thin Provisioning으로 호스트 디스크 용량 최소화 |
| **네트워크** | Management Bridge (1GbE/10GbE) | SimpliVity 노드 및 VME Manager 간 통신 보장 |

---

## 4. 실전 구축 절차

### Step 1. Ubuntu 22.04 LTS ISO 배치
SimpliVity Arbiter는 공식적으로 Windows Server 또는 **Ubuntu 22.04 LTS**만 지원합니다. 관리 호스트에 최신 Ubuntu 22.04 서버 ISO를 배치합니다.

```bash
# ISO 이미지를 libvirt 기본 이미지 경로로 복사 (libvirt 권한 접근 보장)
cp ubuntu-22.04.5-live-server-amd64.iso /var/lib/libvirt/images/

# 파일 배치 확인
ls -lh /var/lib/libvirt/images/
```

### Step 2. `virt-install` 스크립트로 VM 생성
VM 디스크가 저장될 전용 디렉터리를 만들고, KVM 가상머신 생성 스크립트를 작성합니다.

```bash
# VM 이미지 전용 디렉터리 생성 (기존 VME 가상머신 경로와 격리)
mkdir -p /var/morpheus/kvm/vms/arbiter

# 생성 스크립트 작성
vi /root/create_arbiter.sh
```

#### `create_arbiter.sh` 스크립트 내용
```bash
#!/bin/bash
sudo virt-install   --name arbiter   --ram 4096   --vcpus 2   --cpu host-passthrough   --disk path=/var/morpheus/kvm/vms/arbiter/arbiter.qcow2,size=40,format=qcow2,bus=virtio   --cdrom /var/lib/libvirt/images/ubuntu-22.04.5-live-server-amd64.iso   --network network=Management,model=virtio   --graphics vnc,listen=0.0.0.0,password=P@ssw0rd   --boot uefi   --noautoconsole   --autostart
```

> 💡 **각 설정 파라미터의 실무적인 이유**:
> * `--cpu host-passthrough`: 호스트 CPU 명령어 세트를 그대로 전달하여 KVM 가상화 성능 손실 예방
> * `--disk ... format=qcow2,bus=virtio`: virtio 블록 드라이버 사용으로 디스크 I/O 병목 방지
> * `--graphics vnc,listen=0.0.0.0`: 원격 랙 작업 시 로컬 PC의 VNC 클라이언트로 설치 화면에 바로 접근 가능
> * `--autostart`: 관리 서버 재부팅 시 Arbiter VM이 수동 개입 없이 자동 기동되도록 설정

스크립트 실행:
```bash
sh -x /root/create_arbiter.sh
```

### Step 3. VNC 포트 확인 및 OS 설치 진행
VM이 백그라운드에서 실행되었으니, 화면을 보면서 Ubuntu 설치를 진행합니다.

```bash
# VNC 바인딩 포트 번호 확인 (5900 + 포트번호로 매핑됨)
virsh vncdisplay arbiter
```

출력 결과가 `:1`로 나오면 `5901` 포트로 열린 것입니다. MobaXterm이나 RealVNC로 `관리서버IP:5901`에 접속하여 고정 IP를 할당하고 기본 설치를 마칩니다.

### Step 4. 설치 완료 후 자동 시작 및 Serial Console (ttyS0) 개방
설치가 끝나고 VM 내에서 GRUB 설정을 수정해 호스트 CLI에서 `virsh console`로 바로 붙을 수 있게 설정합니다.

```bash
# /etc/default/grub 파일 수정
sudo vi /etc/default/grub

# 시리얼 콘솔 출력 바인딩 추가 (네트워크 장애 시에도 호스트 CLI에서 직접 접근 가능)
GRUB_CMDLINE_LINUX="console=ttyS0"

# GRUB 적용 후 호스트에서 자동 기동 확인
sudo update-grub
```

```bash
# 호스트에서 자동 시작 등록 및 전원 켜기
virsh autostart arbiter
virsh start arbiter
```

### Step 5. HPE SimpliVity Arbiter 패키지 설치
Arbiter VM 내부 접속 후 공식 `svtarb` 패키지를 설치합니다.

```bash
# Arbiter deb 패키지 설치 (라이선스 동의 및 데몬 등록)
sudo dpkg -i ./svtarb_6.0.0.39_amd64.deb
```

---

## 5. 🚨 트러블슈팅 노트 (현장 실무 에러 대처)

### 이슈 1: `virt-install` 실행 시 `Network 'Management' not active` 에러 발생
* **에러 메시지**: `error: Network 'Management' is not active`
* **원인 추정**: KVM 네트워크 가상 스위치 데몬이 서버 부팅 직후 활성화되지 않았거나 비활성화 상태임.
* **해결 방법**:
  ```bash
  # KVM Management 네트워크 강제 기동 및 자동 시작 설정
  sudo virsh net-start Management
  sudo virsh net-autostart Management
  ```

### 이슈 2: VNC 접속 시 화면이 검은색(Black Screen)으로 멈추는 현상
* **에러 메시지**: VNC 연결은 성공하나 설치 화면이 뜨지 않음
* **원인 추정**: `--boot uefi` 설정 시 콘솔 그래픽 출력이 기본 디스플레이 드라이버와 미스매치 발생.
* **해결 방법**: `create_arbiter.sh`에서 `--graphics vnc` 항목 뒤에 `--video virtio` 파라미터를 추가하여 재생성하거나, VNC 대신 `virsh console`로 접근하여 설치 진행.

---

## 6. 최종 검증 방법 (작업 완료 확인)

작업이 제대로 되었는지 현장에서 다음 3가지 포인트로 검증합니다:

1. **호스트 KVM 상태 및 자동 기동 검증**:
   ```bash
   virsh list --all
   # 결과: arbiter | running 상태 및 Autostart: enable 확인
   ```
2. **Arbiter 서비스 데몬 및 포트 바인딩 검증**:
   Arbiter VM 내부에서 데몬이 정상 작동하는지 확인합니다.
   ```bash
   sudo systemctl status svtarb
   # netstat으로 통신 포트(8080 / 443) 상태 확인
   sudo ss -tlpn | grep svtarb
   ```
3. **SimpliVity 노드 간 Ping 및 핑퐁 통신 검증**:
   SimpliVity 호스트 2대에서 Arbiter VM IP로 `ping` 및 `nc -zv <Arbiter_IP> 8080` 포트 통신 테스트를 수행하여 무응답/방화벽 블록이 없는지 최종 검증합니다.

---

## 7. 마무리 및 현장 총평

* **한눈에 보는 핵심**:
  - Arbiter는 2노드 SimpliVity 스플릿 브레인 방지 재판관이므로 **외부 독립 호스트**에 올려야 함.
  - VME Manager GUI 등록 대신 **KVM CLI (`virt-install`)**를 쓰면 라이선스 소모 0원으로 구축 가능.
  - 설치 후 `autostart` 및 `ttyS0` 시리얼 콘솔 개방은 유지보수 필수 절차.

* **관련 글 예고**:
  - 다음 글에서는 이렇게 준비된 Arbiter를 가지고 **[HPE SimpliVity 2노드 클러스터 배포 마스터 가이드]**로 이어집니다.
