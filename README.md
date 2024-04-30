# AzureStackHCI_23H2 展開方法　－DELLサーバー編

## 1. Azure Stack HCI の要件を満たすサーバーやNIC、スイッチなどのハードウェアの準備

- 要件を確実に満たすため、専門家に相談しつつ、Azure Stack HCI カタログからハードウェアを選択する
  -  https://azurestackhcisolutions.azure.microsoft.com/#/catalog
- サーバーのスペックはサイジングツールで概要を把握し、ベンダーと細かな要件を詰めながら調整する
  -　 https://azurestackhcisolutions.azure.microsoft.com/#/sizer 
- その他の要件も確認し、準備に利用する
  - 物理ネットワーク要件：https://learn.microsoft.com/ja-jp/azure-stack/hci/concepts/physical-network-requirements?tabs=overview%2C23H2reqs
  - ホストネットワーク要件：https://learn.microsoft.com/ja-jp/azure-stack/hci/concepts/host-network-requirements
  - ファイアウォール要件：https://learn.microsoft.com/ja-jp/azure-stack/hci/concepts/firewall-requirements
- 導入するハードウェアが Azure Stack HCI セキュリティのどこまで対応しているのかを確認しておく
　- Azure Stack HCI 展開時に各種セキュリティ設定の有効・無効を聞かれるため事前に確認
  - https://learn.microsoft.com/ja-jp/azure-stack/hci/concepts/security-features
  - 導入するハードウェアがすべて対応していることが望ましい　 ※特にTPM の有無
- (オプション) 環境チェッカーツールを利用した環境の事前評価
  - 展開中にも行われるため必須ではないが、作業を開始する前に事前チェックも可能
  - https://learn.microsoft.com/ja-jp/azure-stack/hci/manage/use-environment-checker?tabs=connectivity

## Azure Stack HCI ハードウェア環境設定とOSインストール
    Azure Stack HCI ハードウェアセットアップと ToR スイッチ設定　・・・環境に合わせて VLAN なども設定しておく
    Azure ポータルにアクセスし、検索ボックスに[Azure Stack HCI]と入力、[Azure Stack HCI 管理画面]を表示する
    Azure ポータルのAzure Stack HCI管理画面から Azure Stack HCI OS の英語版の ISOイメージをダウンロード　～不要なローカライズの不具合を回避するため～
    IDRACにて各ノードにアクセスし、Azure Stack HCI OS ISO をマウント
    IDRACのLife Controllerにて Windows Server 2022 ドライバーを使って Azure Stack HCI OSをインストール
    　# OSのインストール画面は Windows Server とほぼ同じなので迷うことはないはず
    　# ISO から直接起動して Azure Stack HCI OSをインストールし、DELL サイトからダウンロードした最新の NICドライバーをインストールしてもよい
        　# ダウンロードしたドライバーを含むフォルダーを共有しておき、Azure Stack HCI ノードから net use v: \\コンピュータ名\共有名 などで接続、ドライバーのインストールを行う
          # QLogix、Mellanox はドライバーのセットアップexeを起動すると Azure Stack HCI OS 上でも GUI が表示され、インストールが可能だった
        　# インストール終了後、net use v: /delete などでマップを削除しておく
    IDRACにてISOイメージをアンマウントしておく　・・・マウントしたままだと Azure Stack HCI 展開中のBitLocker暗号化の画面で進まなくなることがわかっている
    
