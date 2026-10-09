---
title: "[HPE VME] Veeamリストア失敗の解決：VirtIOディスク番号重複(index=0)の修正手順"
description: "HPE VME環境のマルチディスクVMをVeeamでリストアする際に発生する「VirtIOディスクインデックス重複(index=0)」エラーの原因と、Cloud Advanced Optionsを用いた解決手順を解説します。"
date: 2026-10-09T16:10:00+09:00
draft: false
tags: ["HPE", "VME", "SimpliVity", "Veeam", "KVM", "VirtIO", "トラブルシューティング", "ディスクエラー"]
categories:
  - SimpliVityVME
---

KVMベースの **HPE VM Essentials（VME）および HPE SimpliVity HCI** 基盤の構築後、エンタープライズバックアップ標準である **Veeam Backup & Replication** を導入し、イメージバックアップおよびリストア検証（DR訓練）を実施することは、システム運用継続性において極めて重要な実務工程です。

しかし、VME上で既存バックアップから復旧した仮想マシン（VM）や特定テンプレートから展開されたインスタンスのうち、**2つ以上の仮想ディスク（OSディスクおよびデータディスク）を構成するVM**において、Veeamリストア実行時にディスクマッピングの不一致エラーで処理が中断する事象に遭遇することがあります。

本稿では、実務現場で発生した **VeeamのVirtIOディスク照合失敗エラーの原因を分析し、VME Managerの隠れた詳細設定（Cloud Advanced Options）を有効化してVirtIO番号を再割り当てする解決手順** を共有します。

---

## 1. 障害事象とVeeamリストアログのエラー内容

2台の仮想ディスクを持つVMに対してVeeamイメージバックアップを取得後、復旧検証のためにFull VM Restoreを実行したところ、以下のAPIエラーが出力されてリストアが停止しました。

![Veeam Restore Session ディスクマッチングエラーログ](images/veeam_restore_error_log.png)

```text
In the Morpheus external infrastructure API response,
the affected VM (vm-name) has two Virtio disks (id 11 and id 15)
but both disks are reported with the same index value (index=0)
and no diskLabel is provided.

This makes disk identification ambiguous and may cause the restore
workflow to fail disk-to-disk matching, resulting in the error:
“Failed to match the original disk 15 to any disk of the restored VM.”
```

### エラーログの要点分析
1. **Morpheus基盤のVME APIレスポンス**: HPE VME Managerは内部的に実績あるクラウドオーケストレーション基盤（Morpheus）をベースとしてインフラAPIを処理しています。
2. **ディスクインデックスの重複報告（`index=0`）**: 対象VMに接続された2つのVirtIOディスク（`id 11`、`id 15`）が、いずれもインデックス番号 `0` として応答し、個別のディスクラベルが欠落していました。
3. **ディスク識別の競合による復旧遮断**: 元バックアップとリストア先VMの間でディスクを1対1で正確にマッピングする必要があるVeeamにとって、双方が同一のバスアドレスに見えるため `disk 15` を照合できず処理を中断したものです。

---

## 2. VME Manager上の設定確認：同一マウントポイント（VIRTIO 0:0）

原因を究明するため、HPE VME Managerコンソールで対象VMのストレージ詳細設定を確認しました。

![VME Managerでのディスクマウントポイント重複確認画面](images/vme_disk_duplicate_virtio_index.png)
*(写真：2つのディスク vda と vdb の MOUNT POINT がいずれも VIRTIO 0:0 で重複している状態)*

- **Disk 0 (`vda`)**: Storage Controller `VirtIO Block` / Mount Point **`VIRTIO 0:0`**
- **Disk 2 (`vdb`)**: Storage Controller `VirtIO Block` / Mount Point **`VIRTIO 0:0`**

ログの指摘通り、ゲストOS内部では別個のブロックデバイス（`vda`、`vdb`）として認識されていても、**ハイパーバイザー側のコントローラ設定値が双方とも最初のインデックスである `VIRTIO 0:0` に固定**されていました。

