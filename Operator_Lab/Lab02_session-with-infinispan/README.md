# Libertyの構成変更例(1) - セッション共有 (Infinispan)
OpenLiberty Operatorを用いてセッション共有を行うサンプル・アプリケーションをOCP上にデプロイします。<br>
使用しているプロジェクト (本手順では「liberty-on-ocp」を想定)に、既にOpenLiberty Operatorが導入されている前提とします。<br>
本サンプルでは、Infinispanをセッション・ストアとして、JCacheの仕様に従ったセッション共有をアプリケーションに対して構成します。

## 1. サンプル・アプリケーションのソースコードのダウンロード
サンプル・アプリケーションのソースコードをローカルにダウンロードします。

```
$ git clone https://github.com/openliberty/guide-sessions.git
```

## 2. Infinispanの準備
Infinispan Operatorを利用してInfinispan環境をOCP上にデプロイします。<br>
[こちら](https://infinispan.org/docs/infinispan-operator/main/operator.html)を参考に、Infinispanをデプロイします。

## 3. イメージのビルド
Dockerfileを利用し、イメージをビルド(Dockerビルド)します。<br>
[Dockerfile](./Dockerfile)を使用します。<br>

```
$ cat Dockerfile | oc new-build --strategy=docker https://github.com/OpenLiberty/guide-sessions.git --context-dir=finish --name=liberty-infinispan --to='liberty-infinispan-imagestream:1.0-SNAPSHOT' --dockerfile='-' 
```

この時点では、イメージのビルドが失敗します。あとで再度ビルドを行います。

## 4. Libertyの構成ファイル(server.xml)の確認
[src/main/liberty/config/server.xml](./src/main/liberty/config/server.xml)を開き、Infinispanでセッション共有を行う設定を確認します。

```
$ cat src/main/liberty/config/server.xml
:    
    <!-- tag::InfinispanLib[] -->
    <variable name="infinispan.lib" defaultValue="${shared.resource.dir}/infinispan.jar"/>
    <!-- end::InfinispanLib[] -->

    <httpEndpoint httpPort="${default.http.port}" httpsPort="${default.https.port}"
        id="defaultHttpEndpoint" host="*" />
     
    <httpSessionCache libraryRef="InfinispanLib" enableBetaSupportForInfinispan="true">
        <properties infinispan.client.hotrod.server_list="${env.SRV_HOST}:${env.SRV_PORT}"/>
        <properties infinispan.client.hotrod.auth_username="${env.USER_NAME}"/>
        <properties infinispan.client.hotrod.auth_password="${env.USER_PASSWD}"/>
        <properties infinispan.client.hotrod.auth_realm="default"/>
        <properties infinispan.client.hotrod.sasl_mechanism="DIGEST-MD5"/>
        <properties infinispan.client.hotrod.marshaller="org.infinispan.commons.marshall.JavaSerializationMarshaller"/>
    </httpSessionCache>

    <library id="InfinispanLib">
        <fileset dir="${shared.resource.dir}/infinispan" includes="*.jar"/>
    </library>

    <webApplication location="guide-sessions.war" contextRoot="${app.context.root}" />

</server>
```

## 5. イメージのビルド
ローカルにあるソースコードをOCPにアップロードし、イメージのビルドを行います。<br>
Libertyをデプロイしたいプロジェクトは「liberty-on-ocp」であるとして、イメージのビルドを行います。

```
$ cp –rp infinispan guide-sessions/finish

$ cp -rp src/main/liberty/config guide-sessions/finish/src/main/liberty

$ cd guide-sessions

$ oc project liberty-on-ocp
Now using project "liberty-on-ocp" on server "https://c100-e.jp-tok.containers.cloud.ibm.com:30963".

$  oc start-build liberty-infinispan --from-dir=.
Uploading directory "." as binary input for the build ...
.
Uploading finished
build.build.openshift.io/liberty-infinispan-2 started
```

ビルドが開始されたら、ログを確認し、「Push successful」が出力されることを確認します。
```
$ oc logs bc/liberty-infinispan -f
Receiving source from STDIN as archive ...
Replaced Dockerfile FROM image ibmcom/websphere-liberty:full-java11-openj9-ubi
Caching blobs under "/var/cache/blobs".

Pulling image ibmcom/websphere-liberty@sha256:b68923409492b3eeb0c39b2d22bfaf40aa6579e795aad81ed60d95aa2290609c ...
Getting image source signatures
:

Copying config sha256:a1ace259992ed6a0a566ab9c69a4612221f48ff9c28313261074a4a5bdd8d883
Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/liberty-guide-demo/liberty-infinispan-imagestream@sha256:615af5929aed60fa1480dbf813895d83469892d170acfe4a796a419779a770ee
Push successful
```

## 6. Secretの作成
Infinispanの接続情報をSecretとして登録します。

```
$ oc create secret generic secret-infinispan-auth
   --from-literal=username=developer
   --from-literal=password=<Infinispanのパスワードを指定>
   --from-literal=srv_host=<Infinispanのインスタンス名>
   --from-literal=srv_port=11222
```

## 7. OpenLibertyApplicationリソースの編集
OpenLibertyApplicationリソースの作成のため、[olapp.yaml](./olapp.yaml)を編集します。

```
$ cat olapp.yaml
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyApplication
metadata:
  name: cart
spec:
  applicationImage: liberty-guide-demo/liberty-infinispan-imagestream:1.0-SNAPSHOT
  applicationName: cart
  expose: true
  pullPolicy: Always
  replicas: 1
  service:
    port: 9080
    type: ClusterIP
  env:
  - name: USER_NAME
    valueFrom:
      secretKeyRef:
        name: secret-infinispan-auth
        key: username
  - name: USER_PASSWD
    valueFrom:
      secretKeyRef:
        name: secret-infinispan-auth
        key: password
  - name: SRV_HOST
    valueFrom:
      secretKeyRef:
        name: secret-infinispan-auth
        key: srv_host
  - name: SRV_PORT
    valueFrom:
      secretKeyRef:
        name: secret-infinispan-auth
        key: srv_port
```

## 8. OpenLibertyApplicationリソースの作成
7. で編集した[olapp.yaml](./olapp.yaml)をもとに、OpenLibertyApplicationリソースを作成します。

```
$ oc apply -f olapp.yaml
openlibertyapplication.apps.openliberty.io/cart created
```

作成後、アプリケーションに対するRouteを確認します。

```
$ oc get route
NAME                       HOST/PORT                                                                                                                      PATH   SERVICES          PORT       TERMINATION   WILDCARD
cart                       cart-liberty-guide-demo.liberty-on-ocp-guide-d85540d5ac7eb4665421ec4a86b8d368-0000.jp-tok.containers.appdomain.cloud                  cart              9080-tcp                 None
:
```

## 9. アプリケーションの動作確認
8.で確認したRouteをもとに、アプリケーションにWebブラウザでアクセスします。

```
http://<Routeリソースにて指定されたホスト名>/openapi/ui/
```
