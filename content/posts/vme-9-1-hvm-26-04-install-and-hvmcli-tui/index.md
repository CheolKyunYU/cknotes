---
title: "[HPE VME 9.1.0] 신규 HVM OS 26.04 설치 & hvmcli TUI 기반 VME Manager 배포 가이드 (Step 1)"
description: "HPE VME 9.1.0에서 변경된 HVM OS 26.04 설치 절차, Cluster Layout 2.0 지원, Linux Bridge(vs0) 네트워크 스택 전환, 그리고 신규 hvmcli TUI를 활용한 VME Manager 배포 셋업 실무 가이드입니다."
date: 2026-11-27T22:15:00+09:00
draft: false
categories:
  - SimpliVityVME
tags:
  - HPE
  - VME
  - HVM
  - hvmcli
  - Morpheus
  - KVM
---

HPE의 차세대 엔터프라이즈 프라이빗 클라우드 가상화 플랫폼인 **HPE VM Essentials(VME) 9.1.0**이 정식 릴리즈되었습니다. 이번 9.1.0 업데이트는 단순한 마이너 버전 패치가 아니라, 하이퍼바이저 베이스 OS부터 가상 네트워킹 아키텍처, 그리고 관리자 배포 툴킷까지 전면적으로 개편된 대규모 메이저 성격의 릴리즈입니다.

엔지니어링 관점에서 이번 VME 9.1.0 버전에서 반드시 주목해야 할 아키텍처 변경점과 실무 핵심 포인트를 정리하고, 베이스 OS인 **HVM OS 26.04 설치**부터 새롭게 도입된 **`hvmcli tui` 콘솔**을 통한 **VME Manager(Morpheus) 배포 1단계 셋업** 전 과정을 상세히 살펴봅니다.

---

### 1. HPE VME 9.1.0 핵심 아키텍처 변경점

기존 VME나 SimpliVity HVM 환경을 운영해 본 엔지니어라면 이번 9.1.0 버전에서 바뀐 다음 세 가지 핵심 사항을 반드시 숙지해야 합니다.

| 구분 | 이전 방식 (HVM 24.04 기반) | 신규 방식 (HVM 26.04 / VME 9.1.0) | 엔지니어 관점의 실무 의미 |
| :--- | :--- | :--- | :--- |
| **클러스터 레이아웃** | **Layout 1.3** | **Layout 2.0 (26.04 전용)** | 9.1.0을 기존 24.04 위에 올리면 레이아웃이 1.3에 머무르며, **HVM 26.04로 신규 설치해야 차세대 Layout 2.0 활성화 가능** |
| **가상 네트워크 스택** | **OVS (Open vSwitch)** | **`vs` (Linux Bridge 기반)** | 복잡한 플로우 룰 오버헤드가 있던 OVS 의존성을 탈피하고, 직관적이고 경량화된 표준 Linux Bridge(`vs0`) 체계로 전면 개편 |
| **VME Manager 배포 툴** | **`hpe-vm` 콘솔 명령어** | **`hvmcli tui`** 또는 **Windows 마법사** | 기존 `hpe-vm` 명령어가 완전히 폐지되고, 터미널 TUI 환경인 **`hvmcli tui`**와 별도 GUI 환경용 **Windows 배포 위저드** 2-Track 제공 |

#### 왜 굳이 HVM OS 26.04로 새로 설치해야 하는가?
HPE VME 9.1.0 소프트웨어 자체는 기존 HVM OS 24.04 버전에서도 구동할 수 있습니다. 하지만 24.04 위에 설치할 경우 클러스터 레이아웃 구조가 구형인 **Layout 1.3**으로 제한됩니다. VME 9.1.0의 최신 스토리지 오케스트레이션과 네이티브 최적화 기능을 온전히 활용하기 위해서는 반드시 **HVM OS 26.04 설치 미디어를 사용하여 클러스터 레이아웃 2.0(Layout 2.0)**으로 인프라를 구축해야 합니다.

#### OVS에서 Linux Bridge(`vs`)로의 전환
이전 버전까지는 가상 머신 트래픽 격리와 관리를 위해 Open vSwitch(OVS)를 기본 네트워크 스택으로 사용했습니다. 그러나 9.1.0(HVM 26.04)부터는 관리 복잡성을 줄이고 시스템 오버헤드를 최소화하기 위해 **표준 Linux Bridge 기반의 `vs` (Virtual Switch) 방식**으로 전환되었습니다.

