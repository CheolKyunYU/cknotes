---
title: "[HPE VME 9.1.0] 新規HVM OS 26.04インストール＆hvmcli TUIによるVME Manager展開ガイド (Step 1)"
description: "HPE VME 9.1.0におけるHVM OS 26.04の導入手順、Cluster Layout 2.0対応、Linux Bridge(vs0)ネットワークスタック刷新、および新規hvmcli TUIを用いたVME Manager初期展開の実務ガイドです。"
date: 2026-11-27T22:15:00+09:00
draft: false
categories:
  - SimpliVityVME
tags:
  - HPE
  - VME
  - HVM
  - hvmcli
  - Morpheus
  - KVM
---

HPEの次世代エンタープライズプライベートクラウド仮想化基盤である **HPE VM Essentials (VME) 9.1.0** が正式リリースされました。今回の9.1.0アップデートは単なるマイナー修正にとどまらず、ハイパーバイザーのベースOS、仮想ネットワークアーキテクチャ、展開管理ツールキットに至るまで全面的に刷新された大規模なメジャーリリースとなります。

エンジニアリングの観点からVME 9.1.0で注目すべきアーキテクチャ変更点を整理し、ベースOSである **HVM OS 26.04のインストール** から新設された **`hvmcli tui` コンソール** による **VME Manager(Morpheus)展開の第1段階セットアップ** までの全手順を詳細に解説します。

---

### 1. HPE VME 9.1.0 における主要アーキテクチャ変更点

従来のVMEやSimpliVity HVM環境を運用してきたエンジニアにとって、今回の9.1.0リリースで把握すべき重要ポイントは以下の3点です。

| 項目 | 従来方式 (HVM 24.04 ベース) | 新規方式 (HVM 26.04 / VME 9.1.0) | 実務における技術的意義 |
| :--- | :--- | :--- | :--- |
| **クラスターレイアウト** | **Layout 1.3** | **Layout 2.0 (26.04 専用)** | 9.1.0を24.04上で稼働させた場合は1.3にとどまります。**HVM 26.04で新規インストールして初めて次世代Layout 2.0が有効化可能** |
| **仮想ネットワークスタック** | **OVS (Open vSwitch)** | **`vs` (Linux Bridge ベース)** | 複雑なフロー制御オーバーヘッドを伴うOVSを脱却し、軽量かつ標準的なLinux Bridge(`vs0`)体系へ全面移行 |
| **VME Manager 展開ツール** | **`hpe-vm` コマンド** | **`hvmcli tui`** または **Windows ウィザード** | 従来の`hpe-vm`コマンドは完全廃止。ターミナルTUI環境の **`hvmcli tui`** とGUI専用 **Windows展開ウィザード** の2方式を提供 |

#### なぜHVM OS 26.04による新規インストールが必要なのか？
HPE VME 9.1.0ソフトウェア自体は既存のHVM OS 24.04上でも動作可能です。しかし24.04ベース上に導入した場合、クラスターレイアウト構造は旧来の **Layout 1.3** に制限されます。VME 9.1.0の最新ストレージオーケストレーション機能やネイティブ最適化性能を最大限に引き出すためには、必ず **HVM OS 26.04インストールメディアを使用してCluster Layout 2.0** で基盤を構築する必要があります。

#### OVSからLinux Bridge(`vs`)への移行
従来バージョンでは仮想マシン間の通信分離にOpen vSwitch(OVS)を採用していました。しかし9.1.0(HVM 26.04)以降は管理の複雑性を軽減し、システム負荷を抑えるため、**標準Linux Bridgeベースの仮想スイッチ(`vs0`)方式** に改められました。

#### `hpe-vm`の廃止と`hvmcli tui`の導入
従来のVME Manager展開で使用されていたコンソールコマンド`hpe-vm`は完全に削除されました。代わってホストシェルから **`hvmcli tui`** を実行することで、アスキーアートで構成された直感的な **HPE VM Console v1.0.0** が起動します。Windows用の展開ウィザードも提供されていますが、データセンターの閉域網やリモートコンソール作業では、ローカルISOファイルを直接マウントして迅速に展開できる`hvmcli tui`が最も安定して信頼性の高い方法となります。

---

### Step 1. HVM OS 26.04 GRUBブートおよび言語選択

サーバーコンソール(iLO仮想メディア等)に `HVM_OS_26.04` インストールISOをマウントし、UEFIモードで起動します。

ブート直後にGNU GRUBメニューが表示され、最上段に **`HVM Install 26.04`** が表示されます。Enterキーを押してインストーラーへ進みます。

{{< figure src="images/01_grub_hvm_install_26_04.jpg" caption="HVM OS 26.04 GNU GRUB ブートメニュー画面" >}}

ブートカーネルの読み込み後、Ubuntu Subiquityベースの **HPE HVM Installer** 画面が立ち上がります。システム言語として **`[ English (US) ]`** を選択し、次へ進みます。

{{< figure src="images/02_installer_language_select.jpg" caption="HPE HVM Installer 言語選択画面" >}}

---

### Step 2. 管理ネットワークボンディング(bond0)および固定IP設定

