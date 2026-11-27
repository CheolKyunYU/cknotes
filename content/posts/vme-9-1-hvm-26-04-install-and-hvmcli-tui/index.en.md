---
title: "[HPE VME 9.1.0] Fresh HVM OS 26.04 Installation & hvmcli TUI VME Manager Deployment Guide (Step 1)"
description: "A comprehensive enterprise guide covering HVM OS 26.04 installation, Cluster Layout 2.0 enablement, Linux Bridge (vs0) network stack transition, and VME Manager deployment via the new hvmcli TUI."
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

HPE has officially released **HPE VM Essentials (VME) 9.1.0**, its next-generation private cloud virtualization platform for the enterprise. This 9.1.0 release is far more than a routine minor patch—it represents a major architectural overhaul encompassing the underlying hypervisor base OS, the virtual networking subsystem, and the deployment tooling suite.

This guide reviews the architectural shifts introduced in VME 9.1.0 from an engineering perspective, walking through the end-to-end installation of **HVM OS 26.04** and initial deployment parameter setup for **VME Manager (Morpheus)** using the newly introduced **`hvmcli tui` console**.

---

### 1. Key Architectural Changes in HPE VME 9.1.0

Engineers managing previous VME or SimpliVity HVM environments must take note of three fundamental architectural shifts in version 9.1.0:

| Category | Legacy Method (HVM 24.04 Base) | New Method (HVM 26.04 / VME 9.1.0) | Engineering & Practical Impact |
| :--- | :--- | :--- | :--- |
| **Cluster Layout** | **Layout 1.3** | **Layout 2.0 (26.04 Exclusive)** | Installing 9.1.0 atop 24.04 retains Layout 1.3. **A clean HVM OS 26.04 deployment is strictly required to activate Cluster Layout 2.0.** |
| **Virtual Network Stack** | **OVS (Open vSwitch)** | **`vs` (Linux Bridge Based)** | Eliminates Open vSwitch flow-rule overhead in favor of a lightweight, standard Linux Bridge (`vs0`) architecture. |
| **VME Manager Deployment Tool** | **`hpe-vm` CLI Command** | **`hvmcli tui`** or **Windows Wizard** | The `hpe-vm` command has been deprecated and removed. Deployment is now handled via **`hvmcli tui`** or a dedicated **Windows Deployment Wizard**. |

#### Why Must You Perform a Clean Install with HVM OS 26.04?
The HPE VME 9.1.0 software suite can technically run atop the legacy HVM OS 24.04 base. However, running on 24.04 restricts the cluster layout to **Layout 1.3**. To leverage advanced storage orchestration features and native optimization capabilities in VME 9.1.0, you must rebuild the host using the **HVM OS 26.04 installation media to unlock Cluster Layout 2.0**.

#### Transition from OVS to Linux Bridge (`vs`)
Earlier releases relied on Open vSwitch (OVS) as the default network virtualization stack for VM traffic isolation. Starting with 9.1.0 (HVM 26.04), the subsystem transitioned to **standard Linux Bridge virtual switches (`vs0`)**, reducing hypervisor overhead and simplifying network troubleshooting.

#### Deprecation of `hpe-vm` and Introduction of `hvmcli tui`
The legacy console command `hpe-vm` has been completely deprecated. Executing **`hvmcli tui`** from the host shell now launches **HPE VM Console v1.0.0**, an intuitive ASCII-based TUI. While a Windows-based graphical deployment wizard is also offered, `hvmcli tui` remains the fastest and most dependable method for datacenter environments, mounting local ISO images directly without network transfer overhead.

---

### Step 1. HVM OS 26.04 GRUB Boot & Language Selection

Mount the `HVM_OS_26.04` ISO image to the server console (e.g., HPE iLO Virtual Media) and boot in UEFI mode.

Upon boot, the GNU GRUB menu appears with **`HVM Install 26.04`** as the default entry. Press Enter to proceed.

{{< figure src="images/01_grub_hvm_install_26_04.jpg" caption="HVM OS 26.04 GNU GRUB Boot Menu" >}}

Once the installation kernel initializes, the Subiquity-based **HPE HVM Installer** loads. Select **`[ English (US) ]`** as the system language and continue.

