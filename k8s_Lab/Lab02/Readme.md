# Lab02. カスタム・イメージの作成
以下を設定します。
* イメージのビルド（自身のアプリを載せる, 日本語ロケールを追加する） → Dockerfile.defaultAppにロジックを追記
* ビルドしたカスタム・イメージでのデプロイ

動かしたいアプリケーションは以下を使用します。
- [https://github.com/WASdev/sample.DefaultApplication.git](https://github.com/WASdev/sample.DefaultApplication.git)

また、カスタム・イメージとして日本語ロケールが選択できるようにカスタマイズを行います。

## (1) イメージのビルド
作業PCにJavaのビルド環境(maven)が準備されていれば、手元のイメージをビルドし、イメージをアップロードを行うことが可能です。<br>
本手順では作業PC上ではなくOCP上でのイメージのビルドを想定し、OCPのDockerビルド機能を利用して、イメージをビルドします。

実行コマンド(Linuxの場合)
```
cat <Dockerfileのパス> | oc new-build --strategy=docker <リポジトリURL> --context-dir=<ビルド対象のディレクトリ> --name=<BuildConfig名> --to='<イメージストリーム名>' --dockerfile='-'
```

実行例
```
$ cat Dockerfile | oc new-build --strategy=docker https://github.com/WASdev/sample.DefaultApplication.git --context-dir=modernized --name=defaultapp --to='defaultapp-imagestream:1.0-SNAPSHOT' --dockerfile='-'
--> Found container image b50b31b (3 weeks old) from Docker Hub for "ibmcom/websphere-liberty:full-java11-openj9-ubi"

    Red Hat Universal Base Image 8 
    ------------------------------ 
    The Universal Base Image is designed and engineered to be the base layer for all of your containerized applications, middleware and utilities. This base image is freely redistributable, but Red Hat only supports Red Hat technologies through subscriptions for Red Hat products. This image is maintained by Red Hat and updated regularly.

    Tags: base rhel8

    * An image stream tag will be created as "websphere-liberty:full-java11-openj9-ubi" that will track the source image
    * A Docker build using a predefined Dockerfile will be created
      * The resulting image will be pushed to image stream tag "defaultapp-imagestream:1.0-SNAPSHOT"
      * Every time "websphere-liberty:full-java11-openj9-ubi" changes a new build will be triggered

--> Creating resources with label build=defaultapp ...
    imagestream.image.openshift.io "defaultapp-imagestream" created
    buildconfig.build.openshift.io "defaultapp" created
--> Success
```

実行後は以下のコマンドでビルド状況を確認します。 ログ出力を止めたい場合、ビルド自体の処理が終了するのを待つか、Ctrl+Cでログの出力を中断します。
```
oc logs bc/<BuildConfig名> -f
```

以下のように、「Push successful」が表示されたら正常にイメージがビルドされています。
```
$ oc logs bc/defaultapp -f
Cloning "https://github.com/WASdev/sample.DefaultApplication.git" ...
	Commit:	fe3701371fbb0724a663454f6075f7ebb9c5849e (upgrade liberty-maven-plugin version)
	Author:	Cindy High <cthigh@us.ibm.com>
:
Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/liberty-guide-demo/defaultapp-imagestream@sha256:494b43b2715f41ec775864438f08a47d93fa57268eda02204a0a38c34b3b542c
Push successful
```

ビルドしたイメージ(ダイジェスト値あり)を確認します。上記出力の「Successfully pushed」の後に記載されるイメージ・ストリーム情報を確認します。<br>
上記例では「image-registry.openshift-image-registry.svc:5000/liberty-guide-demo/defaultapp-imagestream@sha256:494b43b2715f41ec775864438f08a47d93fa57268eda02204a0a38c34b3b542c」が作成されたイメージになります。

## (2) ビルドしたカスタム・イメージでのデプロイ
(1)の最後に確認したイメージ情報をdeployment.yamlのimageの値に書き込みます。

入力後、デプロイメントを更新します。
```
$ oc apply -f deployment.yaml
```

デプロイメントの更新後、Podが再度Runningになっていることを確認します。
```
$ oc get pod
NAME                                    READY   STATUS      RESTARTS   AGE
defaultapp-1-build                      0/1     Completed   0          19m
:
sample-app-5f4ddd9b46-q8644             1/1     Running     0          2m37s
```

また、Webブラウザで「https://<ドメイン名>/snoop」にアクセスします(ユーザー/パスワードはuser1/change1meになります)。
