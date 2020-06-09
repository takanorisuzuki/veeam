.. _veeam:

---------------------------------------------
Veeam バックアップ&レプリケーション
---------------------------------------------

*このラボの所要時間は60分です。*

概要
++++++++

**この演習では、Veeam Backup Serverに接続し、AHVのバックアッププロキシを展開して構成し、Nutanixクラスターと既存のVeeam Backup＆Replicationインフラストラクチャへの接続を構成し、バックアップおよび復元操作を実行します。**


Nutanix Mineに関するノート
+++++++++++++++++++++++++

.. figure:: images/mineveeam.png

このラボでは、AHVクラスター用のVeeamおよびバックアップジョブの構成に焦点を当てていますが、同じコンポーネントとプリンシパルの多くがVeeamを搭載したNutanix Mineに適用されることは注目に値します。 MINE with Veeamについての重要なポイント:

- Nutanix Mineは、事前検証済みのNX-ハードウェアで実行される専用のセカンダリストレージバックアップアプライアンスです（HXおよびDXサポートは近日提供予定）。

  - XSmall
  - Small
  - Medium
  - Scale-out with the option to leverage Nutanix Objects

- 一般的なFoundationは、AHVクラスターをイメージ化および作成するために実行されます
- "Foundation for Mine"はMINE専用のVMイメージがアップロードされ、AOSとAHVがMineアプライアンスでイメージ化された後に実行されます
- Foundation for Mine VMは、Standard Foundationと同様のWebインターフェイスを介して自動化します:
  - Nutanixコンテナの作成
  - Nutanix Volume Groupの作成
  - MineクラスターへのVeeamバックアップコンポーネントのデプロイ
    - Veeam VBR
    - Nutanix Volume GroupsがアタッチされたVeeam Repository VM
    - Veeam Windows プロキシ
  - Nutanix Mine用のカスタムPrism Elementダッシュボードの導入

.. figure:: images/mine_dashboard.png

今回はNeetanix Mine with Veeamをラボクラスターにデプロイすることはできませんが、詳細、設定手順はこちらにあります。 `<https://ntnx.tips/mine>`_


Veeam Backup Serverのデプロイ
+++++++++++++++++++++++++++++

.. note:: デフォルトでは、Veeam Backup ServerはWindows SQL Server Expressデータベースインスタンスをデプロイします。 本番Veeamデプロイメントでは、外部の高可用性データベースを使用します。

Veeamバックアップサーバーは、Veeamバックアップインフラストラクチャの主要な管理コンポーネントです。 Veeam Backup Serverは、バックアップのターゲットとして使用されるVeeamバックアップリポジトリの管理を担当します。 Veeam Backup＆Replication Consoleは、個々のファイル、ADオブジェクト、Exchangeメールボックス、SQL / Oracleデータベースの復元など、VMの詳細な復元操作にも使用されます。

#. **Prism > VM > Table** を選択し, **+ Create VM** をクリックしてください。

#. 下記のフィールドを入力し **Save** をクリックしてください( *Initials* にはご自身の設定したものと判断できる文字を使ってください):

   - **Name** - *Initials*\ -VeeamServer
   - **Description** - Veeam Backup & Replication 10
   - **vCPU(s)** - 4
   - **Number of Cores per vCPU** - 1
   - **Memory** - 4 GiB
   - Select :fa:`pencil` beside **CD-ROM**

     - **Operation** - Clone from Image Service
     - **Image** - VBR_10.0.0.4442.iso
     - Select **Update**
   - Select **+ Add New Disk**

     - **Operation** - Clone from Image Service
     - **Image** - Windows2016.qcow2
     - Select **Add**
   - Select **+ Add New Disk**

     - **Operation** - Allocate on Storage Container
     - **Storage Container** - Default
     - **Size (GiB)** - 250
     - Select **Add**
   - Select **Add New NIC**

     - **VLAN Name** - Secondary
     - Select **Add**
   - Select **Custom Script**
   - Select **Type or Paste Script**

   .. literalinclude:: VeeamServer-unattend.xml
      :caption: VeeamServer Unattend.xml Custom Script
      :language: xml

   .. note::

    The Unattend script will disable the Windows Firewall.

#. *Initials*\ **-VeeamServer** VMを選択し **Power on** をクリックします。

#. VMが正常に起動したらRDPで接続するか **Launch Console** をクリックします。

   .. note::

     ラボガイドからVMにテキストをコピーして貼り付けることができるように、Microsoft RDP経由でVMにアクセスすることをお勧めします。 Sysprepプロセスは、RDP経由でVMにアクセスできるようになるまでに約2分かかります。

     - **Username** - Administrator
     - **Password** - nutanix/4u

