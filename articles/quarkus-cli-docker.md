---
title: "docker で Quarkus CLI を使えるようにする"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus, docker]
published: true
---
quarkus CLI 便利ですよね。ただ、jbang がうまくインストールできなかったり、バージョンによってはエラー祭りになることが多いと思います。

そこで今回は docker を使って、quakus CLI がインストールされたイメージを作成し、誰でも必ず quakus CLI をインストールできるようしてみます。
# 環境
- Windows 11
- docker 24.0.5
# インストールするバージョン
- jbang 0.125.1 のイメージ (java 11)
- quarkus CLI 3.6.9

# quarkus CLI のインストール方法

## 1. jbang のイメージを pull する
jbang 0.125.1 のイメージを取得します。
```sh
docker pull jbangdev/jbang-action:0.125.1
```
## 2. quarkus CLI がインストールされたイメージをビルドする
quarku CLI 3.6.9 をインストールします。
バージョンを指定したい場合は、以下から特定のバージョンを探して、Dockerfile の4行目の URL を変更してください。
https://repo1.maven.org/maven2/io/quarkus/quarkus-cli/
:::message
quarkus CLI 3.7 以上では [Java 17, 21 のみがサポートされます](https://ja.quarkus.io/blog/java-17/)。
docker hub にある jbang イメージは Java 11 なのでご注意ください。
:::


```Dockerfile:Dockerfile
FROM docker.io/jbangdev/jbang-action:0.125.1

SHELL ["/bin/bash", "-c"]
RUN jbang trust add https://repo1.maven.org/maven2/io/quarkus/quarkus-cli/
RUN jbang app install --name quarkus https://repo1.maven.org/maven2/io/quarkus/quarkus-cli/3.6.9/quarkus-cli-3.6.9-runner.jar

ENTRYPOINT ["/jbang/.jbang/bin/quarkus"]
```

```sh:ビルドコマンド
docker build -t my-quakus-cli:3.6.9 .
```

## 3. alias の登録をする
これで quarkus CLI を利用できるイメージができました。
`docker run` で quarkus CLI を実行するコマンドを quarkus という名前の alias として登録します。

```powershell
# プロファイルスクリプトを開く
notepad $PROFILE

## 以下の内容を書き込む(username 各自指定)
function quarkus() {
docker run --rm -it -v .:/ws  -v C:\Users\<username>\.m2\repository:/root/.m2/repository --workdir=/ws -p 8080:8080 -p 5005:5005 my-quarkus-cli:3.6.9 $args
}
```

# quarkus CLI を試す

## quarkus create app で quarkus AP のテンプレートを作成する
```powershell:テンプレート作成コマンド
$GROUPID = "com.marcha"
$AETIFACTID = "test"
$VERSION = "1.0.0"
 
quarkus create app -P io.quarkus.platform:quarkus-bom:3.6.9  ${GROUPID}:${AETIFACTID}:${VERSION}
```
:::message
`-P` オプションを指定しないと、最新の quarkus BOM を使うようですたぶん。
:::

## extension を追加してみる
```sh
cd ${AETIFACTID}
quarkus extension add quarkus-smallrye-openapi
```
以下の dependency が追加されていれば成功です。
```xml:pom.xml
    <dependency>
      <groupId>io.quarkus</groupId>
      <artifactId>quarkus-smallrye-openapi</artifactId>
    </dependency>
```

## quarkus dev で 開発モードを試す
リスニングアドレスを ip any にしておきます。
```properties:src/main/resources/application.properties
quarkus.http.host=0.0.0.0
```

これで quarkus を動かせます。
```sh:開発モードを開始するコマンド
quarkus dev
```
ブラウザから `localhost:8080/hello` で以下画面が表示されれば OK です。
![](https://storage.googleapis.com/zenn-user-upload/577e13d289ff-20250331.png)
せっかくなので、`localhost:8080/q/swagger-ui` で swagger UI が表示されるかも確認しておきましょう。
![](https://storage.googleapis.com/zenn-user-upload/3b19e4acc1ae-20250331.png)

## quakus build で jar をビルドする
```sh:ビルドコマンド
quarkus build
```
ローカルの Java で作成した jar を実行してみましょう。
```powershell
java -jar .\target\quarkus-app\quarkus-run.jar
```

同様に `loclhost:8080/hello` でアクセスできれば成功です！

# 課題
## 1. コンテナ内に Docker, Podman が入ってないのでコンテナランタイムを使うコマンドを実行できない。
例えば、以下のコマンドを実行できません。
1. quarkus image build
2. quarkus image push
3. quarkus build --native

この課題は結構致命的な気がしています。
これの対処としてコンテナの中に Podman や Docker をインストールする方法があります。
以下に、quarkus CLI についで、Podman もインストールする Dockerfile を記載します。


::::details Podman がインストールされているイメージを作成する方法
### quarkus CLI と Podman が入ったイメージを作成する Dockerfile を作成する。
```Dockerfile:Dockerfile
FROM docker.io/jbangdev/jbang-action:0.125.1

SHELL ["/bin/bash", "-c"]

RUN jbang trust add https://repo1.maven.org/maven2/io/quarkus/quarkus-cli/
RUN jbang app install --name quarkus https://repo1.maven.org/maven2/io/quarkus/quarkus-cli/3.6.9/quarkus-cli-3.6.9-runner.jar

# Podman をインストールする
ENV VERSION_ID 20.04
RUN apt -y update && apt -y upgrade && apt -y install gnupg gnupg1 gnupg2 wget
RUN echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
RUN wget -nv https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/xUbuntu_${VERSION_ID}/Release.key -O- | apt-key add -

RUN apt -y update && apt -y install podman
RUN sed -i 's@#mount_program = "/usr/bin/fuse-overlayfs"@mount_program = "/usr/bin/fuse-overlayfs"@' /etc/containers/storage.conf

ENTRYPOINT ["/jbang/.jbang/bin/quarkus"]
```
### イメージをビルドする
上記の Dockerfile を 以下コマンドでビルドします。
```sh:build コマンド
docker build -t my-quarkus-cli:3.6.9-podman .
```
### quarkus を alias で登録をする
`docker run` で quarkus CLI を実行するコマンドを quarkus という名前の alias として登録します。
コンテナ内で Podman を使うには特権モードが必要なのでご注意ください。

```powershell
# プロファイルスクリプトを開く
notepad $PROFILE

## 以下の内容を書き込む(username 各自指定)
function quarkus() {
docker run --privileged --rm -it -v .:/ws  -v C:\Users\<username>\.m2\repository:/root/.m2/repository --workdir=/ws -p 8080:8080 -p 5005:5005 my-quarkus-cli:3.6.9-podman $args
}
```


### 試しに image build, image push してみる
これで native build や image push もできます。
試しに、native build, image build をして、github のレジストリにイメージ push してみます。
github の token は、[ここ](https://github.com/settings/tokens)で write:package の権限を持ったものを作成してください。
```sh:native image build して github のレジストリに push する
$USER_NAME = "xxx" # github のユーザ名
$GITHUB_TOKEN = "ghp_yyyyyyyyyyyyyyyyyyyyy" # token
quarkus image push --registry=ghcr.io --registry-username=${USER_NAME} --registry-password=${GITHUB_TOKEN}  --also-build --group=${USER_NAME} --native
```
:::message
`docker run` コマンドに `--rm` を付けているのでコンテナはすぐに終了します。
そのため、ローカルの docker には作成したイメージは残らないので注意してください。
残したい場合は、コンテナにログインして操作するなり、マウントするなりしてください。
:::
github の packages タブでイメージが確認できれば OK です！
::::

## 2. 補完が使えない
alias で登録しているだけなので、補完が使えません。
これは、`bash` でコンテナに入って quarkus CLI を使えば OK です。

```sh
docker run --rm -it -v .:/ws  -v C:\Users\<username>\.m2\repository:/root/.m2/repository --workdir=/ws -p 8080:8080 -p 5005:5005 --entrypoint bash my-quarkus-cli:3.6.9
```

上記のように entrypoint を bash に変更し、コンテナに入り、以下のコマンドを実行することで補完を有効にできます。
この後は、コンテナの中で quarkus コマンドを実行してください。
```sh
source <(quarkus completion)
```

# まとめ
ネットワークがちゃんとつながっていて、docker が使える環境であれば誰でも quarkus CLI をインストールできたと思います。それでは楽しい quarkus ライフを！