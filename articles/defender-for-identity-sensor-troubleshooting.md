---
title: "Microsoft Defender for Identity のセンサー サービスが起動に失敗した原因を調査する"
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

![alt text](/images/defender-for-identity-sensor-troubleshooting/1.png)

センサーのインストールが完了すると、Windows サービスとして以下の二つが追加されていることが確認できます。2 つとも実行中になっていることを確認しましょう。

- Azure Advanced Threat Protection Sensor
- Azure Advanced Threat Protection Sensor Updater

![alt text](/images/defender-for-identity-sensor-troubleshooting/2.png)

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

![alt text](/images/defender-for-identity-sensor-troubleshooting/3.png)

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

ドメイン名を入力して [保存] したはずなのに、何度ページをリロードしても、何分待ってもドメイン名が保存されていなくて何でかなぁと半日くらい頭を悩ませたのですが、オチとしては [+ 追加] を押してから [保存] しないとダメだったという罠でした…。複数のフォレストやドメインを登録できるようにするためだとは思うのですが、この UI は知らないとハマる…。(あとでフィードバックします…。)

![alt text](/images/defender-for-identity-sensor-troubleshooting/4.png)


その他にもあれこれトラブって苦労したので、気が向いたら時間のある時にまた追記します。