#### `hpe-vm`의 폐지와 `hvmcli tui`의 도입
기존 VME Manager 배포 시 사용하던 콘솔 명령어 `hpe-vm`은 완전히 제거되었습니다. 대신 호스트 쉘에서 **`hvmcli tui`** 명령어를 입력하면 아스키 아트 기반의 직관적인 **HPE VM Console v1.0.0**이 실행됩니다. 윈도우용 배포 마법사 프로그램도 지원되지만, 데이터센터 현장 폐쇄망이나 원격 콘솔 작업에서는 로컬 ISO 파일을 직접 마운트하여 배포할 수 있는 `hvmcli tui`가 훨씬 빠르고 안정적입니다.

---

### Step 1. HVM OS 26.04 GRUB 부팅 및 언어 설정

서버 콘솔(iLO 등)에 `HVM_OS_26.04` 설치 미디어를 마운트하고 UEFI 모드로 부팅을 진행합니다.

부팅 직후 GNU GRUB 화면이 나타나며 첫 번째 항목으로 **`HVM Install 26.04`**가 표시됩니다. 엔터를 눌러 설치 환경으로 진입합니다.

{{< figure src="images/01_grub_hvm_install_26_04.jpg" caption="HVM OS 26.04 GNU GRUB 부팅 메뉴 화면" >}}

부팅 커널이 로드되면 최신 Ubuntu Subiquity 기반의 **HPE HVM Installer** 화면이 실행됩니다. 기본 언어로 **`[ English (US) ]`**를 선택하고 다음 단계로 진행합니다.

{{< figure src="images/02_installer_language_select.jpg" caption="HPE HVM Installer 언어 선택 화면" >}}

---

### Step 2. 관리 네트워크 본딩(bond0) 및 고정 IP 구성

엔터프라이즈 하이퍼바이저 호스트이므로 물리 네트워크 인터페이스의 고가용성(HA)을 위해 본딩(Bonding) 인터페이스를 구성합니다.

인스톨러 네트워크 화면에서 **`[ Create Bond ]`** 항목을 선택한 후 파라미터를 입력합니다.

* **Name**: `bond0`
* **Devices**: 사용할 물리 인터페이스 선택 (예: `[X] ens33`)
* **Bond mode**: **`active-backup`** (또는 환경에 따라 `802.3ad` LACP 선택)
* **XMIT hash policy**: `layer2`
* **LACP rate**: `slow`

{{< figure src="images/03_network_create_bond0.jpg" caption="관리 네트워크 본딩(bond0) 인터페이스 생성 화면" >}}

생성된 `bond0` 인터페이스를 기본 통신 카드로 지정하고, 호스트 관리용 고정 IP 정보와 시스템 호스트명을 입력합니다.

* **IP Address/CIDR**: 호스트 관리 IP 입력 (예: `192.168.126.200/24`)
* **Gateway**: 게이트웨이 주소 입력 (예: `192.168.126.1`)
* **Hostname**: 호스트명 지정 (예: `hvm01`)
* **DNS Servers**: 네임서버 주소 입력 (예: `8.8.8.8`)

{{< figure src="images/04_network_ip_and_hostname_config.jpg" caption="호스트 고정 IP, 게이트웨이 및 호스트명 설정 화면" >}}

---

### Step 3. Subiquity 스토리지 파티셔닝 & BaseOS 설치 완료

네트워크 설정이 완료되면 디스크 파티셔닝 단계로 넘어갑니다. OS가 설치될 로컬 디스크를 지정하면 데이터 삭제 확인 경고 창이 팝업됩니다.

**`[ Continue ]`**를 선택하여 파티션 포맷 및 OS 패키지 설치를 시작합니다.

{{< figure src="images/05_confirm_destructive_action.jpg" caption="디스크 포맷 및 설치 진행 확인 경고 창 (Confirm destructive action)" >}}

백그라운드에서 Curtin 설치 엔진이 구동되며 파티션 포맷, 부트로더(`grub-efi-amd64-signed`, `shim-signed`), 커널 패키지, 그리고 NVMe over TCP 및 iSCSI 서비스 설정을 자동으로 진행합니다.

설치가 완료되면 하단의 **`[ Reboot Now ]`**를 눌러 시스템을 재부팅합니다.

{{< figure src="images/06_curtin_os_installation_progress.jpg" caption="Curtin 설치 엔진을 통한 OS 패키지 및 커널 드라이버 구성 화면" >}}

설치 마무리 단계(또는 초기 부팅 과정)에서 호스트 시스템의 최고 관리자 계정 정보를 등록합니다.

* **Enter Administrator Username**: `hvmadmin`
* **Enter Administrator Password**: 관리자 비밀번호 입력
* **Confirm Administrator Password**: 비밀번호 재확인

