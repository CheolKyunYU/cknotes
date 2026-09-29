---
title: "【HPE Alletra Storage MP B10K】ArcusOS 10.6.0 新規ホワイトツリーWeb UI全面刷新の徹底解説と運用ガイド"
description: "HPE Alletra Storage MP B10K（B10120）ストレージのOS 10.6.0における新規ホワイトツリーUI構造分析、6大コアメニュー体系および主要管理機能の実務ガイド。"
date: 2026-09-30T07:00:00+09:00
draft: false
tags: ["HPE", "Alletra", "AlletraMP", "B10K", "B10120", "ArcusOS", "WebUI", "Storage", "GreenLake", "Tech"]
categories:
  - Storage
---

> **著者**: 16年目 ITシステムエンジニア (CK notes)  
> **対象機器**: HPE Alletra Storage MP B10120 (B10K 2-Node All-NVMe Block Storage)  
> **オペレーティングシステム**: HPE GreenLake for Block Storage OS (ArcusOS) 10.6.0

---

## 1. はじめに：10.6.0 全面刷新とホワイトツリーUIの導入背景

HPEの次世代モジュラー型オールNVMeストレージである **HPE Alletra Storage MP（B10120 / B10K）** 環境において、OSファームウェアが **10.6.0** へアップデートされたことに伴い、システム管理インターフェースが大幅に刷新されました。

従来の10.5.x系まで採用されていた単一階層のダークサイドバーUIから脱却し、10.6.0からは **クラウドネイティブ感覚のモダンなホワイトテーマと、直感的な多階層ツリーナビゲーション構造** が新たに導入されています。

本記事では、実機環境の Alletra Storage MP B10K をベースに、10.6.0 新UIにおけるメニュー構造の変化点と、エンジニアリング実務の観点から重要なモニタリングポイントを整理して解説します。

---

## 2. 10.5.x vs 10.6.0 UI比較

| 比較項目 | 従来UI (OS 10.5.50 以前) | 新規UI (OS 10.6.0) | 実務におけるメリット |
| :--- | :--- | :--- | :--- |
| **テーマデザイン** | 暗いダークグレーのサイドバー | 明るくモダンなホワイトカードベースUI | 視認性が大幅に向上、最新HPE GreenLake統合ルック＆フィール |
| **左側ナビゲーション** | 単純なアイコン中心の単一階層 | テキストラベル付き多階層折りたたみツリー | 下位メニュー（ドライブ、ポート等）へ1クリックで即時遷移可能 |
| **上部ブランディング** | 基本シンボルアイコンのみ表示 | HPEロゴおよび製品モデル名を明記 | 複数ストレージの同時運用時における識別性が向上 |
| **ポート/スイッチ管理** | 詳細ハードウェア配下を探索 | `Ports` / `Switches` が独立メニューとして常時表示 | SANファブリックのリンク状態やポート速度を即座に確認可能 |
| **セキュリティ・設定統合** | 各所に分散していた設定項目 | `Settings` 配下にネットワーク/テレメトリ/証明書を集約 | セキュリティポリシー（ランサムウェア検知、保持期間）のワンストップ設定 |

### アップグレード前／後の System 画面比較

{{< figure src="fig-01-old-ui-system-dashboard.png" caption="図 1. [アップグレード前] OS 10.5.50 ダークサイドバー採用の従来 System ダッシュボード画面" >}}

{{< figure src="fig-02-new-ui-system-dashboard.png" caption="図 2. [アップグレード後] OS 10.6.0 ホワイトツリー型 新規 System ダッシュボード画面" >}}

---

## 3. 新規 10.6.0 6大コアメニューの詳細解説

### 3.1. Dashboard（システムヘルスおよび性能サマリー）
ログイン時に最初に表示されるトップ画面で、システムヘルス、利用可能容量、リアルタイム性能メトリクスが3つのカード領域に整理されて表示されます。

* **Health**: 新規アラート（New alerts）件数とともに、Storage、Protection、System、Call home、Data Services Cloud Console のステータスアイコンがリアルタイムに表示されます。
* **Usable Capacity**: 全体の利用可能容量（Total Free / Total）を直感的なドーナツチャートで可視化し、Private/Shared容量、スナップショット、システム予約、未割り当て（Unallocated）容量を明確に識別できます。
* **Performance**: オンラインホスト数に加え、リアルタイムIOPS、スループット帯域幅（Bandwidth）、応答遅延（Latency）の推移グラフが提供されます。

