---
title: "[HPE VME] Veeam 백업 복구 실패: VirtIO 디스크 번호 중복(index=0) 해결 가이드"
description: "HPE VME 환경의 다중 디스크 VM을 Veeam으로 복구할 때 발생하는 'VirtIO 디스크 인덱스 중복(index=0)' 오류 원인과 Cloud Advanced Options를 통한 해결 절차를 정리합니다."
date: 2026-10-09T16:10:00+09:00
draft: false
tags: ["HPE", "VME", "SimpliVity", "Veeam", "KVM", "VirtIO", "트러블슈팅", "디스크오류"]
categories:
  - SimpliVityVME
---

KVM 기반의 **HPE VM Essentials(VME) 및 HPE SimpliVity HCI** 환경을 구축한 후, 엔터프라이즈 백업 표준 솔루션인 **Veeam Backup & Replication**을 연동하여 이미지 백업 및 복구 검증(DR Drill)을 진행하는 것은 서비스 연속성 확보를 위한 필수 실무 절차입니다.

하지만 VME에서 기존 백업본을 복구하여 생성한 가상머신(VM)이나 특정 템플릿 기반으로 프로비저닝된 인스턴스 중 **2개 이상의 디스크(Multi-disk)를 장착한 VM**을 Veeam으로 백업받은 뒤 복구를 시도할 때, 예상치 못한 디스크 매핑 오류로 복구 워크플로우가 중단되는 현상이 발생할 수 있습니다.

이번 글에서는 실제 현장에서 마주친 **Veeam의 VirtIO 디스크 매칭 오류 원인을 분석하고, VME Manager의 숨겨진 고급 옵션(Cloud Advanced Options)을 활성화하여 VirtIO 넘버링을 정상 재할당하는 해결 방법**을 공유합니다.

---

## 1. 장애 현상 및 Veeam Restore 세션 에러 로그

2개의 가상 디스크(Root OS 디스크 및 Data 디스크)를 가진 VM을 대상으로 Veeam 이미지 백업을 정상 수신한 뒤, 복구 신뢰성 검증을 위해 Full VM Restore를 실행했을 때 다음과 같은 API 에러 메시지가 기록되며 복구가 중단되었습니다.

![Veeam Restore Session 디스크 매칭 에러 로그](images/veeam_restore_error_log.png)

```text
In the Morpheus external infrastructure API response,
the affected VM (vm-name) has two Virtio disks (id 11 and id 15)
but both disks are reported with the same index value (index=0)
and no diskLabel is provided.

This makes disk identification ambiguous and may cause the restore
workflow to fail disk-to-disk matching, resulting in the error:
“Failed to match the original disk 15 to any disk of the restored VM.”
```

### 에러 로그 핵심 분석
1. **Morpheus 기반 VME API 응답**: HPE VME Manager는 내부적으로 검증된 클라우드 오케스트레이션 엔진(Morpheus)을 기반으로 인프라 API를 처리합니다.
2. **동일한 디스크 인덱스 보고 (`index=0`)**: VM 내부의 VirtIO 디스크 2개(`id 11`, `id 15`)가 모두 인덱스 번호 `0`으로 응답하고 디스크 라벨이 누락되어 있습니다.
3. **디스크 식별 모호성으로 인한 복구 실패**: 원본 디스크와 타겟 VM의 복원 디스크를 1:1로 매핑해야 하는 Veeam 복구 엔진 입장에서, 두 디스크의 버스 주소가 동일하게 인식되므로 `disk 15`를 매칭하지 못하고 에러를 발생시킨 것입니다.

---

## 2. VME Manager 디스크 상태 확인: 동일한 Mount Point (VIRTIO 0:0)

오류 원인을 확인하기 위해 HPE VME Manager 콘솔에서 대상 VM의 스토리지 설정 세부 정보를 확인했습니다.

![VME Manager 디스크 마운트 포인트 중복 확인 화면](images/vme_disk_duplicate_virtio_index.png)
*(사진: 두 개의 디스크 vda와 vdb의 MOUNT POINT가 모두 VIRTIO 0:0으로 중복 지정된 상태)*

- **Disk 0 (`vda`)**: Storage Controller `VirtIO Block` / Mount Point **`VIRTIO 0:0`**
- **Disk 2 (`vdb`)**: Storage Controller `VirtIO Block` / Mount Point **`VIRTIO 0:0`**

로그의 내용 그대로, 두 디스크가 물리적으로 서로 다른 장치(`vda`, `vdb`)임에도 불구하고 **컨트롤러 레벨의 마운트 포인트가 모두 첫 번째 인덱스인 `VIRTIO 0:0`으로 할당**되어 있었습니다. 

게스트 OS 내부에서는 커널이 순차적으로 장치를 잡아 정상 부팅될지 몰라도, 하이퍼바이저 API를 통해 디스크 메타데이터를 정밀하게 대조하는 Veeam 백업/복구 솔루션 관점에서는 치명적인 충돌 요소가 된 것입니다.

---

