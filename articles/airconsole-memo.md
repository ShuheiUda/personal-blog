---
title: "AirConsole の使い方メモ"
emoji: "📝"
type: "tech"
topics: [AirConsole, Console, Gadget]
published: true
---
## Wi-Fi

* SSID: AirConsole-XX
* Password: 12345678

※ 技適はとっていないと思われるので、[技適の特例申請](https://www.tele.soumu.go.jp/j/sys/others/exp-sp/) をするか、電波暗室で。

## Web Console

* URL: http://192.168.10.1
* User: admin
* Pass: admin

## SSH to Serial Port

* 事前に Enable SSH to serial port にチェックを入れて、ポート番号を設定 (既定だと 4001)
* SSH で接続する際のユーザー名を admin:1, admin:2 のように入力することで、直接コンソールアクセスが可能
* 例: ssh admin:1@192.168.10.1

## 参考

* [Manual (PDF)](https://www.get-console.com/airconsole/files/Airconsole-User-Manual-Full-v2.51.pdf)
* [Document](https://support.get-console.com/support/solutions/articles/5000712932-using-the-ssh-to-serial-port-feature-of-airconsole)