{{< figure src="fig-03-dashboard-overview.png" caption="図 3. Dashboard 画面 - Health、Usable Capacity、Performance 指標のリアルタイム統合ビュー" >}}

<br>

### 3.2. Storage（ブロックストレージおよび VMware ボリューム管理）
ホストおよびボリュームに関連するすべてのオブジェクト作成とマッピングを統括するコアメニューです。

* **BLOCK 管理**: Virtual volume sets, Volumes, Host sets, Hosts
* **VMWARE 連携**: VMware storage containers
* **PEER MOTION**: 既存ストレージからの無停止データ移行セッション管理
* **Quick actions**: Create host sets, Create virtual volume sets, Storage tutorial のショートカット

{{< figure src="fig-04-storage-overview.png" caption="図 4. Storage メニュー概要 - ブロックボリューム、ホストグループおよび Quick actions カード" >}}

<br>

#### Virtual Volume Sets およびボリューム管理
作成されたボリュームセット（例: VMware ESXi 連携ボリューム）の保護状態、プロビジョニング容量、エクスポート状態をカードグリッド形式で確認できます。

{{< figure src="fig-05-virtual-volume-sets.png" caption="図 5. Virtual volume sets 詳細画面 - ボリュームセットメンバーおよびエクスポート状況の確認" >}}

<br>

#### Host Sets および接続ホスト状態
SANスイッチ経由で接続されたホストサーバーをグループ化した `Host sets` および各ホストの接続健全性（`Normal`）を直感的に監視できます。

{{< figure src="fig-06-host-sets.png" caption="図 6. Host sets 画面 - 接続ホストメンバー構成およびヘルス状態の確認" >}}

<br>

### 3.3. Protection（データ保護およびレプリケーション／バックアップ管理）
リモートレプリケーション、StoreOnceバックアップアプライアンス連携、仮想化・データベース保護ポリシーをカード形式で一元管理します。

* **REPLICATION**: Replication partners, Replication partner systems（リモートレプリケーションパートナー構成）
* **BACKUP SYSTEMS**: HPE StoreOnce バックアップアプライアンス連携
* **APPLICATION PROTECTION**: Application servers, Datastores, Virtual machines, VM protection groups, MSSQL databases / instances

{{< figure src="fig-07-protection-overview.png" caption="図 7. Protection メニュー概要 - レプリケーション、バックアップ機器およびアプリケーション保護ダッシュボード" >}}

<br>

### 3.4. System（ハードウェア、ファームウェア、物理コンポーネント管理）
ストレージ本体の詳細仕様、デュアルコントローラーノード、エンクロージャーシャーシ、NVMeドライブおよびFCポートの状態を集中管理します。

#### System Details（システム詳細情報）
システムモデル名（HPE Alletra Storage MP B10120）、OSバージョン（10.6.0）、稼働時間（Uptime）、暗号化対応状況、ハードウェア構成サマリー（コントローラー 2、シャーシ 1、ドライブ 8、ポート 8）が一目で確認できます。

{{< figure src="fig-08-system-details.png" caption="図 8. System Details 画面 - モデル仕様、10.6.0 OSバージョンおよびライセンス情報" >}}

<br>

#### System Software（OSファームウェアおよび更新ポリシー）
現在稼働中のOSバージョン（`10.6.0`）の最新状態確認に加え、GreenLakeクラウドからの自動ダウンロードおよびステージングポリシー（Software Update Policy）を設定できます。

{{< figure src="fig-09-system-software.png" caption="図 9. System Software 画面 - 最新OSバージョン確認およびソフトウェア更新ポリシー" >}}

<br>

#### Enclosure Chassis および Controllers
シャーシのフロント／リアグラフィックビューとともに、Node 0（Master, Bay 1）/ Node 1（Bay 2）の稼働健全性（`Normal`）、電源モジュール（Power Supply）、シャーシ検出モジュール（CDM）の完全性を点検できます。

{{< figure src="fig-10-system-enclosure-controllers.png" caption="図 10. Enclosure 詳細画面 - シャーシ背面モジュール配置およびデュアルコントローラーのヘルス状態" >}}

<br>

#### Drives & Ports
ストレージに搭載された8基のNVMe SSDドライブスロット（1:1 〜 1:8、各3.84TB）の稼働状態と、8基の高速FCポート（ホスト接続用およびFreeポート）の状態を即座に把握可能です。

{{< figure src="fig-11-system-drives.png" caption="図 11. System Drives 画面 - 8基のオールNVMe SSDドライブの正常稼働状態" >}}

