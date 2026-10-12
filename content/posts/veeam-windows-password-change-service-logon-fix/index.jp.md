---
title: "[Veeam] Windows サーバーのパスワード変更後に必要なサービスログオン情報の一括更新手順"
description: "セキュリティ診断やポリシー順守のために Windows Administrator のパスワードを変更した後、Veeam サービスのログオン失敗（エラー 1069）および管理コンソール接続不能を解消する実務対処ガイドです。"
date: 2026-10-12T15:52:00+09:00
draft: false
tags: ["Veeam", "Backup", "Windows Server", "セキュリティ", "サービスアカウント", "トラブルシューティング"]
aliases:
  - /jp/posts/veeam-windows-password-change-service-logon-fix/
categories:
  - SimpliVityVME
---

## 1. 背景: Windows パスワード更新直後に発生する Veeam サービス起動停止

企業の IT インフラ運用現場では、セキュリティ診断ガイドラインや定期的な資格情報更新ポリシーに従い、Windows Server のローカル管理者（`Administrator`）パスワードの変更や専用管理アカウントへの移行が実施されます。

しかし、**Veeam Backup & Replication（VBR）**が稼働するバックアップ管理サーバー上で Windows のパスワードを変更した直後、以下のような問題が発生し緊急障害へと発展するケースが多発します。

* **Veeam Backup & Replication Console** 起動時にログオン画面で停止、または接続エラーが発生する。
* Windows イベントログに **Event ID 7000 / 7038（ログオンの失敗によりサービスを開始できませんでした）** が記録される。
* サービス管理画面（`services.msc`）から手動で起動を試みると **エラー 1069（ログオン失敗）** が返される。

```text
[Windows サービスエラーメッセージ]
エラー 1069: ログオンに失敗したため、サービスを開始できませんでした。
(Error 1069: The service did not start due to a logon failure.)
```

この現象は、Veeam のバックアップエンジンやカタログサービス、ブローカーなどの**中核バックグラウンドサービスが、インストール時に指定した管理者アカウント権限で稼働するように登録されているため**に発生します。Windows アカウントのパスワードのみを変更し、サービスプロパティ内の認証情報を更新しない場合、サービスの認証が通らず一斉に停止します。

---

## 2. 事前確認: 影響を受ける主な Veeam サービス一覧

Veeam サーバーには多数の常駐サービスが存在します。パスワード更新が必要となる代表的なサービスは以下の通りです。

| サービス名 (Service Name) | 表示名 (Display Name) | 役割 |
| :--- | :--- | :--- |
| **VeeamBackupSvc** | Veeam Backup Service | バックアップ/リストア処理およびスケジュール制御の中核エンジン |
| **VeeamBrokerSvc** | Veeam Broker Service | インフラ構成情報の通信ブローカーサービス |
| **VeeamCatalogSvc** | Veeam Guest Catalog Service | ゲスト OS ファイルインデックスおよび検索管理 |
| **VeeamCloudSvc** | Veeam Cloud Connect Service | クラウドリポジトリおよびテナント連携サービス |
| **VeeamEnterpriseManagerSvc** | Veeam Backup Enterprise Manager | エンタープライズ統合管理 Web サービス |
| **VeeamRESTSvc** | Veeam Backup Server RESTful API Service | VBR REST API 連携用エンドポイント |

{{< figure src="images/veeam-services-list-properties.png" caption="[図 1] Windows サービス管理画面（services.msc）の Veeam サービス一覧とプロパティメニュー" >}}

---

## 3. 実務対処手順

### Step 1. 起動中の Veeam コンソールを終了
セッションの干渉を防ぐため、開いている Veeam 管理コンソール（`Veeam.Backup.Shell.exe`）をすべて終了します。

### Step 2. サービス管理ツール (services.msc) の起動
Windows のファイル名を指定して実行（`Win + R`）で `services.msc` を入力し、管理者権限で実行します。

### Step 3. Veeam サービスのプロパティを開く
サービス一覧から `Veeam` で始まるサービスを見つけ、右クリックして **プロパティ (Properties)** を選択します。

### Step 4. 「ログオン (Log On)」タブで新パスワードを適用
1. プロパティ画面上部の **ログオン (Log On)** タブを開きます。
2. **アカウント (This account)** 項目に対象アカウント（`.\veeam-admin` または `ドメイン\アカウント名`）が指定されていることを確認します。
3. 変更後の **パスワード (Password)** と **パスワードの確認入力 (Confirm password)** を正しく入力します。
4. **適用 (Apply)** をクリックすると、「アカウントにサービスとしてのログオン権限が付与されました」という旨の通知が表示されます。
5. **OK** をクリックして設定を保存します。

{{< figure src="images/veeam-service-logon-properties.png" caption="[図 2] サービスプロパティのログオンタブにおけるパスワード更新" >}}

> **[実務ポイント: アカウント再指定が必要な場合]**  
> アカウント名を変更した場合などは、`参照 (Browse...)` ボタンからオブジェクトの場所を指定し、`名前の確認 (Check Names)` を実行して確定します。

{{< figure src="images/veeam-select-user-dialog.png" caption="[図 3] ユーザー選択ダイアログにおけるサービス実行アカウントの指定" >}}

### Step 5. 対象となる全 Veeam サービスの更新
上記 Step 3〜4 の手順を、該当ユーザー権限で起動するように設定されているすべての Veeam サービスに対して同様に実施します。

> **注意**: `Local System` や `Network Service` で動作する既定の Windows サービスはパスワード変更の対象外です。

### Step 6. サービスの開始と稼働確認
1. 停止していたサービスを順次右クリックし、**開始 (Start)** を選択します。
2. すべての Veeam サービスの状態（Status）が **実行中 (Running)** に切り替わったことを確認します。

{{< figure src="images/veeam-services-running-status.png" caption="[図 4] パスワード更新完了後の全 Veeam サービス正常稼働（Running）確認" >}}

---

## 4. 最終検証: Veeam コンソール接続とジョブ状態の確認

すべての常駐サービスが起動した後、最終検証を実施します。

1. デスクトップまたはスタートメニューから **Veeam Backup & Replication Console** を起動します。
2. 更新後のパスワードでログオンを実行します。
3. コンソール下部のステータスバーに対象アカウント（`Logged in: .\veeam-admin`）が表示されていることを確認します。
4. バックアップジョブ（Backup Jobs）の一覧およびインフラ情報が正常に読み込まれていることを確認します。

{{< figure src="images/veeam-console-login-verified.png" caption="[図 5] Veeam 管理コンソールへの正常ログオンとバックアップジョブ表示確認" >}}

---

## 5. インフラ運用のベストプラクティス

1. **専用サービスアカウント（gMSA または固定サービスアカウント）の導入**:  
   個人が対話型ログオンに使用する管理者アカウントと、Veeam サービスの実行アカウントを同一にしておくと、定期パスワード変更のたびに全バックアップサービスが停止するリスクが生じます。実務では Veeam 専用のサービスアカウントを用意し、パスワードのライフサイクルを分離管理することが推奨されます。

2. **メンテナンスウィンドウ内での作業徹底**:  
   バックアップやレプリケーションの実行中にサービスアカウントの認証情報が無効化されると、スナップショットの取り残しやリポジトリのロック障害が発生する可能性があります。必ず夜間や休日のジョブ停止時間帯（メンテナンスウィンドウ）を確保して作業を行ってください。
