---
title: "ExpressRoute の検証環境が欲しい！"
emoji: "☁️"
type: "tech"
topics:
  - "azure"
  - "networking"
  - "microsoft"
  - "expressroute"
published: true
published_at: "2023-12-22 11:42"
---

# ExpressRoute / Azure Peering Service は気軽に使えない？

Azure とオンプレミス環境を接続するための ExpressRoute や Azure Peering Service は、構築するのに時間もお金もかかるので気軽に使えない (検証できない) と思っている方も多いのではないでしょうか。実際、ExpressRoute の接続プロバイダーとの契約が必要だったり、プロバイダーからオンプレミス環境までのアクセス回線が必要だったりで、開通までに数週間や数ヶ月がかかったり、費用も高額になりがち (月額 XX 万円、最低利用期間 XX ヶ月、etc...) だと思います。

ただ、ExpressRoute のプロバイダーさんによってはセルフ サービスで (プロバイダー側のポータルをポチポチして) 作成・削除が気軽にできるサービス等も存在します。ExpressRoute 絡みの検証をしたい場合などは、そうしたサービスを利用すると幸せになれるかもしれません。

ということで、[Microsoft Azure Tech Advent Calendar 2023](https://qiita.com/advent-calendar/2023/microsoft-azure-tech) 用に検証用 ExpressRoute 回線の入手方法を書き出してみようかと思います。

**なお、ここに記載した内容の多くは私が個人的に調べたり検証した内容であり、情報が不正確だったり古い可能性もあります。また、所属する組織を代表して見解を述べるものではありません。**

# 検証用の ExpressRoute 回線を入手する方法あれこれ (実際に使ったことがあるサービス)

## Oracle Cloud Infrastructure FastConnect (L3: Private Peering)
[Microsoft と Oracle のパートナーシップ](https://news.microsoft.com/2019/06/05/microsoft-and-oracle-to-interconnect-microsoft-azure-and-oracle-cloud/)の一環として、Azure と OCI では一部のリージョン間が閉域 (ExpressRoute / FastConnect) で接続されています。そのため、Azure 側で ExpressRoute のリソースを、OCI 側で FastConnect のリソースを作成することで、クラウド間を直接 (他のデータセンター事業者や通信キャリアを介さずに) 接続することができます。あくまでも VNet / VCN 間を L3 で接続するのみなので、Private Peering しか構成はできず、Microsoft Peering の検証には利用できません。ただ、OCI のアカウントを持っている方には一番気軽な方法かと思います。

詳細な手順は両者からドキュメントが公開されていますので、そちらをご覧ください。

* [Azure と Oracle Cloud Infrastructure 間の直接相互接続をセットアップする](https://learn.microsoft.com/ja-jp/azure/virtual-machines/workloads/oracle/configure-azure-oci-networking)
* [Microsoft Azureへのアクセス](https://docs.oracle.com/ja-jp/iaas/Content/Network/Concepts/azure.htm)

## アット東京 ATBeX ServiceLink (L2: Private / Microsoft Peering)
アット東京のデータセンター内にある [Cloud Lab](https://www.attokyo.co.jp/connectivity/cloudlab.html) を短期間だけお借りする等して、[ATBeX](https://www.attokyo.co.jp/connectivity/atbex.html) ServiceLink で L2 モデルのクラウド接続を行うことができます。
![](https://storage.googleapis.com/zenn-user-upload/a0b9fe8c635a-20231218.png)

私も何度かお伺いしたことがあるのですが、データセンター内にあるラボ スペースだけでなく、スイッチやルーターも数台貸していただけたり (もちろん自前での持ち込みも可)、一人 1 つ (Private / Microsoft Peering で VLAN 2 つ) の ExpressRoute 回線を人数分用意していただくことも可能でしたので、複数人でオンサイトしてハンズオンやトレーニング等を行う場合に最適かと思います。

## Equinix Metal + Equinix Fabric (L2: Private / Microsoft Peering)

クラウド接続サービスである Equinix Fabric の[料金](https://docs.equinix.com/en-us/Content/Interconnection/Fabric/pricing-billing/Fabric-billing-pricing.htm)は、基本的に月額課金となっており、ポート料金には最低 1 年の契約期間があるので高額だと思われがちです。

ただ、実はベアメタル クラウド サービスである Equinix Metal 経由で Equinix Fabric の VC (Virtual Circuit) を作成すると、一時間単位で利用することができます。[こちらの表](https://deploy.equinix.com/product/interconnection/)の Fabric VC (Metal billed) の列がそれです。
![](https://storage.googleapis.com/zenn-user-upload/c0fea3efa946-20231218.png)

Equinix Metal で借りた物理サーバーでソフトウェア ルーターを動かし、Metal 側のポータルから Interconnections として Fabric VC を作成します。
![](https://storage.googleapis.com/zenn-user-upload/6c35e7385490-20231218.png)

その後、Fabric 側のポータルから接続する際に [Use a Token] を選択して、Metal 側で払い出されたトークンを入力することで、ベアメタル サーバーまで ExpressRoute の VLAN がつながりますので、PE ルーターとして VLAN や BGP の設定を正しく行えば、L2 モデルで ExpressRoute を構成できます。(PE ルーターを自前で構築することになるので、オンプレ側から広報する経路やアトリビュートも好きにいじれます。)
![](https://storage.googleapis.com/zenn-user-upload/45cf54f3df4d-20231218.png)

なお、言うまでもないですがベアメタル サーバーを借りることになるので、クラウド上で安い VM を立てるのと比較するとお値段もそれなりにします。長時間稼働させっぱなしだと高額になるので気を付けましょう。

## Flexibile InterConnect (L2/L3: Private / Microsoft Peering)

Smart Data Platform (SDPF) サービスの 1 つである FIC を利用するためには、[NTTコミュニケーションズ ビジネスポータル](https://b-portal.ntt.com/) のアカウントが必要です。法人向けサービスなので、基本的には法人格がないとアカウントを発行してもらえません。(ただ、とある別のサービスを個人で申し込むと、それに付随してビジネスポータルのアカウントが発行されることがあったりなかったり。)

FIC は L2 / L3 どちらのモデルも対応しており、ExpressRoute の Private / Microsoft Peering 両方が利用できるのはもちろん、Azure Peering Service (MAPS) も利用可能です。FIC 内で完結させようとすると、FIC ルーターを使用して L3 モデルで構成することになります。他方で、UNO 網などを別途契約していれば、オンプレミスまで L2 でもってきて自前の BGP ルーターで終端させることも可能なはず。[構成ガイド](https://sdpf.ntt.com/solution-guide/fic-solution-guides/)にも様々な構成パターンが記載されており、既存のネットワーク サービスと組み合わせて柔軟に利用できるのが FIC の良いところですね。(さすがに個人で UNO は無理ですが…。)

# 直接触ったことはないけど気になっているサービス

## BBIX Open Connectivity eXchange (Private / Microsoft Peering)

BBIX / BBSakura Networks さんが提供するクラウド接続サービスの [OCX](https://docs.ocx-cloud.net/) と、法人向けのインターネット接続サービスの「OCX光 インターネット」のオプションで Azure Peering Service の接続が提供されているらしいとのことで、いずれも気になってます。(あと、AS を取得している身としては IX のポートもいずれは契約してみたいものです…。)

## NTT 東日本 (Private Peering)

[プレス リリース](https://www.ntt-east.co.jp/release/detail/20171004_01.html)を読んだ程度の知識しかありませんが、法人向けのフレッツ VPN サービスを使って Azure と接続できるらしいので、既に利用している方はアクセス回線の費用がかからずオンプレまで引っ張ってこれそうなのが良さそうだなぁと思ったり。ただし、Private Peering のみ対応なのが要注意ですかね。

## Megaport Virtual Cross Connect

Megaport さんの VXC は、MVE (Megaport Virtual Edge) と組み合わせることで各社の SD-WAN と連携ができるので、オンプレのネットワークと統合管理したい人には良さそうです。[Interop Tokyo 2021 の ShowNet](https://www.slideshare.net/InteropTokyo-ShowNet/shownet-shownet2021-studio-20210416nocudasyuhei) では Cisco SD-WAN (旧 Viptela) と連携させてマルチ クラウドに接続していました。こちらも法人格が必要で個人ではダメだったはず…。

## PCCW Console Connect

Interop かなにかの展示会イベントの際に、[ポータル](https://www.consoleconnect.com/ja/)のデモを見せてもらったことがありますが、こちらも法人格が必要で個人ではダメそうでした。残念。
