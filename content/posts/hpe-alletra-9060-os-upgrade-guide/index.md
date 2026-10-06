---
title: "[HPE Alletra 9060] OS 9.6.30 무중단 업그레이드 실무 절차"
description: "HPE Alletra 9060 스토리지 OS 9.6.30 무중단 업그레이드 전 과정을 실제 화면과 함께 쉽게 따라할 수 있도록 정리한 단계별 실무 작업 가이드."
date: 2026-10-04T17:30:00+09:00
draft: false
tags: ["HPE", "Alletra", "Alletra9060", "Alletra9000", "Primera", "OSUpgrade", "Storage", "Tech"]
categories:
  - Storage
---

## 1. 개요: HPE Alletra 9000 계열 펌웨어 업그레이드의 핵심 원칙

엔터프라이즈 미션 크리티컬 환경을 담당하는 **HPE Alletra 9060 (Alletra 9000 / Primera 아키텍처 기반)** 스토리지는 듀얼 또는 쿼드 컨트롤러 노드 구조를 통해 무중단(Online) 펌웨어 업그레이드를 지원합니다.

Alletra 9000 계열의 OS 업그레이드 시 가장 중요한 사전 요구사항은 <strong>최신 Upgrade Tool (UT)</strong>의 준비입니다. 스토리지 OS 본체를 업데이트하기 전에 최신 릴리스에 대응하는 Upgrade Tool이 먼저 시스템에 적재되어야 정확한 사전 점검(Pre-upgrade Readiness Check)과 롤링 노드 재기동 프로세스를 안전하게 실행할 수 있습니다.

이번 포스트에서는 실제 운영 중인 HPE Alletra 9060 장비를 기준으로, OS 9.6.5에서 <strong>OS 9.6.30 (Extended Support Release)</strong>으로의 무중단 업그레이드 전 과정과 현장 엔지니어링 체크포인트를 정리합니다.

---

## 2. 작업 전 준비 사항 및 패키지 획득

### 2.1. 필수 패키지 다운로드 경로
Alletra 9000 및 Primera 장비의 펌웨어와 도구는 HPE 공식 라이선스 포털에서 다운로드합니다:

1. **HPE Enterprise License Portal (My HPE Software Center)** 접속
2. **Software** 메뉴 이동 ➔ 계약된 장비의 SAID 또는 시스템 식별 정보 입력
3. 다운로드 대상:
   * **최신 Upgrade Tool 패키지**: `Upgrade Tool 80 (build 260625)` 이상
   * **목표 OS 펌웨어 패키지**: `HPE Alletra 9000 OS 9.6.30.xx` (tar/iso 형태)

### 2.2. 사전 작업 체크리스트
* **호스트 I/O 다중 경로(Multipath) 정상 상태 확인**: 노드 리부팅 시 잔여 노드로 I/O가 전환되므로 연결 호스트(15개 이상)의 멀티패스 활성 경로 확인
* **스토리지 하드웨어 및 시스템 상태 점검**: 컨트롤러 노드, 전원 모듈, 드라이브 엔클로저 헬스 정상 및 미해결 알림(New alerts: 0) 확인
* **참고 (Remote Copy 및 스케줄 작업)**: 최신 Alletra/Primera OS는 노드 롤링 리부팅 시에도 원격 복제(Remote Copy) 세션과 스케줄을 자체적으로 안전하게 제어하므로, 특정 릴리스 노트 어드바이저리(Advisory)가 없는 한 별도로 수동 중지하지 않고 온라인 상태를 유지하며 진행합니다.

---

## 3. 단계별 무중단 업그레이드 실무 절차

### Step 1. 사전 대시보드 및 시스템 상태 점검
웹 콘솔에 접속하여 전체 시스템의 경고(New alerts: 0), 가용 용량, 호스트 연결 상태를 확인하고, `System` ➔ `Software`에서 현재 구동 중인 기본 펌웨어 버전(9.6.5)과 모델명을 점검합니다.

{{< figure src="fig-01-dashboard-precheck.png" caption="그림 1. Dashboard 사전 점검 - 시스템 헬스 정상, 가용 용량 및 연결 호스트 확인" >}}

{{< figure src="fig-02-system-overview-precheck.png" caption="그림 2. System 화면 - 현재 구동 중인 OS 펌웨어 버전(9.6.5) 및 하드웨어 구성 확인" >}}

