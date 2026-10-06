---
title: "[HPE SN6620C / Cisco MDS] 9.2.2 → 9.4.5 OS 펌웨어 업그레이드 실전 가이드"
description: "HPE SN6620C(Cisco MDS 9148T OEM) 스위치에서 현재 버전 확인부터 Rebex Tiny SCP 파일 전송, install all 펌웨어 설치 및 delete 용량 정리까지의 표준 작업 절차입니다."
date: 2026-09-27T15:00:00+09:00
draft: false
tags: ["HPE", "SN6620C", "Cisco", "MDS", "Firmware", "NX-OS", "SCP", "Storage", "SAN", "Troubleshooting"]
categories:
  - Storage
---

## 1. 배경: 왜 9.4.5 버전 업그레이드가 필요했는가?

HPE SN6620C (Cisco MDS 9148T 기반) 32Gb FC SAN 스위치를 운영 중인 고객사 현장에서 기존 9.2.2 버전의 안정성 보강 요청이 있었습니다. 

구형 9.2.2 버전은 장기 운영 시 환경 센서 폴링 오진단이나 ISSU(In-Service Software Upgrade) 과정에서의 BIOS 타임아웃 가능성이 보고되어 있던 상태였습니다. 이에 HPE 및 Cisco 공식 문서에서 **추천 안정화 버전(Recommended Release)으로 지정된 NX-OS 9.4(5)**로 업그레이드를 결정했습니다.

폐쇄망 환경 특성상 보안 정책에 따라 외부 통신이 차단되어 있었으므로, **작업용 엔지니어 노트북에 Rebex Tiny SFTP/SCP Server를 띄워 로컬 네트워크를 통해 펌웨어를 전송**하는 방식으로 작업을 진행했습니다.

---

## 2. 환경 및 사전조건

| 항목 | 스펙 / 구성 정보 | 실무 설정 이유 |
| --- | --- | --- |
| **대상 스위치** | HPE SN6620C (Cisco MDS 9148T OEM) | 32Gbps 파이버 채널(FC) SAN 스위치 |
| **현재 NX-OS 버전** | 9.2(2) | 기존 운영 버전 |
| **타겟 NX-OS 버전** | 9.4(5) | HPE/Cisco 권장 안정화 버전 (Recommended) |
| **파일 전송 방식** | SCP (Secure Copy Protocol) | 작업 노트북 ↔ 스위치 Management 포트 직결 전송 |
| **SCP 서버 툴** | Rebex Tiny SFTP/SCP Server | 설치 없이 가볍게 실행 가능한 포터블 SCP 서버 |
| **전송 펌웨어 파일** | `m9000-pkg2.9.4.5.bin`, `m9000-kickstart-pkg2.9.4.5.bin` | 시스템(System) 및 킥스타트(Kickstart) 이미지 파일 |

---

## 3. 실전 업그레이드 단계별 절차

### Step 1. 현재 OS 버전 및 스위치 상태 확인
작업 시작 전 스위치 CLI에 접속하여 현재 동작 중인 OS 버전(`show version`)과 모듈 및 포트 헬스 상태(`show module`)를 먼저 점검합니다.

```bash
# 현재 OS 버전 및 시스템 정보 확인
show version

# 모듈 및 포트 동작 상태 점검
show module
```

{{< figure src="step-01-switch-status.jpg" caption="Step 1. 스토리지 스위치 CLI 접속 및 기존 9.2.2 버전 상태 확인" >}}

<br>

### Step 2. Rebex Tiny SCP Server 다운로드 및 구성
폐쇄망 작업 환경에 최적화된 경량 포터블 툴인 **Rebex Tiny SFTP/SCP Server**를 준비하고 설정합니다.
1. 포터블 실행 파일을 다운로드하여 작업 노트북에 준비합니다.
2. 실행 후 **User/Password** 계정 정보(예: `scpuser` / `P@ssw0rd`)를 설정하고, 펌웨어 파일이 위치한 폴더를 **Root Directory**로 지정합니다.
3. `Start Server` 버튼을 눌러 SCP 바인딩 서비스를 즉시 활성화합니다.

{{< figure src="step-02a-rebex-download.jpg" caption="Step 2-1. Rebex Tiny SFTP/SCP Server 포터블 다운로드" >}}

{{< figure src="step-02b-rebex-main.jpg" caption="Step 2-2. Rebex SFTP/SCP Server 메인 실행 화면" >}}

{{< figure src="step-02c-rebex-config.jpg" caption="Step 2-3. Rebex 사용자 계정(User/Password) 및 펌웨어 폴더 경로 설정" >}}

{{< figure src="step-02d-rebex-start.jpg" caption="Step 2-4. Start Server 버튼 클릭을 통한 SCP 서비스 활성화 완료" >}}

<br>

### Step 3. `copy` 명령어를 이용한 펌웨어 파일 전송
스위치 CLI에서 `copy scp:` 명령어를 실행하여 작업 노트북에 저장된 펌웨어 파일(`m9000-pkg2.9.4.5.bin`, `m9000-kickstart-pkg2.9.4.5.bin`)을 스위치 로컬 `bootflash:` 영역으로 땡겨옵니다.

