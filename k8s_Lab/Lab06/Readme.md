# Lab06. ログとダンプの永続化と転送
問題判別やお客様要件上、参照するログやダンプファイルは指定期間は保持するケースが大半です。
本手順では、主にファイルとして出力されるダンプファイルを永続ストレージに保管する、あるいは外部のストレージサービスに転送するための構成例を示します。

## ダンプファイルの出力先を変更する
Libertyのダンプファイルの出力先は環境変数で変更が可能です。

Lab06で用意されている[configmap-liberty-env.yaml](configmap-liberty-env.yaml)を適用し、各ダンプファイルの出力先ディレクトリを/var/data/crashに変更します。
```
$ oc apply -f configmap-liberty-env.yaml
```

## 設定1. 永続ストレージに保管する
このケースでは、Podが稼働するWorker Nodeの外で提供されるストレージにダンプファイルを指定したディレクトリ(/var/data/crash)に出力するための構成を行います。
これまでの構成でDeploymentとして複数台のPodで稼働するように設定しています。この構成の場合、ストレージはRWX(ReadWriteMany)をサポートするストレージが必要です。

以下NFSストレージ(サーバー)が利用可能であるとします。

Lab06で用意しているpvc.yamlを開きます。利用環境に合わせてストレージクラスの設定を変更し、適用します。<br>
IBM Cloudにおいて、東京リージョンのtok02ゾーンでClassic InfrastructureでROKSを利用している場合、以下のような設定を行います。
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: liberty-pvc
 labels:
   billingType: hourly
   region: jp-tok
   zone: tok02
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 20Gi
 storageClassName: ibmc-file-gold-gid
```

ファイルの編集が完了したら、OpenShiftに適用します。
```
$ oc apply -f pvc.yaml
```

以下のコマンドを実行し、ストレージが利用可能であることを確認します。
```
$ oc get pvc
NAME                          STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS            AGE
:
liberty-pvc                   Bound    pvc-5c123e51-4d6e-4d26-bfda-6742e622b687   20Gi       RWX            ibmc-file-gold-gid      86s
```

ストレージが用意できたら、Lab06で用意した[deployment-with-dump-pvc.yaml](./deployment-with-dump-pvc.yaml)を用いてデプロイメントを更新します。imageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-with-dump-pvc.yaml
```

Podが稼働していることを確認します。
```
$ oc get pod
NAME                                     READY   STATUS      RESTARTS   AGE
:
open-liberty-operator-58596c958b-rchkz   1/1     Running     0          15m
```

実際にダンプファイルを出力する想定のディレクトリ(/var/data/crash)が存在しているかを確認するため、Podにログインし、マウント状況を確認します。
```
$ oc rsh sample-app-59b54cb774-5gq9h 
sh-4.4$ mount |grep "/var/data/crash"
fsf-tok0202j-fz.service.softlayer.com:/IBM02SEV2261318_8/data01 on /var/data/crash type nfs4 (rw,relatime,vers=4.1,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=10.133.26.102,local_lock=none,addr=10.201.174.111)
```

さらにダンプが出力されるかを確認します。server javadumpコマンドで各ダンプファイルをまとめて出力します。
```
sh-4.4$ server javadump defaultServer --include=thread,heap,system

サーバー defaultServer をダンプ中です。
サーバー defaultServer のダンプが /var/data/crash/javacore.20211210.005421.1.0004.txt で完了しました。
サーバー defaultServer のダンプが /var/data/crash/heapdump.20211210.005421.1.0005.phd で完了しました。
サーバー defaultServer のダンプが /var/data/crash/core.20211210.005422.1.0006.dmp で完了しました。

sh-4.4$ ls /var/data/crash/
core.20211210.005422.1.0006.dmp  heapdump.20211210.005421.1.0005.phd  javacore.20211210.005421.1.0004.txt
```

ダンプファイル出力後、Libertyプロセスを停止します。server stopコマンドをコンテナ上で実行すると、セッションが切れ、Podの再起動が行われます。
```
sh-4.4$ server stop defaultServer

サーバー defaultServer を停止中です。
command terminated with exit code 137
```

再度Podにログインし、ダンプファイルの出力ディレクトリを確認します。
永続ストレージに出力しているため、ファイルは削除されていません。
```
$ oc get po
NAME                                     READY   STATUS      RESTARTS   AGE
:
open-liberty-operator-58596c958b-rchkz   1/1     Running     1          17m
:

$ oc rsh sample-app-59b54cb774-5gq9h 

sh-4.4$ ls /var/data/crash/
core.20211210.005422.1.0006.dmp  heapdump.20211210.005421.1.0005.phd  javacore.20211210.005421.1.0004.txt
```

