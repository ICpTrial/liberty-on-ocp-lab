# OpenLiberty Operatorを用いたWebSphere Libertyのデプロイ
Kubernetes Operatorを活用して、WebSphere Liberty実行環境のデプロイを行います。

## 目次
以下の構成でLabを用意しています。

| Lab | タイトル | 内容 |
----|----|---- 
| [Lab01_example-app-1](./Lab01_example-app-1/) | Liberty環境のデプロイ - デプロイ例1 | WebSphere Libertyイメージをビルドし、シンプルな構成で WebSphere Liberty をデプロイします |
| [Lab01_example-app-2](./Lab01_example-app-2/) | Liberty環境のデプロイ - デプロイ例2 | メッセージ・ブローカー(Kafka)によって複数マイクロサービスを接続する構成で WebSphere Liberty をデプロイします |
| [Lab02_session-with-infinispan](./Lab02_session-with-infinispan/) | 設定のカスタマイズ例① – セッション共有(Infinispan) | JCache構成の例としてInfinispanをセッションストアとしてセッション共有する構成を行います |
| [Lab02_session-with-db2](./Lab02_session-with-db2/) | 設定のカスタマイズ例① – セッション共有(Db2) <br> 設定のカスタマイズ例② | Db2をセッションストアとしてセッション共有する構成を行います <br> また、構成したDb2の接続情報を環境変数として埋め込むための設定を行います |
| [Lab03_dump-and-trace](./Lab03_dump-and-trace/) | ログとダンプの永続化と転送 | Libertyに対しログとダンプの永続化を行います |
| [Lab04_metrics](./Lab04_metrics/) | モニタリング | OCP提供のPrometheusを利用してLibertyのプラットフォーム情報を取得します |
| [Lab05_autoscaling ](./Lab05_autoscaling) | オートスケーリング | Libertyに対し自動スケーリングの設定を行います |

## 前提
[こちら](https://github.com/ICpTrial/liberty-on-ocp-lab)をご参照ください。<br>
また、各Labの実施にあたってはOpenLiberty Operatorを事前に導入しておく必要があります。詳しくは[こちら](https://github.com/ICpTrial/liberty-on-ocp-lab)をご参照ください。

## 参考URL
* [IBM Document - コンテナーでの Liberty の実行](https://www.ibm.com/docs/ja/was-liberty/base?topic=running-liberty-in-container)
