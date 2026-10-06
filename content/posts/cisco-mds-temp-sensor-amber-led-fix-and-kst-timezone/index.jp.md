---
title: "【Cisco MDS / HPE SN6620C】Tempセンサー誤検知Amber LED解消およびTimezone設定ガイド"
description: "Cisco MDS 9148T / HPE SN6620C NX-OS 9.2.2における偽の温度Amber LEDアラームの原因と9.4.5アップグレードによる解決、タイムゾーン設定の解説。"
date: 2026-09-27T15:30:00+09:00
draft: false
tags: ["Cisco", "MDS", "HPE", "SN6620C", "Temperature", "Amber LED", "Timezone", "Troubleshooting", "Storage", "SAN"]
categories:
  - Storage
---

## 1. 背景: サーバールームは低温なのに、なぜスイッチ前面にAmber LEDが点灯するのか？

データセンター現場での保守作業時、室温は18〜20°Cと十分に涼しく保たれているにもかかわらず、**Cisco MDS 9148T (HPE SN6620C) ストレージスイッチ前面パネルのSYS/ENVステータスLEDがオレンジ色(Amber)に点灯している現象**によく遭遇します。

実際に `show environment` コマンドで内部センサー温度を確認すると `32°C (Normal)` と極めて正常な状態であるにもかかわらず、NX-OS 9.2.2 のセンサーポーリング処理バグ(CSCwo09244等)によって閾値超過アラートが誤検知され、警告灯が点灯してしまうのが原因です。

本記事では、**温湿度センサー誤診断によるAmber LED問題の原因と、推奨安定版9.4.5へのアップグレードによる根本的解決手順**、さらに障害解析時にログのタイムスタンプ齟齬を防ぐための**タイムゾーン(Timezone)設定方法**を整理します。

---

## 2. 環境および前提条件

| 項目 | スペック / 設定情報 | 実務上の設定理由 |
| --- | --- | --- |
| **対象機器** | Cisco MDS 9148T / HPE SN6620C | 32G Enterprise FC SAN スイッチ |
| **発生バージョン** | Cisco NX-OS 9.2(2) | 温湿度センサーポーリング誤診断バグが存在 |
| **対処完了バージョン** | Cisco NX-OS 9.4(5) | センサーポーリングおよびプラットフォーム監視バグ修正済み |
| **適用タイムゾーン** | Timezone (KST/JST, UTC+9) | 標準時基準でのログ解析同期化 |

---

## 3. 実践対処および設定手順

{{< figure src="step-01-temp-amber-led-status.jpg" caption="作業前のスイッチ前面LEDおよびステータス点検画面" >}}

### Step 1. 温湿度センサーおよび環境ステータスの点検
前面のAmber LEDが点灯した際、ハードウェア故障かソフトウェアバグかを切り分けます。

```bash
# 環境センサーおよび温度情報の詳細確認
show environment temperature
```

出力結果で全センサー項目が `OK` かつ正常温度範囲(Normal)内にあるにもかかわらずAmber LEDが消灯しない場合、**NX-OS 9.2.2 のセンサー誤診断バグ**であると判断できます。

<br>

{{< figure src="step-04-temp-amber-led-normal.jpg" caption="ファームウェア更新後のAmber正常化 (show environment センサー状態および正常範囲検証)" >}}

### Step 2. 9.4.5 OS アップグレードによるセンサーバグの根本解決
センサー誤診断バグを解消するため、スイッチOSを推奨安定リリースである **NX-OS 9.4(5)** へアップグレードします。  
アップグレード完了後に機器が再起動すると、プラットフォーム監視デーモンが再初期化され、オレンジ色のAmber LEDが消灯して**正常な緑色(Green) LEDへ復帰**します。

<br>

{{< figure src="step-02-timezone-kst-setting.jpg" caption="clock timezone コマンドによるタイムゾーン設定" >}}

### Step 3. タイムゾーン(Timezone)の設定
スイッチがデフォルトのUTC(協定世界時)のままだと、将来の障害発生時にサーバーやストレージのログと突き合わせる際に9時間の時差が生じ、原因究明に時間がかかります。

```bash
# グローバルコンフィギュレーションモードへ移行
switch# configure terminal

# タイムゾーンの設定 (UTC +9時間)
switch(config)# clock timezone KST 9 0

# 設定の確認
switch(config)# show clock
```

*(理由: ストレージSANスイッチのログタイムスタンプを運用標準時に揃え、トラブルシューティング時間を大幅に短縮するため)*

### Step 4. ランニングコンフィグの保存
設定したタイムゾーン情報が機器再起動後も保持されるよう保存します。

```bash
# 設定の保存
switch# copy running-config startup-config
```

---

## 4. 🚨 トラブルシューティングノート (現場での実務対応)

### 事象 1: 温度は正常(32°C)なのに前面SYS/ENV LEDがオレンジ色(Amber)のまま
* **エラー内容**: `show environment` の結果はすべて `Normal` であるにもかかわらず前面LEDがAmber。
* **原因**: NX-OS 9.2.2 のIOSlice温度センサーポーリング再試行ロジックのバグ(`%PLATFORM-4-MOD_TEMPFAIL` 誤検知)。
* **対処方法**: 
  - コマンドによるソフトリセット(`clear environment history`)では根本解決しません。
  - **NX-OS 9.4(5) バージョンへのアップグレード適用により、100% 正常な緑色(Green) LEDへの復帰を確認完了。**

---

## 5. 検証手順 (作業完了チェック項目)

1. **前面LEDの物理確認**:
   - スイッチ前面パネルの `SYS`、`ENV`、`FAN` LEDがオレンジ色(Amber)から**正常な緑色(Green)**に変更されたことを確認。
2. **センサー状態の確認**:
   ```bash
   show environment
   # 結果: Power, Fan, Temp 等の全項目が OK / Normal であることを確認
   ```
3. **タイムゾーン適用の確認**:
   ```bash
   show clock
   # 出力例: 15:35:12.123 KST Sun Sep 27 2026 (正確な時刻とタイムゾーン表示を確認)
   ```

---

## 6. 実務におけるポイント

* **ハードウェア故障(Fault)とソフトウェアバグ(Bug)の見極め**:  
  前面Amber LEDが点灯したからといってすぐにハードウェア交換(RMA)を申請せず、必ず `show environment` コマンドで状態を確認してください。温度が30°C台の正常範囲であれば、99%ファームウェアのセンサー誤診断バグであるためOSアップグレードで解決できます。
* **初期セットアップ時のタイムゾーン設定は必須**:  
  SANスイッチの初期設定時にタイムゾーン設定を省略すると、リンク障害やエラー発生時にログ解析で混乱します。セットアップ時に `clock timezone` を設定する習慣をつけておきましょう。

---

## 7. まとめ

* **重要ポイント**:
  - NX-OS 9.2.2 温湿度センサー誤診断によるAmber LEDは **9.4.5 アップグレードで根本解決**。
  - `clock timezone` で運用標準時を設定。
  - `copy running-config startup-config` で設定を永続化。
