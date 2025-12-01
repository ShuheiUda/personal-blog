---
title: "改めて Active Directory のセキュリティ対策を考えよう"
emoji: "🛡️"
type: "tech"
topics:
  - "defender"
  - "identity"
  - "activedirectory"
  - "microsoft"
published: true
published_at: "2025-12-01 23:59"
publication_name: "microsoft"
---

## はじめに / アドベント カレンダーの企画趣旨

2025 年も残り一ヶ月となりました。今年も様々なセキュリティ インシデントが話題にのぼりましたが、引き続き Active Directory (以下 AD) を狙った攻撃も数多く発生しているという話を各所で耳にしています。

また、昨今の大規模インシデントで世の中のセキュリティ意識が高まっている影響もあってか、先日 X で何気なくつぶやいた以下のポストには思いのほか多くの反響をいただきました。

https://x.com/syuheiuda/status/1983923727533228226

AD のセキュリティ対策に関心が高まっている (≒ 皆さん困っている) のかな…？と感じたので、クラウド全盛期の 2025 年に改めて Active Directory のセキュリティ対策について考えるべく、今回このアドベント カレンダーを企画しました。是非多くの方の知見を共有していただき、皆さんの身近にある AD 環境のセキュリティ対策に 1 つでも生かしていただけたらと思っています。

https://qiita.com/advent-calendar/2025/active-directory-security

ということで、初日はゆるい感じで皆さんに AD のセキュリティを考えていただくきっかけとして、いくつか改めて考えてみていただきたいポイントを提起してみようかなと思います。

## Active Directory のセキュリティ対策って何があるんだろう…

この投稿を読んでいる方の中には、AD のセキュリティ対策って何から始めたら良いんだろう…？と頭を悩ませている人もいるのではないでしょうか。

AD に対するセキュリティ対策といっても、何に備えるかによって以下にあげるようないくつかのアクションの取り方が考えられるかと思います。

- **Protect**: AD に対する攻撃を受けにくくしたり、被害を最小化するための対策
- **Detect**: 社内に侵入された際に、早期に検出するための対策
- **Respond**: 実際に被害が出てしまった際に、迅速に封じ込めを行うための準備
- **Modernize**: そもそも AD を縮小してクラウド (Entra) 管理に寄せていく
- 等々...

既に実施している対策はなにか、どこが足りていないか等、それぞれの環境によって実施すべき事項や優先度も変わるはずです。今回のアドベント カレンダー シリーズに投稿されるであろう様々なノウハウや TIPS を眺めながら、是非ご自身の管理されている AD 環境には何が必要なのか、どこから始められそうかを今一度じっくり考えてみる機会にしていただけたら幸いです。

なお、Microsoft としてのベスト プラクティスは、以下の Learn にまとまった資料があります。もしお時間があれば、こちらも是非目を通してみていただければと思います。

