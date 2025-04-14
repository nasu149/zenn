---
title: "Quarkus と Keycloak を使ったアクセス制御の実装(ロールベース認可)"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus, keycloak]
published: true
---

今回は Quarkus と Keycloak を組み合わせた **ロールベース認可(RBAC)** を試してみたので残します。
**ロールベース認可** とは、その名の通り **特定のロールを持っているユーザのみリクエストを許可する** というものです。
https://ja.quarkus.io/guides/security-keycloak-authorization
次回、**スコープベースの認可**や**Claim ベースの認可**も試します。

---

# 環境
- Oracle Linux 8.10
- quarkus 3.21.0
- quarkus CLI 3.21.0
- Open JDK 17.0.14
- Podman 4.9.4-rhel
- keycloak 23.0
---

# ロールベース認可の設定手順
今回は keycloak は Podman で起動して、quarkus は 開発モードで実行します。

## keycloak の起動
まずは Podman で keycloak の起動をします。今回は version 23 を使います。
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

## keycloak のユーザとロールを作成する

次に、ユーザとそのパスワードの設定、それとロールを作成します。私は以下のようにコナンっぽい感じで作成しました。(デフォルトのロールは Unassign してます。)

| ユーザ   | ロール              |   
|----------|--------------------|
| `Conan`  | `Child`            | 
| `Kir`    | `BlackOrganization`| 
| `Bourbon`| `BlackOrganization`|

#### ユーザはこんな感じ
![](https://storage.googleapis.com/zenn-user-upload/e271dfe4a5d4-20250413.png)
#### ロールはこんな感じ
![](https://storage.googleapis.com/zenn-user-upload/6c6adbeb57a5-20250413.png)


## quarkus アプリケーションのひな形作成
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
## quarkus アプリケーションの設定
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


## quarkus アプリケーションの実装
さて、実装してきます。
`@RolesAllowed` アノテーションを使って、**ユーザーのロール（役割）に基づいてアクセス制御**を行います。

```java:src/main/java/com/marcha/SecureResource.java
package com.marcha;

import io.quarkus.security.identity.SecurityIdentity;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

@Path("/secure")
public class SecureResource {

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
    @Path("/black")
    @RolesAllowed("BlackOrganization")
    @Produces(MediaType.TEXT_PLAIN)
    public String getBlackResource() {
        String name = securityIdentity.getPrincipal().getName();
        return name + " got the black secret data!";
    }
}
```

/secure/normal は誰でもアクセスできますが、/secure/black は、**`@RolesAllowed("BlackOrganization")`** アノテーションを使って、`BlackOrganization` ロールを持つユーザー(黒の組織)だけがアクセスできるようにしています。とっても簡単ですね。


## 動作確認
curl で動作確認をしてみます。
### `/secure/normal` にアクセスする
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
これを使って、もう一度　`/secure/normal` にアクセスします。
```shell-session
$ curl -s -X GET http://localhost:8080/secure/normal -H "Authorization: Bearer "$access_token
conan got the normal data!
```
OK です！今度は名前が表示されています！

### `/secure/black` にアクセスする
こちらは、黒の組織しか**アクセスできない**はずです。conan ユーザでリクエストしてみましょう。
```shell-session
$ curl -v -X GET http://localhost:8080/secure/black -H "Authorization: Bearer "$access_token
...
 HTTP/1.1 403 Forbidden
```
想定通り **403** で返ってきました。OK です。

では、黒の組織の **Bourbon** でログインしてみます。
```bash
$ export access_token=$(\
    curl -X POST http://localhost:8180/realms/quarkus/protocol/openid-connect/token \
    --user quarkus-client:xxxxxxxxxxx \
    -H 'content-type: application/x-www-form-urlencoded' \
    -d 'username=bourbon&password=bourbon&grant_type=password' | jq --raw-output '.access_token' \
 )
```

```shell-session
$ curl -s -X GET http://localhost:8080/secure/black -H "Authorization: Bearer "$access_token
bourbon got the black secret data!
```
OK です！名前も表示されていて、getBlackResource にアクセスできています！

---

# まとめ
今回はここまでにします。
ロールベースの認可のより詳しい情報は、[ドキュメント](https://ja.quarkus.io/guides/security-keycloak-authorization)を参考にしてください。
次回は、スコープを使った認可制御をしてみたいと思います。ではまた！