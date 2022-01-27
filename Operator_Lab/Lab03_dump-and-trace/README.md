# トレースの取得とダンプの永続化
OpenLiberty Operatorを用いてWebSphere Libertyに対するトレースの取得とダンプの永続化を行います。

## 1. 永続ストレージの準備
環境に合わせて永続ストレージを準備します。各環境のOCP/KubernetesのStorageClassを確認し、作成を行います。<br>
StorageClassが動的プロビジョニングをサポートしている場合、OpenLibertyApplicationリソース内でStorageClassを指定することで、StorageClassに合わせてストレージを準備できます。<br>
<br>
サンプルの[olapp_ dump-and-trace.yaml](./olapp_ dump-and-trace.yaml)を開き、ストレージの設定を行います。
IBM CloudのClassic Infrastructureを利用している場合、StorageClass名に「ibmc-file-bronze-gid」を指定することで、永続ストレージ (ファイルストレージ)を用意できます。

```
$ cat olapp_ dump-and-trace.yaml
apiVersion: apps.openliberty.io/v1beta2
kind: Open Liberty Application
metadata:
	name: dump-and-trace
spec:
	applicationImage: ibmcom/websphere-liberty
	applicationName: dump-and-trace
	expose: true
	serviceability:
		size: 1Gi
		storageClassName: ibmc-file-bronze-gid
	replicas: 1
```

環境に合わせて編集したOpenLibertyApplicationリソースで、Liberty Podをデプロイします。既にOpenLiberty Operatorが導入されているプロジェクト(例では「liberty-on-ocp」)でリソースを作成します。
```
$ oc project liberty-on-ocp
Now using project "liberty-on-ocp" on server "https://c100-e.jp-tok.containers.cloud.ibm.com:30963".

$ oc apply -f olapp_dump-and-trace.yaml 
openlibertydump.openliberty.io/dump-and-trace created
```

各リソースが作成されているかを確認します。本手順では「dump-and-trace-6f5dc769d7-m5b54」という名前のPodが作成されていることが確認できます。
```
$ oc get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                           STORAGECLASS            REASON   AGE
pvc-8a5b209e-bd3d-4a38-9757-d12b029bdf33   20Gi       RWO,RWX        Delete           Bound    liberty-on-ocp/dump-and-trace-serviceability                    ibmc-file-bronze-gid             31s

$ oc get pvc
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS           AGE
dump-and-trace-serviceability     Bound    pvc-8a5b209e-bd3d-4a38-9757-d12b029bdf33   20Gi       RWO,RWX        ibmc-file-bronze-gid   2m16s

$ oc get pod -l app.kubernetes.io/name=dump-and-trace
NAME                              READY   STATUS    RESTARTS   AGE
dump-and-trace-6f5dc769d7-m5b54   1/1     Running   0          7m28s
```

## 2. ダンプの取得
OpenLibertyApplicationに対し、有事の際に問題判別のためダンプを取得することを考えます。
OpenLiberty OperatorではOpenLibertyDumpリソースを作成することで、指定したPodのダンプを取得することが可能になっています。<br>

[oldump.yaml](./oldump.yaml)を開き、podNameに1.で確認したPod名を指定します。本手順の例の場合、以下のようなファイルを作成することになります。
```
$ cat oldump.yaml
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyDump
metadata:
  name: test-dump
spec:
  include:
  - heap
  - thread
  podName: dump-and-trace-6f5dc769d7-m5b54
```

編集した[oldump.yaml](./oldump.yaml)をもとに、リソースを作成します。
```
$ oc apply -f oldump.yaml 
openlibertydump.openliberty.io/test-dump created
```

しばらく待った後で、OpenLibertyDumpリソースの状態を確認します。以下の例のように、COMPLETEDがTrueで、DUMP FILEの項目にファイルパスが指定されていることが確認できれば正常に処理が完了しています。
```
$ oc get oldump
NAME           STARTED   COMPLETED   DUMP FILE
test-dump      True      True        /serviceability/liberty-on-ocp/dump-and-trace-6f5dc769d7-m5b54/2021-12-16_06:35:42.zip
```