{{< figure src="images/02_installer_language_select.jpg" caption="HPE HVM Installer Language Selection" >}}

---

### Step 2. Management Network Bonding (bond0) & Static IP Configuration

For enterprise hypervisor hosts, configuring network interface bonding ensures physical link redundancy and high availability (HA).

Select **`[ Create Bond ]`** on the network configuration screen and configure the parameters:

* **Name**: `bond0`
* **Devices**: Select the physical interfaces (e.g., `[X] ens33`)
* **Bond mode**: **`active-backup`** (or `802.3ad` LACP depending on your switch configuration)
* **XMIT hash policy**: `layer2`
* **LACP rate**: `slow`

{{< figure src="images/03_network_create_bond0.jpg" caption="Creating the Management Network Bond (bond0) Interface" >}}

Designate the newly created `bond0` interface as the primary management link and assign a static IP address, gateway, and hostname:

* **IP Address/CIDR**: Host management IP (e.g., `192.168.126.200/24`)
* **Gateway**: Default gateway (e.g., `192.168.126.1`)
* **Hostname**: System hostname (e.g., `hvm01`)
* **DNS Servers**: Primary DNS resolver (e.g., `8.8.8.8`)

{{< figure src="images/04_network_ip_and_hostname_config.jpg" caption="Configuring Host Static IP, Gateway, and Hostname" >}}

---

### Step 3. Subiquity Disk Partitioning & BaseOS Installation

After network configuration, proceed to disk selection. Specifying the target local installation disk triggers a data destruction confirmation warning.

Select **`[ Continue ]`** to format the target disk and initiate OS package installation.

{{< figure src="images/05_confirm_destructive_action.jpg" caption="Confirm Destructive Action Dialog for Disk Formatting" >}}

The Curtin installation engine executes in the background, handling partition formatting, bootloader installation (`grub-efi-amd64-signed`, `shim-signed`), kernel packages, and driver services for NVMe over TCP and iSCSI.

Once the process finishes, select **`[ Reboot Now ]`** at the bottom to reboot the system.

{{< figure src="images/06_curtin_os_installation_progress.jpg" caption="Curtin Installer Executing Kernel and Storage Driver Packages" >}}

During initial system setup, configure the primary administrator account:

* **Enter Administrator Username**: `hvmadmin`
* **Enter Administrator Password**: Specify a secure password
* **Confirm Administrator Password**: Re-enter the password

{{< figure src="images/07_admin_account_creation.jpg" caption="Configuring the Primary Administrator (hvmadmin) Account" >}}

---

### Step 4. Launching the New hvmcli TUI (HPE VM Console)

After rebooting into HVM OS 26.04, log in as `hvmadmin` via the console or SSH and switch to root.

Verify the presence of the Morpheus Manager software ISO (`HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso`), then execute the new management CLI command:

```bash
root@hvm01:/home/hvmadmin# ls
HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso

root@hvm01:/home/hvmadmin# hvmcli tui
```

{{< figure src="images/08_terminal_morpheus_iso_and_hvmcli_tui.jpg" caption="Verifying the Morpheus ISO Image and Executing hvmcli tui" >}}

The terminal opens the **HPE VM Console (v1.0.0)** TUI interface, displaying an ASCII-art HPE banner.

The menu provides clear management options for host infrastructure and VM operations:
* **`<Host Information>`**: Host hardware details, CPU, and RAM allocation
* **`<Virtual Switch>`**: Linux Bridge virtual switch (`vs0`) port and VLAN status
* **`<Virtual Machines>`**: KVM virtual machine inventory and power operations
* **`<Keyboard Layout / TimeZone>`**: System localization settings
* **`<Install VME Manager>`**: **Automated deployment wizard for VME Manager (Core)**
* **`<Install VME Worker>`**: VME Worker node deployment for distributed compute
* **`<Open Shell>`**: Exit back to the standard Bash shell

{{< figure src="images/09_hpe_vm_console_tui_main_menu.jpg" caption="HPE VM Console (v1.0.0) TUI Main Menu" >}}

---

### Step 5. VME Manager VM Parameter Configuration & Deployment Pipeline

Select **`<Install VME Manager>`** from the main menu to open the deployment configuration dialog.

In the **VME Manager Installation Options** screen, enter the network and hypervisor assignment parameters for the management VM:

* **VM Config Options**:
  * **IP Address**: Dedicated management IP for VME Manager (e.g., `192.168.55.201`)
  * **Netmask**: Subnet mask (e.g., `255.255.255.0`)
  * **Gateway**: Default gateway (e.g., `192.168.55.1`)
  * **DNS Server**: Primary DNS resolver (e.g., `8.8.8.8`)
  * **Appliance URL**: `https://192.168.55.201`
  * **Hostname**: `vme`
  * **Admin User**: Default administrator (`vmeadmin`)
  * **Admin Password**: Web console administrative password
  * **Image URI**: Local path to the Morpheus ISO image (e.g., `file:///mnt/HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso`)
  * **Select VM Size**: Instance specification profile (Test environment: `Small`)
* **Host Config Options**:
  * **Virtual Switch**: Virtual switch bridge (Default: **`vs0`**)
  * **VLAN ID**: Enter VLAN tag if required

After verifying the parameters, select **`<Install>`** to begin.

{{< figure src="images/10_vme_manager_installation_options.jpg" caption="Configuring Network, VM Sizing, and Virtual Switch (vs0) Parameters" >}}

The **VME Manager Deployment** workflow initiates automatically, executing a 13-stage automated pipeline:
1. `[✓] Validating deployment parameters`
2. `[✓] Running pre-flight checks`
3. `[✓] Preparing KVM hypervisor`
4. `[✓] Configuring management network`
5. `[✓] Configuring compute network`
6. `[✓] Creating local storage pool`
7. `[⌛] Downloading QCOW2 image` (Extracting and converting template from ISO)
8. `[ ] Building cloud-init configuration`
9. `[ ] Defining virtual machine`
10. `[ ] Starting virtual machine`
11. `[ ] Waiting for VME services to start`
12. `[ ] Running post-deploy health checks`
13. `[ ] Cleaning up temporary files`

{{< figure src="images/11_vme_manager_deployment_start_7pct.jpg" caption="Automated VME Manager Deployment Starting (Progress 7% - KVM Hypervisor Check)" >}}

The deployment engine creates the storage pool and proceeds through the 40% stage, extracting and converting the QCOW2 image disk template.

{{< figure src="images/12_vme_manager_deployment_download_qcow2_40pct.jpg" caption="Progressing Through QCOW2 Image Extraction and Storage Provisioning (Progress 40%)" >}}

---

### 💡 Engineering Field Notes: Deployment Strategies & Lab Considerations

#### 1. Deployment Tooling: `hvmcli tui` vs. `Windows Deployment Wizard`
HPE VME 9.1.0 includes a standalone graphical **Windows Deployment Wizard**.
* While the Windows wizard offers an intuitive GUI, it requires streaming multi-gigabyte ISO and disk templates over the network from a workstation to the HVM host, creating potential failure points across network boundaries.
* In contrast, **`hvmcli tui`** directly references local storage (`file:///...`) on the HVM host. This avoids remote network transmission bottlenecks, making it the preferred deployment method for secure and isolated enterprise datacenters.

#### 2. Nested Virtualization in VMware Workstation Labs
When deploying HVM OS and VME Manager inside VMware Workstation for validation or proof-of-concept testing, keep the following resource constraints in mind:
* **Nested VT-x/AMD-V**: In VMware Workstation VM settings, enable `Virtualize Intel VT-x/EPT or AMD-V/RVI` under Processor configuration to pass the `/dev/kvm present` pre-flight check in Step 5.
* **Hardware Demands**: VME Manager runs an enterprise private cloud orchestration engine (Morpheus), relational databases, and monitoring services, demanding at least 4–8 vCPUs and 16 GB to 32 GB of memory.
* Nested environments often encounter storage I/O and memory bottlenecks during the QCOW2 disk provisioning and Cloud-init initialization phases (between 40% and 60% progress). For production workloads, deploy directly on physical bare-metal HPE ProLiant Gen10 Plus or Gen11 servers.

---

### What's Next
With HVM OS 26.04 installed and initial deployment parameters configured via `hvmcli tui`, the foundation is set.

In **Step 2**, we will cover connecting to the VME Manager web console (`https://<VME_IP>`), licensing, initial storage and network pool creation, and adding hypervisor host nodes to the cluster.
