---
title: "Azure サブスクリプションの契約形態 (Offer ID) を確認したい"
emoji: "☁️"
type: "tech"
topics:
  - "azure"
  - "subscription"
  - "billing"
  - "microsoft"
published: true
published_at: "2024-11-09 11:37"
publication_name: "microsoft"
---

## Azure の契約種別 (オファー ID / プラン ID)

Azure サブスクリプションの契約形態は、無償評価版やスポンサー プラン、従量課金、EA、CSPなど多岐に及びます。普段あまり契約形態を意識されることは多くないかと思いますが、実はこうした契約方法の違いによって、費用や支払方法、サポート窓口、返金プロセスなどが異なります。

こうした契約形態を識別するため、各サブスクリプションには内部的にオファー ID (プラン ID) というものが設定されており、以下に一覧が掲載されています。
https://azure.microsoft.com/ja-jp/support/legal/offer-details

多数の Azure サブスクリプションを契約・保有している場合、棚卸しの際にどのサブスクリプションがどの契約かを網羅的に確認したいということが稀にあります。(例えば、Microsoft の中の人やパートナー企業の方だと無償枠がついた特典系のサブスクリプションを複数持っていたりとか、事業部やシステム単位で個別に Azure を契約していて契約種別がバラバラだったりとか…。)

そういった際に、オファー ID を確認する方法が Web を検索しても全然ヒットしなかったので、相談を受けて調べたついでに以下にまとめておきます。

## Azure Portal での確認方法

Azure Portal では、サブスクリプションの概要ページや、プロパティのページに以下のようにプラン ID が記載されています。数が少なければポータルで一つずつ確認するのが最もお手軽です。

![](/images/get-offer-id/1.png)

![](/images/get-offer-id/2.png)

なお、権限が不足していたり契約形態によっては、以下のように見えない・表示されない場合もあります。

(権限が不足している例)
![](/images/get-offer-id/3.png)

(個人で契約している CSP サブスクリプションの例)
![](/images/get-offer-id/4.png)

裏取りはしていないですが、権限不足の場合には恐らくこの辺りに書いてある権限をつければ見えるようになるはず…。
https://learn.microsoft.com/ja-jp/azure/cost-management-billing/manage/manage-billing-access#give-others-access-to-view-billing-information

## REST API での確認方法

大量のサブスクリプションでオファー ID を確認したい場合は、REST API を使って以下のように二段階でオファー ID を取得することができそうでした。(もちろん、この場合も各サブスクリプションへの権限が必要で、権限を持っていないサブスクリプションの情報は取得できませんし、契約種別によっては表示されないかもしれないのでご注意を。)

まず、Billing Accounts - List を使って課金アカウントの情報を取得します。
https://learn.microsoft.com/ja-jp/rest/api/billing/billing-accounts/list

```
{
  "value": [
    {
      "id": "/providers/Microsoft.Billing/billingAccounts/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx_2019-05-31",
      "name": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx_2019-05-31",
      "properties": {
        "accountStatus": "Active",
        "accountType": "Individual",
        "accountSubType": "Individual",
        "agreementType": "MicrosoftCustomerAgreement",
        "displayName": "Syuhei Uda",
        "hasReadAccess": true,
        "primaryBillingTenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "billingRelationshipTypes": [
          "Direct"
        ],
        "qualifications": [
          "Commercial"
        ]
      },
      "type": "Microsoft.Billing/billingAccounts"
    },
    {
      "id": "/providers/Microsoft.Billing/billingAccounts/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "name": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
      "properties": {
        "accountStatus": "Active",
        "accountType": "Individual",
        "accountSubType": "Other",
        "agreementType": "MicrosoftOnlineServicesProgram",
        "displayName": "xxxxxxxx",
        "hasReadAccess": true,
        "notificationEmailAddress": "xxxxx@xxxxxxxx.xxx",
        "primaryBillingTenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "soldTo": {
          "addressLine1": "xxxxxxxx",
          "addressLine3": "",
          "city": "xxxxxxxx",
          "companyName": "xxxxxxxx",
          "country": "xxxx",
          "firstName": "xxxxxxxx",
          "lastName": "xxxxxxxx",
          "phoneNumber": "xxxxxxxxxx",
          "postalCode": "xxxxxxxx",
          "region": "xxxxxxxx",
          "isValidAddress": true
        }
      },
      "type": "Microsoft.Billing/billingAccounts"
    }
  ]
}
```

続いて、上記で取得した値をもとに Billing Subscriptions - List By Billing Account で課金アカウントに紐づく Azure サブスクリプションの一覧が取得できます。

```
{
  "totalCount": 3,
  "value": [
    {
      "id": "/providers/Microsoft.Billing/billingAccounts/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/billingSubscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxx",
      "name": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
      "properties": {
        "displayName": "従量課金",
        "offerId": "MS-AZR-0003P",
        "productCategory": "UsageBased",
        "purchaseDate": "0001-01-01T00:00:00.0000000Z",
        "status": "Active",
        "operationStatus": "None",
        "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "suspensionReasons": [],
        "suspensionReasonDetails": []
      },
      "type": "Microsoft.Billing/billingAccounts/billingSubscriptions"
    },
    {
      "id": "/providers/Microsoft.Billing/billingAccounts/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/billingSubscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxx",
      "name": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
      "properties": {
        "displayName": "Visual Studio Enterprise",
        "offerId": "MS-AZR-0063P",
        "productCategory": "UsageBased",
        "purchaseDate": "0001-01-01T00:00:00.0000000Z",
        "status": "Active",
        "operationStatus": "None",
        "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "suspensionReasons": [],
        "suspensionReasonDetails": []
      },
      "type": "Microsoft.Billing/billingAccounts/billingSubscriptions"
    },
    {
      "id": "/providers/Microsoft.Billing/billingAccounts/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/billingSubscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxx",
      "name": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxx",
      "properties": {
        "displayName": "PAYG",
        "offerId": "MS-AZR-0003P",
        "productCategory": "UsageBased",
        "purchaseDate": "0001-01-01T00:00:00.0000000Z",
        "status": "Active",
        "operationStatus": "None",
        "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "suspensionReasons": [],
        "suspensionReasonDetails": []
      },
      "type": "Microsoft.Billing/billingAccounts/billingSubscriptions"
    }
  ]
}
```

こちらで取得した結果の中に、offerId が記載されていますので、あとはお好きなようにスクリプト等を書いていただければと。