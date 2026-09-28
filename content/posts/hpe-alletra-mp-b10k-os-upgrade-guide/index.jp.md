---
title: "[HPE Alletra Storage MP B10K] 10.5.50 → 10.6.0 OSファームウェアアップグレード実践ガイド (新UI刷新と事前Readiness Check必須検証)"
description: "HPE Alletra Storage MP B10K(B10120)における10.6.0 OS無停止アップグレードの全手順、必須となるSystem Readiness Checks事前検証、および大幅刷新された新ホワイトUIの適用ポイントを徹底解説します。"
date: 2026-09-28T21:00:00+09:00
draft: false
tags: ["HPE", "Alletra", "AlletraMP", "B10K", "B10120", "Firmware", "OS Upgrade", "Storage", "GreenLake", "Troubleshooting"]
categories:
  - Storage
---

> **著者**: 16年目 ITシステムエンジニア (CK notes)  
> **対象機器**: HPE Alletra Storage MP B10120 (B10K 2ノードストレージシステム)  
> **OSバージョン**: HPE GreenLake for Block Storage OS 10.5.50 ➔ 10.6.0

---

## 1. 背景: Alletra MP 10.6.0 アップグレードと新UIへの刷新

ミッションクリティカルストレージである **HPE Alletra Storage MP (B10K / B10120)** において、システムの安定性向上と最新機能のサポートを目的とした **10.6.0 OSファームウェアアップグレード** を実施しました。

今回の10.6.0リリースは、バックエンドの性能向上とプラットフォームの安定化だけでなく、**Web管理コンソールが従来のダークサイドバーからモダンなホワイトツリー型新UIへと大幅に刷新される**重要なアップデートとなっています。

ストレージがインターネット(HPE Cloud Connect)に接続されている環境であれば、**10.6.0ファームウェアパッケージが自動的にStaged(事前ダウンロード)** されているため即座に作業を開始できます。閉域網環境の場合でも、ローカルコンソールからファイルを直接アップロードして安全に適用可能です。

---

## 2. 環境および前提条件

| 項目 | スペック / 設定情報 | 実務上の設定理由 |
| --- | --- | --- |
| **対象ストレージ** | HPE Alletra Storage MP B10120 | 2ノード アクティブ-アクティブ エンタープライズブロックストレージ |
| **現行OSバージョン** | OS 10.5.50 | 既存稼働ファームウェアバージョン |
| **ターゲットOSバージョン** | OS 10.6.0 (ArcusOS) | 新UI対応およびプラットフォーム安定化推奨バージョン |
| **パッケージ準備方式** | オンラインStaged / ローカルファイル手動アップロード | インターネット接続時は10.6.0自動受信済み |
| **事前必須検証** | System Readiness Checks | アップグレード前に100% Passed確認が必須 |
| **想定所要時間** | 約60分 〜 90分程度 | ノード順次再起動およびCDMコンポーネント書き込み含む |

---

## 3. 実践アップグレード手順 (ステップ別)

### Step 1. 現在のOSバージョンおよびダッシュボード状態の点検
作業を開始する前にオンプレミスWebコンソールへログインし、**System ➔ Dashboard** にて現行OSバージョン(`10.5.50`)と各コントローラー、シャーシ、IOMモジュールの正常稼働状態(`OK`)を点検します。

{{< figure src="step-01-dashboard-check.png" caption="Step 1. アップグレード前のシステムダッシュボードおよび既存OS 10.5.50稼働状態の確認" >}}

<br>

### Step 2. Software メニューへの移動および10.6.0パッケージの確認
左側メニューの **System ➔ Software** タブへ移動します。  
インターネットに接続されている環境では、推奨リリースの `10.5.60` および最新の `10.6.0` パッケージが **Staged updates** 一覧に自動的にダウンロードされ、適用待機状態になっていることが確認できます。

> 💡 **閉域網(オフライン)環境の場合**:  
> 右上の `Load an update package` メニューをクリックし、事前ダウンロードしたファームウェアISOイメージパッケージ（例：`OS-10.5.50.19.iso`、`OS-10.6.0.xx.iso` 形式）をブラウザ経由でストレージローカル領域へ直接アップロードします。