#. **PowerShell** を開き、以下のコマンドを入力します:

   .. code-block:: Powershell
     :emphasize-lines: 1

     Get-Disk | Where partitionstyle -eq 'raw' | Initialize-Disk -PartitionStyle MBR -PassThru | New-Partition -AssignDriveLetter -UseMaximumSize | Format-Volume -FileSystem NTFS -NewFileSystemLabel "Backups" -Confirm:$false

   .. note:: Windows Explorerがディスクのフォーマットを要求する場合があります-上記のPowershellスクリプトレットがディスクをフォーマットするため、このプロンプトをキャンセルできます

#. Veeamサーバーで、スタートメニューを右クリックして、[**System**]を選択します。 [**Hostname, domain, and workgroup settings**]セクションで、[**Change settings**]をクリックし、[**Change**]をクリックして、Windows内のサーバーの名前をVM名と一致するように変更します (*Initials* \ **-VeeamServer ** ) プロンプトが表示されたら、サーバーを再起動します。

   .. figure:: images/0aa.png

#. マウントされた.isoイメージから **Veeam Backup and Replication 10** セットアップを開きます（ディスク上の実行可能ファイルSetup.exeを開く必要がある場合があります）。 [**Install**]をクリックします。

   .. figure:: images/0a.png

   インストーラーはいくつかの前提条件をインストールし、再起動が必要な場合があります。 プロンプトに従って、Veeam Backup and Replication Serverをインストールします。

#. 使用許諾契約に同意し、[**Next**]をクリックします。

#. `こちら<http://10.42.194.11/images/Veeam/VBRv10RTM/Veeam-100instances-suite-nfr.lic>` _にあるVeeam Backup and Replication ServerのNFRライセンスをダウンロードします。ローカルにファイルをダウンロードできます 次に、ファイルをコピーしてRDPセッションに貼り付けます

#. [**Browse**]をクリックして、ダウンロードしたVeeam NFRライセンスファイルを選択します。 [**Next**] > [**Next**]をクリックします。

#. 欠落している必須コンポーネントを要求されたら、[**Install**]をクリックします。 完了したら、[**Next**]をクリックします。

   .. figure:: images/0b.png

#. 設定を確認し、[**Install**]をクリックします。

   .. figure:: images/0c.png

#. インストールが完了するまでの間、Veeam VBRサーバーに必要なDNSエントリを作成します。 AutoADのコンソールを開き、管理者の資格情報を使用してログインします:
     - **Username:** Administrator
     - **Password:** nutanix/4u

#. [**Start**] > [**Windows Administration Tools**] > [**DNS**]に移動して、DNSコンソールを開きます。 [**DC**] > [**Forward Lookup Zones**] > [**ntnxlab.local**] に移動します。

#. DHCPを介して割り当てられたIPアドレスと一致する **xyz-VeeamServer** のAレコードを作成します。 「**Create associated pointer (PTR) record**」をチェックします。

   .. figure:: images/0d.png

   .. note:: 事前に **Reverse Lookup Zones** が存在していないとエラーが表示されます。

#. インストールが完了したら、Veeam Nutanix AHVプラグインをVeeam Backup and Replication Serverにインストールする必要があります。 この `リンク<http://10.42.194.11/images/Veeam/VBRv10RTM/NutanixAHVPlugin_10.0.0.908.exe>` _を使用して、プラグインをxyz-VeeamServerにダウンロードできます。

#. インストーラーを起動し、プロンプトに従ってNutanix AHVプラグインをVeeamサーバーにインストールします。:

   .. figure:: images/0e.png

デフォルトでは、Veeam Backup ServerはWindows SQL Server Expressデータベースインスタンスをデプロイします。 本番Veeamデプロイメントでは、外部の高可用性データベースを使用します。

インストーラーは、バックアップターゲットとして機能するVeeamバックアップリポジトリも作成します。デフォルトでは、バックアップサーバーに公開されている空き容量が最も多いボリュームが選択されます（*Initials*\ **-VeeamServer** に追加された250GBのローカルディスク）。

Nutanix AHV VMのバックアップを保存するために、Veeamは現在、単純なバックアップリポジトリ（Windows互換のファイルまたはブロックストレージ）、スケールアウトバックアップリポジトリ、およびExaGridアプライアンスの使用をサポートしています。 v10のリリースにより、DellEMC Data Domain DD BoostおよびHPE StoreOnce Catalyst独自のストレージプロトコルが、Veeam Availability for Nutanixでサポートされるようになりました。


