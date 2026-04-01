---
title: "【k8s入門】ノートPC一つでできるKubernetesのオンボーディング"
emoji: "☸️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes"]
published: false
---
# 概要
「Kubernetes（k8s）って難しそう……」
分厚い入門書や解説を読み始めては挫折していませんか？

k8sの仕組みやコンポーネントを座学で完璧に理解することは難しいです。

でも、**まずは動かしてみる**ことから始めれば、k8sが何をしてくれるツールなのか少しずつわかっていくはずです。

この記事では、難しいアーキテクチャの解説は一切しません。
**ノートPC一つで、世界中で使われているオーケストレーションツールの凄さを体感する**ことだけをゴールにします。

## やらないこと
- k8s本体、コンポーネントの解説
- コンテナの解説

## この記事で体験できること
- ローカル環境へのk8sセットアップ
- nginxコンテナのデプロイ
- ブラウザからのアクセス確認

## あるといい前提
- コンテナ（Dockerなど）を触ったことがある
  - コンテナの利点（移植性、再現性、独立性、軽量性、スケーラビリティ）
- ネットワークの基本がわかる
  - TCP/IPについてやDockerでポートフォワードについて
- Linuxコマンドを叩くのに抵抗がない
  - ログの確認やファイルの編集など

## トラブルシューティング
これからDocker Desktopのセットアップやk8sクラスタの起動を行いますが、もしうまくいかない場合は、クラスタをリセットするか、Docker Desktopを再起動してみてください。

# 環境構築
Macでは`OrbStack`、`collima`などの方が高速であるためそれらを使っても構わないが、
本記事では、MacでもWindowsでも同じ手順で導入できる `Docker Desktop` を前提に進めます。（k8sの操作は同じですのでDocker Desktopでなくても構いません）

### Docker Desktopでk8sクラスタを起動
Docker Desktopを起動してサイドバーのKubernetesを選択し、`Create cluster`をクリックします。
![docker-desktop-k8s](/images/docker-desktop-k8s.png)

今回は環境に依存しないよう`kind`(Kubernetes IN Docker)を選択して、詳細設定はデフォルトのままインストールします。
![docker-desktop-k8s-kind](/images/docker-desktop-k8s-kind.png)

インストールが完了すると以下の画面のようになります。`kubectl create deployment`や`kubectl run`などがかかれていますが、今回の記事では命令的な操作は行わず、yamlファイルを用いて宣言的な操作を行います。
![docker-desktop-k8s-success](/images/docker-desktop-k8s-success.png)
### `kubectl`で動作確認
クラスタが起動したら、ターミナルでコマンドを実行して動作確認をしましょう。

今動いているサーバー（ノード）の一覧を表示します。
```bash
kubectl get nodes
```
実行結果の例：
```
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   2m    v1.31.1
```
ここで`STATUS`が`Ready`になっていればOKです。



# nginxをデプロイする
まずはコピペでよいのでとりあえず動かしてみましょう。
nginxで簡単なWebサーバーを立ててみます。
### Deploymentsの作成
好きな場所に `deployment.yaml` というファイルを作成して以下の内容を貼り付けてください。
```yaml:deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

作成したら以下のコマンドを実行してデプロイします。
```bash
kubectl apply -f deployment.yaml
```

続いて、コンテナ(pod)が起動しているか確認しましょう。
```bash
kubectl get pods
```
実行結果の例：
```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-76f89f979c-abcde   1/1     Running   0          1m
```
ここで`STATUS`が`Running`になっていればOKです。

## ブラウザで見えるようにする
### serviceを作成
- 

## 動作確認


# k8sの素晴らしさ
### インフラをコードで管理する（IaC）
従来はコンテナの作成やネットワークの設定などを別々で管理する必要があったが、今回書いたようにyamlでなんでも定義ができる。そして
# 次にやること
### Ingress
