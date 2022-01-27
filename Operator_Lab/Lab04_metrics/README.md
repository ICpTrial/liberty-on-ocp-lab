# Liberty関連メトリクスの取得
WebSphere Libertyに対するメトリクスの取得を行います。<br>
OCPのデフォルトではLibertyに関連するメトリクス(Heapの使用状況, スレッド数など)を取得できませんが、OCP提供のPromethuesに対しユーザー定義プロジェクトの有効化を行うか、別途監視の仕組みと組み合わせることで、Libertyに関連するメトリクスも収集することが可能です。<br>
本手順では、OCP提供のPromethuesに対しユーザー定義プロジェクトの有効化を行った前提でのメトリクスの収集を行います。ユーザー定義プロジェクトの有効化の方法については、[こちら](https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.8/html-single/monitoring/index#enabling-monitoring-for-user-defined-projects)をご参照ください。

## 1. ビルドの準備
メトリクスを有効化するための準備として、Libertyの設定をカスタマイズします。<br>
事前に本リポジトリを作業PCにクローンしている状態で、Operator_Lab/Lab04_metricsに移動し、以下のコマンドを実行します。

```
$ oc project liberty-on-ocp
Now using project "liberty-on-ocp" on server "https://c100-e.jp-tok.containers.cloud.ibm.com:30963".

$ git clone https://github.com/OpenLiberty/guide-getting-started.git

$ cat Dockerfile | oc new-build --strategy=docker https://github.com/OpenLiberty/guide-getting-started.git --context-dir=finish --name=metrics-app --to=‘metrics-app-imagestream:1.0-SNAPSHOT' --dockerfile='-'
```

続いて、Libertyの構成ファイル[server.xml](./server.xml)を確認・必要に応じて編集します。以下のように、Libertyに対し「mpMetrics-3.0」と「monitor-1.0」フィーチャーを指定します。
また、mpMetricsの項目で、authenticationをfalseに指定することで、Prometheus側でメトリクスを取得することを許容します。

```
<server>
    <featureManager>
        :
        <feature>mpMetrics-3.0</feature>
        <feature>monitor-1.0</feature>
        :
    </featureManager>
    
    <mpMetrics authentication="false" />
    :
</server>
```

確認が完了したら、[server.xml](./server.xml)を作業PC上にダウンロードしたアプリケーションのソースコード・リポジトリのフォルダにコピーします。
```
$ cp server.xml guide-getting-started/finish/src/main/liberty/config/
```

その後、改めてイメージをビルドします。
```
$ cd guide-getting-started

$ oc start-build metrics-app --from-dir=.
Uploading directory "." as binary input for the build ...
.
Uploading finished
build.build.openshift.io/metrics-app-2 started
```

以下のコマンドで、実際のビルド状況を確認します。「Push Successful」が表示されたらビルドは正常に完了です。
```
$ oc logs -f bc/metrics-app
```


## 2. OpenLibertyApplicationリソースの作成
1.でビルドしたイメージを用いて、OpenLibertyApplicationリソースを作成します。<br>
[olapp_metrics.yaml](./olapp_metrics.yaml)を開き、1.で作成したイメージを指定します。<br>
また、Prometheusでのモニタリングのため、spec.monitoringの項目を追加します。(指定する値は"{}"のままで問題ありません。)

```
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyApplication
metadata:
  name: metrics-app
spec:
  applicationImage: liberty-on-ocp/metrics-app-imagestream:1.0-SNAPSHOT
  applicationName: metrics-app
  env:
  - name: WLP_LOGGING_MESSAGE_FORMAT
    value: json
  - name: WLP_LOGGING_MESSAGE_SOURCE
    value: message,trace,accessLog,ffdc,audit
  expose: true
  monitoring: {}
```

編集した[olapp_metrics.yaml](./olapp_metrics.yaml)をもとに、リソースを作成します。
```
$ oc apply -f olapp_metrics.yaml 
openlibertydump.openliberty.io/metrics-app created
```

本手順の場合、PrometheusがLibertyのメトリクスを取得するためにServiceMonitorリソースが必要になります。以下のようにServiceMonitorリソースが作成されていることを確認します。
```
$ oc get servicemonitor
NAME          AGE
metrics-app   3m39s
```

Routeの設定を確認し、「http://<RouteのURL>/metrics」にWebブラウザでアクセスします。実際にデータがテキストで返ってきていればLibertyの設定は正しいと判断できます。
```
$ oc get route
NAME           HOST/PORT       PATH   SERVICES         PORT       TERMINATION   WILDCARD
metrics-app    metrics-app-liberty-on-ocp.liberty-on-ocp-guide-xxxx-0000.jp-tok.containers.appdomain.cloud   metrics-app      9080-tcp                 None
```

## 3. Libertyメトリクスの取得確認
OpenShiftダッシュボードより、モニタリング > メトリクス の項目を開きます。<br>
そして、例えば「base_gc_total」という値をクエリーし、値が取得できているかを確認します。