## 3. 원인 해결: 비활성화된 VirtIO 컨트롤러 옵션 활성화하기

문제를 해결하기 위해 VM 편집 메뉴(`Reconfigure Server`)로 진입했으나, 초기 기본 설정 상태에서는 디스크의 마운트 포인트를 사용자가 임의로 변경할 수 있는 드롭다운 메뉴가 나타나지 않거나 비활성화되어 있습니다.

VirtIO 버스 번호를 수동 조정하려면 **상위 클라우드(Cloud) 레벨에서 디스크 및 스토리지 유형 선택 옵션을 먼저 개방**해야 합니다.

### Step 1. Cloud 편집 메뉴의 Advanced Options 활성화

HPE VME 콘솔의 인프라 메뉴로 이동합니다:
`Infrastructure > Clouds > 등록된 SimpliVity Cloud 선택 > 편집(Edit)`

페이지 하단의 **Advanced Options(고급 옵션)** 접기 메뉴를 펼친 뒤, 다음 세 가지 체크박스를 선택하고 저장합니다.

![HPE VME Cloud Advanced Options 활성화 화면](images/vme_cloud_advanced_options_enable.png)
*(사진: Cloud 편집 메뉴에서 ENABLE DISK TYPE SELECTION 및 STORAGE TYPE SELECTION 활성화)*

- **`[x] ENABLE DISK TYPE SELECTION`** (디스크 인터페이스 유형 선택 활성화)
- **`[x] ENABLE STORAGE TYPE SELECTION`** (스토리지 컨트롤러 유형 선택 활성화)
- **`[x] ENABLE NETWORK INTERFACE TYPE SELECTION`** (네트워크 인터페이스 유형 선택 활성화)

---

## 4. VM 재구성: VirtIO 넘버링 재할당 (VIRTIO 0:1)

클라우드 고급 옵션을 저장한 후, 다시 문제가 된 VM의 관리 페이지로 이동하여 서버 재구성을 실행합니다:
`VM 선택 > Actions > Reconfigure Server`

이제 VOLUMES 설정 항목에서 디스크별 컨트롤러 및 마운트 포인트 선택 드롭다운이 활성화됩니다.

![Reconfigure Server 화면에서 VirtIO 마운트 포인트 선택](images/vme_reconfigure_server_virtio_remap.png)
*(사진: Reconfigure Server 팝업에서 두 번째 디스크의 마운트 포인트를 VIRTIO 0:1 등 고유 인덱스로 재할당)*

1. **Root 디스크 (`vda`)**: 기본값인 **`VIRTIO 0:0`** 유지
2. **보조 디스크 (`vdb`)**: 드롭다운을 열어 중복되지 않는 **`VIRTIO 0:1`** (또는 비어있는 고유 인덱스)로 변경 선택
3. 우측 하단의 **`Reconfigure`** 버튼을 클릭하여 설정 적용

설정 적용 후 VM 스토리지 세부 정보를 다시 확인하면, `vda`는 `VIRTIO 0:0`, `vdb`는 `VIRTIO 0:1`로 고유한 버스 인덱스를 갖게 됩니다.

---

## 5. 결과 검증 및 Veeam 복구 성공

VirtIO 넘버링 충돌을 해소한 후 Veeam 환경에서 다음과 같이 후속 작업을 검증했습니다:

1. **새로운 Veeam 백업 세션 실행**: VME API가 두 디스크를 각각 `index=0`과 `index=1`로 명확히 구분하여 메타데이터를 수집합니다.
2. **Full VM Restore 재테스트**: 
   - Veeam의 디스크 매칭 알고리즘이 원본 디스크와 타겟 가상 디스크의 인덱스를 1:1로 완벽하게 매칭합니다.
   - 이전에 발생했던 `Failed to match the original disk 15 to any disk of the restored VM` 오류가 완전히 해소되고 정상적으로 복구가 완료되었습니다.

---

## 6. 실무 핵심 요약 및 권장 점검 사항

1. **템플릿 복제 및 초기 VM 생성 시 디스크 버스 검증 습관**:  
   KVM/QEMU 기반 가상화 환경에서는 수동 생성이나 변환 과정에서 다중 디스크가 동일한 컨트롤러 주소(`0:0`)로 생성되는 경우가 종종 발생합니다. 단일 VM 운영 중에는 체감되지 않더라도, 외부 백업/DR 솔루션 연동 시 이번처럼 API 레벨에서 블로킹이 발생하므로 다중 디스크 VM은 반드시 디스크 인덱스를 사전 점검해야 합니다.
2. **VME 기본 설정과 Cloud Advanced Options**:  
   HPE VME의 직관적인 UI 뒤편에는 세부적인 하드웨어 버스 매핑을 제어할 수 있는 클라우드 정책이 숨겨져 있습니다. 디스크나 네트워크 인터페이스의 세부 속성을 변경할 수 없다면 당황하지 마시고 상위 **`Cloud > Advanced Options`**의 선택 정책들을 확인해 보시기 바랍니다.
