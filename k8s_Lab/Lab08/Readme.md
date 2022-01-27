# Lab08. ライセンスの適用と追跡
最後のLab08では、利用するLibertyに対し、本番での利用を想定し、ライセンスを適用した状態で稼働させることを考えます。
既にLibertyのライセンスがあること、ライセンス・サービスが既にOpenShift上に設定されていることを前提に、以下の作業を行います。
* [【参考】IBM Container License](https://www.ibm.com/software/passportadvantage/containerlicenses.html)

## (1) ライセンスの適用
以下のURLで紹介されているように、ライセンスを有効にして利用するためには環境変数LICENSE=acceptを設定する必要があります。
* [【参考】Using WebSphere Liberty container images in production](https://github.com/WASdev/ci.docker/tree/master/ga/production-upgrade)

LICENSEの環境変数をLab06で作成したConfigMapに追加します。
Lab08で用意されている[configmap-liberty-env.yaml](./configmap-liberty-env.yaml)を適用します。
```
$ oc apply -f configmap-liberty-env.yaml
```

## (2) ライセンスの指定
以下のURLで紹介されているように、購入したLibertyのライセンスに合わせてデプロイメントにアノテーションを追加します。
* [【参考】ライセンス・サービスによる Kubernetes での Liberty 使用量のトラッキング ](https://www.ibm.com/docs/ja/was-liberty/base?topic=liberty-tracking-usage-in-kubernetes-license-service)

Lab08で用意されている[deployment-with-license.yaml](./deployment-with-license.yaml)を適用します。imageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-with-license.yaml
```

新しい設定のLiberty Podが稼働していることを確認します。
```
# Pod名の確認
$ oc get pod
NAME				                    READY	     STATUS   		RESTART	        AGE
:
sample-app-b6b56d887-bchfw        1/1     Running             0          8s
sample-app-b6b56d887-xxxxx        1/1     Running             0          8s
```

## (3) ライセンスの追跡確認
License Operatorを利用して、WebSphere Libertyのライセンス追跡が行えているかを確認します。<br>
License Operatorの利用に関しては[こちら](https://github.com/IBM/ibm-licensing-operator/blob/latest/docs/License_Service_main.md)をご参照ください。
