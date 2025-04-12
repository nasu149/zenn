---
title: "quarkus のメトリクスを収集する"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus]
published: false
---
今回は quarkus におけるメトリクスの収集方法を試してみたので残します。[Micrometer を利用する](https://ja.quarkus.io/guides/telemetry-micrometer)ようです。


# 環境
- Oracle Linux 8.10
- quarkus 3.21.0
- quarkus CLI 3.21.0
- Open JDK 17.0.14

# メトリクスの取得をしてみる
早速デフォルトで用意されているメトリクスを取得してみます。

## quarkus AP のひな形の作成
まずはひな形の作成をします。
```sh
export GROUPID=com.marcha
export AETIFACTID=quarkus-micrometer-test
export VERSION=1.0.0
quarkus create app -P io.quarkus.platform:quarkus-bom:3.21.0 ${GROUPID}:${AETIFACTID}:${VERSION}
```

## extension の追加
次に micrometer-registry-prometheus extension の追加をします。
```sh
cd ${AETIFACTID}
quarkus extension add micrometer-registry-prometheus,rest-jackson
```

## デフォルトで用意されているメトリクスを取得する
ここで quarkus を起動して、実際にメトリクスを取得してみます。
```sh
quarkus dev
```
起動ができたら、curl でメトリクスを取得します。
デフォルトのエンドポイントは **/q/metrics** なのでそこにリクエストを投げるだけです。
```shell-session
$ curl -s localhost:8080/q/metrics | head
# TYPE worker_pool_queue_delay_seconds_max gauge
# HELP worker_pool_queue_delay_seconds_max Time spent in the waiting queue before being processed
worker_pool_queue_delay_seconds_max{pool_name="vert.x-internal-blocking",pool_type="worker"} 0.0
worker_pool_queue_delay_seconds_max{pool_name="vert.x-worker-thread",pool_type="worker"} 0.002507313
# TYPE worker_pool_queue_delay_seconds summary
# HELP worker_pool_queue_delay_seconds Time spent in the waiting queue before being processed
worker_pool_queue_delay_seconds_count{pool_name="vert.x-internal-blocking",pool_type="worker"} 0.0
worker_pool_queue_delay_seconds_sum{pool_name="vert.x-internal-blocking",pool_type="worker"} 0.0
worker_pool_queue_delay_seconds_count{pool_name="vert.x-worker-thread",pool_type="worker"} 6.0
worker_pool_queue_delay_seconds_sum{pool_name="vert.x-worker-thread",pool_type="worker"} 0.005937144
```

いろいろありますが、例えば 404 リクエストの数を見れたりします。

適当に存在しない URL にリクエストを投げてみます。
```shell-session
$ curl -I localhost:8080/sonzaishinai
HTTP/1.1 404 Not Found
```
そうすると以下のように、status="404" のラベルがついたメトリクスが取得できました。これで 404 を返したリクエストの数をカウントできます。
```shell-session
$ curl -s localhost:8080/q/metrics | grep 404
http_server_requests_seconds_count{method="HEAD",outcome="CLIENT_ERROR",status="404",uri="NOT_FOUND"} 1.0
http_server_requests_seconds_sum{method="HEAD",outcome="CLIENT_ERROR",status="404",uri="NOT_FOUND"} 0.055982865
http_server_requests_seconds_max{method="HEAD",outcome="CLIENT_ERROR",status="404",uri="NOT_FOUND"} 0.055982865
```

ちなみに、メトリクスの取得するエンドポイントを変更する場合は application.propeties に `quarkus.http.non-application-root-path` を指定すればよいです。


# 独自のメトリクスを作成する
自分でメトリクスを作成できるようなので、こちらも試してみます。

ということで今回は、[エラトステネスのふるい](https://ja.wikipedia.org/wiki/%E3%82%A8%E3%83%A9%E3%83%88%E3%82%B9%E3%83%86%E3%83%8D%E3%82%B9%E3%81%AE%E7%AF%A9)を使って、特定の数値以下の素数を羅列するアプリケーションを作成して、その中でメトリクスを作成してみようと思います。
以下の quarkus のチュートリアルを参考にしています。
https://ja.quarkus.io/guides/telemetry-micrometer-tutorial


## POJO の作成
まずはレスポンスの形式を POJO で指定します。
limit がクライアントから指定された数字、count が limit より小さい素数の数、primes が limit より小さい素数を羅列する配列です。
```java:src/main/java/com/marcha/prime/PrimeResponsePojo.java
public class PrimeResponsePojo {
    private Integer limit;
    private Integer count;
    private List<Integer> primes;
    // setter getter コンストラクタ省略
}
```

## Rest サービスの作成
次に、Rest サービスを作成します。
```java:src/main/java/com/marcha/PrimeResource.java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;

@Path("/prime")
public class PrimeResource {

    @Inject
    MeterRegistry registry;

    @GET
    @Path("/list")
    @Produces(MediaType.APPLICATION_JSON)
    public PrimeResponsePojo getPrimes(@QueryParam("limit") int limit) {
        if (limit < 2) {
            registry.counter("com.marcha.prime.number", "type", "less-than-two").increment();
            Integer count = 0;
            return new PrimeResponsePojo(limit, count, new ArrayList<Integer>());
        }

        registry.counter("com.marcha.prime.number", "type", "more-than-two-or-equal").increment();

        Timer.Sample sample = Timer.start(registry);
        List<Integer> primes = sieveOfEratosthenes(limit);
        sample.stop(registry.timer("com.marcha.prime.number", "type", "list-seconds"));
        Integer count = primes.size();
        return new PrimeResponsePojo(limit, count, primes);
    }

    private List<Integer> sieveOfEratosthenes(int limit) {
        boolean[] isPrime = new boolean[limit + 1];
        Arrays.fill(isPrime, true);
        isPrime[0] = false;
        isPrime[1] = false;

        for (int i = 2; i * i <= limit; i++) {
            if (isPrime[i]) {
                for (int j = i * i; j <= limit; j += i) {
                    isPrime[j] = false;
                }
            }
        }

        List<Integer> primes = new ArrayList<>();
        for (int i = 2; i <= limit; i++) {
            if (isPrime[i]) {
                primes.add(i);
            }
        }
        return primes;
    }
}
```
エラトステネスのふるいを実装しているだけです。

## ソースコードのポイント
### MeterRegistry インスタンスの作成
まず最初にメトリクスを登録するための `io.micrometer.core.instrument.MeterRegistry` を Inject します。

### カウンターの作成
次に、カウンターを追加しています。カウンターはその名の通り、特定の増加する値を測定するときに利用します。交通量調査バイトの人が押しているカウンターと同じ感じですね。
`registry.counter()` メソッドでカウンターを作成できます。また引数にメトリクスの名前を指定します。
で、 `increment()` メソッドを実行すると、その指定した名前のメトリクスが 1 増加します。
ちなみに `increment(double)` で引数を指定すると、その数分だけ増加します。


では実際に何度か、`curl -s localhost:8080/prime/list?limit=xxx` でリクエストを飛ばしてみます。
その後、もう一度メトリクスの取得をします。
```shell-session
$ curl -s localhost:8080/q/metrics | grep -v "#" |  grep com.marcha.prime.number.total
com_marcha_prime_number_total{type="more-than-two-or-equal"} 2.0
com_marcha_prime_number_total{type="less-than-two"} 1.0
```
このように `registry.counter()` の引数で指定した com_marcha_prime_number という prefix でメトリクスが取得できています。

### タイマーの作成
タイマーは、特定の処理にかかった時間を測定するために利用されます。
`io.micrometer.core.instrument.Timer.start()` メソッドと `io.micrometer.core.instrument.Timer.Sample.stop()` メソッドの間にはさんだ処理にかかった時間を測定します。

これも同様に curl でメトリクスを取得しています。
```shell-session
$ curl -s localhost:8080/q/metrics | grep -v "#" |  grep com.marcha.prime.number.seconds
com_marcha_prime_number_seconds_max{type="list-seconds"} 2.36613E-4
com_marcha_prime_number_seconds_count{type="list-seconds"} 2.0
com_marcha_prime_number_seconds_sum{type="list-seconds"} 2.59494E-4
```

かかった最大時間、測定した回数、かかった合計時間が表示されていますね。

# まとめ
今回は以上になります。
quarkus は何をやるにしても簡単にできる印象があります。
メトリクスを取得できればあとは prometheus や alertmanager を使って障害の監視ができますね！