# Lab05. Podのヘルスチェックの実装
Pod(コンテナ)が異常と判断される状況について、複数のケースが考えられます。

単純なプロセスダウンに関してはOpenShift(Kubernetes)としてデフォルトで再起動を試みます。

それ以外の、例えばアプリケーションが適切なポートをListenしているか、リクエスト応答可能か(プロセスがハングしていないか)など、より実践的にPodに対する正常性を確認するためには以下の設定を行います。
* startupProbe
* livenessProbe
* readinessProbe

## (1) Podが正常に起動したことを確認する (startupProbe)
Lab05で用意されている、正常に稼働監視が行われる定義ファイル[deployment-with-probe.yaml](./deployment-with-probe.yaml)を確認します。
27-31行目のようにstartupProbeの項目を設定します。

本手順では、24-26行目で設定しているTCPポートをLibertyプロセスがListenしているかどうかで正常に起動したかを判断しています。
実際の利用シーンでは、アプリケーションの健全性が判断できるURLをアプリケーション側で用意し、利用するのが望ましいです。
* livenessProbeに設定するようなアプリケーションの稼働が判断できるURLをアプリケーション側で設定する
* /logs/messages.logで「CWWKZ0001I: <自身のアプリケーション名> が開始されました」というメッセージを確認するようなスクリプトを用意し状態を判断する

```
        startupProbe:
          tcpSocket:
            port: http 
          failureThreshold: 30
          periodSeconds: 10
```

## (2) Podが正常に稼働していることを確認する (livenessProbe)
Podが正常に稼働しているかどうかを判断するための項目であり、このProbeで異常と検知された場合はPodの再起動が行われます。
Lab05で用意されている[deployment-with-probe.yaml](./deployment-with-probe.yaml)の32-40行目に以下の設定にlivenessProbeの項目があります。

本手順では、Lab03で設定したメトリクス収集用のURLに対し、定期的にリクエストを送付し、応答を確認します。
実際の利用シーンでは、アプリケーションの健全性が判断できるURLをアプリケーション側で用意し、利用するのが望ましいです。

```
        livenessProbe:
          httpGet:
            path: /metrics
            port: http
            httpHeaders:
            - name: X-Custom-Header
              value: liberty-probe
          initialDelaySeconds: 15
          periodSeconds: 10
```

## (3) Podが正常にリクエストを処理できるかを確認する (readinessProbe)
Podが正常にリクエストを処理できるかを判断するための項目であり、このProbeで異常と検知された場合は一時的にリクエストの割り振り対象から外されます。
livenessProbeと異なり、Podは自動的に再起動されない点には注意が必要です。

他のProbeと同様に、Lab05で用意されている[deployment-with-probe.yaml](./deployment-with-probe.yaml)の41-49行目に以下の設定にreadinessProbeの項目があります。

本手順では、Lab03で設定したメトリクス収集用のURLに対し、定期的にリクエストを送付し、応答を確認します。
実際の利用シーンでは、アプリケーションの健全性が判断できるURLをアプリケーション側で用意し、利用するのが望ましいです。
```
        readinessProbe:
          httpGet:
            path: /metrics
            port: http 
            httpHeaders:
            - name: X-Custom-Header
              value: liberty-probe
          initialDelaySeconds: 15
          periodSeconds: 5
```

## (4) Probe失敗の動作を確認する (失敗例1. livenessProbe)
Lab05で用意されている[deployment-with-probe-fail-liveness.yaml](./deployment-with-probe-fail-liveness.yaml)を適用します。imageの項目を自身のイメージ名に更新した上で適用します。
```
$ oc apply -f deployment-with-probe-fail-liveness.yaml
```

この例の場合、適用後は定期的にPodが再起動されている(RESTARTSカウントの値が0でない)ことが確認できます。
```
$ oc get pod
NAME                          READY             STATUS          RESTARTS        AGE
:
sample-app-776f577766-mvb9j     1/1            Running                 2      2m27s
sample-app-776f577766-z2vdf     1/1            Running                 2       114s
:
```

## (5) Probe失敗の動作を確認する (失敗例2. readinessProbe)
Lab05で用意されている[deployment-with-probe-fail-readiness.yaml](./deployment-with-probe-fail-readiness.yaml)を適用します。imageの項目を自身のイメージ名に更新した上で適用します。
```
oc apply -f deployment-with-probe-fail-readiness.yaml
```

この例の場合、適用後はPodに対するReadyの状態が「1/1」にならないことが確認できます。
```
$ oc get pod
NAME                             READY   STATUS             RESTARTS   AGE
:
sample-app-5b6b74fcf8-lmljk      0/1     Running            0          55s
:
```

実際にWebブラウザでリクエストを送信しても、正常な応答が返らないことが確認できます。
```
https://<アプリのFQDN名>/snoop
```

## (6) Probeを適用する (成功例)
Lab05で用意されている[deployment-with-probe.yaml](./deployment-with-probe.yaml)を適用します。imageの項目を自身のイメージ名に更新した上で適用します。
```
oc apply -f deployment-with-probe.yaml
```

適用後は正常にPodが起動していることを確認します。
```
$ oc get pod
NAME                            READY           STATUS             RESTARTS         AGE
:
sample-app-86d464c8f9-c7gdp       1/1          Running                    0         54s
sample-app-86d464c8f9-zp5f9       1/1          Running                    0         36s
:
```

実際にWebブラウザでリクエストを送信し、アプリケーションの結果が正しく返ることが確認できます。
```
https://<アプリのFQDN名>/snoop
```