Podからログアウトし、Podを削除します。
```
$ oc delete po sample-app-59b54cb774-5gq9h 
pod "sample-app-59b54cb774-5gq9h" deleted
```

削除後、Podの状態を確認し、新しく作成されたPodにログインします。
Podが失われたとしても、永続ストレージにダンプファイルが出力されているため、ダンプファイルが失われることがないことが確認できます。

## 設定2. 複数コンテナでダンプ出力先ディレクトリを共有する
このケースでは、Podの中でLibertyコンテナとは別にコンテナを並行稼働させる構成を取ります。
ダンプ出力先ディレクトリを両コンテナで一時ストレージ(emptyDir)に共有します。<br>
本手順のサンプルでは実装していませんが、並行して稼働しているコンテナで新規にダンプファイルが出力されたら別のディレクトリに送付する構成を実装することでダンプファイルの消失を防ぐ構成を実装可能です。

Lab06で用意した、[deployment-with-dump-emptydir.yaml](./deployment-with-dump-emptydir.yaml)を用いてデプロイメントを更新します。imageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-with-dump-emptydir.yaml
```

Podが稼働していることを確認します。
```
$ oc get pod
NAME                                     READY   STATUS      RESTARTS   AGE
:
open-liberty-operator-58596c958b-rchkz   1/1     Running     0          15m
```

実際にダンプファイルを出力する想定のディレクトリ(/var/data/crash)が存在しているかを確認するため、Podにログインします。
Pod内に2つのコンテナが存在するため、明示的に-cオプションでコンテナ名を指定してログインします。
```
$ oc rsh sample-app-59b54cb774-5gq9h -c app
sh-4.4$ mount |grep "/var/data/crash"
:
ToDo: 結果を反映
```

実際にダンプが出力されるかを確認します。server javadumpコマンドで各ダンプファイルをまとめて出力します。
```
sh-4.4$ server javadump defaultServer --include=thread,heap,system

サーバー defaultServer をダンプ中です。
サーバー defaultServer のダンプが /var/data/crash/javacore.20211210.010153.1.0004.txt で完了しました。
サーバー defaultServer のダンプが /var/data/crash/heapdump.20211210.010154.1.0005.phd で完了しました。
サーバー defaultServer のダンプが /var/data/crash/core.20211210.010155.1.0006.dmp で完了しました。

sh-4.4$ ls /var/data/crash/
core.20211210.010155.1.0006.dmp  heapdump.20211210.010154.1.0005.phd  javacore.20211210.010153.1.0004.txt
```

ダンプファイル出力後、一度exitでコンテナをログアウトし、並行稼働しているコンテナに入ります。
並行稼働しているコンテナ上では「/mount」に共有している一時ストレージ(emptyDir)がマウントされています。
```
$ oc rsh sample-app-59b54cb774-5gq9h -c helper
sh-4.4$ mount |grep "/mount"
:
ToDo: 結果を反映

sh-4.4$ ls -al /mount
core.20211112.142804.1.0003.dmp  heapdump.20211112.142803.1.0002.phd  javacore.20211112.142803.1.0001.txt
```

再度helperコンテナからログアウトし、Libertyコンテナにログインします。そして、server stopコマンドをコンテナ上で実行します。
```
$ oc rsh sample-app-59b54cb774-5gq9h -c app
sh-4.4$ server stop defaultServer

Stopping server defaultServer.
command terminated with exit code 137
```

Libertyコンテナが再起動されるを待ちます。
再度Podにログインし、ダンプファイルの出力ディレクトリを確認します。
永続ストレージに出力しているため、ファイルは削除されていません。
```
$ oc get po
NAME                                     READY   STATUS      RESTARTS   AGE
:
open-liberty-operator-58596c958b-rchkz   1/1     Running     1          17m
:

$ oc rsh sample-app-59b54cb774-5gq9h 

sh-4.4$ ls /var/data/crash/
core.20211112.142804.1.0003.dmp  heapdump.20211112.142803.1.0002.phd  javacore.20211112.142803.1.0001.txt
```

Podからログアウトし、Podを削除します。
```
$ oc delete po sample-app-59b54cb774-5gq9h 
pod "sample-app-59b54cb774-5gq9h" deleted
```

emptyDirを利用しているこちらの例の場合は、Podが削除されるとemptyDirで保管していたデータも削除になります。<br>
設定1と同様に再作成されたPodにログインし、ダンプファイル出力ディレクトリ(/var/data/crash)内のデータが失われていることが確認できます。<br>
設定2の構成の場合は、Podが削除されるケースでは永続化が期待できないため、Libertyコンテナと並行稼働しているコンテナ内で定期的に外部ストレージ・サービスにデータを転送する仕組みの作成が必要です。