Veeam Backup Proxy
++++++++++++++++++++++++++++

バックアッププロキシはLinuxベースの仮想アプライアンスであり、NutanixプラットフォームとVeeam Backup＆Replicationの間のコーディネーターの役割を果たします。 Veeamは、NutanixまたはVANバージョン1の仮想アプライアンス用のVeeam Availabilityを使用して、2018年にNutanix AHVのサポートを導入しました。 この仮想アプライアンスを各AHVクラスターに展開して、Veeam経由でバックアップできます。 最初のリリース以降、3つの主要な更新が行われました。最新の更新は2019年11月のUpdate 3リリースで、パフォーマンスのアップグレードとバグ修正が多数含まれています。

また、VANはAHVで実行されるワークロードに基本的なバックアップ機能を提供しましたが、VeeamはAHVの追加の拡張機能を追加し、バックアップとレプリケーションのバージョン10リリースに合わせています。 新しいアプライアンスは「Veeam Backup and Replication AHV Backup Proxy」と呼ばれます（ただし、多くの場合VANv2と呼ばれます）

v10でリリースされた新機能は次のとおりです:

- Veeamバックアップとレプリケーションコンソールの統合
  - VBRコンソールからのAHVクラスター登録
  - Veeam VBRコンソールからのデプロイ
  - AHVバックアッププロキシのライセンス統合管理

- バックアップ機能
  - Nutanix スナップショット連携
  - Linux ファイルレベルリストア (FLR)
  - ファイルを保持または上書きするオプション
  - インスタントVMリカバリ (リカバリされたVMを実行するにはvSphereホストが必要です)
  - VeeamZipのサポート
  - ネイティブの重複排除アプライアンスのサポート
    - DellEMC Data Domain DD Boost
    - HPE StoreOnce Catalyst
  - UIへのマルチユーザーアクセス
  - メールステータス通知
  - VMのドライブの除外
  - Veeam VBRコンソールを介してアクティブフルバックアップをスケジュールする機能

- Veeam ONEの監視とレポート
  - バックアップジョブのパフォーマンスと統計
  - アラームトリガー
  - 保護されたVM一覧

- NutanixおよびVeeam Community Editionのサポート


バックアッププロキシは、Nutanix REST APIを介してAHVプラットフォームと通信し、バックアップおよび復元操作に必要なリソースを割り当て、Nutanixストレージコンテナーとの間でデータを読み書きし、ターゲットVeeamバックアップリポジトリとの間でVMデータを転送します。 バックアッププロキシは、ジョブの管理とスケジューリング、データの圧縮と重複排除、およびバックアップチェーンへの保持ポリシー設定の適用も行います。

バックアップにVeeamを利用する各Nutanixクラスタには、独自のバックアッププロキシVMが必要です。

新しいAHVバックアッププロキシのリリースにより、バックアップする各クラスターでVMを手動で起動する必要がなく、VBRコンソール自体から自動的に展開できます。 これを行うには、VBR VMにログインしてVeeam VBRコンソールを起動します。

AHVバックアッププロキシの展開
------------------------------

#. Nutanixクラスタから、[Settings]> [Local User Management]に移動し、[+New User]を選択します。 「xyzveeam」という名前のローカルユーザーを作成します。ここで、xyzはイニシャルです:
   - User: xyzveeam
   - Password: nutanix/4u
   - First Name: [Your First Name]
   - Last Name: [Your last name]
   - E-mail: xyz-veeam@ntnxlab.local

#. ユーザーに *Cluster Admin* 権限を付与し、[Save]をクリックします

   .. figure:: images/0.png

#. リモートデスクトップまたはVMコンソールを使用して、以前に展開したVeeam VBR VMに接続し、Veeamバックアップおよびレプリケーションコンソールを起動します。

#. 「Backup Infrastructure」に移動します

#. [Managed Servers]で、[Managed Servers]を右クリックし、[Add Server]を選択します

   .. figure:: images/2.png

#. 「Nutanix AHV」をクリックします

#. クラスタのIPアドレスを入力し、[Next]をクリックします。

   .. figure:: images/3.png

#. 資格情報については、[Add...]をクリックします

#. Nutanixクラスター（xyzveeam / nutanix/4u）で前に指定した資格情報を入力します。 [OK]をクリックし[Next]をクリックします。

   .. figure:: images/5.png

   .. note:: VeeamサーバーがPrismに接続すると、セキュリティ警告が表示されます。 [**Continue**]をクリックします。

