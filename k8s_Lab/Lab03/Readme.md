# Lab03. 利用に合わせたLibertyのカスタマイズ
本手順では、デプロイメントの設定差異は最小限に、環境依存の設定を環境変数や外部からファイルで与えるための設定例を示します。

## (1) ConfigMapでの環境変数の設定

OSの環境変数と、Libertyの環境変数それぞれでConfigMapを用意します。
Lab03で用意されている[configmap-os-env.yaml](./configmap-os-env.yaml)と[configmap-liberty-env.yaml](./configmap-liberty-env.yaml)を適用します。
```
# OSの環境変数
$ oc apply -f configmap-os-env.yaml

# Libertyの環境変数
$ oc apply -f configmap-liberty-env.yaml
```

また、上記のConfigMapを使用するように、デプロイメントの定義ファイルを用意します。
Lab03で用意されている[deployment-with-env-01.yaml](./deployment-with-env-01.yaml)を適用します。imageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-with-env-01.yaml
```

実際にPodにログインして、ログの出力や環境変数に従った動作になっているかを確認します。

```
# Pod名の確認
$ oc get pod
NAME                            READY	     STATUS   		RESTART	        AGE
:
sample-app-b6b56d887-bchfw        1/1       Running               0          8s

# oc rsh <Pod名>でLibertyコンテナにログイン
$ oc rsh sample-app-b6b56d887-bchfw

# 日本標準時(JST)で時刻設定されていることを確認
sh-4.4$ date
Tue Nov 30 23:51:44 JST 2021

# メッセージが日本語で表示されていることを確認
sh-4.4$ cat /logs/messages.log
:
[2021/12/01 12:01:08:922 JST] 0000002b com.ibm.ws.kernel.feature.internal.FeatureManager            A CWWKF0011I: defaultServer サーバーは、Smarter Planet に対応する準備ができました。defaultServer サーバーは 6.380 秒で始動しました。
```

## (2) ConfigMapでのserver.xmlの設定追加/上書き
Libertyのserver.xmlに追記したい設定をConfigMapで追加します。
本手順では、監視用のメトリクスを収集できるように設定を追加します。

Lab03で用意されている[configmap-liberty-config-add.yaml](./configmap-liberty-config-add.yaml) (Libertyの構成フラグメント、すなわち設定差分ファイル)を適用します。
```
$ oc apply -f configmap-liberty-config-add.yaml
```

また、上記のConfigMapを使用するように、デプロイメントの定義ファイルを用意します。
Lab03で用意されている[deployment-with-env-02.yaml](./deployment-with-env-02.yaml)を適用します。imageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-with-env-02.yaml
```

Podが更新されたことを確認します。
```
$ oc get pod
NAME                            READY	     STATUS     RESTART	        AGE
:
sample-app-b6b56d887-flzkk      1/1         Running           0          8s
```

実際に動作を確認します。以下のURLにアクセスし、メトリクスが取得できることを確認します。
```
https://<アプリのFQDN名>/metrics
```

## (3) Secretでの環境変数の埋め込み
例えば、アプリケーションで使用するユーザー情報(ユーザー名, パスワード)を環境変数から設定することを考えます。
OpenShift (Kubernetes)ではConfigMapよりもSecretを用いてユーザー情報を保管します。

以下のコマンドでユーザー情報を「secret-liberty-auth」という名前のSecretとして作成します。
```
oc create secret generic <シークレット名> \
    --from-literal=username=<ユーザー名に置き換え> \
    --from-literal=password=<パスワードに置き換え> \
    --type=kubernetes.io/basic-auth
```

実行例は以下の通りです。
```
$ oc create secret generic secret-liberty-auth \
    --from-literal=username=myroot \
    --from-literal=password=mypasswd \
    --type=kubernetes.io/basic-auth
```

Lab03で用意されている[deployment-with-env-03.yaml](./deployment-with-env-03.yaml)を適用します。imageの項目を自身のイメージ名に更新した上で適用します。
Liberty Podのデプロイ後、Secretで設定されたユーザー情報が利用可能になります。
```
$ oc apply -f deployment-with-env-03.yaml
```

デプロイが完了したら、以下のURLにアクセスします。
デフォルトで設定されていたユーザー情報(ユーザー名: user1, パスワード: change1me)は利用できず、新しいユーザーでログインできることを確認します。
```
https://<アプリのFQDN名>/snoop
```
