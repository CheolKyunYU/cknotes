---
title: "【HPE Alletra 9060】そのまま真似できる OS 9.6.30 無停止アップグレード作業手順"
description: "HPE Alletra 9060 ストレージの OS 9.6.30 無停止アップグレード手順を、実機画面に沿ってそのまま進められるよう分かりやすく解説した作業ガイド。"
date: 2026-10-04T17:30:00+09:00
draft: false
tags: ["HPE", "Alletra", "Alletra9060", "Alletra9000", "Primera", "OSUpgrade", "Storage", "Tech"]
categories:
  - Storage
---

## 1. はじめに：HPE Alletra 9000 系列ファームウェア更新の重要原則

企業の最重要ミッションクリティカル基盤を支える **HPE Alletra 9060（Alletra 9000 / Primera アーキテクチャベース）** ストレージは、デュアルまたはクアッドコントローラー構成により、完全無停止（Online）でのファームウェアアップグレードに対応しています。

Alletra 9000 系列のOS更新において最も重要な前提条件は、<strong>最新の Upgrade Tool（UT）の事前準備</strong>です。ストレージOS本体を適用する前に、対象バージョンに対応した Upgrade Tool をシステムに先行登録しておくことで、正確な事前適合性チェック（Readiness Check）と安全なローリングノード再起動が実現します。

本記事では、実運用中の HPE Alletra 9060 実機をベースに、OS 9.6.5 から **OS 9.6.30（Extended Support Release）** への無停止アップグレード手順と現場での確認ポイントを解説します。

---

## 2. 作業前の事前準備とパッケージ入手

### 2.1. 必須パッケージの入手手順
Alletra 9000 および Primera のファームウェアとツールは、HPE 公式ライセンスポータルより入手します：

1. **HPE Enterprise License Portal（My HPE Software Center）** へアクセス
2. **Software** メニュー ➔ 保守契約情報（SAID）またはシリアル番号を入力
3. 以下のパッケージをダウンロード：
   * **最新 Upgrade Tool パッケージ**: `Upgrade Tool 80（build 260625）` 以降
   * **対象 OS ファームウェアパッケージ**: `HPE Alletra 9000 OS 9.6.30.xx`（tar/iso 形式）

### 2.2. 事前確認チェックリスト
* **ホスト I/O マルチパス（Multipath）正常性の確認**: ノード再起動時に残存ノードへ I/O が集約されるため、接続ホスト（15台以上）のマルチパス稼働パスを確認
* **ストレージハードウェアおよびシステム健全性の点検**: コントローラーノード、電源モジュール、ドライブエンクロージャーの正常性（OK）および未解決アラート（New alerts: 0）の確認
* **備考（Remote Copy およびスケジュールジョブ）**: 最新の Alletra/Primera OS はノードのローリング再起動時にもリモートコピー（Remote Copy）セッションやスケジュールを安全に自律制御するため、特定のリリースノート・アドバイザリ（Advisory）がない限り、手動停止を行わずオンラインのまま作業を進めます。

---

## 3. ステップ別 無停止アップグレード実務手順

### Step 1. 事前ダッシュボードおよびシステム健全性の点検
Webコンソールにログインし、システムアラート（New alerts: 0）、利用可能容量、ホスト接続状態を確認します。`System` ➔ `Software` より現在の稼働バージョン（9.6.5）を確認します。

{{< figure src="fig-01-dashboard-precheck.png" caption="図 1. Dashboard 事前確認 - システム健全性、容量および接続ホスト状態の確認" >}}

{{< figure src="fig-02-system-overview-precheck.png" caption="図 2. System 画面 - 現在稼働中の OS バージョン (9.6.5) およびハードウェア概要" >}}

<br>

### Step 2. Upgrade Tool および OS 9.6.30 パッケージのロード
右側の `Actions` メニューから **Load an update package** をクリックし、事前にダウンロードした <strong>Upgrade Tool（UT 80）</strong> と **OS 9.6.30 パッケージ** をストレージへアップロードします。

{{< figure src="fig-03-load-update-package.png" caption="図 3. Load an update package - アップグレードパッケージの選択とアップロード" >}}

<br>

### Step 3. 事前適合性検査（Pre-upgrade Readiness Checks）および警告対応
本アップグレードを開始する前に、システム健全性と前提条件を検証する事前適合性検査（Readiness Checks）を実行します。

#### ポートトポロジー警告の確認と承認
事前検査において `Check Port Topology Consistency` で Warning が発生する場合があります。これは特定のSANファブリックポート（`0:3:1`, `0:3:2`, `1:3:1`, `1:3:2`）間のトポロジー差異によるもので、実環境上意図された構成であることを確認します。

{{< figure src="fig-05-readiness-warning-topology.png" caption="図 4. 事前 Readiness Checks 結果 - ポートトポロジー整合性警告（Warning）の詳細" >}}

<br>

PD、LD、IOCTL、ホストパス整合性などの主要項目がすべて `Passed` であることを確認後、`Ignore checks` 画面にて内容を承認し **Yes, ignore** を選択して事前検査を通過し次へ進みます。

{{< figure src="fig-06-readiness-passed-checks.png" caption="図 5. 事前 Readiness Checks 項目 - 物理ディスク、ボリューム状態およびホストパスの正常確認" >}}