#. デフォルトのストレージコンテナを選択し、右側の[Choose]ボタンを使用してネットワークを[Secondary]に変更します。 このペインでは静的IPアドレスを指定する必要がないため、[Next]をクリックします。

   .. figure:: images/6.png

#. VBRは、Nutanixクラスターを管理対象サーバーとして追加します。 完了したら、[Next]をクリックします>

   .. figure:: images/7.png

#. 完了をクリックします。 次に、AHVのバックアッププロキシをクラスターに展開する必要があります。 VBRは自動的に展開するように促しますが、プロンプトから[**No**]を選択します

   .. figure:: images/8.png

   .. note:: VBR v10では、VeeamはVBRコンソールからAHVのバックアッププロキシを展開する機能をサポートしますが、現在のリリースでは展開が失敗するため、手動でVeeam Nutanix AHVバックアッププロキシを展開してVBRにインポートします

#. Prismから[**+ Create VM**]をクリックして、新しいVMを作成します。

#. 次のフィールドに入力して、[**Save**]をクリックします:

   - **Name** - *Initials*\ -VeeamAHVProxy
   - **vCPU(s)** - 4
   - **Number of Cores per vCPU** - 1
   - **Memory** - 4 GiB
   - Select **+ Add New Disk**

     - **Operation** - Clone from Image Service
     - **Image** - VeeamAHVProxy2.0.404
     - Select **Add**
   - Select **Add New NIC**

     - **VLAN Name** - Secondary
     - Select **Add**


#. VMの電源を入れます。 VMが起動します。 Cloud-initのエラーが出ますが2分程度待つと起動します。起動が完了したら、Veeam Backup ProxyがDHCPから割り当てられたIPアドレスをメモします。

   .. figure:: images/9.png

#. Veeam VBRサーバーの場合と同様に、AutoAD VMに移動し、DNSコンソールを起動して、[DC] > [Forward Lookup Zones] > [ntnxlab.local]に移動します。

#. Veeam Backup Proxyに割り当てられたIPアドレスを使用してAレコードを作成します:

   .. figure:: images/1.png

#. VMが起動したら、ブラウザーで\ https:// <*VeeamProxy-VM-IP*>：8100 /を開きます。 デフォルトの認証情報を使用してログインします:

   - **Username** - veeam
   - **Password** - veeam

   .. figure:: images/16.png

#. 認証後、インストールするオプションを選択します

   .. figure:: images/installproxy1.png


#. EULAに同意して[Next]をクリックします

#. ユーザー **veeam** の新しい資格情報を指定します:

   - **Login:** veeam
   - **Old password:** veeam
   - **New password:** nutanix/4u
   - **Confirm new password:** nutanix/4u

   .. figure:: images/installproxy2.png

#. VMの作成時に以前に指定したプロキシ名を入力します。 デフォルトのネットワークオプションを選択したままにします

   .. figure:: images/installproxy3.png

#. 概要を確認し、[Finish]をクリックします。 AHVプロキシアプライアンスは設定を適用し、リロードします。

#. Veeam Server Windowsセッション内のVeeam Backup and Replication Consoleに戻ります。 [Backup Infrastructure]をクリックし、[**Backup Proxies**]を右クリックして[**Add Nutanix backup proxy...**]を選択します。

   .. figure:: images/10.png

#. [**Connect proxy**]を選択します

   .. figure:: images/10a.png

#. プロンプトで次のオプションを選択します:

   - **Cluster:** <your cluster>
   - **Name:** *Initials*\ -VeeamAHVProxy

   Click **Next >**

#. デフォルトのネットワークオプションをそのままにして、[**Next >**]をクリックします。

#. [**Add..**]をクリックして、バックアッププロキシの認証情報を追加します:

   - **Username:** veeam
   - **Password:** nutanix/4u

   Click **Next >**

#. デフォルトのアクセス許可のままにします

   .. figure:: images/12.png

   .. note:: VeeamサーバーがPrismに接続すると、セキュリティ警告が表示されます。 [**続行**]をクリックします

#. VBRは、展開したAHVバックアッププロキシを追加します。 [**Next> **]をクリックします

   .. figure:: images/13.png

   .. note:: クラスタを複数人でシェアしている場合、最初の一人以外はエラーが発生します。これ以降の手順ではプロキシを接続できた方のVBR環境を共有で利用してください。接続できた方はインストラクターにIPアドレスと認証情報を共有してください。

#. 概要画面で[**Finish**]をクリックします


VMのバックアップ
+++++++++++++++

