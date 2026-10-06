---
title: "【HPE SN6620C / Cisco MDS】9.2.2 → 9.4.5 OSファームウェアアップグレード実践ガイド"
description: "HPE SN6620C(Cisco MDS 9148T OEM)スイッチにおける現行バージョン確認からRebex Tiny SCPによるファイル転送、install allインストール、deleteによるbootflash容量整理までの標準作業手順です。"
date: 2026-09-27T15:00:00+09:00
draft: false
tags: ["HPE", "SN6620C", "Cisco", "MDS", "Firmware", "NX-OS", "SCP", "Storage", "SAN", "Troubleshooting"]
categories:
  - Storage
---

## 1. 背景: なぜ 9.4.5 バージョンへのアップグレードが必要だったのか？

HPE SN6620C (Cisco MDS 9148Tベース) 32Gb FC SANスイッチを運用中のお客様環境において、安定性向上のためのファームウェア更新を実施しました。

旧バージョンの9.2.2では、長期稼働時の環境センサー誤検知やISSU実行時のBIOSタイムアウトの懸念が報告されていました。そのため、HPEおよびCisco公式の**推奨安定バージョン(Recommended Release)であるNX-OS 9.4(5)**へのアップグレードを決定しました。

外部接続が遮断された閉域網環境であるため、**エンジニアの作業用ノートPCにRebex Tiny SFTP/SCP Serverを起動し、ローカル接続でファームウェアを転送**する方式を採用しました。

---

## 2. 環境および前提条件

| 項目 | スペック / 構成情報 | 実務上の設定理由 |
| --- | --- | --- |
| **対象スイッチ** | HPE SN6620C (Cisco MDS 9148T OEM) | 32Gbps ファイバチャネル(FC) SAN スイッチ |
| **現行 NX-OS バージョン** | 9.2(2) | 既存稼働バージョン |
| **ターゲット NX-OS バージョン** | 9.4(5) | HPE/Cisco 推奨安定版 (Recommended) |
| **転送プロトコル** | SCP (Secure Copy Protocol) | 作業PC ↔ スイッチ Managementポート直結転送 |
| **SCP サーバーツール** | Rebex Tiny SFTP/SCP Server | インストール不要のポータブルSCPサーバー |
| **ファームウェアファイル** | `m9000-pkg2.9.4.5.bin`, `m9000-kickstart-pkg2.9.4.5.bin` | システム(System)およびKickstartバイナリパッケージ |

---

## 3. 実践アップグレード手順 (ステップ別)

### Step 1. 現在のOSバージョンおよびスイッチ状態の確認
作業開始前にスイッチCLIに接続し、現行バージョン(`show version`)とモジュール/ポート状態(`show module`)を点検します。

```bash
# 現在のOSバージョンおよびシステム情報の確認
show version

# モジュールおよびポート稼働状態の確認
show module
```

{{< figure src="step-01-switch-status.jpg" caption="Step 1. ストレージスイッチCLI接続および既存9.2.2バージョン状態確認" >}}

<br>

### Step 2. Rebex Tiny SCP Server のダウンロードと設定
閉域網での作業に適したポータブルツール **Rebex Tiny SFTP/SCP Server** を準備・設定します。
1. ポータブル実行ファイルをダウンロードして作業PCに配置します。
2. 起動後、**User/Password** (例: `scpuser` / `P@ssw0rd`) を設定し、ファームウェアファイル格納フォルダを **Root Directory** に指定します。
3. `Start Server` ボタンを押してSCPサービスを即座に有効化します。

{{< figure src="step-02a-rebex-download.jpg" caption="Step 2-1. Rebex Tiny SFTP/SCP Server ポータブル版のダウンロード" >}}

{{< figure src="step-02b-rebex-main.jpg" caption="Step 2-2. Rebex SFTP/SCP Server メイン画面" >}}

{{< figure src="step-02c-rebex-config.jpg" caption="Step 2-3. Rebex ユーザーアカウントおよびファームウェアフォルダ設定" >}}

{{< figure src="step-02d-rebex-start.jpg" caption="Step 2-4. Start Server ボタンによるSCPサーバー起動完了" >}}

<br>

### Step 3. `copy` コマンドによるファームウェア転送
スイッチCLIから `copy scp:` コマンドを実行し、作業PC内のファームウェアファイル(`m9000-pkg2.9.4.5.bin`, `m9000-kickstart-pkg2.9.4.5.bin`)をスイッチの `bootflash:` 領域へダウンロードします。