## Azure Stack HCI OSインストール後の設定
    OSインストール後にパスワード設定画面が出てくるので、適切なパスワードを入力、Sconfig の画面が表示される
    # SConfig での設定
        9 [Date and time] にて[Internet Time]タブを開き、time.windows.comと同期できていることを確認　－ 通信できない場合は社内のタイムサーバーと同期する必要あり
        7 [Remote desktop] にてリモートデスクトップを Enabled に変更　～リモートデスクトップだとコピー＆ペーストが可能で、生産性が上がるため　・・・Azure Stack HCI展開後は自動でDisableになる
        8 [Network settings] にて管理用のNICにIPアドレスを設定
            # DHCPからIPをもらっている複数のNICがある場合、番号の小さいIPアドレスを管理用にするとよさそう　[静的IPアドレス][サブネットマスク][デフォルトゲートウェイ]の設定後、[DNSサーバー]を追加設定
        2 [Computer name] にてコンピュータ名を変更し、再起動
    NIC の名前設定や DHCP 無効化などを行う
        # リモートデスクトップ mstsc.exe にて各ノードにリモートアクセス　　管理者名は コンピュータ名￥administrator 　パスワードはインストール後に設定したものを利用
            # 通信がうまく行かない場合は 4 [Remote management] にて ping を有効にして確認するなども検討する
            Get-NetIPAddress -AddressFamily IPv4 | select InterfaceAlias,IPAddress,PrefixOrigin　　・・・現時点でのNICの状態を確認
        #--結果例--
        #InterfaceAlias              IPAddress       PrefixOrigin
        #--------------              ---------       ------------
        #NIC2                        10.29.146.4             Dhcp           ・・・管理＋VM通信用に利用するセカンダリNIC
        #NIC1                        10.29.146.13          Manual           ・・・手動でIPアドレスを設定した管理用NIC　管理＋VM通信用に利用
        #SLOT 3 Port 2               169.254.222.78     WellKnown           ・・・Software Defined Storage 用のRDMA NIC１
        #SLOT 3 Port 1               169.254.123.122    WellKnown           ・・・Software Defined Storage 用のRDMA NIC２
        #Ethernet                    169.254.1.2             Dhcp           ・・・サーバーのUSBとホストをつなぐために利用する Ethernet Remote NDIS Compatible Device ・・・後で無効化しないと悪さする
        #Loopback Pseudo-Interface 1 127.0.0.1          WellKnown           ・・・Get-NetAdapter では出てこないし、今回は気にしなくてよい
        #--------
         
        # Azure Stack HCI の Network ATC (インテントベースのネットワーク管理)を利用するため、環境に合わせて NIC名を変更
        # 今回の環境で使うNIC名が NIC1,NIC2,SLOT 3 Port 1,SLOT 3 Port 2 になっていることが前提で書いてあるため、既存環境のNIC名を使って正しく設定する必要あり
        # まずは手動でIPアドレス設定をした管理用NICの名前変更
        Rename-NetAdapter -Name "NIC1" -NewName "MGMT_VM1"
        # それ以外の3つのNICの名前変更
        Rename-NetAdapter -Name "NIC2" -NewName "MGMT_VM2"
        Rename-NetAdapter -Name "SLOT 3 Port 1" -NewName "Storage1"
        Rename-NetAdapter -Name "SLOT 3 Port 2" -NewName "Storage2"

        # 手動設定していない NIC の DHCPを無効化
        Get-NetAdapter -Name "MGMT_VM2" | Set-NetIPInterface -Dhcp Disabled
        Get-NetAdapter -Name "Storage1" | Set-NetIPInterface -Dhcp Disabled
        Get-NetAdapter -Name "Storage2" | Set-NetIPInterface -Dhcp Disabled

        # NICにOS標準のドライバー(Inbox Driver)が残っていないことを確認する  =各ノードで以下のコマンドを実行し、 DriverProvider にMicrosoftが無いことを確認
        Get-NetAdapter -Name * | Select *Driver*
            # Ethernet Remote NDIS Compatible Device というInbox DriverになっているNIC が存在する可能性があります。そちらについては以下を参照し対処をお願いします

        # Enternet Remote NDIS Compatible DeviceというNICについて
            #https://www.dell.com/support/kbdoc/ja-jp/000130077/poweredge-idrac-%E3%83%80%E3%82%A4%E3%83%AC%E3%82%AF%E3%83%88-%E6%A9%9F%E8%83%BD-%E3%81%AE-%E4%BD%BF%E7%94%A8-%E6%96%B9%E6%B3%95
            #pnputil /enum-devices /ids /class net コマンドにて Network Device の Instance ID を確認し、入手したInstance ID を利用して NDIS Compatible Device を削除
            pnputil /remove-device "USB\VID_413C&PID_A102\5678"
            #プラグ＆プレイデバイスのため再起動後には復活しているので、再起動後は上記コマンドを実行すると確実　・・・いずれは対処方法が見つかるかもしれないので

        # Software Defined Storage用の RDMA NICにはVLAN設定をすることになる。VLAN設定されているかどうかの確認は Get-NetAdapter -Name * | fl にて可能

        # もしIPv6が不要ならば Disable にしておくこともできる ・・・有効のままでも Azure Stack HCI クラスター構築には支障なし
        # Disable-NetAdapterBinding -Name * -ComponentID ms_tcpip6

        # Hyper-V の有効化
        Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All　　　　・・・再起動を行う
        # 再起動後リモートデスクトップで再接続し、以下を実行
        pnputil /remove-device "USB\VID_413C&PID_A102\5678"


