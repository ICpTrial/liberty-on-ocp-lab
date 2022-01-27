# Lab01. 利用イメージの取得とデプロイ
提供済みのWebSphere Libertyイメージを用いて、Liberty Podのデプロイを行います。

## (1) プロジェクトの作成
Podをデプロイするためのプロジェクト (Namespace)を作成します。

```
oc new-project <プロジェクト名>
```

本手順では、「liberty-guide-demo」という名前のプロジェクト名を作成し、実施しています。

## (2) サービス・アカウント (Service Account)の作成
Podが稼働しOCPの各リソース（ConfigMapやPVCなど）にアクセスするための認証情報として作成します。
そして、サービス・アカウントに対し、最低限必要と思われる権限を作成・付与します。

```
oc apply -f serviceaccount.yaml
oc apply -f scc.yaml
```

作成したSecurity ContextをService Accountに付与します。
```
oc adm policy add-scc-to-user ibm-websphere-liberty-scc -z sample-app
```

## (3) デプロイメント (Deployment)の作成
プリセット・イメージを使用して、LibertyのPodが稼働するための設定を定義します。
```
oc apply -f deployment.yaml
```

## (4) サービス (Service)の作成
(3) で作成したPodに対し、他の用途で使用しているPodからアクセスするためのエンドポイントを定義します。
```
oc apply -f service.yaml
```

## (5) ルート (Route)の作成
コンポーネントがアプリケーションを外部へ公開するための設定を定義します。
外部からのアクセスのためのRouteの設定は、ファイル内のhostの項目を編集してから適用します。
```
oc apply -f route.yaml
```