```bash
# 작업 노트북(SCP 서버)에서 bootflash: 로 System 및 Kickstart 펌웨어 전송
copy scp://scpuser@192.168.1.100/m9000-pkg2.9.4.5.bin bootflash: vrf management
copy scp://scpuser@192.168.1.100/m9000-kickstart-pkg2.9.4.5.bin bootflash: vrf management
```

{{< figure src="step-03-copy-scp.jpg" caption="Step 3. copy 명령어로 노트북에서 펌웨어 스위치 bootflash로 전송" >}}

<br>

### Step 4. `install all` 명령어로 펌웨어 설치 및 사전 호환성 체크
펌웨어 파일 전송이 완료되면 `install all` 명령어를 실행하여 System 및 Kickstart 이미지의 무결성과 사전 호환성 체크를 수행하고 본 설치를 진행합니다.

```bash
# 9.4.5 OS 펌웨어 설치 시작 (System 및 Kickstart 동시 지정)
install all system bootflash:m9000-pkg2.9.4.5.bin kickstart bootflash:m9000-kickstart-pkg2.9.4.5.bin
```

설치가 진행되는 동안 시스템이 이미지를 검증하고 수퍼바이저/컨트롤러 패치를 순차 적용한 뒤 장비를 재부팅합니다.

{{< figure src="step-04a-install-cmd.jpg" caption="Step 4-1. install all 명령어로 NX-OS 9.4.5 펌웨어 설치 진행 (시작)" >}}

{{< figure src="step-04b-install-impact.jpg" caption="Step 4-2. install all 명령어로 NX-OS 9.4.5 펌웨어 설치 진행 (사전 호환성 체크)" >}}

{{< figure src="step-04c-install-upgrading.jpg" caption="Step 4-3. install all 명령어로 NX-OS 9.4.5 펌웨어 설치 진행 (패치 라이팅 중)" >}}

<br>

### Step 5. 부팅 완료 후 9.4.5 버전 적용 확인
장비 부팅이 완료되면 CLI로 접속해 타겟 버전(NX-OS 9.4.5)이 정상 적용되었는지 확인합니다.

```bash
# OS 버전 최종 확인
show version
```

{{< figure src="step-05-boot-complete.jpg" caption="Step 5. 부팅 완료 후 커널 및 OS 정상 로딩 확인 (NX-OS 9.4.5 정상 적용 완료)" >}}

<br>

### Step 6. 작업 완료 후 `delete` 명령어로 bootflash 정리
펌웨어 업그레이드가 성공적으로 끝나면, bootflash 용량 부족을 방지하기 위해 땡겨왔던 펌웨어 이미지 파일들을 `delete` 명령어로 삭제해 줍니다.

```bash
# bootflash 용량 확인
dir bootflash:

# 전송했던 9.4.5 System 및 Kickstart 이미지 파일 삭제 (용량 확보)
delete bootflash:m9000-pkg2.9.4.5.bin
delete bootflash:m9000-kickstart-pkg2.9.4.5.bin

# 삭제 후 bootflash 여유 공간 재확인
dir bootflash:
```

{{< figure src="step-06-delete-cleanup.jpg" caption="Step 6. 작업 완료 후 delete 명령어로 bootflash 정리" >}}

---

## 4. 🚨 트러블슈팅 노트 (현장 실무 에러 대처)

### 이슈 1: SCP 전송 시 `Host key verification failed` 에러 발생
* **에러 메시지**: `error: ssh connect failed / Host key verification failed`
* **원인 추정**: 스위치 SSH 클라이언트 알고리즘과 노트북 Rebex SCP Server 간 암호화 키 교환 미스매치.
* **해결 방법**:
  ```bash
  # 스위치 CLI에서 SSH 알고리즘 수동 허용 설정
  switch(config)# ip ssh client algorithm key-exchange dh-group14-sha1
  ```

---

## 5. 검증 (작업 완료 확인 방법)

1. **OS 버전 확인**: `show version` ➔ `system: version 9.4(5)` 확인.
2. **SAN 포트 상태 검증**: `show interface fc1/1-48 status` ➔ 모든 FC 포트 Online 정상 유지 확인.
3. **bootflash 용량 검증**: `dir bootflash:` ➔ `delete` 후 여유 공간 확보 확인.

---

## 6. 실무에서는 이렇게 씁니다 (현장 적용 관점)

* **bootflash 파일 정리를 습관화하세요**:  
  MDS 스위치의 bootflash는 용량이 제한되어 있습니다. 업그레이드가 끝난 후 `delete bootflash:` 명령어로 용량을 비워두지 않으면 다음 펌웨어 작업이나 코어덤프 저장 시 용량 부족 에러가 발생하게 됩니다.

---

## 7. 마무리 및 요약

* **한눈에 보는 핵심**:
  - 현재 버전 확인(`show version`) ➜ Rebex SCP 구동 ➜ 파일 전송(`copy scp:`) ➜ 펌웨어 설치(`install all`) ➜ 완료 검증 ➜ 임시 파일 삭제(`delete`).
  - **bootflash 불필요 파일 삭제를 통한 용량 관리**: bootflash 저장 공간 부족으로 인한 장애 및 차기 작업 이슈를 방지하기 위해, 펌웨어 업그레이드 완료 후 업로드했던 설치 파일은 꼭 `delete` 명령어로 지워주세요.