OS単体は起動できても、ハイパーバイザーAPIを経由して厳密にディスク構成を検証する外部バックアップ・リストア製品にとっては致命的な設定不整合となります。

---

## 3. 解決策：非表示のVirtIOコントローラ設定を有効化する

VMの再構成メニュー（`Reconfigure Server`）から修正を試みても、デフォルト設定のままではディスクマウントポイントを変更するドロップダウンが表示されないか、グレーアウトされて編集できません。

VirtIOバス番号を編集可能にするには、**親階層であるクラウド（Cloud）設定側でディスクおよびストレージ種別の選択ポリシーを事前に許可**しておく必要があります。

### Step 1. Cloud詳細オプション（Advanced Options）の有効化

HPE VMEコンソールのインフラメニューへ移動します：
`Infrastructure > Clouds > 対象の SimpliVity Cloud を選択 > 編集（Edit）`

ページ下部にある **Advanced Options（詳細オプション）** を展開し、以下の3つのチェックボックスを有効にして保存します。

![HPE VME Cloud Advanced Options有効化画面](images/vme_cloud_advanced_options_enable.png)
*(写真：Cloud編集メニューで ENABLE DISK TYPE SELECTION および STORAGE TYPE SELECTION を有効化)*

- **`[x] ENABLE DISK TYPE SELECTION`**（ディスク種別選択の有効化）
- **`[x] ENABLE STORAGE TYPE SELECTION`**（ストレージコントローラ種別選択の有効化）
- **`[x] ENABLE NETWORK INTERFACE TYPE SELECTION`**（ネットワークインターフェース種別選択の有効化）

---

## 4. VMの再構成：VirtIO番号の再割り当て（VIRTIO 0:1）

クラウド側のポリシー保存後、問題のVM管理画面へ戻りサーバーの再構成を実行します：
`対象VM選択 > Actions > Reconfigure Server`

これで **VOLUMES** 項目において、ディスクごとのコントローラおよびマウントポイントのドロップダウンがアクティブになります。

![Reconfigure Server画面でのVirtIOマウントポイント変更](images/vme_reconfigure_server_virtio_remap.png)
*(写真：Reconfigure Serverポップアップで2台目ディスクのマウントポイントを VIRTIO 0:1 等の一意なインデックスへ変更)*

1. **Rootディスク (`vda`)**: 既存の **`VIRTIO 0:0`** を維持
2. **追加ディスク (`vdb`)**: ドロップダウンを開き、重複しない **`VIRTIO 0:1`**（または空いている一意のインデックス）を選択
3. 右下の **`Reconfigure`** ボタンをクリックして設定を反映

適用後、ストレージ一覧を確認すると `vda` は `VIRTIO 0:0`、`vdb` は `VIRTIO 0:1` としてそれぞれ一意なバスアドレスが正常に割り当てられます。

---

## 5. 結果確認とVeeamリストアの成功

VirtIO番号の重複を解消した後、Veeam環境で検証を実施しました：

1. **新規Veeamバックアップの実行**: VME APIから2つのディスクがそれぞれ `index=0` と `index=1` として明確に区別されて収集されます。
2. **Full VM Restoreの再実行**:
   - Veeamのディスクマッチング処理が正常に1対1で完了します。
   - 以前発生していた `Failed to match the original disk 15 to any disk of the restored VM` エラーが完全に解消され、正常にリストアが完了しました。

---

## 6. 実務における要点と推奨点検事項

1. **マルチディスク構成時はディスクバスの重複を必ず点検**:  
   KVM/QEMU環境において、テンプレートや簡易プロビジョニングから作成した複数ディスクVMでは、稀にコントローラアドレス（`0:0`）が重複して割り当てられるケースがあります。OS単体では動作しても、API経由のバックアップ・DR連携時に障害要因となるため事前確認が必須です。
2. **詳細オプションが見当たらない時は親Cloud設定を確認**:  
   HPE VMEは直感的な運用を重視して詳細設定を初期状態で隠蔽しています。ディスクやNICのバス設定が変更できない場合は、上位の **`Cloud > Advanced Options`** を確認することが定石です。
