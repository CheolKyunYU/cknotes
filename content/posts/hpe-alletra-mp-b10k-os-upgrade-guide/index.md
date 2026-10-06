---
title: "[HPE Alletra MP B10K] 10.5.50 → 10.6.0 OS 펌웨어 무중단 업그레이드 실전 가이드"
description: "HPE Alletra Storage MP B10K(B10120) 스토리지에서 10.6.0 OS 무중단 업그레이드 전 과정, 필수 System Readiness Checks 검증, 그리고 전면 개편된 신규 화이트 UI 적용 포인트를 정리합니다."
date: 2026-09-28T21:00:00+09:00
draft: false
tags: ["HPE", "Alletra", "AlletraMP", "B10K", "B10120", "Firmware", "OS Upgrade", "Storage", "GreenLake", "Troubleshooting"]
categories:
  - Storage
---

## 1. 배경: Alletra MP 10.6.0 업그레이드와 신규 UI 개편

차세대 미션 크리티컬 스토리지인 **HPE Alletra Storage MP (B10K / B10120)** 환경에서 시스템 안정화와 최신 기능 지원을 위해 **10.6.0 OS 펌웨어 업그레이드** 작업을 진행했습니다.

이번 10.6.0 릴리스는 백엔드 성능 및 플랫폼 안정성 향상뿐만 아니라, **웹 온프렘 관리 콘솔의 UI가 기존 다크 사이드바에서 모던한 화이트 트리형 신규 UI로 전면 개편**되는 중요한 업데이트입니다.

스토리지 시스템이 외부 인터넷(HPE Cloud Connect)에 연결되어 있다면 **10.6.0 펌웨어 패키지가 자동으로 시스템에 Staged(사전 다운로드)**되어 있어 즉시 작업을 시작할 수 있으며, 폐쇄망 환경인 경우에도 포털에서 바이너리를 받아 로컬 콘솔을 통해 손쉽게 업로드하여 진행할 수 있습니다.

---

## 2. 환경 및 사전조건

| 항목 | 스펙 / 설정 정보 | 실무 설정 이유 |
| --- | --- | --- |
| **대상 스토리지** | HPE Alletra Storage MP B10120 | 2-Node 액티브-액티브 엔터프라이즈 블록 스토리지 |
| **기존 OS 버전** | OS 10.5.50 | 기존 운영 펌웨어 버전 |
| **타겟 OS 버전** | OS 10.6.0 (ArcusOS) | 신규 UI 지원 및 플랫폼 안정화 권장 버전 |
| **패키지 준비 방식** | 온라인 Staged / 로컬 패키지 업로드 | 인터넷 연결 시 10.6.0 자동 수신 완료 상태 |
| **사전 필수 검증** | System Readiness Checks | 업그레이드 전 100% Passed 확인 필수 |
| **예상 소요 시간** | 약 60분 ~ 90분 내외 | 노드 순차 리부팅 및 CDM 컴포넌트 펌웨어 라이팅 포함 |

---

## 3. 단계별 실전 업그레이드 절차

### Step 1. 현재 OS 버전 및 스토리지 대시보드 상태 점검
작업을 시작하기 전 스토리지 온프렘 웹 콘솔에 접속하여 **System ➔ Dashboard**에서 현재 동작 중인 OS 버전(`10.5.50`)과 컨트롤러, 섀시, IOM 모듈의 정상 동작 상태(`OK`)를 점검합니다.

{{< figure src="step-01-dashboard-check.png" caption="Step 1. 업그레이드 전 시스템 대시보드 및 기존 OS 10.5.50 버전 상태 확인" >}}

<br>

### Step 2. Software 메뉴 진입 및 10.6.0 펌웨어 패키지 확인
좌측 메뉴의 **System ➔ Software** 탭으로 이동합니다.
스토리지 시스템이 인터넷에 연결되어 있는 환경이라면, 권장 릴리스인 `10.5.60` 및 최신 `10.6.0` 패키지가 **Staged updates** 목록에 이미 안전하게 다운로드되어 대기 중인 것을 확인할 수 있습니다.

> 💡 **폐쇄망(오프라인) 환경인 경우**:  
> 우측 상단의 `Load an update package` 메뉴를 클릭하여 사전 다운로드해 둔 펌웨어 ISO 이미지 패키지 파일(예: `OS-10.6.0.xx.iso` 형태)을 브라우저를 통해 직접 스토리지 로컬 영역으로 업로드하면 됩니다.

{{< figure src="step-02-software-updates.png" caption="Step 2-1. Software 메뉴에서 자동 수신된 10.6.0 Staged 업데이트 패키지 확인" >}}

{{< figure src="step-02b-load-package.png" caption="Step 2-2. 폐쇄망 환경을 위한 Load an update package 수동 업로드 인터페이스" >}}

<br>