{{< figure src="images/07_admin_account_creation.jpg" caption="호스트 최고 관리자(hvmadmin) 계정 생성 화면" >}}

---

### Step 4. 신규 hvmcli TUI (HPE VM Console) 실행

HVM OS 26.04 설치가 완료되고 재부팅되면 콘솔 또는 SSH를 통해 `hvmadmin` 계정으로 로그인한 뒤 root 권한을 획득합니다.

VME Manager 소프트웨어 배포 이미지인 Morpheus ISO 파일(`HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso`)이 위치한 디렉토리를 확인하고, 터미널에서 신규 관리 명령어를 실행합니다.

```bash
root@hvm01:/home/hvmadmin# ls
HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso

root@hvm01:/home/hvmadmin# hvmcli tui
```

{{< figure src="images/08_terminal_morpheus_iso_and_hvmcli_tui.jpg" caption="Morpheus 매니저 ISO 확인 및 hvmcli tui 명령어 실행" >}}

명령어를 실행하면 터미널 화면에 아스키 아트 HPE 로고가 새겨진 **HPE VM Console (v1.0.0)** TUI 창이 나타납니다.

메뉴 구성은 다음과 같이 호스트 인프라 관리 및 VM 배포 항목으로 세분화되어 있습니다:
* **`<Host Information>`**: 호스트 하드웨어, CPU, 메모리 자원 현황
* **`<Virtual Switch>`**: Linux Bridge 기반 가상 스위치(`vs0` 등) 확인 및 포트 매핑
* **`<Virtual Machines>`**: 구동 중인 KVM 가상머신 목록 조회 및 전원 제어
* **`<Keyboard Layout / TimeZone>`**: 키보드 및 타임존 설정
* **`<Install VME Manager>`**: **VME Manager VM 자동 배포 마법사 (핵심)**
* **`<Install VME Worker>`**: 분산 처리를 위한 VME Worker 노드 설치
* **`<Open Shell>`**: 콘솔 일시 탈출 및 Bash 쉘 진입

{{< figure src="images/09_hpe_vm_console_tui_main_menu.jpg" caption="신규 도입된 HPE VM Console (v1.0.0) TUI 메인 메뉴 화면" >}}

---

### Step 5. TUI를 통한 VME Manager VM 설정 및 자동화 배포 파이프라인

메인 메뉴에서 **`<Install VME Manager>`** 항목을 선택하여 VME 관리 VM 배포 설정 창으로 진입합니다.

**VME Manager Installation Options** 화면에서 관리 VM의 네트워크 및 하이퍼바이저 할당 옵션을 정의합니다:

* **VM Config Options**:
  * **IP Address**: VME Manager 웹 콘솔 접속용 IP (예: `192.168.55.201`)
  * **Netmask**: 서브넷 마스크 (예: `255.255.255.0`)
  * **Gateway**: 기본 게이트웨이 주소 (예: `192.168.55.1`)
  * **DNS Server**: 네임서버 주소 (예: `8.8.8.8`)
  * **Appliance URL**: `https://192.168.55.201`
  * **Hostname**: `vme`
  * **Admin User**: 관리자 계정명 (기본값: `vmeadmin`)
  * **Admin Password**: 관리자 접속 비밀번호 설정
  * **Image URI**: Morpheus 배포 ISO 파일 경로 지정 (예: `file:///mnt/HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso`)
  * **Select VM Size**: 가상머신 프로파일 규격 선택 (테스트 환경: `Small`)
* **Host Config Options**:
  * **Virtual Switch**: 신규 Linux Bridge 가상 스위치 지정 (기본값: **`vs0`**)
  * **VLAN ID**: 관리 네트워크 VLAN 태깅 필요 시 입력

설정값을 모두 확인한 후 하단의 **`<Install>`** 버튼을 누릅니다.

{{< figure src="images/10_vme_manager_installation_options.jpg" caption="VME Manager 가상머신 네트워크, 자원 규격 및 가상 스위치(vs0) 옵션 설정 화면" >}}

설치가 시작되면 **VME Manager Deployment** 프로세스가 단계별로 순차 진행됩니다.