<br>

### Step 2. Upgrade Tool 및 OS 9.6.30 업데이트 패키지 로드
우측 `Actions` 메뉴에서 **Load an update package**를 클릭하고, 사전에 다운로드해 둔 <strong>최신 Upgrade Tool(UT 80)</strong>과 **OS 9.6.30 패키지**를 스토리지로 업로드합니다.

{{< figure src="fig-03-load-update-package.png" caption="그림 3. Load an update package - 업그레이드 패키지 선택 및 스토리지 업로드" >}}

<br>

### Step 3. 사전 적합성 검사(Pre-upgrade Readiness Checks) 및 경고 조치
패키지 로드 후 본 업그레이드를 시작하기 전, 시스템 안전성을 검증하는 사전 적합성 검사(Readiness Checks)를 수행합니다.

#### 포트 토폴로지 경고 확인 및 예외 승인
사전 점검 결과 `Check Port Topology Consistency` 항목에서 Warning이 발생할 수 있습니다. 이는 특정 SAN 패브릭 타깃 포트(`0:3:1`, `0:3:2`, `1:3:1`, `1:3:2`) 간의 토폴로지 구성 편차로 인한 것으로, 실제 운영 환경상 의도된 구성임을 확인합니다.

{{< figure src="fig-05-readiness-warning-topology.png" caption="그림 4. 사전 Readiness Checks 결과 - 포트 토폴로지 일관성 경고(Warning) 세부 내역" >}}

<br>

PD, LD, IOCTL, 호스트 경로 일관성(Host Path Consistency) 등 핵심 무중단 점검 항목이 모두 `Passed` 상태임을 확인한 후, `Ignore checks` 창에서 안내 문구를 확인하고 **Yes, ignore**를 선택하여 사전 점검을 통과하고 다음 단계로 진입합니다.

{{< figure src="fig-06-readiness-passed-checks.png" caption="그림 5. 사전 Readiness Checks 항목 - 물리 디스크, 볼륨 상태 및 호스트 경로 정상 확인" >}}

{{< figure src="fig-07-readiness-ignore-proceed.png" caption="그림 6. Ignore checks 다이얼로그 - 경고 확인 및 사전 점검 승인 진행" >}}

<br>

### Step 4. 무중단 롤링 노드(Rolling Node) OS 업그레이드 수행
사전 점검 승인 후 본 업그레이드를 시작하면, **Upgrade Tool 80** 스크립트에 의해 사전 스크립트 구동 및 OS 이미지 설치가 실행됩니다.

{{< figure src="fig-08-upgrade-initiated-ut80.png" caption="그림 7. OS update in progress - Upgrade Tool 80 기반 사전 스크립트 구동" >}}

<br>

#### Node 0 업그레이드 및 리부팅
기본 파일시스템 설치 후 <strong>Node 0</strong>이 클러스터에서 일시 분리(Leaving cluster, `11:13:20`)되어 신규 OS(9.6.30)로 풀 리부팅을 수행합니다. 이때 모든 호스트 I/O는 정상 대기 중인 Node 1으로 원활하게 처리되며, 리부팅 완료 후 Node 0은 클러스터로 자동 복귀(Rejoined, `11:22:13`)합니다.

{{< figure src="fig-09-node0-upgrade-and-reboot.png" caption="그림 8. Node 0 OS 업그레이드 - 클러스터 이탈, 9.6.30 리부팅 및 정상 복귀 완료" >}}

<br>

> 💡 **현장 실무 팁 (관리 IP 페일오버)**:  
> 마스터 컨트롤러 노드가 리부팅되는 시점에는 관리 웹 콘솔(Management GUI)이 백그라운드에서 반대편 노드로 페일오버되면서 일시적인 진행률 팝업(`Update in progress 16%`)이 표시될 수 있습니다. 이는 정상적인 관리 인터페이스 인계 과정이므로 브라우저를 닫거나 조작하지 않고 대기합니다.

{{< figure src="fig-10-management-failover-progress.png" caption="그림 9. 노드 리부팅 중 관리 콘솔 페일오버 진행 안내 화면" >}}

<br>

### Step 5. Node 1 롤링 업그레이드 및 Web UI 재접속 확인
Node 0의 정상 복귀가 확인되면, 동일한 방식으로 <strong>Node 1</strong>이 클러스터를 이탈하여 신규 OS로 리부팅을 진행합니다(`11:27:32`).