{{< figure src="fig-07-readiness-ignore-proceed.png" caption="図 6. Ignore checks ダイアログ - 警告確認および事前検査の承認" >}}

<br>

### Step 4. 無停止ローリングノード（Rolling Node）OSアップグレードの実行
事前検査の承認後、アップグレードを開始すると、**Upgrade Tool 80** エンジンにより事前スクリプト実行およびセカンダリパーティションへのOSファイルシステム展開が行われます。

{{< figure src="fig-08-upgrade-initiated-ut80.png" caption="図 7. OS update in progress - Upgrade Tool 80 による事前スクリプト稼働" >}}

<br>

#### Node 0 のアップグレードおよび再起動
基本ファイルシステム展開後、<strong>Node 0</strong> がクラスタから一時離脱（Leaving cluster, `11:13:20`）し、新OS（9.6.30）でフル再起動します。この間のホストI/Oは Node 1 が継続処理し、再起動後に Node 0 は自動復帰（Rejoined, `11:22:13`）します。

{{< figure src="fig-09-node0-upgrade-and-reboot.png" caption="図 8. Node 0 OS更新 - クラスタ離脱、9.6.30再起動および正常復帰完了" >}}

<br>

> 💡 **現場の実務ヒント（管理IPフェイルオーバー）**:  
> マスターノード再起動時には、管理Webコンソールが対向ノードへ引き継がれるため、一時的に進行中モーダル（`Update in progress 16%`）が表示されます。正常な管理セッション切り替え処理ですので、画面を閉じずに待機します。

{{< figure src="fig-10-management-failover-progress.png" caption="図 9. ノード再起動に伴う管理コンソール切り替え進行画面" >}}

<br>

### Step 5. Node 1 ローリング更新および Web UI 再接続
Node 0 の正常稼働および同期が確認された後、続いて <strong>Node 1</strong> がクラスタを離脱し新OSで再起動します（`11:27:32`）。

Node 1 が再起動している間に Web コンソールへ再接続すると、上部に **メンテナンスモード（Maintenance Mode）** バナーが表示され、全体のOSインストール進捗状況（**Installing HPE Alletra 9000 9.6.30: 69%**）が確認できます。

{{< figure src="fig-04-staged-packages-and-maintenance-mode.png" caption="図 10. Node 1 再起動中の Web UI 再接続画面 - インストール進捗（69%）およびメンテナンスモード稼働" >}}

<br>

Node 1 が再起動を完了してクラスタへ復帰（`11:35:16`）すると、両ノードが 9.6.30 で完全同期され、自動的に事後点検（Post-upgrade, `11:41:23`）フェーズへと移行します。

{{< figure src="fig-11-node1-upgrade-and-postcheck.png" caption="図 11. Node 1 OS更新 - 再起動、クラスタ復帰および事後点検（Post-upgrade）開始" >}}

<br>

### Step 6. 事後点検完了および最終ファームウェアバージョンの検証
両ノード更新完了後、システムは一時ステージングファイル（`OS-9.6.20.6` 等）をクリーンアップします。`Continue update software` 画面で確認後 **Yes, continue** をクリックすると、事後処理が完了（`11:49:45 No issues reported`）します。

{{< figure src="fig-12-continue-update-confirmation.png" caption="図 12. Continue update software - 事後点検承認ダイアログ" >}}

{{< figure src="fig-13-post-upgrade-no-issues.png" caption="図 13. 事後点検完了 - No issues reported 確認による最終終了" >}}

<br>

すべての処理完了後、`Software` 画面にて **Current version: 9.6.30** が表示され、推奨事項に `You're all up to date.` が表示されます。メンテナンスモードが自動解除され、作業完了となります。

{{< figure src="fig-14-software-status-verified.png" caption="図 14. Software 最終画面 - OS 9.6.30 適用完了および健全性確認" >}}

---

## 4. エンジニアリング実務の要点まとめ

| 項目 | 主要チェック内容 | 実務エンジニアリングの推奨事項 |
| :--- | :--- | :--- |
| **Upgrade Tool 先行準備** | UT バージョン整合性 | OS 更新前に必ず最新 Upgrade Tool を登録し、事前検査の信頼性を確保する |
| **パッケージ入手** | HPE Software Portal | HPE Enterprise License Portal の Software 項目より正規契約ベースで入手 |
| **高可用性（HA）検証** | Host Multipath | ローリング再起動時は単一ノードへ I/O が集約されるため、全ホストのマルチパス正常性を事前確認 |
| **トポロジー警告対応** | Port Topology Warning | スイッチゾーニング等の構成差分を事前把握し、意図した設計であることを確認の上で承認 |

---

## 5. まとめおよび技術サポートのご案内

HPE Alletra 9060 の **OS 9.6.30** への更新は、最新のセキュリティパッチと卓越したコントローラー安定性を提供する Extended Support Release です。入念な事前点検と Upgrade Tool の事前準備を行うことで、100% 無停止で安全に完了できます。

> ⚠️ **テクニカルサポートに関するご案内**:  
> Alletra 9000 ストレージは企業の基幹ミッションクリティカルデータを担うコアインフラです。アップグレードの事前検証、SANトポロジー警告の精査、またはファームウェア適合性についてご不明な点がある場合は、<strong>HPE Pointnext 公式サポート窓口または認定パートナー企業の専任エンジニア</strong>へご相談されることを強く推奨いたします。
