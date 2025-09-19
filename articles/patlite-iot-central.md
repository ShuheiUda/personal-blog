---
title: "PATLITE NHV4 を Azure IoT Central に接続してみる"
emoji: "🚨"
type: "tech"
topics:
  - "azure"
  - "microsoft"
  - "iot"
  - "IoTCentral"
  - "PATLITE"
published: true
published_at: "2025-09-20 12:00"
publication_name: "microsoft"
---

## パトライトをクラウド経由制御したい
先日 Azure IoT Central への接続に対応した NHV4 を購入したので、セットアップ手順やクラウド経由の制御方法をまとめておきます。

## IoT Central Application を作成
Azure Portal の検索ボックスから [IoT Central Applications] のページを開き、リソース名やリージョン等を設定して作成します。
![](/images/patlite-iot-central/1.png)

作成完了後、[概要] ページに表示されたリンクから、IoT Central アプリケーションのページにアクセスします
![](/images/patlite-iot-central/2.png)

## デバイス登録
[デバイス] - [新規] から、デバイス名やデバイス ID を設定して新しいデバイスを作成します。
![](/images/patlite-iot-central/3.png)

作成したデバイスのページで [接続] をクリックすると、パトライト側に設定する各種情報が表示されます。
![](/images/patlite-iot-central/4.png)

パトライトの [クラウド設定] - [Azure 接続設定] に上記の情報を入力して、[設定] をクリックします。
(SAS トークンは、プライマリ・セカンダリのどちらを設定しても問題ありません。)

![](/images/patlite-iot-central/5.png)

1-2 分程度で [接続済み] の状態になり、パトライトのプロパティ情報を取得したことが [生データ] のページで確認出来ます。
![](/images/patlite-iot-central/6.png)

## テンプレートの作成・編集
接続済になったことを確認した後、[テンプレートの管理] - [テンプレートの自動作成] を開き、[テンプレートの作成] をクリックします。
![](/images/patlite-iot-central/7.png)

テンプレートの作成が完了したら、[+ 機能の追加] から "Method_Control1" 等の名前でコマンドを追加し、[保存] したのち、忘れずに [公開] をクリックします。
※ パトライトの説明書を見る限り、コマンド名は Method_ControlX というフォーマットで固定のようです。適当な名前にしたら動きませんでした。
![](/images/patlite-iot-central/8.png)

## トークンの生成
[アクセス許可] - [API トークン] - [+ 新規] より、アプリケーション等から API を呼び出すためのトークンを生成します。
SharedAccessSignature sr=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX といったトークンが表示されるため、こちらを忘れずにメモしておきましょう。
(ここで表示されたトークンをあとから再表示することはできません。)
![](/images/patlite-iot-central/8.png)

## API 経由でパトライトを操作してみる
上記で作成したトークンを使用して、PowerShell の Invoke-RestMethod から登録済のデバイスに対してコマンドを実行してみます。

ちなみに API のリファレンスはこちら。curl や Postman でも同様に出来るはずです。
[Devices - Run Command - REST API (Azure IoT Central) | Microsoft Learn](https://learn.microsoft.com/ja-jp/rest/api/iotcentral/dataplane/devices/run-command)

Authorization ヘッダーには SAS トークンを、Body には JSON 形式でパトライトへ投げるコマンドを記載します。(コマンド名を間違えたり、テンプレート編集後に公開してないとエラーになってハマります。)

```PowerShell
# LED を全色点灯させる場合
Invoke-RestMethod -Method Post -Uri "https://<サブドメイン>.azureiotcentral.com/api/devices/<デバイス名>/commands/<コマンド名>?api-version=2022-07-31" -Headers @{"Authorization" = "SharedAccessSignature sr=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" } -Body (@{request = @{"led" = "11111" } } | ConvertTo-Json) -ContentType 'application/json'

# LED を全色消灯させる場合
Invoke-RestMethod -Method Post -Uri "https://<サブドメイン>.azureiotcentral.com/api/devices/<デバイス名>/commands/<コマンド名>?api-version=2022-07-31" -Headers @{"Authorization" = "SharedAccessSignature sr=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX" } -Body (@{request = @{"led" = "00000" } } | ConvertTo-Json) -ContentType 'application/json'
```

ということで、API 経由でパトライトを制御できるようになりました。

あとは好きな監視製品から API を呼び出すようにするだけですが、トークンの期限は一年間らしいので運用環境で利用する場合は忘れずに更新する仕組みを作らないといけなそうです。

[Azure IoT Central で REST API 呼び出しを認証する - Azure IoT Central | Microsoft Learn](https://learn.microsoft.com/ja-jp/azure/iot-central/core/howto-authorize-rest-api)

> API トークンの有効期間は、約 1 年間です。 IoT Central アプリケーションでは、組み込みとカスタムの両方のロールに対してトークンを生成できます。 API トークンの作成時に選択した組織によって、API がアクセスできるデバイスが決まります。 アプリケーションに組織を追加する前に作成された API トークンは、ルート組織に関連付けられています。

ご注意ください。