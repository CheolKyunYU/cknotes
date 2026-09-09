---
title: "Comparison of HPE SimpliVity VME storage architecture and SimpliVity on VMware"
description: "Compare the licensing savings and software-defined storage strengths of HPE SimpliVity's RAID+RAIN dual protection architecture, NFS datastores, and VME versus VMware-based."
date: 2025-12-31T22:00:00+09:00
draft: false
categories: ["Tech"]
tags: ["HPE", "SimpliVity", "VME", "Storage", "VMware", "NFS", "RAID", "RAIN"]
---


## 1. Overview

HPE SimpliVity is a solution that provides integrated computing, storage, and network based on hyperconverged infrastructure (HCI).

The storage architecture is based on **RAID + RAIN structure** and **NFS protocol**, and VMware-based SimpliVity and VME have similarities and differences.

---

## 2. Storage architecture core composition

### 1) RAID + RAIN structure
* **RAID (Local)**: Configure RAID inside each node to prepare for disk failure
* **RAIN (Cluster)**: Data replication to other nodes in the cluster → Data integrity is guaranteed even when node failures
* Double protection structure ensures high level of **high availability (HA)**

### 2) NFS protocol based data store
* Use **NFS datastore** in a VMware ESXi or VME environment
* OmniStack (based on VMware) or VME software virtualizes local storage to provide a **logical pool of storage**
* Virtual machines (VMs) operate reliably through this NFS datastore.

### 3) Data efficiency
* Reduce storage space through **Deduplication** + **Compression** technology
* Provides built-in backup and recovery functions → Self-DR (disaster recovery) possible without external storage

---

## 3. VMware-based SimpliVity vs HPE SimpliVity VME comparison table

| item | SimpliVity on VMware | HPE SimpliVity VME |
| --- | --- | --- |
| **Storage Protocol** | NFS-based | NFS-based |
| **How data is protected** | RAID + RAIN + OmniStack Acceleration Card | RAID + RAIN (Software Defined) |
| **Hardware Dependencies** | OmniStack Accelerator Card required | No additional hardware required, based on standard x86 servers |
| **Management Platform** | VMware vCenter integration | VME Manager + Morpheus (can be managed simultaneously with VMware) |
| **Scalability** | Expandable by adding nodes | Equally node-based scalability |
| **Backup/Restore** | Built-in features provided | Provides the same built-in features |
| **Cost Structure** | VMware License + OmniStack Card Cost | Socket-based licensing (up to 70% savings compared to VMware) |

---

## 4. Additional advantages of VME architecture

* **Hardware simplification**: Reduce points of failure and ease maintenance by eliminating OmniStack cards
* **Software-defined approach**: Utilizes the latest OS (based on Ubuntu) and open source technology to improve flexibility and update speed
* **IPv6 and modern network support**: Optimized for next-generation network environments
* **Management efficiency**: Morpheus integration provides a hybrid operating environment that can manage VMware and VME simultaneously
* **Cloud-friendliness**: Enhanced on-premises + cloud scalability through HPE GreenLake integration

---

## 5. Key takeaways

* **Common features**: RAID + RAIN structure, NFS protocol-based datastore, built-in backup/recovery function
* **Difference**: VMware-based requires OmniStack card, VME is software-defined for greater simplicity and cost-effectiveness

---

## 6. Reference sources

* [HPE SimpliVity official product page](https://www.hpe.com/us/en/integrated-systems/simplivity.html)
* [HPE VME Release Notes Documentation](https://support.hpe.com/hpesc/public/docDisplay?docId=a00156081en_us&docLocale=en_US)
* [HPE InfoSight Solutions Guide](https://www.hpe.com/us/en/solutions/infosight.html)
