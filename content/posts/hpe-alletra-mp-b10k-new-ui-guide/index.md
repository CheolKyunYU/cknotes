---
title: "[HPE Alletra MP B10K] ArcusOS 10.6.0 신규 화이트 트리 웹 UI 분석 및 실무 가이드"
description: "HPE Alletra Storage MP B10K(B10120) 스토리지의 10.6.0 OS 신규 화이트 트리 UI 구조 분석, 6대 핵심 메뉴 체계 및 주요 관리 기능 실무 가이드."
date: 2026-09-30T07:00:00+09:00
draft: false
tags: ["HPE", "Alletra", "AlletraMP", "B10K", "B10120", "ArcusOS", "WebUI", "Storage", "GreenLake", "Tech"]
categories:
  - Storage
---

## 1. 개요: 10.6.0 전면 개편과 화이트 트리 UI의 도입 배경

HPE의 차세대 모듈러 올-NVMe 스토리지인 **HPE Alletra Storage MP (B10120 / B10K)** 환경에서 OS 펌웨어가 **10.6.0**으로 업그레이드되면서 시스템 관리 인터페이스가 전면적으로 개편되었습니다.

기존 10.5.x 버전까지 유지되던 단일 계층의 다크 사이드바 기반 UI에서 벗어나, 10.6.0부터는 **클라우드 네이티브 감성의 모던한 화이트 테마와 직관적인 다계층 트리 네비게이션 구조**가 새롭게 적용되었습니다.

이번 포스트에서는 실제 현장에 구축된 Alletra Storage MP B10K 장비를 기준으로, 10.6.0 신규 UI의 메뉴별 주요 변화점과 엔지니어링 실무 관점에서의 주요 모니터링 포인트를 정리합니다.

---

## 2. 10.5.x vs 10.6.0 UI 비교

| 비교 항목 | 기존 UI (OS 10.5.50 이하) | 신규 UI (OS 10.6.0) | 실무 관점의 장점 |
| :--- | :--- | :--- | :--- |
| **테마 디자인** | 어두운 다크 그레이 사이드바 | 밝고 현대적인 화이트 카드 기반 UI | 가독성 대폭 향상, 최신 HPE GreenLake 통합 룩앤필 |
| **좌측 네비게이션** | 단순 아이콘 중심 단일 레벨 | 텍스트 라벨 기반 다계층 접이식 트리 | 하위 메뉴(드라이브, 포트 등)로 1-클릭 즉시 이동 가능 |
| **상단 브랜딩** | 기본 심볼 아이콘만 노출 | HPE 로고 및 제품 모델명 명시 | 다중 스토리지 동시 운영 시 시스템 식별성 강화 |
| **포트/스위치 관리** | 세부 하드웨어 메뉴 내부 탐색 | `Ports` / `Switches` 독립 메뉴 상시 노출 | SAN 패브릭 링크 상태 및 포트 속도 즉각 점검 가능 |
| **보안 및 설정 통합** | 산재된 설정 항목 | `Settings` 하위에 네트워크/텔레메트리/인증서 통합 | 보안 정책(랜섬웨어 탐지, 볼륨 보존) 원스톱 설정 |

### 업그레이드 전/후 System 화면 비교

{{< figure src="fig-01-old-ui-system-dashboard.png" caption="그림 1. [업그레이드 전] OS 10.5.50 다크 사이드바 기반 기존 System 대시보드 화면" >}}

{{< figure src="fig-02-new-ui-system-dashboard.png" caption="그림 2. [업그레이드 후] OS 10.6.0 화이트 트리형 신규 System 대시보드 화면" >}}

---

## 3. 신규 10.6.0 6대 핵심 메뉴 상세 분석

### 3.1. Dashboard (시스템 상태 및 성능 요약)
로그인 시 가장 먼저 마주하는 첫 화면으로, 시스템 헬스 상태, 가용 용량, 실시간 성능 지표를 3개 영역의 카드로 일목요연하게 제공합니다.

* **Health**: 새로운 알림(New alerts) 건수와 함께 Storage, Protection, System, Call home, Data Services Cloud Console의 상태 아이콘을 실시간 표시합니다.
* **Usable Capacity**: 전체 가용 용량(Total Free / Total)을 직관적인 도넛 차트로 시각화하며, Private/Shared 용량, 스냅샷, 시스템 예약, 미할당(Unallocated) 용량을 구분합니다.
* **Performance**: 온라인 호스트 수와 함께 실시간 IOPS, 전송 대역폭(Bandwidth), 응답 지연 시간(Latency) 추이 그래프를 제공합니다.

{{< figure src="fig-03-dashboard-overview.png" caption="그림 3. Dashboard 화면 - Health, Usable Capacity, Performance 지표 실시간 종합 뷰" >}}

<br>