<br>

### 3.5. Settings（ネットワーク、セキュリティ、テレメトリの統合設定）
従来は複数のメニューに分散していたインフラ管理設定を、単一メニュー配下に体系的に集約しました。

* **主要設定項目**: System, Telemetry, Users, Domains, LDAP configuration, Contacts, VMware vCenter, HPE VM Essentials, Network services, Array certificates, Trusted certificates.

{{< figure src="fig-12-settings-overview.png" caption="図 12. Settings メニュー概要 - ネットワーク、ユーザー、証明書および外部連携の統合一覧" >}}

<br>

#### Settings ▶ System（管理ネットワークおよびデータセキュリティポリシー）
管理用IPv4アドレス、サブネットマスク、ゲートウェイ、DNSサーバー情報とともに、ボリューム保持期間（Volume Retention）やランサムウェア検知（Ransomware Detection）ポリシーを中央から一括設定します。

{{< figure src="fig-13-settings-system-network.png" caption="図 13. Settings-System 画面 - 管理ネットワークIP設定およびランサムウェア検知セキュリティポリシー" >}}

<br>

#### Settings ▶ Telemetry（Call Home & Data Services Cloud Console）
リモート障害自動検知およびHPEサポートセンターへのメトリクス送信を担う Call Home の状態と、HPE GreenLake Data Services Cloud Console（DSCC）の連携状態を確認できます。

{{< figure src="fig-14-settings-telemetry-dscc.png" caption="図 14. Settings-Telemetry 画面 - Call Home および Data Services Cloud Console 接続ステータス" >}}

<br>

#### Settings ▶ Contacts（顧客担当者およびサポート窓口管理）
システムアラートや障害発生時に迅速なテクニカルサポートを受けるための担当者連絡先を登録・管理します。

{{< figure src="fig-15-settings-contacts.png" caption="図 15. Settings-Contacts 画面 - サポート担当者プロファイルの管理" >}}

<br>

### 3.6. Reports & Activities（レポーティングおよびタスク監視）

#### Reports（リアルタイム性能／容量レポート）
`Create report` からCPU、メモリ、IOPS、スループット帯域幅の推移を任意の時間軸で即座に抽出し、保存済みレポートを簡単に再ロードできます。

{{< figure src="fig-16-reports.png" caption="図 16. Reports 画面 - リアルタイムカスタムレポートの作成および保存済みレポートのロード" >}}

<br>

#### Activities（アラート、タスクおよびスケジュール管理）
システム内で発生したイベントアラート（Alerts）、バックグラウンドで実行されたファームウェア更新やボリューム作成等のタスク履歴（Tasks）、定期点検スケジュール（Schedules）を統合追跡します。

{{< figure src="fig-17-activities.png" caption="図 17. Activities 画面 - リアルタイムアラート、タスク実行履歴および予約スケジュール管理" >}}

---

## 4. エンジニアリング実務における重要ポイントまとめ

1. **メニュー遷移の手間を大幅削減**:  
   従来のダークUIと比較して、左側のテキストベース多階層ツリーが常時展開されているため、特定ボリュームの確認中であっても `System ➔ Drives` や `System ➔ Ports` へ即座に切り替え可能となり、トラブルシューティング時の初動速度が向上しました。

2. **ポートおよびスイッチの可視化強化**:  
   SANファブリックの物理リンク障害を分析する際、深い階層のタブを探し回ることなく、左側ナビゲーションから `Ports` と `Switches` の状態をダイレクトに確認できます。

3. **セキュリティポリシーの一元管理**:  
   `Settings ➔ System` にて Volume Retention と Ransomware Detection ポリシーが一元化され、ストレージレベルでのデータ改ざん防止・保護設定が容易に設定・検証できるようになりました。

---

## 5. まとめおよび技術サポートのご案内

HPE Alletra Storage MP B10K の **ArcusOS 10.6.0** は、直感的なホワイトUIへの刷新により、オンプレミスのWebコンソール環境においてもGreenLakeクラウドと同等の優れた操作性と視認性を提供します。

> ⚠️ **テクニカルサポートに関するご案内**:  
> Alletra Storage MP は企業の最重要ミッションクリティカルデータを担うコアインフラです。ボリューム設計、ホストマッピング、リモートレプリケーション構成、または新規ファームウェアの適用において設定に不安がある場合や構成検証が必要な場合は、**HPE Pointnext 公式サポート窓口または認定パートナー企業の専任エンジニア**へご相談されることを強く推奨いたします。
