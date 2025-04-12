---
title: "prometheus のアラートを alertmanager に飛ばして Line に通知する"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus, prometheus, alertmanager]
published: true
---
前回、[quarkus でメトリクスを取得する](https://zenn.dev/marcha/articles/quarkus-prometheus-line)方法を試しました。今回はこのメトリクスを prometheus に取り込んで、アラームを設定し、alertmanager にアラートを飛ばしてみたいと思います。
で、せっかくなので前々回作成した [quarkus から Line に通知を飛ばすアプリケーション](https://zenn.dev/marcha/articles/quarkus-line-jwt)を合わせて、さらに Line にも通知を飛ばしてみます。

# 環境
- Oracle Linux 8.10
- quarkus 3.21.0
- quarkus CLI 3.21.0
- Open JDK 17.0.14
- [Prometheus v3.2.1](https://hub.docker.com/layers/prom/prometheus/v3.2.1/images/sha256-5d34817348f69ddb956be509cafa4d4c2b5ecbbb5b786dd99d5a62e686b2ece4)
- [alertmanager v0.28.1](https://hub.docker.com/layers/prom/alertmanager/v0.28.1/images/sha256-220da6995a919b9ee6e0d3da7ca5f09802f3088007af56be22160314d2485b54)

# 作成するシステムの構成
下図のような感じです。
![](https://storage.googleapis.com/zenn-user-upload/be2409328e05-20250412.png)

素数表示アプリケーションがメインとなるアプリケーションで、そのアプリケーションのメトリクスを prometheus が監視しています(②)。で、異常を感知すると、Prometheus は alertmanager に通知を飛ばします(③)。通知を受け取った alertmanager は、Line に通知する quarkus アプリケーションにリクエストを飛ばします(④)。
Line に通知する quarkus アプリケーションは、Messaging API を利用して Line に通知します(⑤⑥)。

# Prometheus の設定と起動
[前回](https://zenn.dev/marcha/articles/quarkus-prometheus-line)、[前々回](https://zenn.dev/marcha/articles/quarkus-line-jwt)作成した、素数表示アプリケーションと Line に通知するアプリケーションはそれぞれ、192.168.56.130:8080, 192.168.56.130:8081 で動いているものとします。

## Prometheus の設定ファイルを定義する
ひとまず Prometheus に素数表示アプリケーションからメトリクスを取得させるための設定を書きます。
```yaml:prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'quarkus-app'
    static_configs:
      - targets: ['192.168.56.130:8080']
    metrics_path: '/q/metrics'
```
`scrape_interval` で、メトリクスを取得する間隔をしています。
`scrape_configs > static_configs > targets` で メトリクスを取得するサーバの IP:Port, `metrics_path` でパスを指定しています。

## アラートのルールを作成する
次に、アラートのルールを定義します。
素数表示アプリケーションが、1分間の間に50回以上 404 を返した場合に、アラームを出すと指定してみます。
```yaml:alert-rules.yml
groups:
  - name: quarkus-alerts
    rules:
      - alert: TooMany404s
        expr: |
          increase(http_server_requests_seconds_count{instance="192.168.56.130:8080", job="quarkus-app", method="GET", outcome="CLIENT_ERROR", status="404", uri="NOT_FOUND"}[1m]) > 50
        for: 0s
        labels:
          severity: warning
```
`increase(XXXX)[yyy] > zz` で yyy 時間の間に、XXXX というメトリクスが zz 数増加したらアラートを出すという指定になります。

## Prometheus の設定ファイルに作成したアラートルールを追記する
上で作成した alert-rules.yml を prometheus.yml にアラートルールとして追記して、アラートを発火したら 9093 ポートで動いている alertmanager に通知するという設定をします。
```yaml:prometheus.yml に追記
rule_files:
  - "/etc/prometheus/alert-rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["192.168.56.130:9093"]
```
alertmanager の IP:port を `alerting > alertmanagers > static_configs > targets` に指定するだけです。

## Prometheus の起動
ここまでできたら、Prometheus を起動しましょう。上で作成した設定ファイルを Prometheus のイメージに入れて、起動します。

```Dockerfile:Dockerfile
FROM docker.io/prom/prometheus:v3.2.1

COPY --chown=nobody:nobody prometheus.yml /etc/prometheus/prometheus.yml
COPY --chown=nobody:nobody alert-rules.yml /etc/prometheus/alert-rules.yml
```

ビルドして実行します。
```sh:イメージの build & コンテナの起動
podman build -t my-prometheus:v3.2.1 .
podman run --rm --name prometheus -p 9090:9090 \
  my-prometheus:v3.2.1 --config.file=/etc/prometheus/prometheus.yml
```

# alertmanager の設定と起動
次に、alertmanager に webhooks の設定をして、Prometheus からアラートがきたら Line に通知を飛ばす quarkus アプリケーションにリクエストを飛ばすようにします。

## alertmanager の設定ファイルを作成する
alertmanager.yml をという名前で作成します。
以下のように、`webhook_configs > url` にアラートを受け取った時にリクエストを送信する URL を指定します。今回は 8081 でリッスンしている Line に通知を飛ばす quarkus アプリケーションにリクエストを飛ばします。
```yaml:alertmanager.yml
route:
  receiver: 'webhook_receiver'

receivers:
  - name: 'webhook_receiver'
    webhook_configs:
      - url: 'http://192.168.56.130:8081/notify-message'
```

## alertmanager を起動する
上記の設定ファイルを alertmanager のイメージに入れます。
```Dockerfile:Dockerfile
FROM docker.io/prom/alertmanager:v0.28.1

COPY --chown=nobody:nobody alertmanager.yml /etc/alertmanager/alertmanager.yml
```

ビルドして起動します。
```sh
podman build -t my-alertmanager:v0.28.1 .
podman run --rm --name alertmanager -p 9093:9093 \
  my-alertmanager:v0.28.1 --config.file=/etc/alertmanager/alertmanager.yml
```

# Line に通知する quarkus アプリケーションを修正する
alertmanager の webhook の[ドキュメント](https://prometheus.io/docs/alerting/latest/configuration/#webhook_config)を見てみると、alertmanager は Prometheus からアラートを受け取ると、POST で通知先のサーバに送信するようでした。
前々回作成した Line に通知する quarkus アプリケーションは GET しか対応していなかったので、POST に対応するように修正します。
ただ、リクエスト時の JSON 形式も書いてありますが、その形式の POJO を作成するのが面倒なので String でデータを受け取っちゃいます、、、

そのため修正は簡単で、@POST アノテーションで作成したメソッド内に GET で動かしていたメソッドを呼ぶだけです。
```java:src/main/java/com/marcha/NotifyMessage.java
    @POST
    @Produces(MediaType.TEXT_PLAIN)
    @Consumes(MediaType.APPLICATION_JSON)
    public String notifyMessagePost(String alertManagerPojo) {
        this.notifyMessage(alertManagerPojo); //GET のメソッド
        
        return "OK! I sent the word '" + message + "'!";
    }
```

# 実際にアラートを出してみる
実際にアラートを出してみます。
404 が大量に送られてきた場合に発火するので、とりあえず素数表示アプリケーションが動く 8080 ポートの、存在しない URL へ大量にリクエストを送信してみます。
Apache Bench を利用するのが簡単ですね。以下のように100回くらいリクエストしてみます。
```sh
podman run --rm docker.io/library/httpd:2.4.63 ab -n 300 -c 1 192.168.56.130:8080/sonzaishinai
```

そうすると、Prometheus が過去1分間に50回以上 404 を返した回数が増加したことを検知して、alertmanager にアラートを投げます。さらに alertmanager は、アラートを受け取ると、8081 で動く Line に通知する quarkus アプリケーションにリクエストを送信します。
これで、無事 Line に通知が来ました！

# まとめ
今回は、[前回](https://zenn.dev/marcha/articles/quarkus-prometheus-line)、[前々回](https://zenn.dev/marcha/articles/quarkus-line-jwt)に作成したアプリケーションを組み合わせて障害を通知するシステムを構築してみました。
ほとんどコーディングしなくても簡単に色々できるので、やっぱりミドルウェアって楽しいですよね。
というところで今回は以上です！