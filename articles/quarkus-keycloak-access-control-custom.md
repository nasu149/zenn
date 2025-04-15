---
title: "Quarkus と Keycloak を使ったカスタムクレームによる認可の実装"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus, keycloak]
published: true
---
[前回]()、[前々回](https://zenn.dev/marcha/articles/quarkus-keycloak-access-control)で、ロールベースの認可(アクセス制御)と、スコープベースの認可(アクセス制御)を試しました。
今回は、アクセストークンに **独自のクレーム** を定義して、それを使ったアクセス制御を試します。

例えば、ユーザの職業を表す **occupation** というクレームをアクセストークンに付けて、これが **student** の人だけアクセス可能にする みたい感じです。

これを実現する場合、quarkus の extension `keycloak-authorization` を使って Keycloak に [**Regex-Based Policy**](https://www.keycloak.org/docs/23.0.7/authorization_services/index.html#_policy_regex) の設定すれば、簡単にできます。
ただ今回はせっかくなので、これを使わずに Quarkus の実装でやってみます。

Quarkus の実装でやれば、アクセス制御の条件を独自にコーディングすることができます。

さて、**ロールベースのアクセス制御** や、**スコープベースのアクセス制御** の場合は、**アノテーションを使えば簡単** にできましたが、特定のクレームによってアクセス制御をするためのアノテーションはないので、*`io.quarkus.vertx.http.runtime.security.HttpSecurityPolicy`* を implements して、**独自の Policy を作成** し、`@AuthorizationPolicy` アノテーションにその作成した Policy を指定して、アクセス制御をします。

以下が参考になります。

https://ja.quarkus.io/guides/security-authorize-web-endpoints-reference#custom-http-security-policy

上のドキュメントを参考に以下に手順を残します。

# 環境
- Oracle Linux 8.10
- quarkus 3.21.0
- quarkus CLI 3.21.0
- Open JDK 17.0.14
- Podman 4.9.4-rhel
- keycloak 23.0

# カスタムクレームによるアクセス制御の設定手順

**ロールベース認可** の場合は、[`@RolesAllowed` アノテーション](https://zenn.dev/marcha/articles/quarkus-keycloak-access-control)、**スコープベース認可** の場合は [`@PermissionsAllowed` アノテーション](https://zenn.dev/marcha/articles/quarkus-keycloak-access-controle-scope) を利用しました。

今回は、`@AuthorizationPolicy` アノテーションを利用します。こちらは一般的な認可判断に利用できます。つまり、カスタムクレームに限らず、自分で **独自の認可判断ロジックを作成** する場合に利用します。

その認可判断のロジックは、*`io.quarkus.vertx.http.runtime.security.HttpSecurityPolicy`* を implements したクラス内に記述します。

## Quarkus によるクレームによるアクセス制御の例
[前回](https://zenn.dev/marcha/articles/quarkus-keycloak-access-control)、Bourbon と Kir という名探偵コナンで黒の組織のスパイとして潜入しているユーザを定義しました。

2人とも、role は、**BlackOrganization(黒の組織)** ですが、本当の顔(RealFace) は、公安と CIA です。
そのため、アクセストークンに **RealFaceClaim** という新しいクレームを定義して(定義方法は[後述](#補足keycloak-の設定)、**Bourbon には Zero**、**Kir には CIA** という値を入れたとします。
※ 公安警察は、ゼロ科ともいうので。

その前提で、RealFace が CIA の人しかアクセスできない、API を作成してみます。

以下のような感じで実装します。

```java:src/main/java/com/marcha/CustomNamedhttpSecPolicy.java
package com.marcha;

import io.quarkus.security.identity.SecurityIdentity;
import io.quarkus.vertx.http.runtime.security.HttpSecurityPolicy;
import io.smallrye.jwt.auth.principal.JWTCallerPrincipal;
import io.smallrye.mutiny.Uni;
import io.vertx.ext.web.RoutingContext;
import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class CustomNamedhttpSecPolicy implements HttpSecurityPolicy{

    @Override
    public String name() {
        return "custom";
    }

    @Override
    public Uni<CheckResult> checkPermission(RoutingContext request, Uni<SecurityIdentity> identityUni,
            AuthorizationRequestContext requestContext) {
        Uni<CheckResult> isPermission = identityUni.onItem().transform(securityIdentity -> {
            JWTCallerPrincipal principal = (JWTCallerPrincipal)securityIdentity.getPrincipal();
            String realFaceClaim = principal.getClaim("RealFaceClaim");

            if (realFaceClaim.equals("CIA")) {
                return CheckResult.PERMIT;
            } else {
                return CheckResult.DENY;
            }
        });
        return isPermission;
    }   
}
```
*`io.quarkus.vertx.http.runtime.security.HttpSecurityPolicy`* を implements して、`checkPermission()` メソッドに、どのような条件の時にアクセス許可するか、ないしはどのような条件の時にアクセス拒否するかを記述します。

引数で受け取れる、`SecurityIdentity` 型は、Quarkus では、`JWTCallerPrincipal` 型で渡されるので、これにキャストしています。
そうすると、アクセストークンのすべてのクレームが取得できます。

これを使って、**RealFaceClaim** が **CIA** の場合はアクセス許可、それ以外の場合はアクセス拒否するという感じにしています。

で、`name()` メソッドでこの Policy の名前を定義しています。
Rest サービスに、 `@AuthorizationPolicy` アノテーションを付けるのですが、このアノテーションの引数に Policy の名前を指定するわけです。
以下のような感じです。

```java:src/main/java/com/marcha/CustomClaimResource.java
package com.marcha;

import io.quarkus.security.identity.SecurityIdentity;
import io.quarkus.vertx.http.security.AuthorizationPolicy;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;

@Path("/claim")
public class CustomClaimResource {
    
    @Inject
    SecurityIdentity securityIdentity;

    @GET
    @Path("/custom")
    @AuthorizationPolicy(name = "custom")
    @Produces(MediaType.TEXT_PLAIN)
    public String getResourceRequireCIAClaim() {
        String name = securityIdentity.getPrincipal().getName();
        return name + " read the data!";
    }
}
```

## 動作確認
上記のソースコードで `quarkus dev` などで起動して、curl で動作確認をしてみます。
(Quarkus の設定は、[前回の記事](https://zenn.dev/marcha/articles/quarkus-keycloak-access-control#quarkus-%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AE%E8%A8%AD%E5%AE%9A)を見てください)

### `/claim/custom` にアクセスする
このリソースは **RealFaceClaim というクレームが CIA でないとアクセスできない** はずです。

試しに、**RealFaceClaim が Zero** である Bourbon でログインして試します。

#### アクセストークンの取得
```bash
export access_token=$(\
    curl -X POST http://localhost:8180/realms/quarkus/protocol/openid-connect/token \
    --user quarkus-client:xxxxxxxxxxx \
    -H 'content-type: application/x-www-form-urlencoded' \
    -d 'username=bourbon&password=bourbon&grant_type=password' | jq --raw-output '.access_token' \
 )
```
取得したアクセストークンを [jwt.io](https://jwt.io/) などで中身を見てみます。
```json:access token 抜粋
{
  ...
  "scope": "email profile",
  ...
  "preferred_username": "bourbon",
  ...
  "RealFaceClaim": "Zero"
}
```
このように **RealFaceClaim が Zero** になっています。

そのため、curl してみると 403 が返されます。
```shell-session
$ curl -v -X GET http://localhost:8080/claim/custom -H "Authorization: Bearer "$access_token
...
 HTTP/1.1 403 Forbidden
```

#### Kir でアクセストークンを取得
次に、**RealFaceClaim が CIA** である Kir でログインします。
```bash
export access_token=$(\
    curl -X POST http://localhost:8180/realms/quarkus/protocol/openid-connect/token \
    --user quarkus-client:xxxxxxxxxxx \
    -H 'content-type: application/x-www-form-urlencoded' \
    -d 'username=kir&password=kir&grant_type=password' | jq --raw-output '.access_token' \
 )
```
取得したアクセストークンを [jwt.io](https://jwt.io/) などで中身を見てみます。
```json:access token 抜粋
{
  ...
  "scope": "email profile",
  ...
  "preferred_username": "kir",
  ...
  "RealFaceClaim": "CIA"
}
```
このように **RealFaceClaim が CIA** になっています。

そのため、curl してみるとアクセスができるわけです。
```shell-session
$ curl -s -X GET http://localhost:8080/claim/custom -H "Authorization: Bearer "$access_token
kir read the data!
```
---












# [補足]Keycloak の設定
Keycloak にカスタムクレームを設定する方法を補足します。

カスタムクレームを付けるには、ユーザに属性(Attributes) を付けて、特定のスコープ(以下では profile scope)に [User Attribute Type のマッパー](https://www.keycloak.org/docs/23.0.7/server_admin/#_ldap_mappers)を追加するだけです。

簡単に手順を残します。

## Keycloak の起動
まずは Podman で Keycloak の起動をします。今回は version 23 を使います。
```bash
podman run -p 8180:8080 --env KEYCLOAK_ADMIN=admin --env KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:23.0 start-dev
```
Quarkus を 8080 ポートで起動するので、8180 にしておきます。

## Keycloak の realm, Client を作成する
まずは `localhost:8180` で接続し `admin/admin` で **管理コンソール** に入って、必要な設定をしてきます。

ということでまずは、Realm と Client を作成します。
今回は、**quarkus** という Realm と、**quarkus-client** という名前の Client を作成しました。
で、Client は以下のように **Client authentication** と **Authorization** を On にしてください。他はデフォルトでよいです。
![](https://storage.googleapis.com/zenn-user-upload/a6491d1b62d9-20250414.png)

またあとで、Client Secret は使うので、控えておいてください。

## keycloak のユーザを作成する

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



## 各ユーザに属性を付与する
ここで **RealFace** という属性を付与してみます。この次の説で、これをクレームに付与するわけです。

こちらは、**`Users > [ユーザ名] > Attributes > Add an attribute `** で作成できます。
![](https://storage.googleapis.com/zenn-user-upload/84a021e0c4b0-20250415.png)

今回は **RealFace** という属性を付与します。Bourbon には Zero、Kir には CIA です。
![](https://storage.googleapis.com/zenn-user-upload/d068af65dce2-20250415.png)
![](https://storage.googleapis.com/zenn-user-upload/2241673dcf50-20250415.png)

## Keycloak の profile scope に Mapper を追加する
すでに用意されている profile というスコープに RealFaceClaim を関連付けます。


Mapper は、Scope に付与するので、
こちらは、**`Client scopes > profile > Mappers > Add mapper > By configuration > User Attribute`** で作成できます。
![](https://storage.googleapis.com/zenn-user-upload/19423da36c80-20250416.png)
![](https://storage.googleapis.com/zenn-user-upload/ec57cb318f35-20250416.png)

これで、**profile scope** がついていて、**RealFace attribute** をもつユーザでログインすれば、**RealFaceClaim** クレームがアクセストークンに付与されます。
なお、**profile scope** は **デフォルトで付与される** スコープです。

ここまできたら Keycloak の設定はひとまず OK です。







---

# まとめ
今回は、カスタムクレームでアクセス制御をしてみました。
独自に Policy を定義できるので、カスタムクレーム以外にもなんでもできそうですね。

今回はこのあたりで！
