---
title: "Microsoft Defender for Identity センサー関連のトラブルシューティング"
emoji: "🔍"
type: "tech"
topics:
  - "defender"
  - "identity"
  - "microsoft"
  - "troubleshooting"
published: true
published_at: "2024-11-18 14:24"
publication_name: "microsoft"
---

## はじめに
2024/11 より、Azure Networking のサポート チームから、Security の CSA (Cloud Solution Architect) に鞍替えしました。ロールも担当製品も変わったので、まだ右も左も分からないことが多い状況ではありますが、そんな時期だからこそユーザー目線でトラブルに遭遇する機会も多いので、自分用のメモを兼ねてトラシューの記録を書き残しておこうと思います。

## MDI センサーの大まかな仕組み
Microsoft Defender for Identity (通称 MDI) では、Windows Server (2016 以降) にセンサーをインストールして、ADDS / ADFS / ADCS / Entra Connect から情報 (イベント ログやパケット) をクラウド側へと吸い上げます。センサーのセットアップは、[Defender Portal](https://security.microsoft.com/) の [設定] - [ID] - [全般] - [センサー] から、[センサーの追加] をクリックして、[インストーラーをダウンロード] し、[アクセス キー] を使って行います。

![Defender ポータルで MDI センサーをダウンロード画面のスクリーンショット](/images/defender-for-identity-sensor-troubleshooting/1.png)

センサーのインストールが完了すると、Windows サービスとして以下の二つが追加されていることが確認できます。2 つとも実行中になっていることを確認しましょう。

- Azure Advanced Threat Protection Sensor
- Azure Advanced Threat Protection Sensor Updater

![Azure ATP Sensor のサービス 2 つを含む services.msc のスクリーンショット](/images/defender-for-identity-sensor-troubleshooting/2.png)

ちなみに、インストールやサービスに関するログは以下のファイルに記録されています。

### インストール関連のログ
- %USERPROFILE%\AppData\Local\Temp
- C:\Windows\Temp
  - Azure Advanced Threat Protection Microsoft.Tri.Sensor.Deployment.Deployer_YYYYMMDDHHMMSS.log
  - Azure Advanced Threat Protection Sensor_YYYYMMDDHHMMSS.log
  - Azure Advanced Threat Protection Sensor_YYYYMMDDHHMMSS_001_MsiPackage.log

### センサー サービス関連のログ
- C:\Program Files\Azure Advanced Threat Protection Sensor\\<version number>\Logs
  - Microsoft.Tri.Sensor.log
  - Microsoft.Tri.Sensor-Errors.log
  - Microsoft.Tri.Sensor.Updater.log
  - Microsoft.Tri.Sensor.Updater-Errors.log

https://learn.microsoft.com/ja-jp/defender-for-identity/troubleshooting-using-logs

## 実際に遭遇したトラブルあれこれ

### インストールに失敗する

複数の検証用テナントをセットアップしていた際に、別テナントでダウンロードしたインストーラーを使いまわそうとした結果、"センサーを登録できませんでした。アクセス キーをご確認ください。" というエラーに遭遇…。Defender Portal からダウンロードした ZIP ファイルの中に含まれる SensorInstallationConfiguration.json に、接続先の情報 (FQDN や Workspace ID) が含まれているため、別テナントのアクセス キーを入力しても当然弾かれるというだけのお話でした。

![インストール時のエラー画面のスクリーンショット](/images/defender-for-identity-sensor-troubleshooting/3.png)

### サービスが起動しない

AD 等ではない独立した Windows Server VM に、スタンドアロン センサーとして構成しようとした際、以下のような "DomainControllerDnsNames is empty or not configured" のエラーに悩まされることがありました。

```txt:Microsoft.Tri.Sensor-Errors.log
2024-11-13 01:20:52.0850 Error DirectoryServicesDomainNetworkCredentialsManager Microsoft.Tri.Infrastructure.ExtendedException: DomainControllerDnsNames is empty or not configured
   at void Microsoft.Tri.Sensor.DirectoryServicesDomainNetworkCredentialsManager.UpdateConfigurations(ConfigurationCollection configurations)
   at Func<Task> Microsoft.Tri.Infrastructure.ActionExtension.ToAsyncFunction(Action action)+(TItem _) => { }
   at async Task Microsoft.Tri.Infrastructure.ConfigurationManager.RegisterConfigurationAsync(Func<ConfigurationCollection, Task> onConfigurationsUpdateAsync, Type[] configurationTypes)
   at void Microsoft.Tri.Infrastructure.TaskExtension.Await(Task task)
   at new Microsoft.Tri.Sensor.DirectoryServicesDomainNetworkCredentialsManager(IConfigurationManager configurationManager, IDomainTrustMappingManager domainTrustMapping, IMetricManager metricManager, ISensorSecretManager secretManager, IWorkspaceApplicationSensorApiJsonProxy workspaceApplicationSensorApi)
   at object lambda_method(Closure, object[])
   at object Autofac.Core.Activators.Reflection.ConstructorParameterBinding.Instantiate()
   at void Microsoft.Tri.Infrastructure.ModuleManager.AddModules(Type[] moduleTypes)
   at new Microsoft.Tri.Sensor.SensorModuleManager()
   at ModuleManager Microsoft.Tri.Sensor.SensorService.CreateModuleManager()
   at async Task Microsoft.Tri.Infrastructure.Service.OnStartAsync()
   at void Microsoft.Tri.Infrastructure.TaskExtension.Await(Task task)
   at void Microsoft.Tri.Infrastructure.Service.OnStart(string[] args)
```

DNS Name が空ということで、"DomainControllerDnsNames is empty or not configured" で検索して、Defender Portal のセンサー設定の画面でドメインを設定すれば良いらしいと判明。

https://techcommunity.microsoft.com/discussions/azureadvancedthreatprotection/microsoft-tri-sensor-error---defender-indentitiy-standalone/3966091

ドメイン名を入力して [保存] したはずなのに、何度ページをリロードしても、何分待ってもドメイン名が保存されていなくて何でかなぁと半日くらい頭を悩ませたのですが、オチとしては [+ 追加] を押してから [保存] しないとダメだったという罠でした…。複数のフォレストやドメインを登録できるようにするためだとは思うのですが、この UI は知らないとハマる…。(一応、フィードバックしておきました…。)

![センサーの設定画面のスクリーンショット](/images/defender-for-identity-sensor-troubleshooting/4.png)


### サービスが起動しない Part.2

ひたすら CreateLdapConnectionAsync failed が出て、サービスが起動しないエラーに遭遇しました。Test-NetConnection 等で試す限り TCP での疎通はできているのに…。

```txt:Microsoft.Tri.Sensor-Errors.log
2025-11-04 08:31:29.1572 Error DirectoryServicesClient+<CreateLdapConnectionAsync>d__49 Microsoft.Tri.Infrastructure.ExtendedException: CreateLdapConnectionAsync failed [DomainControllerDnsName=ADDC1.contoso.com]
   at async Task<LdapConnection> Microsoft.Tri.Sensor.DirectoryServicesClient.CreateLdapConnectionAsync(DomainControllerConnectionData domainControllerConnectionData, bool isGlobalCatalog, bool isTraversing)
   at async Task<bool> Microsoft.Tri.Sensor.DirectoryServicesClient.TryCreateLdapConnectionAsync(DomainControllerConnectionData domainControllerConnectionData, bool isGlobalCatalog, bool isTraversing)
2025-11-04 08:31:29.1774 Error DirectoryServicesClient Microsoft.Tri.Infrastructure.ExtendedException: Failed to communicate with configured domain controllers [ _domainControllerConnectionDatas=ADDC1.contoso.com]
   at new Microsoft.Tri.Sensor.DirectoryServicesClient(IConfigurationManager configurationManager, IDirectoryServicesDomainNetworkCredentialsManager domainNetworkCredentialsManager, IDomainTrustMappingManager domainTrustMappingManager, IRemoteImpersonationManager remoteImpersonationManager, IMetricManager metricManager, IWorkspaceApplicationSensorApiJsonProxy workspaceApplicationSensorApiJsonProxy)
   at object lambda_method(Closure, object[])
   at object Autofac.Core.Activators.Reflection.ConstructorParameterBinding.Instantiate()
   at void Microsoft.Tri.Infrastructure.ModuleManager.AddModules(Type[] moduleTypes)
   at new Microsoft.Tri.Sensor.SensorModuleManager()
   at ModuleManager Microsoft.Tri.Sensor.SensorService.CreateModuleManager()
   at async Task Microsoft.Tri.Infrastructure.Service.OnStartAsync()
   at void Microsoft.Tri.Infrastructure.TaskExtension.Await(Task task)
   at void Microsoft.Tri.Infrastructure.Service.OnStart(string[] args)
```

オチとしては、このサーバーは MDI 導入後に新規で立てた環境だったので、gMSA を作成する際にあわせて作成したグループに対象のホストが含まれていなかったというだけでした。

ドキュメントの手順でいうと、以下の PowerShell で作っている mdiSvc01Group に対象のホスト名が入っておらず、gMSA が利用不可だったという感じ。

https://learn.microsoft.com/ja-jp/defender-for-identity/deploy/create-directory-service-account-gmsa#create-the-gmsa-account

```PowerShell
# Variables:
# Specify the name of the gMSA you want to create:
$gMSA_AccountName = 'mdiSvc01'
# Specify the name of the group you want to create for the gMSA,
# or enter 'Domain Controllers' to use the built-in group when your environment is a single forest, and will contain only domain controller sensors.
$gMSA_HostsGroupName = 'mdiSvc01Group' ### このグループに対して、
# Specify the computer accounts that will become members of the gMSA group and have permission to use the gMSA. 
# If you are using the 'Domain Controllers' group in the $gMSA_HostsGroupName variable, then this list is ignored
$gMSA_HostNames = 'DC1', 'DC2', 'DC3', 'DC4', 'DC5', 'DC6', 'ADFS1', 'ADFS2' ### ここで列挙したサーバーしか含まれておらず、後日新設したサーバーを足していなかった…orz

# Import the required PowerShell module:
Import-Module ActiveDirectory

# Set the group
if ($gMSA_HostsGroupName -eq 'Domain Controllers') {
    $gMSA_HostsGroup = Get-ADGroup -Identity 'Domain Controllers'
} else {
    $gMSA_HostsGroup = New-ADGroup -Name $gMSA_HostsGroupName -GroupScope DomainLocal -PassThru
    $gMSA_HostNames | ForEach-Object { Get-ADComputer -Identity $_ } |
        ForEach-Object { Add-ADGroupMember -Identity $gMSA_HostsGroupName -Members $_ }
}

# Create the gMSA:
New-ADServiceAccount -Name $gMSA_AccountName -DNSHostName "$gMSA_AccountName.$env:USERDNSDOMAIN" `
 -PrincipalsAllowedToRetrieveManagedPassword $gMSA_HostsGroup
```


その他にもあれこれトラブって苦労したので、気が向いたら時間のある時にまた追記します。