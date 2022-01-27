# Libertyの構成変更例(2) - セッション共有 (Db2)
OpenLiberty Operatorを用いてセッション共有を行うサンプル・アプリケーションをOCP上にデプロイします。<br>
使用しているプロジェクト (本手順では「liberty-on-ocp」を想定)に、既にOpenLiberty Operatorが導入されている前提とします。<br>
本サンプルでは、Db2をセッション・ストアとしてアプリケーションを構成します。

## 1. サンプル・アプリケーションのソースコードのダウンロード
サンプル・アプリケーションのソースコードをローカルにダウンロードします。

```
$ git clone https://github.com/openliberty/guide-sessions.git
```

## 2. Db2の準備
Db2コンテナをOCP上にデプロイします。<br>
[こちら](https://github.com/IBM/application-modernization-javaee-quarkus/tree/master/db2/deployment)を参考に、Db2コンテナをデプロイします。

```
$ oc new-project db2-test
Now using project "db2-test" on server "https://c100-e.jp-tok.containers.cloud.ibm.com:30963".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app ruby~https://github.com/sclorg/ruby-ex.git

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=gcr.io/hello-minikube-zero-install/hello-node

$ oc create sa mysvcacct
serviceaccount/mysvcacct created

$ oc get sa
NAME        SECRETS   AGE
builder     2         2m53s
default     2         2m53s
deployer    2         2m53s
mysvcacct   2         3s

$ oc adm policy add-scc-to-user privileged -z mysvcacct
clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "mysvcacct”"

$ oc apply -f db2-dc.yaml 
deploymentconfig.apps.openshift.io/db2 created

$ oc get all -n db2-test
NAME                          READY   STATUS      RESTARTS   AGE
pod/db2-1-8s9jn    1/1     Running     0          46h
pod/db2-1-deploy   0/1     Completed   0          46h

NAME                                     DESIRED   CURRENT   READY   AGE
replicationcontroller/db2-1   1         1         1       46h

NAME                                                REVISION   DESIRED   CURRENT   TRIGGERED BY
deploymentconfig.apps.openshift.io/db2   1          1         1         config

$ oc apply -f db2-svc.yaml 
service/db2 created

$ oc get svc -n db2-test
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
db2   ClusterIP   172.21.231.14   <none>        50000/TCP   46h
```

## 3. セッションDBの準備
必須ではないですが、先にDb2にセッション共有のためのデータベースを作成します。

```
# Db2コンテナに対するrshログイン
$ oc get po -n db2-test
NAME                      READY   STATUS      RESTARTS   AGE
db2-1-8s9jn    1/1     Running     0          2m50s
db2-1-deploy   0/1     Completed   0          2m52s
$ oc rsh db2-1-8s9jn
sh-4.2#

# セッションデータ保管用のDBを作成
sh-4.2# mkdir /tablespace
sh-4.2# chown db2inst1:db2iadm1 /tablespace
sh-4.2# su - db2inst1
Last login: Mon Nov  8 06:21:01 UTC 2021
[db2inst1@db2-1-8s9jn ~]$ db2 create db session on /tablespace
DB20000I  The CREATE DATABASE command completed successfully.
[db2inst1@db2-1-8s9jn ~]$ db2 connect to session

   Database Connection Information

 Database server        = DB2/LINUXX8664 11.5.5.0
 SQL authorization ID   = DB2INST1
 Local database alias   = SESSION

[db2inst1@db2-1-8s9jn ~]$
```

## 4. イメージのビルド
Dockerfileを利用し、イメージをビルド(Dockerビルド)します。<br>
[Dockerfile.db2](./Dockerfile.db2)を使用します。<br>
```
$ cat Dockerfile.db2 | oc new-build --strategy=docker https://github.com/openliberty/guide-sessions.git --context-dir=finish --name=test-db2 --to='test-db2-imagestream:1.0-SNAPSHOT' --dockerfile='-'
```

この時点では、イメージのビルドが失敗します。あとで再度ビルドを行います。

## 5. Libertyの構成ファイル(server.xml)の用意
[src/main/liberty/config/server.xml](./src/main/liberty/config/server.xml)を編集し、作成したDb2をセッションDBとして使用する設定を追記します。

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
        <properties.db2.jcc databaseName="SESSION" serverName="db2.db2-test.svc.cluster.local" portNumber="50000" 
                        currentLockTimeout="30s" user="db2inst1" password="db2inst1"/>        
    </dataSource>
    <httpSessionDatabase id="SessionDB" dataSourceRef="SessionDS"/>
    <httpSession storageRef="SessionDB" cloneId="101"/>
    <!--  DB2 session database -->
</server>
```

編集したファイルは1. でダウンロードしたサンプル・アプリケーションのリポジトリにコピーします。

```
$ cp -p src/main/liberty/config/server.xml guide-sessions/start/src/main/liberty/config/ 
```

## 6. JDBCドライバーのインストール
JDBCドライバー（db2jcc4.jar）を[こちらのサイト](https://www.ibm.com/support/pages/db2-jdbc-driver-versions-and-downloads)からダウンロードします。<br>
ダウンロードしたJDBCドライバーを適当なディレクトリに配置します。<br>
本手順では guide-sessions/final/src/main/liberty/config/以下にJDBCドライバー(db2jcc4.jar) を配置します。

## 7. イメージのビルド
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
test-db2-2   Docker   Binary@1fc48cd           Complete   7 minutes ago       5m34s
```

## 8. OpenLibertyApplicationリソースの編集
OpenLibertyApplicationリソースの作成のため、[olapp_db2.yaml](./olapp_db2.yaml)を編集します。

```
$ cat olapp_db2.yaml
apiVersion: apps.openliberty.io/v1beta2
kind: Open Liberty Application
metadata:
  name: liberty-db2-session
  labels:
    name: liberty-db2-session
  namespace: liberty-on-ocp
spec:
  applicationImage: liberty-on-ocp/test-db2-imagestream:1.0-SNAPSHOT
  env:
    - name: WLP_LOGGING_MESSAGE_FORMAT
      value: "json"
    - name: WLP_LOGGING_MESSAGE_SOURCE
      value: "message,trace,accessLog,ffdc,audit"
  replicas: 2
  expose: true
```

## 9. OpenLibertyApplicationリソースの作成
8. で編集した[olapp_db2.yaml](./olapp_db2.yaml)をもとに、OpenLibertyApplicationリソースを作成します。

```
$ oc apply -f olapp_db2.yaml 
Open Liberty Application.openliberty.io/liberty-db2-session created
```

作成後、リソースの状態を確認します。<br>
合わせて、アプリケーションに対するRoute(以下の例では出力の一番下の行)を確認します。

```
$ oc get all -l app.kubernetes.io/name=liberty-db2-session
NAME                           READY   STATUS    RESTARTS   AGE
pod/liberty-db2-session-dc45f9bfc-lpzq9   1/1     Running   0          47h
pod/liberty-db2-session-dc45f9bfc-xq682   1/1     Running   0          2d

NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/liberty-db2-session   ClusterIP   172.21.163.213   <none>        9080/TCP   2d

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/liberty-db2-session   2/2     2            2           2d

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/liberty-db2—session-dc45f9bfc   2         2         2       2d

NAME                                HOST/PORT                                                                                                              PATH   SERVICES   PORT       TERMINATION   WILDCARD
route.route.openshift.io/liberty-db2-session   liberty-db2-session-liberty-on-ocp.liberty-on-ocp-guide-xxxx-0000.jp-tok.containers.appdomain.cloud      liberty-db2-session   9080-tcp         None
```

## 10. アプリケーションの動作確認
9.で確認したRouteをもとに、アプリケーションにWebブラウザでアクセスします。

```
http://<Routeリソースにて指定されたホスト名>/openapi/ui/
```

[後続の手順](./Config.md)でOpenLiberty Applicationの設定のカスタマイズ例に進んでください。