### Step 3. ★ 필수 사전 단계: System Readiness Checks (사전 호환성 검증)
업그레이드 버튼을 누르기 전, **반드시 거쳐야 하는 가장 중요한 단계**입니다.  
Staged 목록 우측의 **`View readiness checks`** 링크를 클릭하여 시스템 사전 무결성 검증을 확인합니다.

* **점검 핵심 항목**:
  - `Check System Status` & `Restart system logger`: Passed
  - `Ensure Node Disk Free Space`: 노드 디스크 여유 공간 충분 여부
  - `Verify Host Connectivity`: 연결된 호스트 경로 정상 여부
  - `Check VV` & `Check VLUNs`: 볼륨 및 LUN 매핑 무결성 확인

모든 항목이 초록색 **Passed**로 완료된 것을 확인한 후 다음 단계로 진행합니다.

{{< figure src="step-03-readiness-checks.png" caption="Step 3. System readiness checks 실행 결과 - 모든 시스템/호스트/볼륨 항목 100% Passed 검증" >}}

<br>

### Step 4. Update Software 실행 및 10.6.0 패키지 선택
사전 점검이 성공적으로 확인되면 우측 상단의 **`Update software`** 액션 버튼을 클릭합니다.  
설치할 패키지 선택 창에서 **`HPE GreenLake for Block Storage 10.6.0`**을 체크하고 하단의 **`Install`** 버튼을 누릅니다.

{{< figure src="step-04-select-package.png" caption="Step 4. Update software 창에서 HPE GreenLake for Block Storage 10.6.0 선택 및 Install 실행" >}}

<br>

### Step 5. 10.6.0 OS 펌웨어 업그레이드 백그라운드 태스크 기동
설치 명령이 실행되면 상단에 `Starting installation of HPE GreenLake for Block Storage 10.6.0` 알림 배너가 표시되며 백그라운드 업그레이드 세션이 활성화됩니다.

{{< figure src="step-05-install-started.png" caption="Step 5. 10.6.0 설치 시작 및 백그라운드 Activities 태스크 활성화" >}}

<br>

### Step 6. 무중단 OS 업그레이드 진행 및 신규 UI 자동 전환
Alletra MP의 OS 업그레이드는 컨트롤러 노드를 순차적으로 페일오버하면서 무중단으로 진행됩니다.

1. **사전 체크 단계 (Pre-update checks)**: 시스템 무결성 최종 검증.
2. **노드 소프트웨어 및 펌웨어 라이팅**: Node 0 및 Node 1에 패키지 설치.
3. **노드 순차 리부팅 (Node Reboot)**: Node 0 재부팅 후 정상 기동 확인 ➔ Node 1 재부팅 진행.
4. **버전 전환 및 웹 콘솔 재시작**: 시스템 버전이 10.6.0으로 전환되는 동안 잠시 `Update in progress` 팝업이 표시됩니다.
5. **컴포넌트 펌웨어 라이팅 및 신규 UI 전환**:  
   버전 스위치가 완료되는 순간, **웹 콘솔이 모던한 화이트 트리형 신규 UI로 즉시 자동 갱신**되며 백그라운드에서 드라이브/디스크 및 CDM(Chassis Discovery Module) 컴포넌트 펌웨어가 마무리 적용됩니다.

{{< figure src="step-06a-prechecks-running.png" caption="Step 6-1. OS update in progress - Initializing pre-update checks 진행" >}}

{{< figure src="step-06b-node-reboot.png" caption="Step 6-2. 36% 진행 - Node 0 정상 검증 후 순차 재부팅 및 페일오버 유지" >}}

{{< figure src="step-06c-version-switch.png" caption="Step 6-3. 44% 버전 스위치 진행 - 웹 관리 콘솔 세션 자동 갱신 팝업" >}}

{{< figure src="step-06d-new-ui-component-firmware.png" caption="Step 6-4. 96% 신규 화이트 UI 자동 전환 및 CDM 컴포넌트 펌웨어 최종 라이팅" >}}

<br>

### Step 7. OS 업그레이드 완료 확인
모든 단계(Pre-checks ➔ Node S/W ➔ Node F/W ➔ Reboot ➔ Post-checks ➔ Component F/W)가 완료되면 **`OS update successful`** 화면이 표시되며 `HPE Alletra Storage 10.6.0 update completed successfully` 메시지가 출력됩니다.

{{< figure src="step-07-update-success.png" caption="Step 7. OS update successful - 10.6.0 업그레이드 전 항목 Completed 완료" >}}

<br>

### Step 8. 신규 UI 대시보드 검증 및 10.6.0 최종 운영 확인
업그레이드 완료 후 **System ➔ Details / Software** 대시보드로 이동하여 시스템 상태를 최종 점검합니다.

