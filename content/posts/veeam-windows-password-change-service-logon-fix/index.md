---
title: "[Veeam] Windows 서버 비밀번호 변경 후 서비스 계정 일괄 갱신 가이드"
description: "보안 컴플라이언스 및 정기 점검 규정으로 Veeam 서버의 Windows Administrator 비밀번호를 변경한 뒤 발생하는 서비스 로그인 실패(Error 1069) 및 콘솔 접속 불가를 해결하는 실무 조치 가이드입니다."
date: 2026-10-12T15:52:00+09:00
draft: false
tags: ["Veeam", "Backup", "Windows Server", "보안컴플라이언스", "비밀번호변경", "서비스계정", "트러블슈팅"]
aliases:
  - /posts/veeam-windows-password-change-service-logon-fix/
categories:
  - Storage
---

## 1. 배경: 보안 조치 후 마주하는 Veeam 콘솔 접속 불가 현상

기업 및 공공기관 인프라 운영 환경에서는 **정보보안 취약점 진단 가이드(기본 관리자 계정 관리 등)**나 내부 컴플라이언스 기준에 따라 Windows Server의 로컬 관리자(`Administrator`) 비밀번호를 주기적으로 변경하거나 전용 관리 계정으로 전환하는 작업을 진행합니다.

하지만 **Veeam Backup & Replication(VBR)**이 구동되는 백업 관리 서버에서 Windows 계정 비밀번호를 변경한 직후, 갑자기 다음과 같은 문제가 발생하여 긴급 장애 상황으로 이어지는 경우가 많습니다.

* Veeam 백업 콘솔(`Veeam Backup & Replication Console`) 실행 시 로그인 창에서 멈추거나 접속 불가 오류 발생
* Windows 이벤트 로그에 **Event ID 7000 / 7038 (로그온 실패로 인해 서비스를 시작할 수 없습니다)** 기록
* 서비스 관리자(`services.msc`)에서 서비스를 시작하려 하면 **Error 1069 (로그온 실패로 서비스를 시작할 수 없습니다)** 발생

```text
[Windows 서비스 오류 메시지]
오류 1069: 로그온 실패로 서비스를 시작할 수 없습니다.
(Error 1069: The service did not start due to a logon failure.)
```

이 문제는 Veeam의 핵심 백업 엔진과 카탈로그, 브로커 등 **Veeam 백그라운드 서비스들이 설치 시 지정된 Windows 관리자 계정 권한으로 상시 구동**되도록 등록되어 있기 때문에 발생합니다.  
Windows 계정의 비밀번호만 바꾸고 서비스에 등록된 인증 정보를 갱신하지 않으면 서비스가 일제히 중단됩니다.

---

## 2. 사전 확인: 영향받는 Veeam 서비스 목록

Veeam 서버에는 백업 코어 엔진부터 REST API, 라이선스, 카탈로그까지 10여 개 이상의 백그라운드 서비스가 등록되어 동작합니다. 서비스 계정 갱신이 필요한 대표적인 서비스들은 다음과 같습니다.

| 서비스 이름 (Service Name) | 표시 이름 (Display Name) | 설명 |
| :--- | :--- | :--- |
| **VeeamBackupSvc** | Veeam Backup Service | 백업/복원 작업 및 스케줄링을 총괄하는 코어 엔진 |
| **VeeamBrokerSvc** | Veeam Broker Service | 인프라 구성 정보 및 통신 브로커 서비스 |
| **VeeamCatalogSvc** | Veeam Guest Catalog Service | 게스트 OS 파일 인덱싱 및 카탈로그 관리 |
| **VeeamCloudSvc** | Veeam Cloud Connect Service | 클라우드 레포지토리 및 테넌트 연동 서비스 |
| **VeeamEnterpriseManagerSvc** | Veeam Backup Enterprise Manager | 엔터프라이즈 통합 관리 웹 서비스 (구성 시) |
| **VeeamRESTSvc** | Veeam Backup Server RESTful API Service | VBR REST API 연동 엔드포인트 |

{{< figure src="images/veeam-services-list-properties.png" caption="[그림 1] Windows 서비스 관리자(services.msc)의 Veeam 서비스 목록 및 속성 메뉴" >}}

---

## 3. 단계별 실무 조치 절차

### Step 1. 실행 중인 Veeam 콘솔 종료
작업 전 열려 있는 Veeam 관리 콘솔(`Veeam.Backup.Shell.exe`) 창을 모두 닫아 세션 간섭을 방지합니다.

