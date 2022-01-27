# Lab04. Podの分散配置
[Lab03](../Lab03/)まででデプロイしたアプリケーションに対し、可用性を高めるためにPodを複数作成・分散配置を行うための設定例を説明します。

## (1) replica数の変更
現在はPodのレプリカ数は1の状態です。レプリカ数を2に変更します。

Lab04で用意されているdeployment-change-replicas.yamlを適用します。imageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-change-replicas.yaml
```

実行後、Pod数を確認します。
```
# デプロイメントに対する指定Pod数の確認
$ oc get deployment

# Pod数の確認
$ oc get pod
```

## (2) Pod配置の設定
これまでの設定で複数のPodを配置しましたが、Worker Nodeの障害を考慮し、可能な限り別々のWorker Nodeに分散配置を行わせるための設定を行います。<br>
Lab04で用意されている[deployment-with-affinity.yaml](./deployment-with-affinity.yaml)を適用します。[deployment-change-replicas.yaml](./deployment-change-replicas.yaml)との違いは、以下のPodAntiAffinityの項目です。<br>
Podに対しappラベルの値が「sample-app」であるPodは可能な限り別々のWorker Nodeに配置することを以下の設定で保証します。
```
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - sample-app
              topologyKey: kubernetes.io/hostname
```

上記の設定の意味を理解した上で、[deployment-with-affinity.yaml](./deployment-with-affinity.yaml)のimageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-with-affinity.yaml
```

実行後、Podの配置を確認し、別々のノードにPodが配置されていることを確認します。
```
# Pod数の確認
$ oc get pod -o wide
```

可能であれば、以下のコマンドでPod数を増やし、動作を確認します。<br>
Podの台数がWorker Nodeの台数を超えるまでは別々のWorker Nodeに配置しようと動作し、台数を超えた場合でもエラーにならずスケール可能であることが確認できます。
```
# Pod数の変更
$ oc scale deployment --replicas <Pod数> sample-app
```