## Azure Stack HCI 展開用の Active Directry 事前設定　　　・・・Active Directryに管理者としてアクセスできるマシンであればどこからでも実施可能
    Install-Module AsHciADArtifactsPreCreationTool -Repository PSGallery -Force
    $NewOU = "作成するOU名"   ・・・作成する OU 名は OU=xx,DC=xxx,DC=xxx という形式で入力すること
    # 以下のスクリプト実行時にHCIクラスター展開用のユーザー名とパスワード(12 文字以上で、小文字、大文字、数字、特殊文字を含む)を入力するため、事前にユーザー名とパスワードを決めておく
    # 既存OU名の指定、既存ユーザー名の入力も可能だが、OU内にはホストやクラスターオブジェクトが追加され、入力したユーザーに対してはHCI展開用の権限設定が自動的に行われるため他のユーザー管理とは分けることをおすすめする
    # サーバーボリュームが暗号化されている場合、OUを削除するとBitLocker回復キーも削除されるため、繰り返しになるが専用のOUが望ましい
    New-HciAdObjectsPreCreation -AzureStackLCMUserCredential (Get-Credential) -AsHciOUName $NewOU
    # [Active Directoryユーザーとコンピュータ] ツールにて新しいOUと展開用のユーザーができていることを確認する

## Azure Stack HCI 各ノードを Azure Arc で Azure と接続
    まずは Azure ポータルを英語に変更　　・・・不要なトラブルを回避するため、本手順書作成時は日本語ポータルだと仮想スイッチ名の文字化けなどに遭遇
    Azure Stack HCIに関連するオブジェクトを登録するリソースグループを用意、なければ新規作成
    そのリソースグループに対し、Azure 側の作業をするアカウントに以下の管理権限が付与されていることを確認し、足りなければ追加
    　・Azure Connected Machine Onboarding
    　・Azure Connected Machine Resource Administrator

    サブスクリプションに以下のリソースプロバイダーが登録されていることを確認し、登録されていなければ登録する
        Microsoft.HybridCompute
        Microsoft.GuestConfiguration
        Microsoft.HybridConnectivity
        Microsoft.AzureStackHCI

    # 各Azure Stack HCIノードをAzure Arcに登録するための手順１　モジュールのインストール
        #Register PSGallery as a trusted repo　・・・入力を求められたら Y を入力。。。そして、既に設定されているというエラーが出たら気にしなくてよい
        Register-PSRepository -Default -InstallationPolicy Trusted
        #Install Arc registration script from PSGallery 　・・・入力を求められたら All の A を入力
        Install-Module AzsHCI.ARCinstaller
        #Install required PowerShell modules in your node for registration
        Install-Module Az.Accounts -Force
        Install-Module Az.ConnectedMachine -Force
        Install-Module Az.Resources -Force

    # 各Azure Stack HCIノードをAzure Arcに登録するための手順２　登録に必要な情報の入力と収集
        # AzureサブスクリプションIDを指定　・・・Azureポータルのサブスクリプション管理画面から入手可能
        $Subscription = "利用するサブスクリプションID"
        # リソースグループ名を指定　・・・日本にアクセスポイントがないため、まずはEast USなどで作成しておくとトラブルが減らせる
        $RG = "利用するリソースグループ名"
        # テナントIDを指定　・・・ Azureポータルの Microsoft Entra ID管理画面から入手可能
        $Tenant = "利用するテナントID"
        # デバイスコードを使ってサブスクリプションにログオン　・・・Azure Stack HCI が Server Coreベースでログオンのポップアップが出せないための仕組み
        # 他のマシンのブラウザーで https://login.microsoftonline.com/common/oauth2/deviceauth にアクセスし、Azure Stack HCI画面に表示されたコードを使ってAzureユーザーをログオンさせている
        Connect-AzAccount -SubscriptionId $Subscription -TenantId $Tenant -DeviceCode
        #ログオンしたAzureユーザーのトークンを取得
        $ARMtoken = (Get-AzAccessToken).Token
        #Arcに登録するためのアカウントのIDを取得
        $id = (Get-AzContext).Account.Id

    # 各Azure Stack HCIノードをAzure Arcに登録するための手順３　実際の登録作業
        Invoke-AzStackHciArcInitialization -SubscriptionID $Subscription -ResourceGroup $RG -TenantID $Tenant -Region eastus -Cloud "AzureCloud" -ArmAccessToken $ARMtoken -AccountID $id

        Azure Stack HCI の画面で登録完了したら、Azureポータルの[Azure Arc]-[Machines]にて登録作業を行ったAzure Stack HCIマシン名をクリックし、Azure側の管理画面を表示
        [Extensions] をクリックして、以下の4つの拡張機能が追加され、成功するまで待つ
            # AzureEdgeDeviceManagement
            # AzureEdgeLifecycleManager
            # AzureEdgeRemoteSupport
            # AzureEdgeTelemetryAndDiagnostics
        # すべての拡張機能のインストールが完了すると、Arc への登録も完了  　　・・・しばらく放置しておくと5つ目(MDE.Windows)も増えていることがわかる

        ## もしも Azure Stack HCI クラスター展開がうまく進まなかったり、AzureからAzure Stack HCIクラスターを削除した場合、Azure Arc管理画面から Azure Stack HCIノードも削除される
        ## しかし、Azure Stack HCI 各ノードには既に設定された情報が残っているため、以下のコマンドを使って削除を行う
            #$Subscription = "利用するサブスクリプションID"
            #$RG = "利用するリソースグループ名"
            #$Tenant = "利用するテナントID"
            #Connect-AzAccount -SubscriptionId $Subscription -TenantId $Tenant -DeviceCode
            #$ARMtoken = (Get-AzAccessToken).Token
            #$id = (Get-AzContext).Account.Id
            #Remove-AzStackHciArcInitialization -SubscriptionID $Subscription -ResourceGroup $RG -TenantID $Tenant -Cloud "AzureCloud" -ArmAccessToken $ARMtoken -AccountID $id


