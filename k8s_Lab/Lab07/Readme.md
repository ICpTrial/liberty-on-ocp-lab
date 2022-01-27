# Lab07. 割り当てリソースの制限
これまで作成してきたLibertyのPodに対し、Lab07では使用するリソースの制限を与えます。

Lab07で用意されている[deployment-with-resources.yaml](./deployment-with-resources.yaml)の中身を確認します。
23−29行目にあるリソースの設定で、実行するコンテナのCPUとメモリーに対し、制限を加えます。
```
        resources:
          requests:
            memory: "256Mi"
            cpu: 1
          limits:
            memory: "512Mi"
            cpu: 2
```
この設定では、Libertyコンテナに対し、requestsの項目で「必ず1cpu分のCPUリソース、256MiBのメモリー・リソースを確保」し、limitsの項目で「最大で2cpu分のCPUリソース、512MiBのメモリー・リソースをまで利用可能」な設定となります。

実際に[deployment-with-resources.yaml](./deployment-with-resources.yaml)を利用してデプロイメントの構成を更新します。imageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-with-resources.yaml
```

デプロイが完了したら、以下のコマンドでrequests/limitsが設定されていることを確認します。
```
oc describe pod <デプロイされたPod名>
```