### Step 2. 서비스 관리자(services.msc) 실행
Windows 실행 창(`Win + R`)을 열고 `services.msc`를 입력한 뒤 관리자 권한으로 실행합니다.

### Step 3. Veeam 서비스 속성 열기
서비스 목록에서 `Veeam`으로 시작하는 서비스를 찾고, 마우스 우클릭 후 **속성(Properties)**을 클릭합니다.

### Step 4. 로그온(Log On) 탭에서 변경된 계정/비밀번호 갱신
1. 속성 창 상단의 **로그온(Log On)** 탭으로 이동합니다.
2. **계정 지정(This account)** 항목에서 사용자 계정(`.\veeam-admin` 또는 `도메인\계정명`)을 확인합니다.
3. 변경된 **암호(Password)**와 **암호 확인(Confirm password)**을 정확히 입력합니다.
4. 하단의 **적용(Apply)** 버튼을 누르면 *"사용자에게 서비스로 로그온 권한이 부여되었습니다"*라는 메시지가 표시됩니다.
5. **확인(OK)**을 눌러 설정을 저장합니다.

{{< figure src="images/veeam-service-logon-properties.png" caption="[그림 2] Veeam 서비스 속성의 로그온(Log On) 탭 계정 비밀번호 갱신" >}}

> **[실무 팁: 사용자 계정 브라우징이 필요한 경우]**  
> 계정명이 변경되었거나 도메인 계정 조회가 필요한 경우, `찾아보기(Browse...)` 버튼을 눌러 위치(Location)를 지정하고 계정 이름을 확인(`Check Names`)한 뒤 적용합니다.

{{< figure src="images/veeam-select-user-dialog.png" caption="[그림 3] Windows 사용자 선택(Select User) 대화상자에서 서비스 실행 계정 지정" >}}

### Step 5. 대상 Veeam 서비스 전체 일괄 갱신
위 Step 3~4 절차를 로컬/도메인 계정 권한으로 실행되는 모든 Veeam 서비스에 동일하게 반복 적용합니다.

> **참고**: `Local System`이나 `Network Service`로 실행되는 기본 Windows 서비스는 비밀번호 변경 대상이 아니므로 수정하지 않아도 됩니다.

### Step 6. 서비스 시작 및 정상 구동 확인
1. 중지되어 있던 서비스들을 순서대로 마우스 우클릭 후 **시작(Start)**을 누릅니다.
2. 모든 Veeam 관련 서비스의 **상태(Status)**가 **실행 중(Running)**으로 정상 전환되었는지 확인합니다.

{{< figure src="images/veeam-services-running-status.png" caption="[그림 4] 전체 Veeam 서비스 계정 갱신 후 정상 실행(Running) 상태 확인" >}}

---

## 4. 최종 검증: Veeam 콘솔 접속 및 백업 인프라 상태 확인

모든 서비스가 정상 기동되면 최종 검증을 진행합니다.

1. 바탕화면 또는 시작 메뉴에서 **Veeam Backup & Replication Console**을 실행합니다.
2. 새 비밀번호(또는 변경된 관리 계정)로 로그인을 진행합니다.
3. 콘솔 하단 상태 표시줄의 접속 계정 정보(`Logged in: .\veeam-admin`)가 정상 표시되는지 확인합니다.
4. 등록된 백업 작업(Backup Jobs) 목록과 인프라 인벤토리가 에러 없이 정상 로드되는지 최종 점검합니다.

{{< figure src="images/veeam-console-login-verified.png" caption="[그림 5] 변경된 계정으로 Veeam 관리 콘솔 로그인 완료 및 백업 Job 정상 조회 확인" >}}

---

## 5. 엔지니어 운영 권고사항

1. **서비스 계정(gMSA 또는 전용 Service Account) 분리 권장**:  
   엔지니어가 대화형 로그인에 사용하는 개인 관리자 계정과 Veeam 서비스 구동 계정을 동일하게 사용할 경우, 주기적 비밀번호 변경 주기마다 전체 백업 서비스가 멈추는 장애 위험이 상존합니다. 실무에서는 Veeam 전용 서비스 계정을 별도로 생성하고 비밀번호 만료 정책을 분리 운영하는 것이 안전합니다.

2. **비밀번호 변경 작업 시 정기 점검 윈도우(Maintenance Window) 활용**:  
   백업 작업(Job)이 한창 구동 중일 때 서비스 계정 암호가 만료되거나 변경되면 진행 중인 스냅샷 및 전송 세션이 비정상 종료될 수 있습니다. 반드시 백업 스케줄이 비어 있는 정기 점검 시간대에 작업을 수행해야 합니다.