{{< figure src="step-02-software-updates.png" caption="Step 2-1. Softwareメニューで自動受信された10.6.0 Stagedアップデートパッケージの確認" >}}

{{< figure src="step-02b-load-package.png" caption="Step 2-2. 閉域網環境向け Load an update package 手動アップロード画面" >}}

<br>

### Step 3. ★ 必須事前手順: System Readiness Checks (事前互換性・健全性検証)
アップデートを実行する前に、**必ず確認しなければならない最も重要なステップ**です。  
Staged一覧の右側にある **`View readiness checks`** リンクをクリックし、事前検証結果を確認します。

* **重要チェック項目**:
  - `Check System Status` & `Restart system logger`: Passed
  - `Ensure Node Disk Free Space`: ノード内ディスク空き容量の十分性
  - `Verify Host Connectivity`: 接続されているホストパスの正常性
  - `Check VV` & `Check VLUNs`: ボリュームおよびLUNマッピングの整合性

すべての項目が緑色の **Passed** となっていることを確認した上で次のステップへ進みます。

{{< figure src="step-03-readiness-checks.png" caption="Step 3. System readiness checks 実行結果 - システム/ホスト/ボリューム全項目 100% Passed の検証" >}}

<br>

### Step 4. Update Software の実行および10.6.0パッケージの選択
事前点検の成功を確認したら、右上の **`Update software`** アクションボタンをクリックします。  
パッケージ選択画面で **`HPE GreenLake for Block Storage 10.6.0`** にチェックを入れ、下部の **`Install`** ボタンを押します。

{{< figure src="step-04-select-package.png" caption="Step 4. Update software 画面で HPE GreenLake for Block Storage 10.6.0 を選択し Install を実行" >}}

<br>

### Step 5. 10.6.0 OSファームウェアアップグレードのバックグラウンド起動
インストールが開始されると、画面上部に `Starting installation of HPE GreenLake for Block Storage 10.6.0` の通知バナーが表示され、バックグラウンドでのアップグレードセッションが有効化されます。

{{< figure src="step-05-install-started.png" caption="Step 5. 10.6.0 インストール開始およびバックグラウンド Activities タスクの起動" >}}

<br>

### Step 6. 無停止OSアップグレードの進行と新UIへの自動切り替え
Alletra MPのOSアップグレードは、コントローラーノードを順次フェイルオーバーしながらI/Oを停止することなく進行します。

1. **事前チェック (Pre-update checks)**: システム整合性の最終検証。
2. **ノードソフトウェア・ファームウェア書き込み**: Node 0 および Node 1 へのパッケージ適用。
3. **ノード順次再起動 (Node Reboot)**: Node 0 再起動および健全性復帰確認 ➔ Node 1 再起動。
4. **バージョン切り替えとコンソール再起動**: システムバージョンが10.6.0へ切り替わる間、一時的に `Update in progress` ポップアップが表示されます。
5. **コンポーネントファームウェア適用と新UIへの刷新**:  
   バージョンスイッチが完了した瞬間、**Webコンソールがモダンなホワイトツリー型の新UIへ自動的に更新**され、バックグラウンドでドライブおよびCDM(Chassis Discovery Module)ファームウェアの最終適用が行われます。

{{< figure src="step-06a-prechecks-running.png" caption="Step 6-1. OS update in progress - Initializing pre-update checks 進行" >}}

{{< figure src="step-06b-node-reboot.png" caption="Step 6-2. 36% 進行 - Node 0 健全性確認後の順次再起動およびフェイルオーバー維持" >}}

{{< figure src="step-06c-version-switch.png" caption="Step 6-3. 44% バージョンスイッチ進行 - Web管理コンソールセッション自動更新ポップアップ" >}}

{{< figure src="step-06d-new-ui-component-firmware.png" caption="Step 6-4. 96% 新ホワイトUIへの自動切り替えおよびCDMコンポーネントファームウェア最終書き込み" >}}

<br>

