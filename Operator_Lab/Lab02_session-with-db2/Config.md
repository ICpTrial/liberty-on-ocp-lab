# Libertyのカスタマイズ例
Db2をセッション・ストアとして構成したOpenLiberty Applicationにおいて、ConfigMapやSecretを利用して構成情報を環境に合わせて設定変更しやすい形にカスタマイズします。

## 1. Libertyの構成ファイル(server.xml)の用意
[src/main/liberty/config/server.xml](./src/main/liberty/config/server.xml)を編集し、Db2への接続情報を環境変数に変更します。

```
$ cat src/main/liberty/config/server.xml
<server description="Liberty Server for Sessions Management">
    <featureManager>
        <feature>servlet-4.0</feature>
        <feature>jaxrs-2.1</feature>
        <feature>jsonp-1.1</feature>
        <feature>mpOpenAPI-2.0</feature>
        <feature>sessionDatabase-1.0</feature>
    </featureManager>

    <variable name="default.http.port" defaultValue="9080"/>
    <variable name="default.https.port" defaultValue="9443"/>
    <variable name="app.context.root" defaultValue="guide-sessions"/>

    <httpEndpoint httpPort="${default.http.port}" httpsPort="${default.https.port}"
        id="defaultHttpEndpoint" host="*" />

    <webApplication location="guide-sessions.war" contextRoot="${app.context.root}" />

    <!-- DB2 session database-->
    <library id="DB2JCC4Lib">
       <fileset dir="/opt/ibm/wlp/usr/shared/resources/" includes="db2jcc4.jar"/>
    </library>

    <dataSource id="SessionDS">
        <jdbcDriver libraryRef="DB2JCC4Lib"/>
        <properties.db2.jcc databaseName="SESSION" serverName="${SERVER_NAME}" portNumber="50000" 
                        currentLockTimeout="30s" user="${DB2_USER}" password="${DB2INST1_PASSWORD}"/>        
    </dataSource>
    <httpSessionDatabase id="SessionDB" dataSourceRef="SessionDS"/>
    <httpSession storageRef="SessionDB" cloneId="101"/>
    <!--  DB2 session database -->
</server>
```

編集したファイルは1. でダウンロードしたサンプル・アプリケーションのリポジトリにコピーします。

```
$ cp -p src/main/liberty/config/server.xml guide-sessions/start/src/main/liberty/config/ 
```

## 2. イメージのビルド
ローカルにあるソースコードをOCPにアップロードし、イメージのビルドを行います。<br>
Libertyをデプロイしたいプロジェクトは「liberty-on-ocp」であるとして、イメージのビルドを行います。

```
$ cd guide-sessions

$ oc project liberty-on-ocp
Now using project "liberty-on-ocp" on server "https://c100-e.jp-tok.containers.cloud.ibm.com:30963".

$ oc start-build test-db2 --from-dir=.
..
Uploading finished
build.build.openshift.io/test-db2-2 started
```

ビルド実施後、ビルドのSTATUSが「Complete」となることを確認します。
```
$ oc get build
NAME         TYPE     FROM                     STATUS     STARTED             DURATION
test-db2-3   Docker   Binary@6hc4rdd           Complete   7 minutes ago       5m34s
```



## 3. ConfigMapの作成
環境変数として参照するため、Db2のホスト名「db2.db2-test.svc.cluster.local」をConfigMapに登録します。

```
$ oc create cm db2-cm --from-literal=server_name=db2.db2-test.svc.cluster.local
configmap/db2-cm created

$ oc get cm db2-cm -o yaml
apiVersion: v1
data:
  server_name: db2.db2-test.svc.cluster.local
kind: ConfigMap
metadata:
  creationTimestamp: "2021-12-09T10:39:45Z"
  managedFields:
＊＊（略）＊＊
  name: db2-cm
  namespace: liberty-on-ocp
  resourceVersion: "28072586"
  selfLink: /api/v1/namespaces/liberty-on-ocp/configmaps/db2-cm
  uid: f4b87cc0-f4a9-4ed3-a6e0-10ecc5d04b4b
```

## 4. Secretの作成
Db2のユーザー情報に関しては、ConfigMapではなくSecretで登録します。<br>
本手順では、ユーザー名は「db2inst1」, パスワードは「db2inst1」とします。

```
$ oc create secret generic db2-secret --from-literal=db2_user=db2inst1 --from-literal=db2_password=db2inst1
secret/db2-secret created

$ oc get secret db2-secret -o yaml
apiVersion: v1
data:
  db2_password: ZGIyaW5zdDE=
  db2_user: ZGIyaW5zdDE=
kind: Secret
metadata:
  creationTimestamp: "2021-11-16T01:30:32Z"
  managedFields:
＊＊（略）＊＊
  name: db2-secret
  namespace: liberty-on-ocp
  resourceVersion: "17175740"
  selfLink: /api/v1/namespaces/liberty-on-ocp/secrets/db2-secret
  uid: 903513ad-aae2-4762-928b-e7d92732cc73
type: Opaque
```

## 5. OpenLibertyApplicationリソースの編集
OpenLibertyApplicationリソースの作成のため、[olapp_db2_env.yaml ](./olapp_db2_env.yaml)を編集します。

```
$ cat olapp_db2_env.yaml 
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyApplication
metadata:
  name: liberty-db2-session-env
  labels:
    name: liberty-db2-session-env
spec:
  applicationImage: liberty-on-ocp/test-db2-imagestream:1.0-SNAPSHOT
  env:
    - name: WLP_LOGGING_MESSAGE_FORMAT
      value: "json"
    - name: WLP_LOGGING_MESSAGE_SOURCE
      value: "message,trace,accessLog,ffdc,audit"
    - name: DB2_USER
      valueFrom:
        secretKeyRef:
          key: db2_user
          name: db2-secret
    - name: DB2INST1_PASSWORD
      valueFrom:
        secretKeyRef:
          key: db2_password
          name: db2-secret
  replicas: 2
  expose: true
```

## 6. OpenLibertyApplicationリソースの作成
5. で編集した[olapp_db2_env.yaml ](./olapp_db2_env.yaml)をもとに、OpenLibertyApplicationリソースを作成します。

```
$ oc apply -f olapp_db2_env.yaml
Open Liberty Application.openliberty.io/liberty-db2-session configured
```

## 7. アプリケーションの動作確認
再度デプロイし、Podが稼働状態になっていることを確認した後で、アプリケーションにWebブラウザでアクセスします。

```
http://<Routeリソースにて指定されたホスト名>/openapi/ui/
```
