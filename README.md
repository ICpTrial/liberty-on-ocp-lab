# WebSphere Liberty on OpenShift Container Platform 利用ガイド
OCP (OpenShift Container Platform) 上でWebSphere Libertyを利用する際の設定をStep by Stepで学ぶLab資料です。<br>
本リポジトリでは、Kubernetesリソースを直接記述してデプロイする場合と、Kubernetes Operatorを利用する場合の２通りの方法でWebSphere Libertyのデプロイを学びます。<br>
<br>
Helmによるデプロイは「[WebSphere Liberty on OpenShift Container Platform 利用ガイド ~ Helm編 ~](https://community.ibm.com/HigherLogic/System/DownloadDocumentFile.ashx?DocumentFileKey=62f6245d-4511-da1f-e67a-695473cfb707)」を参照してください。

## 目次

| ディレクトリ | 内容 |
----|---- 
| [k8s_Lab](./k8s_Lab/) | Kubernetesの仕様にしたがって、LibertyデプロイのためのDeploymentの定義を作成し、デプロイを行います |
| [Operator_Lab](./Operator_Lab/) | Kubernetes OperatorであるOpenLiberty Operatorを活用して、Libertyのデプロイを行います |

## 前提
* [Red Hat OpenShift on IBM Cloud](https://cloud.ibm.com/docs/openshift?topic=openshift-getting-started) などのOCP環境が用意されていること
* Cluster Adminの権限を持つユーザーで操作可能であること (OpenLiberty Operatorを導入するための必要)
* OpenShift CLI (oc)が操作端末に導入されていること ([参考](https://access.redhat.com/documentation/ja-jp/openshift_container_platform/4.9/html/cli_tools/_openshift-cli-oc))
* Git Clientが導入済みであること