Node 1이 리부팅되는 동안 웹 콘솔(Web UI)에 다시 접속하면, 상단에 **유지보수 모드(Maintenance Mode)** 알림 배너와 함께 전체 OS 설치 진행률(**Installing HPE Alletra 9000 9.6.30: 69%**)이 표시되는 메인 Software 화면을 확인할 수 있습니다.

{{< figure src="fig-04-staged-packages-and-maintenance-mode.png" caption="그림 10. Node 1 리부팅 중 Web UI 재접속 화면 - 설치 진행률(69%) 및 유지보수 모드 활성화" >}}

<br>

Node 1이 리부팅을 마치고 클러스터로 성공적으로 복귀(`11:35:16`)하면, 양 노드가 모두 9.6.30으로 동기화되며 자동으로 사후 점검(Post-upgrade, `11:41:23`) 및 구버전 임시 패키지 정리 단계로 진입합니다.

{{< figure src="fig-11-node1-upgrade-and-postcheck.png" caption="그림 11. Node 1 OS 업그레이드 - 9.6.30 리부팅 및 사후 점검(Post-upgrade) 개시" >}}

<br>

### Step 6. 사후 점검(Post-upgrade) 완료 및 최종 펌웨어 버전 검증
양 노드 업데이트 완료 후 시스템은 자동으로 사후 점검을 수행하며 임시 스테이징 파일(`OS-9.6.20.6` 등)을 정리합니다. `Continue update software` 창에서 최종 승인 체크 후 **Yes, continue**를 누르면 절차가 마무리(`11:49:45 No issues reported`)됩니다.

{{< figure src="fig-12-continue-update-confirmation.png" caption="그림 12. Continue update software - 사후 점검 승인 다이얼로그" >}}

{{< figure src="fig-13-post-upgrade-no-issues.png" caption="그림 13. 사후 점검 완료 - No issues reported 확인 후 절차 마무리" >}}

<br>

모든 프로세스가 종료되면 `Software` 화면에서 **Current version: 9.6.30**이 정상 표시되며, 권장 사항에 `You're all up to date.` 상태가 나타납니다. 유지보수 모드가 자동 해제되며 모든 업그레이드 작업이 완벽하게 완료됩니다.

{{< figure src="fig-14-software-status-verified.png" caption="그림 14. Software 최종 화면 - OS 9.6.30 적용 완료 및 최신 상태 검증" >}}

---

## 4. 엔지니어링 실무 핵심 요약

| 구분 | 주요 점검 항목 | 실무 엔지니어링 권고사항 |
| :--- | :--- | :--- |
| **Upgrade Tool 선행** | UT 버전 적합성 | OS 설치 전 반드시 최신 릴리스 Upgrade Tool을 먼저 적재하여 사전 점검 신뢰성 확보 |
| **패키지 획득 경로** | HPE Software Portal | HPE Enterprise License Portal의 Software 탭에서 공인 계약 기반으로 패키지 수급 |
| **무중단 이중화 검증** | Host Multipath | 롤링 리부팅 시 단일 노드로 I/O가 집중되므로 사전 다중 경로 정상 상태 확인 필수 |
| **토폴로지 경고 조치** | Port Topology Warning | 스위치 조닝/포트 속도 불일치 여부를 사전 파악하고, 의도된 환경일 경우 예외 승인 후 진행 |

---

## 5. 마무리 및 기술 지원 안내

HPE Alletra 9060의 **OS 9.6.30** 업그레이드는 최신 보안 패치와 컨트롤러 안정성을 제공하는 Extended Support Release로, 철저한 사전 점검과 Upgrade Tool 준비가 뒷받침된다면 100% 무중단으로 안전하게 완료할 수 있습니다.

> ⚠️ **현장 기술 지원 안내**:  
> Alletra 9000 스토리지는 엔터프라이즈 최상위 미션 크리티컬 데이터가 집중된 핵심 인프라입니다. 업그레이드 사전 검증, 토폴로지 경고 분석 또는 펌웨어 적합성에 대한 지원이 필요하신 경우 <strong>HPE Pointnext 공식 기술지원 센터 또는 공인 파트너사의 전담 엔지니어</strong>를 통해 지원받으실 것을 권장합니다.