```bash
# 作業PC(SCPサーバー)から bootflash: へファイル転送 (System & Kickstart)
copy scp://scpuser@192.168.1.100/m9000-pkg2.9.4.5.bin bootflash: vrf management
copy scp://scpuser@192.168.1.100/m9000-kickstart-pkg2.9.4.5.bin bootflash: vrf management
```

{{< figure src="step-03-copy-scp.jpg" caption="Step 3. copyコマンドでPCからファームウェアをスイッチbootflashへ転送" >}}

<br>

### Step 4. `install all` コマンドによるファームウェアインストールと事前互換性チェック
ファイル転送が完了したら、`install all` コマンドでSystemおよびKickstartイメージを同時指定して実行し、イメージ整合性および事前互換性チェックを行って本番適用を開始します。

```bash
# 9.4.5 OS ファームウェアのインストール開始 (System 및 Kickstart 동시 지정)
install all system bootflash:m9000-pkg2.9.4.5.bin kickstart bootflash:m9000-kickstart-pkg2.9.4.5.bin
```

インストール中はシステムが自動でモジュール検証とパッチ適用を行い、完了後に再起動が実行されます。

{{< figure src="step-04a-install-cmd.jpg" caption="Step 4-1. install all コマンドによるNX-OS 9.4.5 ファームウェアインストール開始" >}}

{{< figure src="step-04b-install-impact.jpg" caption="Step 4-2. install all コマンドによる事前互換性チェック (Impact Analysis)" >}}

{{< figure src="step-04c-install-upgrading.jpg" caption="Step 4-3. ファームウェアパッチ適用中 (Writing System Image)" >}}

<br>

### Step 5. 再起動完了後の 9.4.5 バージョン適用確認
スイッチの再起動完了後、CLIに接続してターゲットバージョン(NX-OS 9.4.5)が正常に適用されていることを確認します。

```bash
# OSバージョンの最終確認
show version
```

{{< figure src="step-05-boot-complete.jpg" caption="Step 5. 再起動完了後のカーネルおよびOS正常ロード確認 (NX-OS 9.4.5 適用完了)" >}}

<br>

### Step 6. 作業完了後の `delete` コマンドによる bootflash 容量整理
アップグレード完了後、bootflashの容量枯渇を防ぐため、転送したファームウェアバイナリを `delete` コマンドで削除します。

```bash
# bootflash の容量確認
dir bootflash:

# 転送した 9.4.5 SystemおよびKickstartバイナリの削除
delete bootflash:m9000-pkg2.9.4.5.bin
delete bootflash:m9000-kickstart-pkg2.9.4.5.bin

# 削除後の空き容量再確認
dir bootflash:
```

{{< figure src="step-06-delete-cleanup.jpg" caption="Step 6. 作業完了後に delete コマンドで bootflash 容量整理" >}}

---

## 4. 🚨 トラブルシューティングノート (現場での実務対応)

### 事象 1: SCP 転送時に `Host key verification failed` エラーが発生
* **エラー内容**: `error: ssh connect failed / Host key verification failed`
* **原因**: スイッチのSSHクライアントと作業PC(Rebex)間の暗号化アルゴリズム不一致。
* **対処方法**:
  ```bash
  # スイッチCLIでSSHアルゴリズムを手動許可
  switch(config)# ip ssh client algorithm key-exchange dh-group14-sha1
  ```

---

## 5. 検証手順 (作業完了チェック項目)

1. **OSバージョン確認**: `show version` ➔ `system: version 9.4(5)` であることを確認。
2. **SANポート状態確認**: `show interface fc1/1-48 status` ➔ 全FCポートが Online を維持していること。
3. **bootflash容量確認**: `dir bootflash:` ➔ `delete` 実行後の空き容量を確認。

---

## 6. 実務におけるポイント

* **bootflash のクリーンアップを習慣化する**:  
  MDSスイッチのbootflash容量には限りがあります。作業完了後に大容量バイナリを削除しておかないと、次回のファームウェア更新やコアダンプ出力時に容量不足エラーが発生します。

---

## 7. まとめ

* **重要ポイント**:
  - バージョン確認(`show version`) ➜ Rebex SCP起動 ➜ ファイル転送(`copy scp:`) ➜ 適用(`install all`) ➜ 完了確認 ➜ 不要ファイル削除(`delete`)。
  - **bootflash の不要ファイル削除による容量管理**: 空き容量不足によるトラブルや次回作業時のエラーを防ぐため、アップデート完了後は必ず `delete` コマンドで作業用ファイルを削除してください。