[Active Directory をセキュリティで保護するためのベスト プラクティス](https://learn.microsoft.com/ja-jp/windows-server/identity/ad-ds/plan/security-best-practices/best-practices-for-securing-active-directory)

## Microsoft Defender for Identity ってご存じですか？

先の X のポストでも言及しましたが、Microsoft では Microsoft Defender for Identity (通称: MDI) という AD 向けのセキュリティ製品を提供しています。

今回 MDI を始めて知ったという方もいると思うのでざっくり説明すると、Active Directory 関連の各サーバー (ADDC, ADFS, ADCS, Entra Connect) に対して MDI のセンサー (エージェント) をオンボードしていただくことで、認証関連の情報 (監査ログやトラフィック、構成情報など) を収集して、クラウド側で自動的に分析を行い、怪しいふるまいが見られた際に[アラート](https://learn.microsoft.com/ja-jp/defender-for-identity/alerts-overview)をあげてくれるような製品です。また、AD の構成をチェックして[推奨事項](https://learn.microsoft.com/ja-jp/defender-for-identity/security-assessment)を提案してくれますので、検出された項目を可能な範囲で一つずつ対処いただくことで攻撃面を減らし、より堅牢な構成にしていただくこともできるかと思います。

![セキュア スコアの例](/images/active-directory-security-advent-calendar-2025-1/1.png)*セキュア スコアの例*

なお、MDI は M365 E5 をはじめとする以下のライセンスをお持ちのお客様がご利用いただけます。E5 のライセンスをお持ちにもかかわらずまだ導入されていない方 (使わないのはもったいない！) や、これから AD のセキュリティ対策を実施していこうと思われている方は、是非検討してみてください。

https://learn.microsoft.com/ja-jp/defender-for-identity/deploy/prerequisites-sensor-version-2

```- Enterprise Mobility + Security E5 (EMS E5/A5)
- Microsoft 365 E5 (Microsoft E5/A5/G5)
- Microsoft 365 E5/A5/G5/F5* Security
- Microsoft 365 F5 Security + Compliance*
- A standalone Defender for Identity license
- 
*両方の F5 ライセンスは、Microsoft 365 F1/F3 または F3とEnterprise Mobility + Security E3 Office 365 必要があります。
```

## Tier モデルやエンタープライズ アクセス モデルは採用されていますか？

AD が残っている、すなわちオンプレミスがまだまだ現役の皆様、AD や特権アカウント、それらを管理する端末は適切に階層を分けて管理できているでしょうか。

もう少し具体的に言うと、

- Domain Admins 等の強い特権を有するアカウントが多数存在していないでしょうか
- IT 管理者の通常のユーザー アカウント (メールやブラウジングをするユーザー) で AD を管理・操作されていないでしょうか
- IT 管理者の通常の業務端末 (メールやブラウジングをする端末) で AD を管理・操作されていないでしょうか

こんな風に問われたら、皆さんは自信を持って問題ないと即答できますか？(などと偉そうなことを書いていますが、[私のコンテナ データセンター](https://thinkit.co.jp/article/13243)ではまだ実装しきれていません…ｗ)

今となっては少し古い考え方とされていますが、AD 環境を守るベスト プラクティスの一つに [Tier モデル (archive.org)](https://web.archive.org/web/20201210154206/https://docs.microsoft.com/en-us/windows-server/identity/securing-privileged-access/securing-privileged-access-reference-material) という考え方があります。(最近だと、クラウド環境まで加味した[エンタープライズ アクセス モデル](https://learn.microsoft.com/ja-jp/security/privileged-access-workstations/privileged-access-access-model)もありますが、オンプレが主軸な方には Tier モデルもまだまだ有効な考え方だと個人的には思っていますので、今回はあえて Tier モデルで…。)

Tier モデルを非常にざっくり説明すると、

- IT 資産を Tier 0 / 1 / 2 の 3 つの階層に分け、
- 各階層を管理する端末やユーザーは、他の Tier へのアクセスを禁止するように構成します

つまり、最も守るべき Tier 0 の資産にあたる AD の管理作業をする際には Tier 0 用の管理端末と管理アカウントを使い、Tier 1 や Tier 2 からのアクセスは遮断しておくといった構成を取ります。(下図のように、その他の Tier でも同様。)

![Tier モデルの概要図](/images/active-directory-security-advent-calendar-2025-1/2.png)*Tier モデルの概要図*

これにより、仮に一般社員のユーザーや端末 (Tier 2) が侵害されたとしても、横移動 (ラテラル ムーブメント) で上位の Tier へ侵害を広げられにくくするといったことができます。(Pass-the-Hash や Pass-the-Ticket のような攻撃手法で権限昇格されるリスクを低減することができます。)

こうした管理用の専用端末のことを PAW (Privileged Admin Workstation) や SAW (Secure Admin Workstation) と呼び、Learn にも[ドキュメント](https://learn.microsoft.com/ja-jp/security/privileged-access-workstations/privileged-access-devices)も用意されています。管理者が端末を何台も持つことになるのはコスト的にも物理的にも荷が重いかもしれませんが、Tier モデルは AD 環境を保護する上で非常に有用ですので、Learn の関連ドキュメントにも是非とも目を通していただければと思います。

---

ということで、Active Directory Security Advent Calendar 2025 - Day 1 は以上です。12/25 まで無事完走できるように盛り上げていけたらと思いますので、皆様も是非ご参加くださいー。(フォロー、いいね、SNS での拡散等での応援も是非よろしくお願いします！)