エンタープライズハイパーバイザーホストとして物理ネットワークの冗長性(HA)を確保するため、ボンディング(Bonding)インターフェースを構成します。

ネットワーク画面で **`[ Create Bond ]`** を選択し、パラメータを入力します。

* **Name**: `bond0`
* **Devices**: 使用する物理インターフェースを選択 (例: `[X] ens33`)
* **Bond mode**: **`active-backup`** (またはスイッチ構成に応じて `802.3ad` LACP を選択)
* **XMIT hash policy**: `layer2`
* **LACP rate**: `slow`

{{< figure src="images/03_network_create_bond0.jpg" caption="管理ネットワークボンディング(bond0)インターフェース作成画面" >}}

作成した `bond0` をプライマリ通信カードに指定し、ホスト管理用の固定IPアドレス、ゲートウェイ、ホスト名を入力します。

* **IP Address/CIDR**: ホスト管理IPを入力 (例: `192.168.126.200/24`)
* **Gateway**: デフォルトゲートウェイ (例: `192.168.126.1`)
* **Hostname**: ホスト名 (例: `hvm01`)
* **DNS Servers**: DNSネームサーバー (例: `8.8.8.8`)

{{< figure src="images/04_network_ip_and_hostname_config.jpg" caption="ホスト固定IP、ゲートウェイおよびホスト名設定画面" >}}

---

### Step 3. SubiquityストレージパーティショニングとBaseOSインストール完了

ネットワーク設定完了後、ディスクパーティショニング設定へ進みます。OSを導入するローカルディスクを指定すると、データ消去の確認警告ウィンドウが表示されます。

**`[ Continue ]`** を選択し、パーティション初期化およびOSパッケージのインストールを開始します。

{{< figure src="images/05_confirm_destructive_action.jpg" caption="ディスク初期化およびインストール実行の確認警告画面 (Confirm destructive action)" >}}

バックグラウンドでCurtinインストールエンジンが動作し、パーティションフォーマット、ブートローダー(`grub-efi-amd64-signed`, `shim-signed`)、カーネルパッケージ、NVMe over TCPおよびiSCSIドライバーの設定が自動実行されます。

インストール完了後、画面下部の **`[ Reboot Now ]`** を選択してシステムを再起動します。

{{< figure src="images/06_curtin_os_installation_progress.jpg" caption="CurtinエンジンによるOSパッケージおよびストレージドライバー導入画面" >}}

初期セットアップ画面において、ホストシステムの管理者アカウントを登録します。

* **Enter Administrator Username**: `hvmadmin`
* **Enter Administrator Password**: 管理者パスワードを入力
* **Confirm Administrator Password**: パスワード再確認

{{< figure src="images/07_admin_account_creation.jpg" caption="ホスト管理者(hvmadmin)アカウント登録画面" >}}

---

### Step 4. 新規 hvmcli TUI (HPE VM Console) の起動

HVM OS 26.04のインストールが完了して再起動したら、コンソールまたはSSH経由で `hvmadmin` としてログインし、root権限に昇格します。

VME Managerソフトウェア展開用ISO(`HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso`)の配置を確認し、新規管理コマンドを実行します。

```bash
root@hvm01:/home/hvmadmin# ls
HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso

root@hvm01:/home/hvmadmin# hvmcli tui
```

{{< figure src="images/08_terminal_morpheus_iso_and_hvmcli_tui.jpg" caption="Morpheus ISOの配置確認とhvmcli tuiコマンドの実行" >}}

コマンドを実行すると、ターミナル上にアスキーアートのHPEロゴが表示された **HPE VM Console (v1.0.0)** TUI画面が立ち上がります。

メニューはホスト基盤管理とVM展開機能に整理されています:
* **`<Host Information>`**: ハードウェア仕様、CPU、メモリリソース状態
* **`<Virtual Switch>`**: Linux Bridge仮想スイッチ(`vs0`等)のポート状態確認
* **`<Virtual Machines>`**: 稼働中のKVM仮想マシン一覧および電源管理
* **`<Keyboard Layout / TimeZone>`**: キーボードおよびタイムゾーン設定
* **`<Install VME Manager>`**: **VME Manager VM自動展開ウィザード (中核機能)**
* **`<Install VME Worker>`**: 分散処理用VME Workerノード展開
* **`<Open Shell>`**: 一時的にBashシェルへ復帰

{{< figure src="images/09_hpe_vm_console_tui_main_menu.jpg" caption="新設されたHPE VM Console (v1.0.0) TUIメインメニュー画面" >}}

---

### Step 5. TUIによるVME Manager VM設定と自動展開パイプライン

メインメニューから **`<Install VME Manager>`** を選択し、展開パラメータ設定ウィンドウへ進みます。

**VME Manager Installation Options** 画面で管理VMのネットワーク構成およびハイパーバイザー割り当てを定義します:

