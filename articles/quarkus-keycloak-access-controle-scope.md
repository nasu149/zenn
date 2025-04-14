---
title: "Quarkus と Keycloak を使ったアクセス制御の実装(スコープベース認可)"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus, keycloak]
published: true
---
[前回](https://zenn.dev/marcha/articles/quarkus-keycloak-access-control) Quarkus と Keycloak を組み合わせた **ロールベース認可** を試してみました。

https://zenn.dev/marcha/articles/quarkus-keycloak-access-control

今回は Quarkus と Keycloak を組み合わせた **スコープベースの認可** を試してみたので残します。
**スコープベースの認可** とは、アクセストークンに **特定のスコープを持っている場合のみリクエストを許可する** というものです。

**スコープ**（`Client scope`）は、OAuth2 の一部であり、ユーザーやクライアントが何を実行できるかを指定します。Keycloak などの認可サーバーは、JWT トークンに `scope` クレームを含め、クライアントに与えた権限を管理します。

Quarkus のドキュメントだと、以下が参考になります。
https://ja.quarkus.io/guides/security-authorize-web-endpoints-reference

といっても、Keycloak の設定さえできていれば、Quarkus の実装はとても簡単で、**REST サービスに `@PermissionsAllowed` アノテーションを付けるだけ** です。
Quarkus 側の実装でいつもと違うところはそれだけです。(設定ファイルに Keycloak の情報を書く必要はありますが、、)

そのため、Keycloak に詳しい方は、[quarkus アプリケーションの設定](#quarkus-アプリケーションの実装) 以降を読めば十分かと思います。

---

# 環境
- Oracle Linux 8.10
- quarkus 3.21.0
- quarkus CLI 3.21.0
- Open JDK 17.0.14
- Podman 4.9.4-rhel
- keycloak 23.0
---

# スコープベース認可の設定手順
[前回](https://zenn.dev/marcha/articles/quarkus-keycloak-access-control)とほぼ同じ手順になります。
**ロールベース認可** の場合は、**`@RolesAllowed`** アノテーションを利用して resource を保護しましたが、**スコープベース認可** の場合は **`@PermissionsAllowed`** アノテーションを利用します。
それぞれのアノテーションの引数に、ロール名を指定するのか、スコープ名を指定するかだけの違いです。とっても簡単ですね。

さて前回と被るところも多いですが、以下に手順を記載します。

## Keycloak の起動
まずは Podman で Keycloak の起動をします。今回は version 23 を使います。
```bash
podman run -p 8180:8080 --env KEYCLOAK_ADMIN=admin --env KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:23.0 start-dev
```
この後 quarkus を 8080 ポートで起動するので、8180 にしておきます。

## Keycloak の realm, Client を作成する
まずは `localhost:8180` で接続し `admin/admin` で **管理コンソール** に入って、必要な設定をしてきます。

ということでまずは、Realm と Client を作成します。
今回は、**quarkus** という Realm と、**quarkus-client** という名前の Client を作成しました。
で、Client は以下のように **Client authentication** と **Authorization** を On にしてください。他はデフォルトでよいです。
![](https://storage.googleapis.com/zenn-user-upload/a6491d1b62d9-20250414.png)

またあとで、Client Secret は使うので、控えておいてください。

## keycloak のユーザを作成する

次に、ユーザとそのパスワードの設定、それとロールを作成します。私は前回、**conan** ユーザを作っているので、そいつを使います。パスワードも **conan** です。

| ユーザ   | ロール              |   
|----------|--------------------|
| `Conan`  | `Child`            | 

## Keycloak の Client scope を作成する
今回のメインである Client scope の作成をします。

こちらは、**`Client scopes > Create client scope `** で作成できます。

今回は **read** という Client scope を作成しました。雰囲気的には、「特定のリソースを読み込む権限がある」みたいなイメージです。
で、設容は以下にしました。Type は **Optional** にして、**デフォルトでは付与されない** 設定です。
![](https://storage.googleapis.com/zenn-user-upload/e71905f8a06f-20250414.png)


次に、この作成した Client Scope を Client に割り当てます。

こちらは **`Client > quarkus-client > Client scopes > Add client scope`** で割り当て画面を表示して、先に作成した **read** scope をチェックして、Add ボタンを押下します。
この時、**`Optional`** で Add してください。
#### quarkus-client の画面
![](https://storage.googleapis.com/zenn-user-upload/0c71050b0460-20250414.png)
#### 割り当て画面
![](https://storage.googleapis.com/zenn-user-upload/9314a4944a2c-20250414.png)


ここまできたら Keycloak の設定はひとまず OK です。





## Quarkus アプリケーションのひな形作成
ここからは quarkus の設定をしていきます。
とりあえずいつもの通り、ひな形の作成をします。
```bash
export GROUPID=com.marcha
export AETIFACTID=access-control
export VERSION=1.0.0
quarkus create app -P io.quarkus.platform:quarkus-bom:3.21.0 ${GROUPID}:${AETIFACTID}:${VERSION}
```
次に、**oidc extension** を追加します。

```bash
cd ${AETIFACTID}
quarkus extension add oidc,rest-jackson
```
## Quarkus アプリケーションの設定
実装する前に、quarkus の設定ファイルに、認可サーバとして keycloak を利用するという設定をする必要があります。

```properties:src/main/resources/application.properties
quarkus.oidc.auth-server-url=http://localhost:8180/realms/quarkus
quarkus.oidc.client-id=quarkus-client
quarkus.oidc.credentials.secret=xxxxxxxxxxxxx
quarkus.oidc.tls.verification=none
quarkus.keycloak.devservices.enabled=false
```
**quarkus.oidc.credentials.secret** は、管理コンソールから `clients > quarkus-client > Credentials > Client Secret` で確認できます。
また、**quarkus.keycloak.devservices.enabled=false** を設定しないと、quarkus がdocker を使って keycloak コンテナを立ち上げてくれちゃいます。私の環境は docker が入っていないので **false** にしています。


## Quarkus アプリケーションの実装

さて、実装してきます。といっても、前回とほぼ同じです。
前回は、`@RolesAllowed` アノテーションを使って、**ロール（役割）に基づいてアクセス制御**を行いましたが、
今回は、**`@PermissionsAllowed`** を使って、**スコープ（権限）に基づいてアクセス制御**を行います。

```java:src/main/java/com/marcha/SecureResource.java
package com.marcha;

import io.quarkus.security.PermissionsAllowed;
import io.quarkus.security.identity.SecurityIdentity;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

@Path("/scope")
public class ScopeResource {

    @Inject
    SecurityIdentity securityIdentity;

    @GET
    @Path("/normal")
    @Produces(MediaType.TEXT_PLAIN)
    public String getNormalResource() {
        String name = securityIdentity.getPrincipal().getName();
        return name + " got the normal data!";
    }

    @GET
    @Path("/read")
    @PermissionsAllowed("read")
    @Produces(MediaType.TEXT_PLAIN)
    public String getResourceRequireScope() {
        String name = securityIdentity.getPrincipal().getName();
        return name + " read the data!";
    }
}
```

/scope/normal は誰でもアクセスできますが、/scope/read は、**`@PermissionsAllowed("read")`** アノテーションを使って、`read` scope を持つリクエストだけ許可するようにしています。とっても簡単ですね。


## 動作確認
curl で動作確認をしてみます。

### `/scope/normal` にアクセスする
```shell-session
$ curl -s -X GET http://localhost:8080/secure/normal
 got the normal data!
```
ログインしてないので、名前は表示されませんが、**アクセスできました**。

次は、**Conan** ユーザとしてログインして、**アクセストークン**を取得します。
```bash
export access_token=$(\
    curl -X POST http://localhost:8180/realms/quarkus/protocol/openid-connect/token \
    --user quarkus-client:xxxxxxxxxxx \
    -H 'content-type: application/x-www-form-urlencoded' \
    -d 'username=conan&password=conan&grant_type=password' | jq --raw-output '.access_token' \
 )
```
これを使って、もう一度　`/scope/normal` にアクセスします。
```shell-session
$ curl -s -X GET http://localhost:8080/secure/normal -H "Authorization: Bearer "$access_token
conan got the normal data!
```
OK です！今度は名前が表示されています！

### `/scope/read` にアクセスする
ここからが本題です。
こちらは、read scope をもっていないと **アクセスできない** はずです。
まずは read scope を要求せずに(先ほど取得したアクセストークンで)、リクエストしてみます。
```shell-session
$ curl -v -X GET http://localhost:8080/scope/read -H "Authorization: Bearer "$access_token
...
 HTTP/1.1 403 Forbidden
```
想定通り **403** で返ってきました。OK です。

この時の アクセストークン を jwt.io(https://jwt.io/) などで中身を見てみます。
すると、scope に **`read`** はありません。
```json:access token 抜粋
{
  "exp": 1744623967,
  ...
  "realm_access": {
    "roles": [
      "Child"
    ]
  },
  "scope": "email profile",
  ...
}
```
scope は、**email profile** だけですね。

では、**read** scope を要求してみます。
```bash
$ export access_token=$(\
    curl -X POST http://localhost:8180/realms/quarkus/protocol/openid-connect/token \
    --user quarkus-client:xxxxxxxxxxx \
    -H 'content-type: application/x-www-form-urlencoded' \
    -d 'username=bourbon&password=bourbon&grant_type=password&scope=read' | jq --raw-output '.access_token' \
 )
```
`-d` オプションの最後に **`scope=read`** として、read scope を要求しています。
これで、アクセストークンに read scope がつきます。

```shell-session
$ curl -s -X GET http://localhost:8080/scope/read -H "Authorization: Bearer "$access_token
conan read the data!
```
OK です！名前も表示されていて、getResourceRequireScope にアクセスできています！

最後に、この時のアクセストークンを jwt.io(https://jwt.io/) で中身を見てみます。
すると、scope に **`read`** があります。
```json:access token 抜粋
{
  "exp": 1744624627,
  ...
  "realm_access": {
    "roles": [
      "Child"
    ]
  },
  "scope": "email profile read",
  ...
}
```
scope は、**email profile read** で、**read** がついていますね！

---

# まとめ
今回はここまでにします。
ロールベース認可と組み合わせれば、より細かいアクセス制御ができそうですよね。
スコープベースの認可のより詳しい情報は、[ドキュメント](https://ja.quarkus.io/guides/security-authorize-web-endpoints-reference)を参考にしてください。
次回は、カスタムクレームを使った認可制御をしてみたいと思います。ではまた！