Veeam Backup＆Replicationは、VMware vSphereやMicrosoft Hyper-V VMと同様に、Nutanix AHV VMをイメージレベルでバックアップします。 バックアッププロキシは、Nutanix AHVと通信してVMスナップショットをトリガーし、VMをホストしているストレージコンテナーからブロックごとにVMデータを取得し、データを圧縮して重複排除し、Veeam独自の形式でバックアップリポジトリに書き込みます。

AHV VMの場合、Veeam Backup＆ReplicationバックアッププロキシはVMのコンテンツ全体をコピーし、ターゲットの場所に完全バックアップファイル（VBK）を作成します。 フルバックアップファイルは、バックアップチェーンの開始点として機能し、後続のバックアップセッションをフォーマットします。Veeamは、前回のバックアップ以降に変更されたデータブロックのみをコピーし、これらのデータブロックをターゲットの場所の増分バックアップファイルに保存します。 増分バックアップファイルは、完全バックアップファイルと、バックアップチェーン内の先行する増分バックアップファイルに依存しています。 バックアッププロキシは、NutanixのChange Block Tracking（CBT）APIと統合して、VMのデータの変更された部分を特定し、効率的な増分バックアップを可能にします。 AHVバックアッププロキシの新しいバージョンでは、管理者は完全バックアップまたは増分バックアップの両方をスケジュールできるようになりました（以前のバージョンでは、最初の完全バックアップが作成された後、後続のすべてのバックアップは増分バックアップでした）。

#. Prism > VM > Table **で、[**+ Create VM**]をクリックします。

#. 次のフィールドに入力して、[**Save**]をクリックします:

   - **Name** - *Initials*\ -VeeamBackupTest
   - **vCPU(s)** - 2
   - **Number of Cores per vCPU** - 1
   - **Memory** - 4 GiB
   - Select **+ Add New Disk**

     - **Operation** - Clone from Image Service
     - **Image** - Windows2016
     - Select **Add**
   - Select **Add New NIC**

     - **VLAN Name** - Secondary
     - Select **Add**

#. *Initials*\ **-VeeamBackupTest** VMを選択し、[**Power on**]をクリックします。

#. VMが起動したら、[**Launch Console**]をクリックします。 Sysprepプロセスを完了し、ローカル管理者アカウントのパスワードを入力します。

#. ローカル管理者としてログインし、デスクトップに複数のファイル（ドキュメント、画像など）を作成します。

   .. figure:: images/17.png

#. Veeam Backup Proxy Webコンソール（https:// <ip_address>：8100）にログインします。 **Veeam Backup Proxy Webコンソール**で、ツールバーから[**Jobs**]を選択します。

   .. figure:: images/18.png

   .. note:: 複数人でクラスタをシェアしている場合、プロキシ設定が最初に完了した方のVBRコンソールのIPアドレスを入力してください。

#. [**+Add**]をクリックし、バックアップジョブの名前（*Initials*\ -DevVMsなど）を入力し、デフォルトのオプションである[Backup job]のままにして、[**Next**]をクリックします。

   .. figure:: images/19.png

#. [**+Add**]をクリックして、この演習用に作成したVMを検索します。 [Add] > [Next]をクリックします。

   .. figure:: images/20.png

.. note::

  動的モードでは、Nutanix保護ドメイン内のすべてのVMをバックアップできます。 これにより、すでにNutanix PDを利用している場合、バックアップジョブの構成が簡単になります。また、PDに追加されたすべての新しいVMが、ジョブを変更せずにVeeamによってバックアップされるようになります。

[**Default Backup Repository**]を選択し、[**Next**]をクリックします。 これは、*Initials*\ **-VeeamServer** VMに接続されている250GBのディスクですが、環境で使用可能な場合は、サポートされている他のVeeamバックアップリポジトリを選択できます。

.. figure:: images/21.png

次のフィールドに入力して、[**Next**]をクリックします。:

- Select **Run this job automatically**
- Select **Periodically every:**
- Select **1**
- Select **Hour**
- **Restore Points to keep on disk** - 5

.. figure:: images/22.png

[**Run backup job when I click Finish**]を選択し、[**Finish**]をクリックします。

最初の完全バックアップが正常に完了するまでの進行状況を監視します。 最初のバックアップには約2〜5分かかります。 [**Close**]をクリックします。

.. figure:: images/23.png

.. note::

  バックアップジョブを中断せずに[**Close**]をクリックできます。 ジョブの進行状況を再度表示するには、バックアップジョブの[**Status**]の下にある[**Running**]リンクをクリックします。