* **VM Config Options**:
  * **IP Address**: VME Manager Webコンソール用IP (例: `192.168.55.201`)
  * **Netmask**: サブネットマスク (例: `255.255.255.0`)
  * **Gateway**: デフォルトゲートウェイ (例: `192.168.55.1`)
  * **DNS Server**: ネームサーバー (例: `8.8.8.8`)
  * **Appliance URL**: `https://192.168.55.201`
  * **Hostname**: `vme`
  * **Admin User**: 管理者アカウント名 (デフォルト: `vmeadmin`)
  * **Admin Password**: 管理者パスワード設定
  * **Image URI**: Morpheus展開用ISOのローカルパス (例: `file:///mnt/HPE_Morpheus_Manager_SW_image_v9.1.0-1_S5Q83-11089.iso`)
  * **Select VM Size**: 仮想マシンリソースプロファイル (テスト環境: `Small`)
* **Host Config Options**:
  * **Virtual Switch**: 仮想スイッチブリッジ指定 (デフォルト: **`vs0`**)
  * **VLAN ID**: VLANタギングが必要な場合に指定

設定内容を確認後、下部の **`<Install>`** ボタンを選択します。

{{< figure src="images/10_vme_manager_installation_options.jpg" caption="VME Manager VMのネットワーク、サイジングおよび仮想スイッチ(vs0)設定画面" >}}

展開処理が開始されると、**VME Manager Deployment** プロセスが全13段階の自動化パイプラインに従って順次進行します:
1. `[✓] Validating deployment parameters` (パラメータ妥当性検証)
2. `[✓] Running pre-flight checks` (ホスト要件事前点検)
3. `[✓] Preparing KVM hypervisor` (KVMハイパーバイザー準備)
4. `[✓] Configuring management network` (管理ネットワークブリッジ構成)
5. `[✓] Configuring compute network` (コンピュートネットワーク構成)
6. `[✓] Creating local storage pool` (ローカルストレージプール生成)
7. `[⌛] Downloading QCOW2 image` (ISOからのQCOW2抽出および展開)
8. `[ ] Building cloud-init configuration` (Cloud-init設定構築)
9. `[ ] Defining virtual machine` (libvirt VM定義)
10. `[ ] Starting virtual machine` (仮想マシン起動)
11. `[ ] Waiting for VME services to start` (Morpheusサービス待機)
12. `[ ] Running post-deploy health checks` (展開後健全性検証)
13. `[ ] Cleaning up temporary files` (一時ファイルクリーンアップ)

{{< figure src="images/11_vme_manager_deployment_start_7pct.jpg" caption="VME Manager自動展開シーケンス開始画面 (Progress 7% - KVM事前検証)" >}}

ローカルストレージプールの作成が完了し、QCOW2ディスクイメージを変換・展開する40%の段階まで正常に遷移することを確認できます。

{{< figure src="images/12_vme_manager_deployment_download_qcow2_40pct.jpg" caption="ローカルストレージプール生成後のQCOW2イメージ展開処理画面 (Progress 40%)" >}}

---

### 💡 エンジニア現場Note: VME 9.1.0展開手法と検証ラボの留意点

#### 1. 展開方式の比較: `hvmcli tui` vs `Windows展開ウィザード`
HPE VME 9.1.0にはGUIベースの **Windows展開ウィザード** も同梱されています。
* WindowsウィザードはGUI操作に慣れた管理者に適していますが、作業端末からホストへ数ギガバイトに及ぶOSイメージをネットワーク経由で転送する必要があるため、帯域負荷やセッション切断による失敗リスクが伴います。
* 一方 **`hvmcli tui`** はHVMホスト上のローカルストレージ(`file:///...`)を直接参照するため、ネットワーク転送ボトルネックがなく迅速かつ確実に展開できます。セキュリティの厳しいデータセンター閉域網では`hvmcli tui`の利用が推奨されます。

#### 2. VMware Workstation等のネスト仮想化(Nested Lab)における制限事項
検証目的でVMware Workstation上にHVM OSを構築しVME Managerを展開する場合、以下の点に留意が必要です:
* **ネスト仮想化の有効化**: VMware Workstationの仮想マシン設定でプロセッサの `Virtualize Intel VT-x/EPT or AMD-V/RVI` が有効化されていないと、Step 5の `/dev/kvm present` チェックを通過できません。
* **リソース要求仕様**: VME Managerは内部でMorpheusオーケストレーションエンジンや監視DBが稼働するため、最低4〜8 vCPUおよび16GB〜32GB以上のメモリ割り当てが必須です。
* Workstation環境では仮想ディスクI/O負荷やメモリ競合により、QCOW2イメージ展開からCloud-init初期起動(40%〜60%付近)でタイムアウトが発生する場合があります。本番環境の構築にはHPE ProLiant Gen10 PlusまたはGen11の物理ベアメタルサーバーを採用することが鉄則です。

---

### 次のステップ
ベースOSであるHVM 26.04の導入と、`hvmcli tui` を用いたVME Manager VMの展開パラメータ設定が完了しました。

次回の **Step 2** では、正常起動したVME ManagerのWeb管理コンソール(`https://<VME_IP>`)へ初回ログインし、ライセンス適用、ストレージプール構築、およびホストノードのクラスター登録手順を詳細に解説します。