배포 엔진은 관리자가 입력한 파라미터를 기반으로 다음 13단계 자동화 파이프라인을 실행합니다:
1. `[✓] Validating deployment parameters` (입력 파라미터 유효성 검사)
2. `[✓] Running pre-flight checks` (호스트 시스템 요구사항 점검)
3. `[✓] Preparing KVM hypervisor` (KVM 가상화 모듈 로드)
4. `[✓] Configuring management network` (관리 네트워크 브릿지 연동)
5. `[✓] Configuring compute network` (컴퓨트 네트워크 구성)
6. `[✓] Creating local storage pool` (VM 디스크용 로컬 스토리지 풀 생성)
7. `[⌛] Downloading QCOW2 image` (ISO 내부의 QCOW2 템플릿 추출 및 변환)
8. `[ ] Building cloud-init configuration` (Cloud-init 초기화 데이터 생성)
9. `[ ] Defining virtual machine` (libvirt KVM VM 정의)
10. `[ ] Starting virtual machine` (가상머신 기동)
11. `[ ] Waiting for VME services to start` (Morpheus 관리 서비스 구동 대기)
12. `[ ] Running post-deploy health checks` (배포 후 건전성 점검)
13. `[ ] Cleaning up temporary files` (임시 설치 파일 정리)

{{< figure src="images/11_vme_manager_deployment_start_7pct.jpg" caption="VME Manager 자동화 배포 시퀀스 시작 화면 (Progress 7% - KVM 점검)" >}}

배포 엔진이 로컬 스토리지 풀 구성을 마치고 QCOW2 이미지를 변환/다운로드하는 40% 단계까지 정상적으로 진행되는 것을 확인할 수 있습니다.

{{< figure src="images/12_vme_manager_deployment_download_qcow2_40pct.jpg" caption="로컬 스토리지 풀 생성 후 QCOW2 이미지 배포 단계 진행 화면 (Progress 40%)" >}}

---

### 💡 엔지니어 현장 Note: VME 9.1.0 배포 방식 및 랩(Lab) 환경 가이드

#### 1. VME Manager 배포 도구: `hvmcli tui` vs `Windows 배포 위저드`
HPE VME 9.1.0은 Windows 환경에서 실행할 수 있는 GUI 기반의 **Windows 배포 마법사(Wizard)**를 함께 제공합니다. 
* 윈도우 마법사는 GUI 환경이 익숙한 관리자에게 편리하지만, 원격 PC에서 HVM 호스트로 수 기가바이트에 달하는 관리자 이미지를 네트워크로 밀어 넣어야 하므로 대역폭 제한이나 네트워크 단절에 따른 배포 실패 위험이 있습니다.
* 반면 **`hvmcli tui`**는 HVM 호스트 로컬 디스크나 마운트된 스토리지에 위치한 ISO 파일(`file:///...`)을 직접 참조하므로, 네트워크 전송 병목 없이 가장 안전하고 빠른 배포가 가능합니다. 폐쇄망 데이터센터 환경이라면 무조건 `hvmcli tui` 방식을 권장합니다.

#### 2. VMware Workstation 중첩 가상화(Nested Lab) 환경의 주의점
개발 테스트나 사전 검증을 위해 VMware Workstation 상에 HVM OS를 올리고 VME Manager를 배포할 경우 다음 사항을 염두에 두어야 합니다:
* **중첩 가상화 활성화**: VMware Workstation VM 설정에서 CPU의 `Virtualize Intel VT-x/EPT or AMD-V/RVI` 옵션이 반드시 켜져 있어야 Step 5의 `/dev/kvm present` 사전 점검을 통과할 수 있습니다.
* **하드웨어 자원 한계**: VME Manager는 내부적으로 고성능 프라이빗 클라우드 오케스트레이션 엔진(Morpheus)과 데이터베이스, 모니터링 데몬이 구동되므로 최소 4~8 vCPU와 16GB~32GB 이상의 메모리를 필요로 합니다.
* VMware Workstation 기반의 중첩 환경에서는 가상 디스크 I/O 병목 및 메모리 부족으로 인해 QCOW2 이미지 전개 및 Cloud-init 부팅 과정(40%~60% 구간)에서 타임아웃이 발생할 수 있습니다. 완벽한 프로덕션 배포 및 클러스터 구성은 실제 ProLiant Gen10 Plus 또는 Gen11 물리 베어메탈 서버에서 진행하는 것이 이상적입니다.

---

### 다음 단계 예고
베이스 OS인 HVM 26.04 구축과 VME Manager VM의 배포 파라미터 구성이 완료되었습니다. 

다음 포스팅인 **Step 2**에서는 정상적으로 기동된 VME Manager의 웹 관리 콘솔(`https://<VME_IP>`)에 최초 접속하여 기본 라이선스 등록, 서브넷 및 스토리지 풀 셋업, 그리고 호스트 노드를 클러스터로 등록하는 상세 절차를 다루겠습니다.
