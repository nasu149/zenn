---
title: "quarkus CLI で作成したイメージを GitHub のイメージレジストリに push する"
emoji: "🎃"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [quarkus]
published: true
---
今回は quarkus CLI を使って、quarkus で作成した AP を含んだイメージを作成し、さらにそのイメージを GitHub のイメージレジストリに push する方法を説明します。

# 環境
- Oracle Linux 8.10
- Podman 4.9.4-rhel
- quarkus CLI 3.21.0
- Open JDK 17.0.14 

# 1. quarkus AP の作成
まずは、quarkus AP を作成します。
```sh
# 適当に名前を決めて
export GROUPID=com.marcha
export AETIFACTID=quarkus-app
export VERSION=1.0.0
# テンプレを作成する
quarkus create app -P io.quarkus.platform:quarkus-bom:3.21.0 ${GROUPID}:${AETIFACTID}:${VERSION}
```
開発モードで実行ができて「Hello from Quarkus REST」が表示できれば OK です。
```sh
cd ${AETIFACTID}
quarkus dev
```

# 2. イメージをビルドする
開発モードを終了して、イメージ作成します。イメージの作成コマンドは `quarkus image build` です。
```shell-session
$ quarkus image build --group=quarkus --name=${AETIFACTID} --tag=${VERSION}
```
実際にイメージが作成されたことを確認してみます。
```shell-session
$ podman images ${AETIFACTID}
REPOSITORY                     TAG         IMAGE ID      CREATED         SIZE
localhost/quarkus/quarkus-app  1.0.0       3054073df3a8  34 seconds ago  421 MB 
```

ちなみに上記イメージは、以下の Dockerfile から作成されます。
```Dockerfile:src/main/docker/Dockerfile.jvm
FROM registry.access.redhat.com/ubi9/openjdk-17:1.21

ENV LANGUAGE='en_US:en'

COPY --chown=185 target/quarkus-app/lib/ /deployments/lib/
COPY --chown=185 target/quarkus-app/*.jar /deployments/
COPY --chown=185 target/quarkus-app/app/ /deployments/app/
COPY --chown=185 target/quarkus-app/quarkus/ /deployments/quarkus/

EXPOSE 8080
USER 185
ENV JAVA_OPTS_APPEND="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
ENV JAVA_APP_JAR="/deployments/quarkus-run.jar"

ENTRYPOINT [ "/opt/jboss/container/java/run/run-java.sh" ]
```
「registry.access.redhat.com/ubi9/openjdk-17:1.21」という Java のイメージに quarkus AP の jar を入れます。
せっかくなので実行してみます。
```shell-session
$ podman run --rm localhost/quarkus/quarkus-app:1.0.0
INFO exec -a "java" java -XX:MaxRAMPercentage=80.0 -XX:+UseParallelGC -XX:MinHeapFreeRatio=10 -XX:MaxHeapFreeRatio=20 -XX:GCTimeRatio=4 -XX:AdaptiveSizePolicyWeight=90 -XX:+ExitOnOutOfMemoryError -Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager -cp "." -jar /deployments/quarkus-run.jar 
INFO running in /deployments
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2025-04-02 10:27:50,643 INFO  [io.quarkus] (main) quarkus-app 1.0.0 on JVM (powered by Quarkus 3.21.0) started in 1.382s. Listening on: http://0.0.0.0:8080
```
実行できました！

では次は、ネイティブ実行可能ファイルをビルドして、それを含むイメージを作成します。先ほどのコマンドに `--native` オプションを付けるだけです。
ただしこちらはこの後 GitHub のイメージレジストリに push するので、レジストリ名を github レジストリの名前にしておきます。
```sh
export GITHUB_USERNAME=yyyyyyy
quarkus image build --native --registry=ghcr.io --group=${GITHUB_USERNAME} --name=${AETIFACTID} --tag=${VERSION}-native
```
同様に、イメージができたことを確認します。
```shell-session
$ podman images ${AETIFACTID}
REPOSITORY                     TAG           IMAGE ID      CREATED             SIZE
ghcr.io/yyyyyyy/quarkus-app    1.0.0-native  7946627ca24e  About a minute ago  145 MB
localhost/quarkus/quarkus-app  1.0.0         3054073df3a8  5 hours ago         421 MB
```
native の方は、Java が入っていないのでめっちゃ軽いですね。
ちなみにこのイメージは以下の Dockerfile から作成されます。
```Dockerfile:src/main/docker/Dockerfile.native
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.10
WORKDIR /work/
RUN chown 1001 /work \
    && chmod "g+rwX" /work \
    && chown 1001:root /work
COPY --chown=1001:root --chmod=0755 target/*-runner /work/application

EXPOSE 8080
USER 1001

ENTRYPOINT ["./application", "-Dquarkus.http.host=0.0.0.0"]
```

# 3. GitHub のイメージレジストリに push する
### a. GitHub で token を発行する
まずは以下のサイトで、write:package の権限を持った token を発行し、token を控えてください。
https://github.com/settings/tokens

### b. quarkus CLI で push する
次に、quarkus CLI で 2. で作成した native イメージを push します。image を push するコマンドは `quarkus image push` です。この時に a. で作成した token が必要です。
```sh
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxx
quarkus image push --registry=ghcr.io --registry-username=${GITHUB_USERNAME} --registry-password=${GITHUB_TOKEN} --group=${GITHUB_USERNAME} --name=${AETIFACTID} --tag=${VERSION}-native
```
`--alse-build` オプションを付けることで image build も一気にやってくれます。`quarkus image build` するより楽ですね。

以下のサイトでイメージの push できていることを確認できれば OK です。
https://github.com/<username>?tab=packages

あとは好きな場所から、pull して使ってください。
今回は以上です！