実際にファイルが作成されているかは、直接Podにログインして確認することが可能です。
```
$ oc rsh dump-and-trace-6f5dc769d7-m5b54
bash-4.4$ ls -l /serviceability/
total 4
drwxr-xr-x. 3 1000660000 99 4096 Oct 12 04:57 liberty-on-ocp

bash-4.4$ ls -l /serviceability/liberty-on-ocp/
total 4
drwxr-xr-x. 2 1000660000 99 4096 Oct 12 04:57 dump-and-trace-6f5dc769d7-m5b54

bash-4.4$ ls -l /serviceability/liberty-on-ocp/dump-and-trace-6f5dc769d7-m5b54/2021-12-16_06:35:42.zip 
total 7044
-rw-r-----. 1 1000660000 99 7178880 Oct 12 04:57 2021-12-16_06:35:42.zip 
bash-4.4$
```

作業PCにダンプファイルをダウンロードしたい場合は、以下のようにoc cpコマンドで実施します。<br>
<br>
コマンド例
```
oc cp <Pod名>:<Pod上にあるダンプファイルのパス> ./<ローカルに保存するファイル名>
```

実行例
```
$ oc cp liberty-test-app-856d6855ff-bs8b9: /serviceability/liberty-on-ocp/dump-and-trace-6f5dc769d7-m5b54/2021-12-16_06:35:42.zip ./2021-10-12_04:57:29.zip
```

## 3. トレースの有効化
OpenLibertyApplicationに対し、有事の際に問題判別のためトレースを有効化・トレースレベルの変更を行うことを考えます。
OpenLiberty OperatorではOpenLibertyTraceリソースを作成することで、指定したPod上のLibertyに対するトレースの有効化・トレースレベルの変更を行うことが可能になっています。<br>

[oltrace.yaml](./oltrace.yaml)を開き、podNameに1.で確認したPod名を指定します。本手順の例の場合、以下のようなファイルを作成することになります。

```
$ cat oldump.yaml
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyTrace
metadata:
  name: test-trace
spec:
  podName: dump-and-trace-6f5dc769d7-m5b54
  traceSpecification: '*=info:com.ibm.ws.webcontainer*=all'
```

編集した[oltrace.yaml](./oltrace.yaml)をもとに、リソースを作成します。
```
$ oc apply -f oltrace.yaml 
openlibertytrace.openliberty.io/test-trace created
```

しばらく待った後で、OpenLibertyTraceリソースの状態を確認します。以下の例のように、TRACINGがTrueになっていることを確認します。
```
$ oc get oltrace
NAME         PODNAME                             TRACING
test-trace   liberty-test-app-856d6855ff-bs8b9   True
```

実際にトレースファイルが作成されているかは、直接Podにログインして確認することが可能です。永続ストレージにmessages.log, trace.logが作成されていることを確認します。
```
$ oc rsh dump-and-trace-6f5dc769d7-m5b54 
sh-4.4$ ls -l /serviceability/liberty-on-ocp/dump-and-trace-6f5dc769d7-m5b54/
total 7096
-rw-rw----. 1 1000660000 99 7222274 Dec 16 06:36 2021-12-16_06:35:42.zip
-rw-r-----. 1 1000660000 99     714 Dec 16 06:57 messages.log
-rw-r-----. 1 1000660000 99     784 Dec 16 06:57 trace.log
```

作業PCにダンプファイルをダウンロードしたい場合は、以下のようにoc cpコマンドで実施します。<br>

```
$ oc cp liberty-test-app-856d6855ff-bs8b9:serviceability/liberty-on-ocp/dump-and-trace-6f5dc769d7-m5b54/trace.log ./trace.log
$ oc cp liberty-test-app-856d6855ff-bs8b9:serviceability/liberty-on-ocp/dump-and-trace-6f5dc769d7-m5b54/message.log ./message.log
```

トレースを無効化したい場合は、以下のようにOpenLibertyTraceリソースを編集し、無効化します。
```
$ oc get oltrace
NAME         PODNAME                           TRACING
test-trace   dump-and-trace-6f5dc769d7-m5b54   True　　　　　　 ##トレースが有効

$ oc edit oltrace test-trace
openlibertytrace.openliberty.io/test-trace edited

===変更内容===
spec:
  disable: true　　　　　　　　　　　　## disableパラメータを追加しtrueに設定します
  podName: dump-and-trace-6f5dc769d7-m5b54
  traceSpecification: '*=info:com.ibm.ws.webcontainer*=all'
============

$ oc get oltrace
NAME         PODNAME                           TRACING
test-trace   dump-and-trace-6f5dc769d7-m5b54   False　　　　　　　##トレースが無効
```
