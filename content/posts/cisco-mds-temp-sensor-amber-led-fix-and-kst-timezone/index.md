---
title: "[Cisco MDS] Temp 센서 Amber LED 해결 및 타임존 설정"
description: "Cisco MDS 9148T / HPE SN6620C 9.2.2 버전 운영 시 발생하는 가짜 온도 경고(Amber LED)의 원인과 9.4.5 OS 업그레이드를 통한 조치, 타임존(Timezone) 설정 방법을 다룹니다."
date: 2026-09-27T15:30:00+09:00
draft: false
tags: ["Cisco", "MDS", "HPE", "SN6620C", "Temperature", "Amber LED", "Timezone", "Troubleshooting", "Storage", "SAN"]
categories:
  - Storage
---

## 1. 배경: 항온항습실은 시원한데 왜 스위치 전면에 Amber LED가 들어올까?

데이터센터 현장에 출장을 나가보면, 전산실 온도는 18 ~ 20°C로 아주 서늘하게 유지되고 있는데도 **Cisco MDS 9148T (HPE SN6620C) 스토리지 스위치 전면 패널의 SYS/ENV 상태 LED가 주황색(Amber)으로 켜져 있는 현상**을 종종 목격하게 됩니다.

실제 `show environment` 명령어로 내부 센서 온도를 찍어보면 `32°C (Normal)`로 아주 정상적인 상태임에도 불구하고, 9.2.2 버전의 센서 폴링 메커니즘 버그(CSCwo09244 등)로 인해 임계치 초과 경고를 잘못 감지하여 주황색 경고등을 띄우는 것입니다.

이번 글에서는 **온습도 센서 오진단 Amber LED 이슈의 원인과 9.4.5 업그레이드를 통한 근본 해결 과정**, 그리고 장애 분석 시 로그 타임스탬프 혼선을 막기 위한 **스위치 타임존(Timezone) 설정 방법**을 정리합니다.

---

## 2. 환경 및 사전조건

| 항목 | 스펙 / 설정 정보 | 실무 설정 이유 |
| --- | --- | --- |
| **대상 장비** | Cisco MDS 9148T / HPE SN6620C | 32G Enterprise FC SAN Switch |
| **이슈 발생 버전** | Cisco NX-OS 9.2(2) | 온습도 센서 폴링 오진단 코스메틱 버그 존재 |
| **조치 완료 버전** | Cisco NX-OS 9.4(5) | 센서 폴링 및 플랫폼 모니터링 버그 수정 적용 |
| **적용 타임존** | Timezone (KST, UTC+9) | 국내 한국 표준시 기준 로그 분석 일치화 |

---

## 3. 실전 조치 및 설정 절차

{{< figure src="step-01-temp-amber-led-status.jpg" caption="작업 전 스위치 전면 LED 및 상태 점검 화면" >}}

### Step 1. 온습도 센서 및 환경 상태 점검
전면 Amber LED가 켜졌을 때 하드웨어 고장인지 버그인지 1차 판단을 수행합니다.

```bash
# 환경 센서 및 온도 정보 상세 확인
show environment temperature
```

출력 결과에서 모든 센서 항목이 `OK` 및 정상 온도 범위(Normal) 내에 있음에도 Amber LED가 꺼지지 않는다면 **NX-OS 9.2.2 센서 오진단 버그**임을 확정할 수 있습니다.

<br>

{{< figure src="step-04-temp-amber-led-normal.jpg" caption="펌웨어 작업 후 엠버 정상화 (show environment 센서 상태 및 정상 범위 검증)" >}}

### Step 2. 9.4.5 OS 업그레이드를 통한 센서 버그 근본 해결
센서 오진단 버그를 해결하기 위해 스위치 OS를 추천 릴리스인 **NX-OS 9.4(5)**로 업그레이드합니다.  
업그레이드가 완료되고 장비가 재부팅되면 플랫폼 모니터링 데몬이 재시작되면서 주황색 Amber LED가 꺼지고 **정상 초록색(Green) LED로 전환**됩니다.