### Step 7. OSアップグレード完了の確認
全工程が完了すると、**`OS update successful`** 画面が表示され、`HPE Alletra Storage 10.6.0 update completed successfully` のメッセージが出力されます。

{{< figure src="step-07-update-success.png" caption="Step 7. OS update successful - 10.6.0 アップグレード全項目 Completed 完了" >}}

<br>

### Step 8. 新UIダッシュボードの検証および10.6.0最終稼働確認
アップグレード完了後、新しくなった **System ➔ Details / Software** ダッシュボードへ移動し、システム状態を最終点検します。

* **OS version**: `10.6.0` の正常適用を確認。
* **ハードウェア健全性**: Enclosure chassis、Controllers、Drive IOMs、Drives、Ports、Switches の全項目が緑色チェック(`OK`)であることを確認。
* **新ナビゲーションツリー**: Storage、Protection、System、Settings、Reports の階層が直感的に整理されていることを確認します。

{{< figure src="step-08-final-dashboard-10-6-0.png" caption="Step 8. 新ホワイトUIが適用された System ダッシュボードで OS 10.6.0 正常稼働を検証" >}}

---

## 4. 🚨 トラブルシューティングおよび注意事項

### 注意 1: Readiness Check で Warning または Failed が発生した場合
* **原因**: ホストのマルチパス切断、バックアップタスクの実行中、ノード空き容量不足など。
* **対処**: Readiness Checks でエラーや警告が出ている場合は、絶対に `Ignore` をチェックして強制実行せず、原因を解消してから `Re-run checks` を行い、**全項目が Passed になってからアップデートを開始**してください。

### 注意 2: 44% バージョンスイッチ時のWebコンソール一時応答停止について
* **事象**: ノード再起動とWebデーモンの再起動に伴い、ブラウザの応答が1〜2分程度停止したり更新ポップアップが表示されたままになる場合があります。
* **対処**: 新しいWebサービスへの正常な切り替え処理中であるため、ブラウザを閉じたりF5を連打せず2〜3分待機してください。処理完了後に自動で新ホワイトUI画面へリフレッシュされます。

### 注意 3: 自社での対応が困難・不安な場合（HPE専門エンジニアへの支援要請の必須推奨）
* **推奨事項**: HPE Alletra MP ストレージは企業のミッションクリティカルな中核データを支える基幹インフラです。自社の運用体制で対応が困難な場合、事前 Readiness Check で警告やエラーが解消しない場合、またはオフライン手動アップロードなど作業に不安がある場合は、**無理に自社単独で実行せず、必ずHPE公式サポート（Pointnext）またはHPE認定パートナーの専任エンジニアによる技術支援を依頼**し、安全に作業を実施してください。

---

## 5. 検証手順 (作業完了チェック項目)

1. **OSバージョンの確認**: `System` ➔ `Software` で `OS version: 10.6.0` を確認。
2. **コンポーネント健全性の確認**: コントローラー2ノードおよびIOM、電源、ファンモジュールの正常(`OK`)を確認。
3. **ホストマルチパスの確認**: 接続サーバー側でSANストレージボリュームへのアクセスが無停止で継続されていることを確認。

---

## 6. 実務におけるポイント

* **クラウド接続(Staged)機能を活用する**:  
  Alletra MPはクラウドコネクテッド設計のため、外部通信が許可された環境であれば手動でファイルを探してダウンロードする手間なく、Staged機能によりワンクリックで安全にパッケージを準備できます。
* **10.6.0 新UIへの適応**:  
  従来の単一階層から `System` 配下に `Details`、`Software`、`Controllers`、`Drives`、`Ports` などがツリー状に体系化されたため、日常点検や障害時の切り分け動線が大幅に効率化されています。

---

## 7. まとめ

* **重要ポイント**:
  - 既存 10.5.50 確認 ➜ Staged 10.6.0 パッケージ確認 ➜ **`System Readiness Checks` 100% Passed 検証 (必須)** ➜ `Update software` 実行 ➜ ノード無停止再起動 & 新ホワイトUI自動切り替え ➜ 10.6.0 最終稼働検証。
  - 事前チェックを徹底することで、約1時間で安全に最新バージョンと新UIへの移行が完了します。
