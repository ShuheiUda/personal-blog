---
title: "Raritan PX3 の消費電力を Grafana Agent Flow で Grafana Cloud に飛ばして可視化する"
emoji: "🔍"
type: "tech"
topics: [Grafana, Raritan, Prometheus]
published: false
---
## 概要

自前で Grafana とか Prometheus 運用したくないので、Grafana Cloud に投げつけてみました

* Grafana Cloud のアカウント (Free で問題なし)
* Grafana Agent Flow を入れる適当な端末 (今回は Windows OS)
* Raritan PX3 (PX3-5460R, PX3-5145R, ファームウェア: 4.0.20.5-49038)

## セットアップ

1. [Grafana Agent Flow](https://github.com/grafana/agent/releases/latest) から grafana-agent-flow-installer.exe.zip (grafana-agent-installer.exe.zip ではないので注意) をダウンロードして、インストール

2. Grafana Cloud 側で Home > Connections > Add new connection > Hosted Prometheus metrics から API token を作成
![Alt text](/images/raritan-px3-grafana-cloud-1.png)
![Alt text](/images/raritan-px3-grafana-cloud-2.png)

3. "C:\Program Files\Grafana Agent Flow\config.river" を編集

```river:config.river
logging {
 level = "info"
}
prometheus.remote_write "default" {
  endpoint {
    url = "https://prometheus-XXXXX.grafana.net/api/prom/push"
    basic_auth {
      username = "xxxxxxxx"
      password = "API secret"
    }
  }
}

prometheus.scrape "PDU" {
  targets = [{
    __address__ = "172.16.x.1:443",
  },
  {
    __address__ = "172.16.x.2:443",
  },
  {
    __address__ = "172.16.x.3:443",
  }]
  scrape_interval = "1m"
  scrape_timeout = "1m"
  scheme = "https"
  tls_config {
    insecure_skip_verify = true
  }
  metrics_path = "/cgi-bin/dump_prometheus.cgi"
  params = {"include_names" = ["1"]}
  basic_auth {
    username = "UserName"
    password = "Password"
  }
  forward_to = [prometheus.remote_write.default.receiver]
}
```

4. services.msc から "Grafana Agent Flow" を再起動

5. ダッシュボードを作成

* PDU 毎の電圧、周波数

```:Display name
${__field.labels.pduname}
```

![Alt text](/images/raritan-px3-grafana-cloud-3.png)
![Alt text](/images/raritan-px3-grafana-cloud-4.png)

* Outlet 毎の消費電力

```:Transform data
^.*outletid=".*?", (.*)pduid=".*".*$
$1N/A

^.*outletname="(.*?)".*$
$1
```

![Alt text](/images/raritan-px3-grafana-cloud-5.png)

## 完成

![Alt text](/images/raritan-px3-grafana-cloud-6.png)

## 参考

* [Raritan PDUでPrometheus+Grafanaでメトリクスを取る PX3-5138JR](https://blog.nishi.network/2023/01/09/intelligent-pdu-part3/)
* [RaritanのPDUのデータをGrafanaで可視化する](https://zenn.dev/cocotte/articles/748dc59ab4053f)