* **OS version**: `10.6.0` 정상 적용 확인.
* **하드웨어 헬스**: Enclosure chassis, Controllers, Drive IOMs, Drives, Ports, Switches 전 항목 초록색 체크(`OK`) 확인.
* **좌측 신규 네비게이션 트리**: Storage, Protection, System, Settings, Reports 등 신규 레이아웃이 깔끔하게 동작함을 확인합니다.

{{< figure src="step-08-final-dashboard-10-6-0.png" caption="Step 8. 신규 화이트 UI가 적용된 System 대시보드에서 OS 10.6.0 정상 동작 검증" >}}

---

## 4. 🚨 트러블슈팅 및 작업 시 주의사항

### 주의 1: Readiness Check에서 Warning 또는 Failed 발생 시
* **원인**: 호스트 멀티패스 단선, 백업 태스크 실행 중, 노드 디스크 여유 공간 부족 등.
* **대처**: Readiness Checks 결과에 실패나 경고가 있다면 절대 `Ignore` 옵션을 체크하고 강제 진행하지 마시고, 원인을 먼저 해결한 뒤 `Re-run checks`를 수행하여 **모든 항목이 Passed가 된 상태에서 업그레이드를 시작**해야 합니다.

### 주의 2: 44% 버전 스위치 단계에서 웹 콘솔 일시 끊김 현상
* **증상**: 노드 리부팅 및 웹 관리 서버 데몬 재기동 중 브라우저 응답이 1 ~ 2분가량 멈추거나 팝업이 유지됨.
* **대처**: 스토리지 내부적으로 신규 웹 데몬으로 전환되는 정상 과정이므로 브라우저 창을 강제로 닫거나 F5를 난타하지 마시고 2 ~ 3분간 대기하시면 신규 화이트 UI로 자동 리프레시됩니다.

### 주의 3: 자체 작업이 어렵거나 불확실한 경우 (HPE 전문 엔지니어 지원 필수 권고)
* **권고 사항**: Alletra MP 스토리지는 엔터프라이즈 미션 크리티컬 데이터가 운영되는 핵심 인프라입니다. 만약 사내 엔지니어가 직접 작업을 수행하기 어렵거나, 사전 Readiness Check에서 반복적인 오류/경고가 해결되지 않는 경우, 혹은 오프라인 수동 패키지 로드 등 작업 절차에 확신이 서지 않는 경우에는 **절대로 무리하게 단독으로 진행하지 마시고, 반드시 HPE 공식 기술지원(Pointnext) 또는 HPE 공인 파트너사 전담 엔지니어의 지원을 받아 안전하게 작업을 진행**하시기 바랍니다.

---

## 5. 검증 (작업 완료 확인 체크리스트)

1. **OS 버전 확인**: `System` ➔ `Software`에서 `OS version: 10.6.0` 확인.
2. **컴포넌트 헬스 검증**: 컨트롤러 2노드 및 IOM, 전원, 팬 모듈 정상(`OK`) 확인.
3. **호스트 I/O 및 멀티패스 정상 유지**: 서버 측에서 SAN 스토리지 볼륨 액세스 무단절 검증.

---

## 6. 실무에서는 이렇게 씁니다 (현장 적용 관점)

* **인터넷 연결 환경을 적극 활용하세요**:  
  Alletra MP는 클라우드 커넥티드 아키텍처로 설계되어 있어, 외부 네트워크가 열려 있다면 번거롭게 펌웨어 파일을 찾아서 다운받고 올릴 필요 없이 Staged 기능을 통해 원클릭으로 안전하게 패키지를 준비할 수 있습니다.
* **10.6.0 신규 UI 적응 포인트**:  
  기존 단일 레벨의 다크 좌측 바에서 `System` 하위에 `Details`, `Software`, `Controllers`, `Drives`, `Ports` 등이 트리 형태로 체계화되었으므로, 장애 분석이나 일상 점검 시 메뉴 이동 동선이 훨씬 직관적으로 개선되었습니다.

---

## 7. 마무리 및 요약

* **한눈에 보는 핵심 워크플로우**:
  - 기존 10.5.50 대시보드 확인 ➜ Staged 10.6.0 패키지 확인 ➜ **`System Readiness Checks` 100% Passed 검증(필수)** ➜ `Update software` 실행 ➜ 무중단 노드 리부팅 & 신규 화이트 UI 자동 전환 ➜ 10.6.0 최종 정상 검증.
  - 사전 체크리스트만 철저히 확인하면 1시간 내외로 안전하게 최신 버전과 신규 UI를 확보할 수 있습니다.

> 📢 **Next Post 예고**:  
> 이번 포스트에서는 10.6.0 OS 펌웨어 업그레이드 실무 절차를 집중적으로 살펴보았습니다. **다음 시간에는 10.6.0 버전에서 완전히 새로워진 화이트 트리형 신규 웹 UI(대시보드, 메뉴 동선, 볼륨 및 포트 세부 관리 기능 등)의 주요 변화와 실무 활용 팁**을 상세하게 소개해 드리겠습니다!
