# HPAを利用した自動スケーリング
Libertyに対し、「Horizontal Pod Autoscaler (HPA)」によるオートスケーリングの定義を行います。<br>
HPAを設定することで、CPU使用率に応じて自動でPodのスケーリングを行うことが可能になります。

## 1. OpenLibertyApplicationリソースの作成
[olapp_hpa.yaml](./olapp_hpa.yaml)を開き、設定を確認します。<br>
spec.autoscalingの項目で自動スケーリングの動作を指定します。本手順の例の場合、最小Pod数は1, 最大Pod数は3, CPU使用率の閾値は80%を指定しています。

```
$ cat olapp_hpa.yaml
apiVersion: apps.openliberty.io/v1beta2
kind: OpenLibertyApplication
metadata:
  name: hpa-app
  labels:
    name: hap-app
spec:
  applicationImage: ibmcom/websphere-liberty
  env:
    - name: WLP_LOGGING_MESSAGE_FORMAT
      value: "json"
    - name: WLP_LOGGING_MESSAGE_SOURCE
      value: "message,trace,accessLog,ffdc,audit"
  resources:
    requests:
      cpu: 100m
    limits:
      cpu: 300m
  autoscaling:
    maxReplicas: 3
    minReplicas: 1
    targetCPUUtilizationPercentage: 80
```

確認した[olapp_hpa.yaml](./olapp_hpa.yaml)をもとに、リソースを作成します。
```
$ oc apply -f olapp_hpa.yaml
openlibertydump.openliberty.io/hap-app created

$ oc get po -l app.kubernetes.io/name=hpa-app
NAME                      READY   STATUS    RESTARTS   AGE
hpa-app-ddd8cb974-dfwjz   1/1     Running   0          5m4s
```

HPAリソースが合わせて作成されていることを確認します。
```
$ oc get hpa
NAME      REFERENCE            TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hpa-app   Deployment/hpa-app   9%/80%          1         3         1          4m50s
```

## 2. 自動スケーリングの動作確認
例えば、以下のようにPodにログインし、無限ループ処理を実行させてCPU負荷を高くする方法が考えられます。

```
$ oc rsh hpa-app-ddd8cb974-dfwjz 

sh-4.4$ while true;do true ; done &
[1] 110749
sh-4.4$
```

上記実行後、別ターミナルでPodの一覧を確認します。しばらく待つと新しいPodが作成されます。
```
$ oc get hpa
NAME       REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hpa-app    Deployment/hpa-app    51%/80%         1         3         3          9m55s

$ oc get pod                                                        
NAME                                                READY   STATUS       RESTARTS   AGE
hpa-app-ddd8cb974-dfwjz                             1/1     Running      0          4m55s
hpa-app-ddd8cb974-bdn2v                             1/1     Running      0          9m55s
hpa-app-ddd8cb974-nw74p                             1/1     Running      0          4m55s
```

また、実行したループ処理を停止させます。Podにログインしたターミナルで以下を実行します。
```
# 上記でwhileコマンドを実行した際のPIDを指定してループ処理を停止
sh-4.4$ kill -9 110749
```

上記実行後、しばらく待つとPodが削除されていき、1台に戻ります。
```
$ oc get hpa
NAME       REFERENCE             TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hpa-app    Deployment/hpa-app    9%/80%          1         3         1          14m

$ oc get pod
NAME                                                READY   STATUS       RESTARTS   AGE
hpa-app-ddd8cb974-nw74p                             1/1     Running      0          14m
```