<br>

{{< figure src="step-02-timezone-kst-setting.jpg" caption="clock timezone 명령어를 통한 한국 표준시(KST, UTC+9) 타임존 설정" >}}

### Step 3. 타임존(Timezone) 설정
스위치가 기본 UTC(협정 세계시)로 설정되어 있으면 나중에 장애 발생 시 서버/스토리지 로그와 시간을 대조할 때 9시간의 시차가 생겨 분석이 매우 번거로워집니다. 타임존을 설정합니다.

```bash
# 글로벌 설정 모드 진입
switch# configure terminal

# 타임존(Timezone) 설정 (KST 한국 표준시 기준 UTC +9시간)
switch(config)# clock timezone KST 9 0

# 설정 확인
switch(config)# show clock
```

*(이유: 스토리지 SAN 스위치의 로그 타임스탬프를 표준시로 맞춰 장애 분석 시간을 절반으로 단축하기 위함)*

### Step 4. 러닝 컨피그 저장
설정한 타임존 값이 장비 재부팅 후에도 유지되도록 저장합니다.

```bash
# 설정 저장
switch# copy running-config startup-config
```

---

## 4. 🚨 트러블슈팅 노트 (현장 실무 에러 대처)

### 이슈 1: 온도는 정상(32°C)인데 전면 SYS/ENV LED가 주황색(Amber)으로 남아있는 현상
* **에러 증상**: `show environment` 결과는 전부 `Normal`인데 전면 LED가 주황색임.
* **원인 추정**: NX-OS 9.2.2 버전의 IOSlice 온도 센서 폴링 재시도 로직 버그(`%PLATFORM-4-MOD_TEMPFAIL` 경고 코스메틱 오작동).
* **해결 방법**: 
  - 단순 소프트웨어 리셋(`clear environment history`)으로는 해결되지 않음.
  - **NX-OS 9.4(5) 버전 업그레이드 적용 후 100% 정상 초록색(Green) LED로 원복 확인 완료.**

---

## 5. 검증 (작업 완료 확인 방법)

1. **전면 LED 물리적 모니터링**:
   - 스위치 전면 패널의 `SYS`, `ENV`, `FAN` LED가 주황색(Amber)에서 **정상 초록색(Green)**으로 변경되었는지 확인.
2. **센서 상태 2차 검증**:
   ```bash
   show environment
   # 결과: Power, Fan, Temp 등 모든 항목이 OK / Normal 상태임을 확인
   ```
3. **타임존 적용 검증**:
   ```bash
   show clock
   # 출력 결과 예시: 15:35:12.123 KST Sun Sep 27 2026 (정확한 시간 및 타임존 표시 확인)
   ```

---

## 6. 실무에서는 이렇게 씁니다 (현장 적용 관점)

* **하드웨어 고장(Fault)과 소프트웨어 버그(Bug)의 구별법**:  
  전면 Amber LED가 켜졌다고 무작정 벤더에 RMA(하드웨어 교체)를 신청하지 마시고, 반드시 `show environment` 명령어를 먼저 쳐보셔야 합니다. 온도가 30°C 대의 정상 범위라면 99% 펌웨어 센서 오진단 버그이므로 OS 업그레이드로 해결할 수 있습니다.
* **초기 셋업 시 타임존 설정 필수**:  
  SAN 스위치 셋업 시 타임존을 빼먹으면 나중에 FC 포트 Flapping이나 주황색 LED 에러가 났을 때 "이 로그가 새벽 3시 로그인가, 낮 12시 로그인가" 헤매게 됩니다. 셋업 초기 단계에서 `clock timezone KST 9 0` 명령어를 넣어주는 습관이 중요합니다.

---

## 7. 마무리 및 요약

* **한눈에 보는 핵심**:
  - NX-OS 9.2.2 온습도 센서 오진단 Amber LED는 **9.4.5 업그레이드로 깔끔하게 해결**.
  - `clock timezone KST 9 0`으로 표준시 설정 완료.
  - `copy running-config startup-config`로 영구 저장 필수.