*Initials*\ **-VeeamBackupTest** VMコンソールに戻り、いくつかの小さな変更（インターネットからの壁紙画像のダウンロード、アプリケーションのインストールなど）を行います。

**Veeam Backup Proxy Webコンソール > Backup Jobs**からジョブを選択し、[**Start**]をクリックして手動で増分バックアップをトリガーし、バックアップチェーンに追加します。

.. figure:: images/24.png

元の完全バックアップと新しい増分バックアップの差分は最小限であるため、2番目のバックアップジョブは1分以内に完了するはずです。 VMのディスクの全容量が処理された（40GB）ことに注意してください。ただし、Change Block Tracking APIにより、実際に読み取られてバックアップリポジトリに転送されたデータはごくわずかです。 これは、ハイパーバイザーレベルのスナップショットを実行するためにVMを「一時停止」する必要がないことも実現しました。

.. note::

  管理者は、ジョブを選択して[**Active Full**]をクリックすることにより、VMの新しいフルバックアップを手動でトリガーすることもできます。 この新しい完全バックアップはバックアップチェーンをリセットし、その後のすべての増分バックアップはそれを開始点として使用します。 以前の完全バックアップは、構成された保存期間に基づいてバックアップチェーンから削除されるまで、リポジトリに残ります。

**Dashboard**に戻って、クラスターの最も重要なバックアップメトリックの概要を確認します。 Veeam Backup＆Recoveryは、大規模な環境全体でバックアップを管理するためのソリューションを提供しますが、AHVバックアッププロキシは、Nutanix管理者がバックアップを制御し、データ保護に影響する可能性のある主要な問題を特定するための合理化されたHTML5 UIを提供します。

.. figure:: images/25.png

VMのリストア
++++++++++++++

バックアッププロキシWebコンソールを使用して、バックアップからNutanix AHVクラスターにVMを復元できます。 Veeam Backup＆Replication v10では、Nutanixクラスター間での復元がサポートされるようになりました。 復元プロセス中に、バックアッププロキシはVeeamバックアップリポジトリのバックアップからVMディスクデータを取得し、元のVMのディスクが配置されていたストレージコンテナーにコピーして、復元されたVMをNutanix AHVクラスターに登録します。

**Veeam Back Proxy Web Console**で、ツールバーから[**Protected VMs**]を選択します。

テストバックアップVM *Initials*\ **-VeeamBackupTest**を選択し、[**Restore**]をクリックします。

**Add**、**Remove**、および **Point** オプションを使用して、目的のVMを特定の時間に選択的に復元できます。 デフォルトでは、VMは最新のバックアップに基づいて復元されます。

[**Next**]をクリックします。

.. figure:: images/26.png

[**Restore to a new location**]を選択し、[**Next**]をクリックして、既存のVMを上書きするのではなく、バックアップデータからVMのクローンを作成します。

*Initials*\ **-VeeamBackupTest**を選択して、[**Name**]をクリックします。 [**Add suffix**]を選択します。 [**Preserve virtual machine ID**]オプションのチェックを外し、[OK] > [Next]をクリックします：

.. figure:: images/27.png

必要に応じて、VMを拡張し、復元されたVMを代替のNutanixストレージコンテナーにリダイレクトできます。 デフォルトでは、VMは元のストレージコンテナーに復元されます。

[**Next**]をクリックします。

必要に応じて、ネットワークを拡張し、復元したVMをクラスター上の代替ネットワークに割り当てることができます。 この演習では、デフォルトネットワークを選択したままにします（セカンダリネットワークにする必要があります）。 [**Next**]をクリックします。

復元操作の理由を指定して、[**Next**]をクリックします。

.. figure:: images/28.png

[**Finish**]をクリックし、正常に完了するまで復元操作を監視します。

.. figure:: images/29.png

.. note::

  最新の復元ポイントが選択されている場合、復元操作は非常に迅速に完了します。 Veeamは、各VMの最新のローリングスナップショットを保持し、バックアップターゲットストレージではなくローカルスナップショットから直接復元できます。

Prismで復元されたVMの電源を入れ、最新の手動バックアップを反映していることを確認します。

**おめでとうございます！** 単一のWebコンソールから、NutanixクラスターのVeeamバックアップ操作を管理および監視することができました。

完全なVMリストアに加えて、**Veeam Backup Proxy Webコンソール**は、クラスター内の任意のVMにマップできる個々の仮想ディスクをリストアすることもできます。 この機能は、データを含む仮想ディスクが破損した場合（たとえば、cryptolocker、ウイルスなど）に役立ちます。