## Azure Stack HCI クラスター展開
    #事前設定
    Azure ポータルを英語のままで作業を実施  ・・・不要なトラブルを回避するため、本手順書作成時は日本語ポータルだと追加するロール名で違う表現が使われたりで、わかりにくいなどあり
    サブスクリプションに対し、Azure 側の作業をするアカウントに以下の管理権限が付与されていることを確認し、足りなければ追加
    　・Azure Stack HCI Administrator
    　・Cloud Application Administrator
    　・Reader
    リソースグループに対し、Azure 側の作業をするアカウントに以下の管理権限が付与されていることを確認し、足りなければ追加
    　・Key Vault Data Access Administrator (Key Vault データアクセス管理者)
    　・Key Vault Secrets Officer (キーコンテナーシークレット責任者)
    　・Key Vault Contributor (Key Vault の共同作成者)
    　・Storage Account Contributor (ストレージアカウント共同作成者)

    # クラスター構築作業
    Azureポータルの [Azure Arc]-[Azure Stack HCI] 管理画面にて、[All Clusters (PREVIEW)]を選択
        # PREVIEW が不要な場合は画面内の [Old Experience]をクリックするとGA済みの画面を表示
    1: [+Create]メニューから[Azure Stack HCI Cluster]を選択
    2: Basics タブ
        2-1: 展開に利用するサブスクリプションとリソースグループを選択
            # リソースグループが違うと画面一番下にArcに登録したサーバー一覧が表示されないので、Azure Stack HCI ノードをAzure Arcに登録したリソースグループを選択すること
        2-2: Cluster name を入力
        2-3: Region は East US    ※今後Japanにもサービスが来れば日本を指定可能になる
        2-4: Key vault name では[Create a new key vault]をクリックし、右に出てくる画面で[Create]をクリック        ※必要に応じてKey Vault nameなどを変更する
        2-5: リソースグループ内の Azuer Stack HCI ノードの一覧が表示されるので、クラスターに参加させるノードをチェックし、[Validate selected servers]をクリック
        2-6: Validateが完了したら [Next: Configuration]をクリック
    3: Configuration タブ
        [New configuration] が選択されていることを確認し[Next: Networking]をクリック　　　　　※テンプレートが用意できている場合はテンプレートを利用
    4: Networking タブ
        ※ ここは実際の環境に合わせて設定をする必要がある
        ※ 以下は NICx4 環境にて、管理とVMが使うネットワークに NICx2 (最初にIPアドレスを設定したNICを含む)、ストレージ用に NICx2 (VLAN 設定必須) を利用する想定
        4-1: [Network switch for storage] をクリック
        4-2: [Group management and compute traffic] をクリック
        4-3: インテント名「Compute_Management」に対して [MGMT_VM1]を選択
        4-4: [+ Select another adapter for this traffic]をクリックして[MGMT_VM2]を追加
        4-4: インテント名「Storage」に対して　[Storage1]を選択
        4-5: 必須項目となっている VLAN ID には[環境に合わせたVLAN ID] を入力
            # ノード間の通信で利用するためのもの。
            # スイッチレスやスイッチ側ですべての VLAN ID の通信を許可していればデフォルトのままでOK、スイッチ側で設定がされていればそのVLAN IDを間違わずに入力すること
        4-6: [+ Select another adapter for this traffic]をクリックして[Storage2]追加
        4-7: [環境に合わせた VLAN ID]を入力
        4-8: Azure Stack HCI が利用する最低7つのIP アドレス範囲を用意し、[Starting IP]~[Ending IP] として入力
        4-9: [サブネットマスク　例 255.255.255.0]を入力
        4-10: [デフォルトゲートウェイのIPアドレス]を入力
        4-11: [DNS Server のIPアドレス]を入力
        [Next: Management]をクリック
    5: Management タブ
        5-1: Azure から Azure Stack HCI クラスターに指示を出す際に利用するロケーション名として[任意のCustom location name]を入力
            # 良く使うので、プロジェクト名や場所、フロアなどを使って、わかりやすい名前を付けましょう　　※思い浮かばない時はクラスター名に-cl とつけておくとわかりやすいかも
        5-2: Cloud witness 用に[Create new]をクリック、さらに右に出てきた内容を確認・必要に応じて修正のうえ、[Create]をクリックし、Azure Storage Accountを作成
        5-3: [ドメイン名 例 contoso.com] を入力
        5-4: [OU名 　例 OU=test,DC=contoso,DC=com ] を入力　　　※Active Directoryの準備の際に設定したOU
        5-5: Deployment用の[Username]を入力　　※Active Directoryの準備の際に指定したDeployment用のユーザー名
        5-6: [Password] [Confirm password] を間違えないように入力
        5-7: Local administratorとしての [Username]を入力　　※特別な設定をしていなければ AdministratorでOK
        5-8: [Password] [Confirm password] を間違えないように入力
        5-9: [Next: Security]をクリック
    6: Security タブ
        [Recommended security settings] が選択されていることを確認し [Next: Advanced] をクリック
            # 古いサーバーを使って展開をする場合など、推奨設定の機能を満たせない場合は [Custommized security settings]をクリックして有効にしたい項目のみを選択
    7: Advanced タブ
        [Create workoad volumes and required infrastructure volumes]が選択されていることを確認し[Next: Tags]をクリック
            # 既定で、Software Defined Storage プールに Infrastructure ボリュームと、Azure Stack HCI各ノードをOwnerとする論理ボリュームを自動作成してくれる
    8: Azure 上のオブジェクトを管理しやすくする任意のタグをつけ、[Next: Validation]をクリック
    9: Validation タブ
        9-1: 特に問題が無ければ Resource Creation として6つの処理を行うため全て Succeeded になることを確認
            # サブスクリプションやリソースグループへの管理権限が足りない場合に Key Vault のエラーに遭遇する
        9-2: [Start Validation]をクリック
        9-3: 更に11個のチェックが行われ Validationが完了したら [Next: Preview + Create]をクリック
    10: [Create] をクリックし、Azure Stack HCI Cluster Deployment を開始
            # 画面が Azure Stack HCI 管理画面の ”Settings” にある「Deployments」が選択された状態に遷移するので [Refresh]をクリックして状況を確認
            # 数時間かかるが、OSの更新やドメイン参加を含め Azure Stack HCI 23H2 クラスター作成作業が自動で行われ、終了するとAzure から管理可能な状態になる
            # 途中エラーが出た場合はログを確認するなどして対処し[Rerun deployment] を実施
            # 何度か再起動が行われるため Remote NDIS Device を削除するなどの作業が必要になることもある ※現時点
