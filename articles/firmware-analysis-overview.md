---
title: "Firmware 分析を試してみる"
emoji: "🛡️"
type: "tech"
topics:
  - "azure"
  - "security"
  - "microsoft"
  - "firmware"
  - "firmware"
published: true
published_at: "2025-10-12 02:00"
publication_name: "microsoft"
---

## はじめに
Azure 上で IoT / OT 製品の Firmware 分析を行ってくれるサービスがリリースされたので、試してみました。

[Firmware Analysis now Generally Available](https://techcommunity.microsoft.com/blog/iotblog/firmware-analysis-now-generally-available/4458413)

本サービスでは、Linux ベースで 1 GB 未満のファームウェア イメージをアップロードすることで、そのイメージに含まれるコンポーネントを解析して SBOM (Software Bill of Materials / ソフトウェア部品表) を作成し、既知の脆弱性情報 (CVE) と照らし合わせてチェックしてくれます。その他、ハードコードされた認証情報 (ユーザー アカウント、パスワード ハッシュ) や、SSL/TLS 証明書などの埋め込まれたシークレットを検出してくれたり、レポートの生成も行ってくれるようです。

[チュートリアル: ファームウェア分析を使用して IoT/OT ファームウェア イメージを分析する](https://learn.microsoft.com/ja-jp/azure/firmware-analysis/tutorial-analyze-firmware)

## 試してみる

リリース ブログには Azure Arc を介して提供されるような旨が書かれているのですが、Azure Arc のページにはファームウェア分析のメニューらしきものは見当たりません。今のところ独立したサービスとして専用のページが用意されているようなので、検索ボックスから Firmware analysis 等で検索します。

![検索ボックスに Firmware analysis と入力](/images/firmware-analysis-overview/1.png)

ファームウェア分析を始めるにあたって、まずはファームウェア ワークスペースを作成します。2025/10 時点では Free レベルしか用意されておらず、1 サブスクリプションあたり 1 ワークスペース、一ヶ月当たり 5 イメージまで無償で利用できるようです。いずれは有償レベルが用意され、スキャン回数等に応じて課金されるものと思われます。

![ファームウェア ワークスペースを作成](/images/firmware-analysis-overview/2.png)

ワークスペースができたら、イメージ ファイルをアップロードします。あとから探しやすいように、ベンダー名や機種名、バージョン等も適切に入力しておきます。

![ファームウェアをアップロード](/images/firmware-analysis-overview/3.png)

アップロード後、状態が Extracting から Ready に変わるまで数分程度待ちます。抽出されたファイルのサイズやファイル数などが表示されています。

![ファームウェア分析結果の概要](/images/firmware-analysis-overview/4.png)

[結果を表示する] から、分析結果の詳細なレポート画面に遷移します。イメージ内に含まれているコンポーネントの情報や、既存の CVE にマッチするものがグラフィカルに表示されます。

![分析結果のレポート](/images/firmware-analysis-overview/5.png)

SBOM のタブを開くと、イメージに含まれている各種コンポーネントの情報が確認できます。

![SBOM タブ](/images/firmware-analysis-overview/6.png)

弱点タブでは既知の CVE にマッチしたものの一覧が表示されます。今回は、意図的に少し古いファームウェアを解析させたので、openssl の Critical な脆弱性を検出してくれています。

![弱点 タブ](/images/firmware-analysis-overview/7.png)

バイナリ強化タブは、実行可能ファイルが推奨されるセキュリティ設定をしようしてコンパイルされたかを確認できるらしいです。(Linux 詳しくないので、あんまり分かってない…)

![バイナリ強化タブ](/images/firmware-analysis-overview/8.png)

パスワード ハッシュ タブでは、ファームウェアに含まれるユーザー名やパスワード ハッシュの情報が表示されます。

![パスワード ハッシュ タブ](/images/firmware-analysis-overview/9.png)

証明書タブには、ファームウェア内で検出された SSL/TLS 証明書の一覧が表示されます。これはファームウェアに埋め込まれた証明書の期限切れに起因するトラブルを未然に防ぐ用途でも役に立つかも…？(2018 年に某携帯キャリアでベンダー埋め込みの証明書に起因する大規模な通信障害とかありましたよね…。)

![証明書タブ](/images/firmware-analysis-overview/10.png)

キー タブには、公開鍵や秘密鍵の情報が表示されます。

![キー タブ](/images/firmware-analysis-overview/11.png)

## 非対応イメージの場合

手持ちのネットワーク機器のファームウェアをアップロードしてみたところ、Linux ベースではないファームウェアをアップロードした場合には、解析できずに特に情報が出てこないようです。

![Linux ベースではないファームウェアの場合](/images/firmware-analysis-overview/12.png)

## レポートの保存・印刷

概要タブの [PDF としてダウンロードする] から、PDF 保存や印刷も可能です。今のところ、概要ページに表示されているグラフ等が出力されるのみで、詳細までは含まれていませんでした。

![レポートの印刷](/images/firmware-analysis-overview/13.png)

## 余談
リリース ブログの Azure Arc という記載が違和感あったので少し調べてみたところ、どうも Defender for IoT 側で同様の機能が提供されていた雰囲気がありました。

[デバイス ビルダー向け Microsoft Defender for IoT とは](https://learn.microsoft.com/ja-jp/azure/defender-for-iot/device-builders/release-notes)

確かにそういわれてみれば、そんな項目ありましたね。こちらからもファームウェア分析の画面と同じものが見れそうです。(結局 Azure Arc を介して～というのは謎のまま…)

![Defender for IoT の画面](/images/firmware-analysis-overview/14.png)

2021 年に Refirm Labs を買収して、Defender for IoT チームを強化すると言っていたので、その系譜のサービスということでしょうかね。

[Microsoft acquires ReFirm Labs to enhance IoT security](https://www.microsoft.com/en-us/security/blog/2021/06/02/microsoft-acquires-refirm-labs-to-enhance-iot-security/)

IoT / OT デバイスを提供しているベンダーさんがファームウェアをリリースする前のチェックで使ってもいいだろうし、ユーザー企業側の IoT / OT セキュリティ担当の人にとっても役に立ちそうですね。

個人的には、自宅に有象無象の検証機器を持っている身なので、いろんなファームウェアをアップロードして解析させてみるのも面白そうだなぁと思いました。
