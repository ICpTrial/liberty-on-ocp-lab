# Kubernetesリソースの基本とLibertyのデプロイ
Kubernetesの仕様にしたがって、LibertyデプロイのためのDeploymentの定義を作成し、デプロイを行います。

## 目次
以下の構成でLabを用意しています。

| Lab | タイトル | 内容 |
----|----|---- 
| [Lab01](./Lab01/) | 利用イメージの取得とデプロイ | 提供済みの WebSphere Liberty イメージでデプロイします |
| [Lab02](./Lab02/) | カスタムイメージの作成 | アプリケーションを含めたWebSphere Libertyイメージをビルドし、デプロイします |
| [Lab03](./Lab03/) | 利用に合わせたLibertyのカスタマイズ | アプリケーションを含めたWebSphere Libertyイメージをビルドし、デプロイします |
| [Lab04](./Lab04/) | Podの分散配置 | Liberty Podの複数デプロイと分散配置を行います |
| [Lab05](./Lab05/) | Podのヘルスチェックの実装 | Podの健全性の確認のため、ヘルスチェック(Probe)を設定します |
| [Lab06](./Lab06/) | ログとダンプの永続化と転送 | Libertyのログやダンプを永続化するために永続ストレージのマウントを行います |
| [Lab07](./Lab07/) | 割り当てリソースの制限 | Liberty Podに割り当てるリソースに制限を与えます |
| [Lab08](./Lab08/) | ライセンスの適用と追跡 | WebSphere Liberyの利用に合わせ、ライセンスの追跡のための設定を行います |

## 前提
[こちら](https://github.com/ICpTrial/liberty-on-ocp-lab)をご参照ください。

## 参考URL
* [IBM Document - コンテナーでの Liberty の実行](https://www.ibm.com/docs/ja/was-liberty/base?topic=running-liberty-in-container)
