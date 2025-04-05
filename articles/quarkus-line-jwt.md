---
title: "quarkus で JWT の作成と署名をして Line Messaging API のアクセストークンを取得する"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus]
published: false
---
今回は **quarkus-smallrye-jwt-build** を使って JWT の作成と署名をしてみます。
さらにせっかくなので、それを使って [Line Messaging API](https://developers.line.biz/ja/docs/messaging-api/generate-json-web-token/) のアクセストークンを取得して、Line のメッセージ送信をしてみます。

# 環境
- Oracle Linux 8.10
- quarkus CLI 3.21.0
- Open JDK 17.0.14 

# 概要
0. Line 公式アカウントの作成
1. quarkus の ひな形を作成
2. JWK のキーペアを作成して、公開鍵を Line Developer に登録
3. JWT の作成と署名
4. アクセストークンを得るための POJO, Rest Client を作成
5. Line Messaging API でメッセージを送信するための POJO, Rest Client を作成
6. Rest サービスの作成

# 手順詳細
## 0. Line 公式アカウントの作成
Line messaging API を試すにはまず、以下のドキュメントを見て Line の公式アカウントを作成しておく必要があります。
https://developers.line.biz/ja/docs/messaging-api/getting-started/

また公式アカウントを作成したら、[Line Developers console](https://developers.line.biz/console/) から作成した公式アカウントの[チャネル基本設定] の **チャネルID**, **あなたのユーザーID** を控えておいてください。

## 1. quarkus の ひな形を作成
まずは quarkus AP のひな形を作成します。また **quarkus-smallrye-jwt-build** extension を追加します。
```sh
# 適当に名前を決めて
export GROUPID=com.marcha
export AETIFACTID=line-notify-app
export VERSION=1.0.0
# テンプレを作成する
quarkus create app -P io.quarkus.platform:quarkus-bom:3.21.0 ${GROUPID}:${AETIFACTID}:${VERSION}
cd ${AETIFACTID}
# extension の追加
quarkus extension add quarkus-smallrye-openapi,quarkus-rest-client-jackson, quarkus-smallrye-jwt-build
```

`quarkus dev` で起動できれば OK です。

## 2. JWK のキーペアを作成して、公開鍵を Line Developer に登録
Line のドキュメント内にある以下の図が分かりやすいです。この手順通り進めていきます。
![](https://developers.line.biz/assets/img/channel-access-token-issue-flow.7854f23b.svg)
https://developers.line.biz/ja/docs/messaging-api/generate-json-web-token/

ということで、まずはキーペアを作成します。私は[ここ](https://developers.line.biz/ja/docs/messaging-api/generate-json-web-token/#use-browser)を見て、ブラウザから JWK のキーペアを作成しました。

で、私は秘密鍵を `/etc/secrets/privatekey.key` に保存しました。が、生データをそのまま保存するのはセキュリティ的にまずいので vault で暗号化するなど対策してください。

また公開鍵は、[LINE Developersコンソール](https://developers.line.biz/console/) の ［チャネル基本設定］タブ > [公開鍵を登録する］で登録します。
公開鍵の登録に成功すると、**kid**(=アサーション署名キー) を取得することができるので、それを控えます。

## 3. quarkus-smallrye-jwt-build で JWT の作成と署名
quarkus の[ドキュメント](https://ja.quarkus.io/guides/security-jwt-build)を読みながら進めます。以下のように実装すれば良いでしょう。
```java:src/main/java/com/marcha/token/utils/JwtUtil.java
package com.marcha.token.utils;

import io.smallrye.jwt.algorithm.SignatureAlgorithm;
import io.smallrye.jwt.build.Jwt;
import io.smallrye.jwt.build.JwtClaimsBuilder;

public class JwtUtil {
    private String kid;
    private String channelId;

    public JwtUtil(String kid, String channelId){
        this.kid = kid;
        this.channelId = channelId;
    }
    public String getJwt(){
        JwtClaimsBuilder builder = Jwt.claims();
        String signedJwt = builder.subject(channelId)//.claim("token_exp", 86400)
            .jws().keyId(kid).algorithm(SignatureAlgorithm.RS256).sign();
        return signedJwt;
    }
}
```
### [*JwtUtil.java* の Point]
#### 1. ヘッダーの設定
[ドキュメント](https://developers.line.biz/ja/docs/messaging-api/generate-json-web-token/#header)によるとヘッダーは以下にする必要があるという記載があります。
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "536e453c-aa93-4449-8e90-add2608783c6"
}
```
*JwtUtil.java* では `keyId()` メソッド、`algorithm()` メソッドで指定しています。

#### 2. ペイロードの設定
[ドキュメント](https://developers.line.biz/ja/docs/messaging-api/generate-json-web-token/#payload)によるとペイロードは以下にする必要があるという記載があります。
```json
{
  "iss": "1234567890",
  "sub": "1234567890",
  "aud": "https://api.line.me/",
  "exp": 1559702522,
  "token_exp": 86400
}
```
ペイロードは **quarkus-smallrye-jwt-build** を使っている場合、以下の項目は application.properties や 環境変数に設定すれば quarkus が自動で設定してくれます。
- iss: smallrye.jwt.new-token.issuer
- aud: smallrye.jwt.new-token.audience
- exp: smallrye.jwt.new-token.lifespan

なので .env に以下のように指定しています。
```sh:.env
smallrye_jwt_new-token_issuer=${channel_id}
smallrye_jwt_new-token_audience=https://api.line.me/
```
で、**sub** は環境変数で登録できないっぽいので *JwtUtil.java* では **sub** を `subject()` メソッドで指定しています。

なお、token_exp は[ステートレスチャネルアクセストークン](https://developers.line.biz/ja/reference/messaging-api/#issue-stateless-channel-access-token)を使う場合は不要なのでコメントアウトしています。必要なら `claim()` メソッドで追加すれば良いかと思います。

#### 3. `sign()` メソッドで署名する
最後に引数なしの `sign()` メソッドで署名をしています。引数がない場合、`smallrye_jwt_sign_key_location` という環境変数で指定している場所に、JWK や PEM がないか見に行きます。今回は /etc/secrets/privatekey.key に JWK をおいているので、.env に以下のように指定します。ただセキュリティ的にまずいので、暗号化して保存するなり対策してくださいね。
```sh:.env
smallrye_jwt_sign_key_location=/etc/secrets/privatekey.key
```

## 4. アクセストークンを得るための POJO, Rest Client を作成
### アクセストークンを受け取るレスポンス時の POJO の作成
アクセストークンのレスポンスは Json で返ってくるようなので、POJO を作成するかと思いますが、その場合は [jsonschema2pojo](https://www.jsonschema2pojo.org/) などを使うと便利です。
Line のドキュメントに [レスポンスの例](https://developers.line.biz/ja/reference/messaging-api/#issue-channel-access-token-v2.1-response) が書いてあるので、これを[jsonschema2pojo](https://www.jsonschema2pojo.org/) に貼り付けるだけで POJO ができます。


### Rest Client の作成
Rest Client の[ドキュメント](https://ja.quarkus.io/guides/rest-client#create-the-jakarta-rest-resource)を見て、以下のようなインタフェースを作成すれば良いかと思います。エラーで返ってきた場合の処理は面倒なので省きます。
```java:src/main/java/com/marcha/clients/AccessTokenClient.java
package com.marcha.clients;

import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;
import com.marcha.token.AccessTokenPojo;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.FormParam;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.MediaType;

@Path("/token")
@RegisterRestClient(configKey="access-token-api")
public interface AccessTokenClient {
    @POST
    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
    AccessTokenPojo getAccessToken(@FormParam("grant_type") String grantType, @FormParam("client_assertion_type") String clientAssertionType, @FormParam("client_assertion") String clientAssertion);
}
```
上で作成した POJO(AccessTokenPojo) で受けます。とっても簡単ですね。
あ、Rest 先の URL も .env に書いてます。これで[ステートレスチャネルアクセストークン](https://developers.line.biz/ja/reference/messaging-api/#issue-stateless-channel-access-token)を得ます。
```sh:.env
quarkus_rest-client_access-token-api_url=https://api.line.me/oauth2/v3/
```

## 5. Line Messaging API でメッセージを送信するための POJO, Rest Client を作成
さてアクセストークンが取れたらあとは、Line Messaging API でメッセージの送信をしてみます。ドキュメントは[ここ](https://developers.line.biz/ja/reference/messaging-api/#send-push-message)を見れば良いかと思います。今回は簡単なテキストメッセージを送るだけにとどめますが、絵文字とかスタンプとかいろいろ送れるみたいですね。

### POJO の作成
リクエストもレスポンスも JSON のようなので、どちらも [ドキュメント](https://developers.line.biz/ja/reference/messaging-api/#send-push-message) を参考に [jsonschema2pojo](https://www.jsonschema2pojo.org/) などで POJO を作成すれば良いです。

### Rest Client の作成
以下のようなインタフェースを作成すれば良いかと思います。
```java:src/main/java/com/marcha/clients/SendMessageClient.java
package com.marcha.clients;

import org.eclipse.microprofile.openapi.annotations.parameters.RequestBody;
import org.eclipse.microprofile.rest.client.inject.RegisterRestClient;
import com.marcha.message.request.SendMessagePojo;
import com.marcha.message.response.SentMessagePojo;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.HeaderParam;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.core.MediaType;

@Path("/push")
@RegisterRestClient(configKey="send-message-api")
public interface SendMessageClient {
    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    SentMessagePojo sendMessage(@HeaderParam("Authorization") String accessTokenWithBearer, @RequestBody SendMessagePojo sendMessagePojo);
}
```
リクエストボディーの POJO を SendMessagePojo、レスポンスボディの POJO を SentMessagePojo としています。(我ながら分かりづらい名前にしてしまったと後悔しています、、、)
また、プッシュメッセージの URL を .env にしています。
```sh:.env
quarkus_rest-client_send-message-api_url=https://api.line.me/v2/bot/message/
```

## 6. Rest サービスの作成
ここまでくれば完成です。ほとんどコーディングせず完成してしまいました。すごい！
Rest サービスは手順1.~5.で作成したものを呼び出すだけです。以下のような感じです。
```java:src/main/java/com/marcha/NotifyMessage.java
package com.marcha;

import java.util.Arrays;
import java.util.List;

import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.eclipse.microprofile.rest.client.inject.RestClient;

import com.marcha.clients.AccessTokenClient;
import com.marcha.clients.SendMessageClient;
import com.marcha.message.request.Message;
import com.marcha.message.request.SendMessagePojo;
import com.marcha.message.response.SentMessagePojo;
import com.marcha.token.AccessTokenPojo;
import com.marcha.token.utils.JwtUtil;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

@Path("/notify-message")
public class NotifyMessage {

    @ConfigProperty(name = "notify.message.jwt.private.key.id")
    String kid;
    @ConfigProperty(name = "notify.message.jwt.channel.id")
    String channelId;
    @ConfigProperty(name = "notify.message.client.id")
    String clientId;

    @Inject @RestClient
    AccessTokenClient accessTokenClient;
    @Inject @RestClient
    SendMessageClient sendMessageClient;

    @GET @Path("{message}")
    @Produces(MediaType.TEXT_PLAIN)
    public String notifyMessage(String message) {

        // get an access token
        AccessTokenPojo accessTokenPojo = createAccessTokenPojo();
        String accessToken = accessTokenPojo.getAccessToken();  
        String accessTokenWithBearer = "Bearer " + accessToken;

        // create the messages
        SendMessagePojo sendMessagePojo = creatSendMessagePojo(message);

        // Rest to Line API to notify messages.
        SentMessagePojo sentMessagePojo = sendMessageClient.sendMessage(accessTokenWithBearer, sendMessagePojo);
        logger.debug("sentMessagePojo = " + sentMessagePojo);

        return "OK! I sent the word '" + message + "'!";
    }

    private AccessTokenPojo createAccessTokenPojo(){
        // create a JWT
        JwtUtil jwtUtil = new JwtUtil(kid, channelId);
        String signedJwt = jwtUtil.getJwt();

        // Rest to Line API to get AccessToken
        String grantType = "client_credentials";
        String clientAssertionType = "urn:ietf:params:oauth:client-assertion-type:jwt-bearer";
        AccessTokenPojo accessTokenPojo = accessTokenClient.getAccessToken(grantType, clientAssertionType, signedJwt);

        return accessTokenPojo;
    }

    private SendMessagePojo creatSendMessagePojo(String message){
        // create a message to send to Line users
        Message message1 = new Message("text", message);
        List<Message> messages = Arrays.asList(message1);
        SendMessagePojo sendMessagePojo = new SendMessagePojo(clientId, messages);
        return sendMessagePojo;
    }
}
```
手順 0. で取得した **チャネルID**, **あなたのユーザーID** をそれぞれ .env に `notify_message_jwt_channel_id`, `notify_message_client_id` として登録してあります。
また手順 2. で取得した **kid** は、`notify_message_jwt_private_key_id` として .env に登録してあります。
```sh:.env
notify_message_jwt_channel_id=0000
notify_message_client_id=yyyy
notify_message_jwt_private_key_id=xxxx
```

# まとめ
説明しながら書くと長いですが、ほとんどコーディングはしてないです。
こんなに簡単にかける quarkus はやはり素晴らしいです！
まだまだ試してみたいこと(ex. 秘密鍵を vault で暗号化したり)はたくさんあるので、また時間を見つけて試してみます。今回は以上です！