---
title: "管理者ロールを持っているのに MFA が有効になっていないユーザーを見つける方法"
emoji: "🔑"
type: "tech"
topics:
  - "azure"
  - "microsoft"
  - "entra"
  - "microsoft 365"
  - "security"
  - "defender"
published: true
published_at: "2025-06-24 11:42"
publication_name: "microsoft"
---

## 管理者権限を持つユーザーには、MFA を設定しましょう

管理者権限を持つユーザーを攻撃者から守る上では、多要素認証 (MFA) の導入は非常に重要です。

今回は、Defender ポータルの[セキュア スコア](https://security.microsoft.com/securescore)にも出てくる、管理者ロールが付与されているにもかかわらず MFA が有効になっていないユーザーを探す方法をご紹介します。

![Defender ポータルのセキュアスコアの画面](/images/check-admin-users-without-mfa/1.png)

## Microsoft Graph PowerShell を使用して、対象のユーザーを抽出する

Entra ID 上に存在するユーザーやロールの情報は、Microsoft Graph PowerShell を使うと簡単に取得することが可能です。

インストール方法などは以下のドキュメントに記載がありますので、参照ください。

[Microsoft Graph PowerShell を使用して Microsoft 365 に接続する](https://learn.microsoft.com/ja-jp/microsoft-365/enterprise/connect-to-microsoft-365-powershell)

そのうえで、いわゆる管理者ロール (以下) の権限が付与されているユーザーを抽出し、そのうち MFA が有効になっていないユーザーを見つけ出すスクリプトが後述の通りとなります。

冒頭の Tenant ID の部分だけご自身の環境にあわせて書き換えてご利用ください。 (複数のテナントを管理されている場合など、明示的にテナント ID を指定する必要があります。)

(本スクリプトは Copilot にベースを作ってもらい、多少デバッグや修正を加えて概ね期待した動作をすることを確認していますが、あくまでもサンプルですので必要に応じて書き換えていただくようお願いします。)

- グローバル管理者 
- 認証管理者 
- 課金管理者 
- 条件付きアクセス管理者 
- Exchange 管理者 
- ヘルプデスク管理者 
- セキュリティ管理者 
- SharePoint 管理者 
- ユーザー管理者 

### サンプル スクリプト
```
# Connect to Microsoft Graph
Connect-MgGraph -Scopes "User.Read.All", "RoleManagement.Read.Directory", "Directory.Read.All" -TenantId xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Define the list of admin roles to check
$adminRoles = @(
    "Global Administrator",
    "Authentication Administrator",
    "Billing Administrator",
    "Conditional Access Administrator",
    "Exchange Administrator",
    "Helpdesk Administrator",
    "Security Administrator",
    "SharePoint Administrator",
    "User Administrator"
)

# Get all directory roles
$roles = Get-MgDirectoryRole

# Filter roles to only those in the list
$targetRoles = $roles | Where-Object { $adminRoles -contains $_.DisplayName }

# Initialize an array to store users without MFA
$usersWithoutMFA = @()

# Loop through each role and get members
foreach ($role in $targetRoles) {
    $members = Get-MgDirectoryRoleMember -DirectoryRoleId $role.Id
    foreach ($member in $members) {
        if ($member.AdditionalProperties.'@odata.type' -eq '#microsoft.graph.user') {
            $user = Get-MgUser -UserId $member.Id
            $authMethods = Get-MgUserAuthenticationMethod -UserId $user.Id
            $hasMFA = $authMethods | Where-Object { $_.AdditionalProperties.'@odata.type' -match 'authenticationMethod' -and $_.AdditionalProperties.'@odata.type' -ne '#microsoft.graph.passwordAuthenticationMethod' }
            if (-not $hasMFA) {
                $usersWithoutMFA += [PSCustomObject]@{
                    DisplayName = $user.DisplayName
                    UserPrincipalName = $user.UserPrincipalName
                    Role = $role.DisplayName
                }
            }
        }
    }
}

# Output the users without MFA
$usersWithoutMFA | Format-Table -AutoSize
```


### 追記

と、ここまで書いていたら Entra Portal の [概要] - [推奨事項] にも同じような項目があり、[影響を受けたリソース] で対象ユーザーが見れることが判明しましたとさ。とほほ。

![Entra ポータルの推奨事項の一覧](/images/check-admin-users-without-mfa/2.png)

![Entra ポータルの推奨事項で対象のユーザーが確認出来る画面](/images/check-admin-users-without-mfa/3.png)