### 3.2. Storage (블록 스토리지 및 VMware 볼륨 관리)
호스트와 볼륨에 관련된 모든 오브젝트를 생성하고 매핑을 관리하는 핵심 영역입니다.

* **BLOCK 관리**: Virtual volume sets, Volumes, Host sets, Hosts
* **VMWARE 연동**: VMware storage containers
* **PEER MOTION**: 기존 스토리지로부터의 무중단 데이터 마이그레이션 세션 관리
* **Quick actions**: Create host sets, Create virtual volume sets, Storage tutorial 바로가기 지원

{{< figure src="fig-04-storage-overview.png" caption="그림 4. Storage 메뉴 개요 - 블록 볼륨, 호스트 그룹 및 Quick actions 카드" >}}

<br>

#### Virtual Volume Sets 및 볼륨 관리
생성된 볼륨셋(예: VMware ESXi 연동 볼륨)의 보호 상태, 할당 용량, 프로비저닝 상태를 카드 그리드 형태로 확인할 수 있습니다.

{{< figure src="fig-05-virtual-volume-sets.png" caption="그림 5. Virtual volume sets 상세 화면 - 볼륨셋 멤버 및 익스포트 상태 확인" >}}

<br>

#### Host Sets 및 연결 호스트 상태
SAN 스위치를 통해 연결된 호스트 서버들을 그룹화한 `Host sets` 및 개별 호스트들의 연결 상태(`Normal`)를 직관적으로 모니터링할 수 있습니다.

{{< figure src="fig-06-host-sets.png" caption="그림 6. Host sets 화면 - 연결된 호스트 멤버 구성 및 헬스 상태 점검" >}}

<br>

### 3.3. Protection (데이터 보호 및 복제/백업 관리)
원격지 복제(Replication), StoreOnce 백업 시스템 연동, 가상화 및 데이터베이스 보호 정책을 카드형으로 통합 관리합니다.

* **REPLICATION**: Replication partners, Replication partner systems (원격 복제 파트너 구성)
* **BACKUP SYSTEMS**: HPE StoreOnce 백업 어플라이언스 연동
* **APPLICATION PROTECTION**: Application servers, Datastores, Virtual machines, VM protection groups, MSSQL databases / instances

{{< figure src="fig-07-protection-overview.png" caption="그림 7. Protection 메뉴 개요 - 복제, 백업 장비 및 애플리케이션 데이터 보호 대시보드" >}}

<br>

### 3.4. System (하드웨어, 펌웨어, 물리 컴포넌트 관리)
스토리지 본체의 세부 스펙, 2개 컨트롤러 노드, 섀시, 드라이브 및 FC 포트 상태를 집중적으로 관리합니다.

#### System Details (시스템 상세 정보)
시스템 모델명(HPE Alletra Storage MP B10120), OS 버전(10.6.0), 가동 시간(Uptime), 암호화 지원 여부, 하드웨어 구성 요약(컨트롤러 2, 섀시 1, 드라이브 8, 포트 8)을 한눈에 확인할 수 있습니다.

{{< figure src="fig-08-system-details.png" caption="그림 8. System Details 화면 - 모델 스펙, 10.6.0 OS 버전 및 라이선스 정보" >}}

<br>

#### System Software (OS 펌웨어 및 업데이트 정책)
현재 적용된 OS 버전(`10.6.0`)의 최신 상태 여부와 함께, GreenLake 클라우드로부터의 자동 다운로드 및 스테이징 정책(Software Update Policy)을 구성할 수 있습니다.

{{< figure src="fig-09-system-software.png" caption="그림 9. System Software 화면 - 최신 OS 버전 확인 및 소프트웨어 업데이트 정책" >}}

<br>

#### Enclosure Chassis 및 Controllers
섀시 전면/후면 그래픽 뷰와 함께 Node 0(Master, Bay 1) / Node 1(Bay 2)의 실시간 정상 동작 상태(`Normal`), 전원 공급 장치(Power Supply), 섀시 탐색 모듈(CDM)의 무결성을 점검합니다.

{{< figure src="fig-10-system-enclosure-controllers.png" caption="그림 10. Enclosure 상세 화면 - 섀시 후면 모듈 배치 및 듀얼 컨트롤러 헬스 상태" >}}

<br>

#### Drives & Ports
스토리지에 장착된 8개의 NVMe SSD 드라이브 슬롯(1:1 ~ 1:8, 각 3.84TB) 상태와 8개의 고속 FC 포트(Host 연결용 및 Free 포트) 상태를 즉시 파악할 수 있습니다.

{{< figure src="fig-11-system-drives.png" caption="그림 11. System Drives 화면 - 8개 올-NVMe SSD 드라이브의 정상 동작 상태" >}}

<br>

### 3.5. Settings (네트워크, 보안 및 텔레메트리 통합 설정)
기존에 여러 메뉴로 분산되어 있던 인프라 관리 설정을 단일 메뉴 하위로 체계적으로 집약했습니다.

