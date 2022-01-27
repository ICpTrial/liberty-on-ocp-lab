# サンプル・アプリケーションのデプロイ(2)
OpenLiberty Operatorを用いてサンプル・アプリケーションをOCP上にデプロイします。<br>
使用しているプロジェクト (本手順では「liberty-on-ocp」を想定)に、既にOpenLiberty Operatorが導入されている前提とします。<br>
本サンプルでは、Apache KafkaのKubernetes OperatorであるStrimzi Operatorを利用したサンプルを扱います。

## 1. Strimzi Operatorのデプロイ
[こちら](https://github.com/strimzi/strimzi-kafka-operator)のリンクを参照し、OCPにKafkaクラスターを導入します。<br>
作成後、KafkaクラスターのBootstrapアドレスを確認します。<br>
<br>
コマンド例
```
Macの場合： oc get kafka kafka-cluster -o=jsonpath='{.status.listeners[?(@.type=="plain")].bootstrapServers}{"\n"}'
Windowsの場合： oc get kafka kafka-cluster -o=jsonpath="{.status.listeners[?(@.type=='plain')].bootstrapServers}{'n'}"
```

実行例
```
$ oc get kafka kafka-cluster -o=jsonpath='{.status.listeners[?(@.type=="plain")].bootstrapServers}{"\n"}'
kafka-cluster-kafka-bootstrap.liberty-on-ocp.svc:9092
```

## 2. イメージのビルド
Dockerfileを利用し、イメージをビルド(Dockerビルド)します。<br>
[Dockerfile.inventory](./Dockerfile.inventory)および[Dockerfile.system](./Dockerfile.system)をそれぞれ使用します。<br>
<br>
コマンド例
```
cat <Dockerfileのパス> | oc new-build --strategy=docker <リポジトリURL> --context-dir=<ビルド対象のディレクトリ> --name=<BuildConfig名> --to='<イメージストリーム名>' --dockerfile='-'
```

実行例
```
$ cat Dockerfile.system | oc new-build --strategy=docker https://github.com/OpenLiberty/guide-cloud-openshift-operator.git --context-dir=start --name=system --to='system-imagestream:1.0-SNAPSHOT' --dockerfile='-'
--> Found container image b50b31b (45 hours old) from Docker Hub for "ibmcom/websphere-liberty:full-java11-openj9-ubi"

    Red Hat Universal Base Image 8 
    ------------------------------ 
    The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.

    Tags: base rhel8

＊＊（略）＊＊

--> Creating resources with label build=system ...
    imagestream.image.openshift.io "websphere-liberty" created
    imagestream.image.openshift.io "system-imagestream" created
    buildconfig.build.openshift.io "system" created
--> Success

$ cat Dockerfile.inventory | oc new-build --strategy=docker https://github.com/OpenLiberty/guide-cloud-openshift-operator.git --context-dir=start --name=inventory --to='inventory-imagestream:1.0-SNAPSHOT' --dockerfile='-'
--> Found container image b50b31b (45 hours old) from Docker Hub for "ibmcom/websphere-liberty:full-java11-openj9-ubi"

    Red Hat Universal Base Image 8 
    ------------------------------ 
    The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.

    Tags: base rhel8

＊＊（略）＊＊

--> Creating resources with label build=system ...
    imagestream.image.openshift.io "websphere-liberty" created
    imagestream.image.openshift.io "inventory-imagestream" created
    buildconfig.build.openshift.io "inventory" created
--> Success

```

## 3. ビルド結果の確認
ビルド結果を確認します。以下のように「pod/inventory-1-build」および「pod/system-1-build」のSTATUSが「Completed」になっていること、およびイメージストリームが作成されていることを確認します。
以下の実行例では、「system-imagestream 」および「inventory-imagestream」という名前の、タグが「1.0-SNAPSHOT」のイメージストリームがそれぞれ作成されていることを確認します。

```
$ oc get all
NAME                                                    READY   STATUS      RESTARTS   AGE
pod/inventory-1-build                                   0/1     Completed   0          83m
pod/open-liberty-operator-5b755b4d78-gwwl5              1/1     Running     0          2d4h
pod/strimzi-cluster-operator-v0.25.0-644dd5d9d9-w2wg2   1/1     Running     0          87s
pod/system-1-build                                      0/1     Completed   0          83m

＊＊（略）＊＊

NAME                                       TYPE     FROM             LATEST
buildconfig.build.openshift.io/inventory   Docker   Dockerfile,Git   1
buildconfig.build.openshift.io/system      Docker   Dockerfile,Git   1

NAME                                   TYPE     FROM                     STATUS     STARTED             DURATION
build.build.openshift.io/system-1      Docker   Dockerfile,Git@ca6448e   Complete   About an hour ago   5m12s
build.build.openshift.io/inventory-1   Docker   Dockerfile,Git@ca6448e   Complete   About an hour ago   5m2s

NAME                                                   IMAGE REPOSITORY                                                                        TAGS                     UPDATED
imagestream.image.openshift.io/inventory-imagestream   image-registry.openshift-image-registry.svc:5000/liberty-on-ocp/inventory-imagestream   1.0-SNAPSHOT             About an hour ago
imagestream.image.openshift.io/system-imagestream      image-registry.openshift-image-registry.svc:5000/liberty-on-ocp/system-imagestream      1.0-SNAPSHOT             About an hour ago
```

## 4. OpenLibertyApplicationリソースの編集
OpenLibertyApplicationリソースの作成のため、[olapp_example-app-2.yaml](./olapp_example-app-2.yaml)を編集します。

```
$ cat olapp_example-app-2.yaml
apiVersion: apps.openliberty.io/v1beta2
kind: Open Liberty Application
metadata:
  name: system
  labels:
    name: system
spec:
  applicationImage: liberty-on-ocp/system-imagestream:1.0-SNAPSHOT
  env:
    - name: WLP_LOGGING_MESSAGE_FORMAT
      value: "json"
    - name: WLP_LOGGING_MESSAGE_SOURCE
      value: "message,trace,accessLog,ffdc,audit"
    - name: MP_MESSAGING_CONNECTOR_LIBERTY_KAFKA_BOOTSTRAP_SERVERS
      value: “kafka-cluster-kafka-bootstrap.liberty-on-ocp.svc:9092”
---
apiVersion: apps.openliberty.io/v1beta2
kind: Open Liberty Application
metadata:
  name: inventory
  labels:
    name: inventory
spec:
  applicationImage: liberty-on-ocp/inventory-imagestream:1.0-SNAPSHOT
  service:
    port: 9085
  expose: true
  env:
    - name: WLP_LOGGING_MESSAGE_FORMAT
      value: "json"
    - name: WLP_LOGGING_MESSAGE_SOURCE
      value: "message,trace,accessLog,ffdc,audit"
    - name: MP_MESSAGING_CONNECTOR_LIBERTY_KAFKA_BOOTSTRAP_SERVERS
      value: "kafka-cluster-kafka-bootstrap.liberty-on-ocp.svc:9092"
```

## 5. OpenLibertyApplicationリソースの作成
4. で編集した[olapp_example-app-2.yaml](./olapp_example-app-2.yaml)をもとに、OpenLibertyApplicationリソースを作成します。

```
$ oc apply -f olapp_example-app-2.yaml 
Open Liberty Application.openliberty.io/system created
Open Liberty Application.openliberty.io/inventory created
```

作成後、リソースの状態を確認します。<br>
合わせて、アプリケーションに対するRoute(以下の例では出力の一番下の行)を確認します。

```
$ oc get olapp
NAME               IMAGE                                                     EXPOSED   RECONCILED   AGE
inventory          liberty-on-ocp/inventory-imagestream:1.0-SNAPSHOT         true      True         13s
system             liberty-on-ocp/system-imagestream:1.0-SNAPSHOT                      True         13s

$ oc get all -l app.kubernetes.io/managed-by=open-liberty-operator
NAME                                    READY   STATUS    RESTARTS   AGE
pod/inventory-55b4bf4fc4-ljqjd          1/1     Running   0          4d22h
pod/system-767f5bff54-rwz9k             1/1     Running   0          4d19h

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/inventory          ClusterIP   172.21.183.120   <none>        9085/TCP   4d22h
service/system             ClusterIP   172.21.23.4      <none>        9080/TCP   4d19h

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/inventory          1/1     1            1           4d22h
deployment.apps/system             1/1     1            1           4d19h

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/inventory-55b4bf4fc4          1         1         1       4d22h
replicaset.apps/system-767f5bff54             1         1         1       4d19h

NAME                                        HOST/PORT                                                                                                                      PATH   SERVICES           PORT       TERMINATION   WILDCARD
route.route.openshift.io/inventory          inventory-liberty-on-ocp.liberty-on-ocp-guide-xxxx-0000.jp-tok.containers.appdomain.cloud                 inventory          9085-tcp                 None
```

## 6. アプリケーションの動作確認
5.で確認したRouteをもとに、アプリケーションにWebブラウザでアクセスします。

```
http://<Routeリソースにて指定されたホスト名>/inventory/systems
```