.. figure:: images/30.png

バックアップテストVMディスクをWindows Tools VMに直接復元してみてください！

ファイルレベルリストア
+++++++++++++++++++++++++++

**Veeam Backup Proxy Web Console**はインフラストラクチャ管理者が必要とするすべての基本的なデータ保護機能を提供しますが、**Veeam Backup＆Replication Console*を使用して**Veeam Backup Server**で追加の高度な機能にアクセスできます。

データの復元の一般的な使用例は、誤って変更または削除されたゲスト内の個々のファイルにアクセスすることです。 VM全体をプロビジョニングして単一のファイルにアクセスする必要がなくなるため、必要な時間とリソースを大幅に削減できます。

*Initials*\ **-VeeamServer**コンソール（またはRDPセッション）から**Veeam Backup＆Replication Console**を開きます。

[**Home**]タブで[**Backups**]を展開し、[**Disk**]をクリックします。 個々のファイルを復元するゲストVMディスク（xyz-VeeamBackupTest）を右クリックし、[**Restore guest files**] > [**Microsoft Windows**]を選択します

.. figure:: images/31.png

ファイルを復元するバックアップを選択して、[**Next**]をクリックします。 必要に応じて、復元の理由を入力し、[**Next**]をクリックします

.. figure:: images/31a.png

ファイルレベルのリストアの概要を確認し、[**Finish**]をクリックします

.. figure:: images/31b.png

Veeamは、バックアップに関連付けられたVMディスクを仮想的にマウントし、それらを**Backup Browser**アプリに表示します。

.. note::

  また、「C：\ VeeamFLR」の下にある*Initials*\ **-VeeamServer**でローカルにファイルレベルの復元マウントを探索することもできます。

復元するファイルに移動して選択します。 右クリックして[**Restore**]を選択します。 **Overwrite** または **Keep** のオプションと、別の場所にコピーを作成する **Copy To** のオプションがあることに注意してください
リストアするVMのWindows Firewallが無効化できていない場合はクレデンシャル入力後に接続エラーがでます、手動でリストア対象のVMにログインしてFirewallを無効化してください。

.. figure:: images/31c.png

**Backup Browser** を閉じて、バックアップをアンマウントします。

**Backup Browser** を **Veeam Explorer** アプリケーションと組み合わせて使用して、Microsoft Active Directory、Exchange、SharePoint、SQL Server、およびOracleワークロードのアプリケーション対応のリストアを実行することもできます。

.. _veeam-objects:
(オプション) ターゲットとしてのNutanixオブジェクトの設定
++++++++++++++++++++++++++++++++++++++++++++++++++

Veeamは、ワークロードをS3互換オブジェクトストアにバックアップする機能をサポートしています。 これはNutanixオブジェクトの主要なユースケースであり、Nutanix MINEで大規模なバックアップワークロードに対応する1つの方法です。最初の鉱山セカンダリストレージクラスターと、Veeam内のターゲットとして構成できる別のNutanixオブジェクトクラスターのサイズを決定します。 Veeam内でのオブジェクトの構成はシンプルで簡単であり、従来のiSCSIバックアップターゲットを使用する場合と比較して、オンプレミスオブジェクトを使用してもパフォーマンスがほとんどまたはまったく低下しません

.. note:: 時間を節約するために、Prism Central内でオブジェクトを有効にし、「ntnx-objects」という名前のオブジェクトストアを事前にステージングしました。 そのオブジェクトストア内にバケットを作成します


アクセスキーの作成
-------------------

#. Prism Central > Services > Objectsに移動します

#. 左上のメニューの「Access Keys」をクリックしてください

#. [+Add People]をクリックし、[Add people not in a directory service,]を選択して、 "xyzveeam@ntnxlab.local"という名前を指定します。 [Next]をクリックします。

   .. note:: ローカルユーザーではなく、ここでユーザー認証用のディレクトリサービスを構成できます。

   .. figure:: images/32.png

#. [Download Keys]をクリックして、ユーザー認証キーをローカルマシンにダウンロードします。 次に、「Close」をクリックします。 これらのキーは、後でVeeam内でバケットを構成するときに使用します

   .. figure:: images/33.png


バケットの設定
---------------------

Object StorageはAPIキーを使用してさまざまなバケットへのアクセスを許可するため、上記で作成したAPIキーを使用してバケットを作成します。
バケットは、バージョン管理、WORMなどのポリシーを適用できるオブジェクトストア内のサブリポジトリです。デフォルトでは、新しく作成されたバケットは作成者に対するプライベートリソースです。 バケットの作成者にはデフォルトで読み取り/書き込み権限があり、他のユーザーに権限を付与できます。