* **주요 설정 메뉴**: System, Telemetry, Users, Domains, LDAP configuration, Contacts, VMware vCenter, HPE VM Essentials, Network services, Array certificates, Trusted certificates.

{{< figure src="fig-12-settings-overview.png" caption="그림 12. Settings 메뉴 개요 - 네트워크, 사용자, 인증서 및 외부 연동 통합 목록" >}}

<br>

#### Settings ▶ System (관리 네트워크 및 데이터 보안 정책)
관리용 IPv4 주소, 서브넷 마스크, 게이트웨이, DNS 서버 정보와 함께 데이터 볼륨 보존 기간(Volume Retention), 랜섬웨어 탐지(Ransomware Detection) 정책을 중앙에서 설정합니다.

{{< figure src="fig-13-settings-system-network.png" caption="그림 13. Settings-System 화면 - 관리 네트워크 IP 구성 및 랜섬웨어 탐지 보안 정책" >}}

<br>

#### Settings ▶ Telemetry (Call Home & Data Services Cloud Console)
원격 장애 자동 감지 및 HPE 기술 지원 센터로 메트릭을 전송하는 Call Home 상태와 HPE GreenLake Data Services Cloud Console(DSCC) 클라우드 연동 상태를 확인합니다.

{{< figure src="fig-14-settings-telemetry-dscc.png" caption="그림 14. Settings-Telemetry 화면 - Call Home 및 Data Services Cloud Console 연결 상태" >}}

<br>

#### Settings ▶ Contacts (고객사 및 기술 지원 담당자 관리)
장비 알림 및 장애 시 신속한 기술 지원을 받기 위한 담당자 연락망을 등록하고 관리합니다.

{{< figure src="fig-15-settings-contacts.png" caption="그림 15. Settings-Contacts 화면 - 지원 담당자 프로필 관리" >}}

<br>

### 3.6. Reports & Activities (리포팅 및 작업 모니터링)

#### Reports (실시간 성능/용량 리포트)
`Create report`를 통해 CPU, 메모리, IOPS, 대역폭 추이를 원하는 시간대별로 즉시 추출하고, 저장된 리포트를 간편하게 다시 로드할 수 있습니다.

{{< figure src="fig-16-reports.png" caption="그림 16. Reports 화면 - 실시간 커스텀 리포트 생성 및 저장된 리포트 로드" >}}

<br>

#### Activities (경고, 태스크 및 스케줄 관리)
시스템에서 발생한 이벤트 경고(Alerts), 백그라운드에서 실행된 펌웨어 업데이트나 볼륨 생성 등의 작업 이력(Tasks), 정기 점검 스케줄(Schedules)을 통합 추적합니다.

{{< figure src="fig-17-activities.png" caption="그림 17. Activities 화면 - 실시간 경고, 작업 진행 이력 및 예약 스케줄 관리" >}}

---

## 4. 엔지니어링 실무 관점에서의 핵심 정리

1. **메뉴 이동 동선 단축**:  
   기존 다크 UI 대비 좌측 텍스트 기반 다계층 트리가 상시 노출되어, 특정 볼륨 점검 중에도 `System ➔ Drives` 또는 `System ➔ Ports`로 즉시 전환할 수 있어 장애 대응 속도가 향상되었습니다.

2. **포트 및 스위치 시각화 강화**:  
   SAN 패브릭 물리 연결 문제를 분석할 때 하위 상세 탭을 복잡하게 찾을 필요 없이, 좌측 네비게이션에서 `Ports`와 `Switches` 상태를 바로 확인할 수 있습니다.

3. **중앙 집중식 보안 정책 관리**:  
   `Settings ➔ System`에서 Volume Retention과 Ransomware Detection 정책이 일원화되어, 스토리지 레벨에서의 데이터 위변조 방지 정책을 손쉽게 구성하고 검증할 수 있습니다.

---

## 5. 마무리 및 기술 지원 안내

HPE Alletra Storage MP B10K의 **ArcusOS 10.6.0**은 직관적인 화이트 UI 개편을 통해 온프레미스 웹 콘솔 환경에서도 GreenLake 클라우드 수준의 뛰어난 시각적 완성도와 관리 편의성을 제공합니다.

> ⚠️ **현장 기술 지원 안내**:  
> Alletra Storage MP는 엔터프라이즈의 가장 핵심적인 미션 크리티컬 데이터를 담당하는 인프라입니다. 볼륨 설계, 호스트 매핑, 원격 복제 구성 또는 신규 펌웨어 적용 시 설정에 어려움이 있거나 환경 검증이 필요한 경우 **HPE Pointnext 공식 기술지원 센터 또는 공인 파트너사의 전담 엔지니어**를 통해 지원받으실 것을 권장합니다.
