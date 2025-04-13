---
title: "Quarkus と Keycloak を使ったアクセス制御の実装(スコープベースの認可)"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus, keycloak]
published: false
---
# スコープ制御を使った認可

**スコープ**（`scope`）は、OAuth2 の一部であり、ユーザーやクライアントが何を実行できるかを指定します。Keycloak などの認可サーバーは、JWT トークンに `scope` クレームを含め、クライアントに与えた権限を管理します。

### 例：`user.read` スコープを持つユーザーだけがアクセスできる API

```java
package org.acme;

import io.quarkus.security.identity.SecurityIdentity;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.ForbiddenException;

@Path("/secure")
public class ScopedResource {

    @Inject
    SecurityIdentity identity;

    @GET
    @Path("/scoped")
    @Produces(MediaType.TEXT_PLAIN)
    public String scopedHello() {
        if (!hasScope("user.read")) {
            throw new ForbiddenException("Missing required scope: user.read");
        }
        return "You have the required scope!";
    }

    private boolean hasScope(String scope) {
        return identity.getAttributes()
                       .getOrDefault("scope", "")
                       .toString()
                       .contains(scope);
    }
}
```

ここでは、`SecurityIdentity` を使って、JWT トークン内の `scope` クレームに `user.read` が含まれていることを確認しています。

---

## ロールとスコープの組み合わせでアクセス制御

ロールとスコープを**AND 条件**で組み合わせて、アクセス制御をより細かく設定できます。

例えば、`admin` ロールを持ち、かつ `data.write` スコープを持つユーザーのみがアクセスできる API を作成します。

```java
package org.acme;

import io.quarkus.security.identity.SecurityIdentity;
import jakarta.annotation.security.RolesAllowed;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.ForbiddenException;

@Path("/secure")
public class CombinedSecurityResource {

    @Inject
    SecurityIdentity identity;

    @GET
    @Path("/admin-write")
    @RolesAllowed("admin")
    @Produces(MediaType.TEXT_PLAIN)
    public String adminWithWriteScope() {
        // ロールチェックは Quarkus が自動でやってくれる

        // スコープチェックは自前で実装
        if (!hasScope("data.write")) {
            throw new ForbiddenException("You must have the 'data.write' scope");
        }

        return "You are an admin AND you have the 'data.write' scope!";
    }

    private boolean hasScope(String scope) {
        return identity.getAttributes()
                       .getOrDefault("scope", "")
                       .toString()
                       .contains(scope);
    }
}
```

このコードでは、`admin` ロールと `data.write` スコープの両方を持つユーザーだけが `/secure/admin-write` にアクセスできるようにしています。

---

## Claim ベースの認可

**Claim ベースの認可**では、JWT トークンに含まれる任意のクレーム（属性情報）を使ってアクセス制御を行います。これにより、ロールやスコープだけではなく、ユーザーの属性（例えば `tenant_id` や `email`）を基にした認可が可能になります。

### 例：`tenant_id = "companyA"` のユーザーだけがアクセスできる API

```java
@Inject
SecurityIdentity identity;

@GET
@Path("/multi-tenant")
public String accessForTenant() {
    String tenantId = identity.getPrincipal().getAttribute("tenant_id");
    if (!"companyA".equals(tenantId)) {
        throw new ForbiddenException("Access denied for tenant: " + tenantId);
    }
    return "Welcome, companyA user!";
}
```

このコードでは、JWT トークン内の `tenant_id` クレームを使って、特定のテナントに属するユーザーだけがアクセスできるようにしています。

---

## まとめ

- **ロールベースの認可**（`@RolesAllowed`）は、ユーザーの役割に基づいてアクセスを制限します。
- **スコープ制御**は、アクセストークンに含まれる権限情報に基づいてアクセスを制限します。
- **ロールとスコープを組み合わせる**ことで、より細かいアクセス制御が可能です。
- **Claim ベースの認可**を使えば、ユーザー属性（`tenant_id` など）を基にしたアクセス制御ができます。

これらの認可方法を組み合わせることで、柔軟で強力なアクセス制御を実現できます。Keycloak や Quarkus を使用した認証・認可の実装において、必要に応じて適切な方法を選んでください。

---

### 参考リンク

- [Quarkus 公式ドキュメント](https://quarkus.io/guides/)
- [Keycloak 公式ドキュメント](https://www.keycloak.org/documentation)

---