#. Object Storeを選択してから、 **Create Bucket** をクリックします

   .. figure:: images/buckets-1.png

#. バケットに *INITIALS*-**veeam-bucket** という名前をつけ **Create** をクリックします

   .. note::

      バケット名は小文字でなければならず、文字、数字、ピリオド、ハイフンのみを含める必要があります。
      さらに、すべてのバケット名は、特定のオブジェクトストア内で一意である必要があります。 既存のバケット名（*your-name* -my-bucketなど）でフォルダーを作成しようとすると、フォルダーの作成は成功しないことに注意してください。
      この方法でバケットを作成すると、資格のあるユーザーにセルフサービスが可能になり、Prism Buckets UIで作成したバケットと同じです。

   .. figure:: images/buckets-2.png

#. 作成したバケットをクリックし、 **Edit User Access** をクリックします

   .. figure:: images/buckets-3.png

   .. figure:: images/buckets-4.png

#. ユーザーを検索し **Read** と **Write** アクセスを付与します

   .. figure:: images/buckets-5.png

VeeamにNutanix Objectsを設定
---------------------------------------

#. Veeam VBRコンソールで **Backup Infrastructure** > **Backup Repositories** をクリックします

   .. figure:: images/36.png

#. Backup Repositoriesを右クリック、 **Add Backup Repository** を選択し、 "Object storage" をクリックします

   .. figure:: images/37.png

#. "S3 Compatible"を選択します。 プロンプトが表示されたら、前に作成したバケットと一致する新しいオブジェクトストレージリポジトリの名前を指定します - *Initials*veeam-bucket, その後 **Next >** をクリックします


#. Accountセクションで、次のように情報を指定します:

   - Service Point: https://<IP of Object Store Client IP>
   - Region: <leave default>
   - Credentials: **Add**をクリックしAccess keyとSecret keyを入力します。これはNutanixオブジェクトでバケットを作成するときにダウンロードしたファイルになります。

   .. note:: **Services** > **Objects**に移動してPrism Centralに接続すると、オブジェクトからサービスポイントアドレスを見つけることができます。 テーブル内には、サービスエンドポイントである「Client Used IPs」があります。

      .. figure:: images/38.png

   .. figure:: images/39.png

    [Next >] をクリックして、証明書のセキュリティ警告を受け入れます

#. 前のセクションで作成したバケットが表示されるはずです。 フォルダの「Browse」をクリックして、「backup」という名前の新しいフォルダを作成します

   .. figure:: images/40.png

#. Finishをクリックします

Nutanixオブジェクトをアーカイブ層として活用するようにバックアップジョブを構成できるようになりました。

VMバックアップがVeeamバックアップリポジトリに格納されると、Veeamはバックアップコピー機能を提供して、同じ場所に同じバックアップデータの複数のインスタンスを作成します。

AHVバックアッププロキシを介して構成されたプライマリバックアップと同様に、バックアップコピーはジョブ主導のプロセスです。 Veeam Backup＆Replicationは、バックアップコピープロセスを完全に自動化し、保存設定を指定して、目的の数の復元ポイントを維持し、アーカイブ目的で完全バックアップを行うことができます。

バックアップコピーにより、バックアップの専門家が推奨する「3-2-1」ルールに従うことが簡単になります:

- **3** - 元の本番データと2つのバックアップの3つ以上のデータのコピーが必要です。

- **2** - データのコピーを保存するには、少なくとも2種類のメディア（ローカルディスクとテープ/クラウドなど）を使用する必要があります。

- **1** - 少なくとも1つのバックアップをオフサイト（クラウドまたはリモートサイト）に保持する必要があります。

まとめ
+++++++++

VeeamとAHVのバックアッププロキシについて知っておくべき重要なこと

- Veeamは広く採用されているバックアップテクノロジーで、Nutanix AHVのネイティブサポートを備えています。

- AHV用のVeeam Backup Proxyは、Nutanix管理者がVeeam Backup＆Replication Consoleにアクセスせずにバックアップと復元操作をすばやく実行できるようにスタンドアロンのHTML5 UIを提供します。

- VeeamはエージェントレスVMバックアップを提供し、APIを介してNutanixスナップショットと直接統合します。

- Veeamには、ファイルレベルの復元、Microsoft Active Directory、Microsoft Exchange、Microsoft SQL Server、Oracleのサポートを含む高度な復元機能があります。
