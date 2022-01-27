# サンプル・アプリケーションのデプロイ(1)
OpenLiberty Operatorを用いてサンプル・アプリケーションをOCP上にデプロイします。<br>
使用しているプロジェクト (本手順では「liberty-on-ocp」を想定)に、既にOpenLiberty Operatorが導入されている前提とします。

## 1. イメージのビルド
Dockerfileを利用し、イメージをビルド(Dockerビルド)します。<br>
<br>
コマンド例
```
cat <Dockerfileのパス> | oc new-build --strategy=docker <リポジトリURL> --context-dir=<ビルド対象のディレクトリ> --name=<BuildConfig名> --to='<イメージストリーム名>' --dockerfile='-'
```

実行例
```
$ cat Dockerfile | oc new-build --strategy=docker https://github.com/OpenLiberty/guide-getting-started.git --context-dir=start --name=app --to='app-imagestream:1.0-SNAPSHOT' --dockerfile='-'
--> Found container image 15c384e (10 days old) from Docker Hub for "ibmcom/websphere-liberty:full-java11-openj9-ubi"
    Red Hat Universal Base Image 8 
    ------------------------------ 
    The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.
    Tags: base rhel8
    * An image stream tag will be created as "websphere-liberty:full-java11-openj9-ubi" that will track the source image
    * A Docker build using a predefined Dockerfile will be created
      * The resulting image will be pushed to image stream tag "app-imagestream:1.0-SNAPSHOT"
      * Every time "websphere-liberty:full-java11-openj9-ubi" changes a new build will be triggered
--> Creating resources with label build=app ...
    imagestream.image.openshift.io "app-imagestream" created
    buildconfig.build.openshift.io "app" created
--> Success
```

## 2. ビルド・ログの確認
ビルドの状況を確認し、「Push successful」が表示されることを確認します。

```
$ oc logs -f bc/app
Receiving source from STDIN as archive ...
Replaced Dockerfile FROM image ibmcom/websphere-liberty:full-java11-openj9-ubi
Caching blobs under "/var/cache/blobs".

Pulling image ibmcom/websphere-liberty@sha256:112d7f01f3e3e2395e337bd1d50717f12c4a610896b3bb7dc0034c6fef5dcbec ...
Getting image source signatures
＊＊（略）＊＊
Copying blob sha256:70b0f25802251e5a708df61291ba957f1b1c59beeb781466821b4cfec60e5ada
Copying config sha256:eafe03e89130c4b9056beb73c10d73587621be25daaf36ff159280c9173b1e50
Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/test-yamamoto/app-imagestream@sha256:7638e62c4313b23621a0e0969458bc9bee8d99cab75237d180f2d1ac65aa96ba
Push successful
```

## 3. ビルドされたイメージの確認
ビルドが成功し、イメージストリームが作成されていることを確認します。<br>
以下の実行例では、「app-imagestream」という名前の、タグが「1.0-SNAPSHOT」のイメージストリームが作成されていることを確認します。

```
$ oc get is 
NAME                IMAGE REPOSITORY                                                                   TAGS                     UPDATED
app-imagestream     image-registry.openshift-image-registry.svc:5000/liberty-on-ocp/app-imagestream    1.0-SNAPSHOT             2 minutes ago
websphere-liberty   image-registry.openshift-image-registry.svc:5000/liberty-on-ocp/websphere-liberty  full-java11-openj9-ubi   7 minutes ago
```

## 4. OpenLibertyApplicationリソースの編集
OpenLibertyApplicationリソースを用意します。[olapp_example-app-1.yaml](./olapp_example-app-1.yaml)を開き、applicationImageの項目に3. で確認したイメージを指定します。

```
$ cat olapp_example-app-1.yaml
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyApplication
metadata:
  name: sample-app
  labels:
    name: sample-app
spec:
  applicationImage: liberty-on-ocp/app-imagestream:1.0-SNAPSHOT
  env:
    - name: WLP_LOGGING_MESSAGE_FORMAT
      value: "json"
    - name: WLP_LOGGING_MESSAGE_SOURCE
      value: "message,trace,accessLog,ffdc,audit"
  expose: true
```

## 5. OpenLibertyApplicationリソースの作成
4. で編集した[olapp_example-app-1.yaml](./olapp_example-app-1.yaml)をもとに、OpenLibertyApplicationリソースを作成します。

```
$ oc apply -f olapp_example-app-1.yaml
openlibertyapplication.openliberty.io/sample-app created
```

作成後、リソースの状態を確認します。<br>
合わせて、アプリケーションに対するRoute(以下の例では出力の一番下の行)を確認します。

```
$ oc get olapp -l name=sample-app
NAME         IMAGE                                                 EXPOSED   RECONCILED   AGE
sample-app   liberty-on-ocp/test-sample-imagestream:1.0-SNAPSHOT   true      True         7m23s

$ oc get all -l app.kubernetes.io/name=sample-app
NAME                              READY   STATUS    RESTARTS   AGE
pod/sample-app-6b94759855-lll6d   1/1     Running   0          2m54s
NAME                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/sample-app   ClusterIP   172.21.124.209   <none>        9080/TCP   2m54s
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/sample-app   1/1     1            1           2m55s
NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/sample-app-6b94759855   1         1         1       2m55s

NAME                          HOST/PORT                                   PATH   SERVICES     PORT       TERMINATION   WILDCARD
route.route.openshift.io/sample-app   sample-app-liberty-on-ocp.liberty-on-ocp-guide-xxxxx-0000.jp-tok.containers.appdomain.cloud          sample-app   9080-tcp                 None
```

## 6. アプリケーションの動作確認
5.で確認したRouteをもとに、アプリケーションにWebブラウザでアクセスします。

```
http://<Routeリソースにて指定されたホスト名>/
```
