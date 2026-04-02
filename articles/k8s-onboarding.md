---
title: "【k8s入門】ノートPC一つでできるKubernetesのオンボーディング"
emoji: "☸️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes"]
published: false
---
# 1. 概要
「Kubernetes（k8s）って難しそう……」
分厚い入門書や解説を読み始めては挫折していませんか？

k8sの仕組みやコンポーネントを座学で完璧に理解することは難しいです。

でも、**まずは動かしてみる**ことから始めれば、k8sが何をしてくれるツールなのか少しずつわかっていくはずです。

この記事では、難しいアーキテクチャの解説は一切しません。サーバーやクラウドは必要ありません。
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
  - TCP/IPの基本概念やDockerでのポートフォワードの仕組みについて
- Linuxコマンドを叩くのに抵抗がない
  - ログの確認やファイルの編集など

## トラブルシューティング
これからDocker Desktopのセットアップやk8sクラスタの起動を行いますが、もしうまくいかない場合は、クラスタをリセットするか、Docker Desktopを再起動してみてください。

# 2. 環境構築
Macでは`OrbStack`や`Colima`などのツールのほうが動作が高速なため、そちらをお使いいただいても構いませんが、
本記事では、MacでもWindowsでも同じ手順で導入できる `Docker Desktop` を前提に進めます。（k8sの振る舞いは同じであるため、Docker Desktopでなくても問題ありません）

## Docker Desktopでk8sクラスタを起動
Docker Desktopを起動してサイドバーのKubernetesを選択し、`Create cluster`をクリックします。
![docker-desktop-k8s](/images/docker-desktop-k8s.png)

今回は環境に依存しないよう`kind`(Kubernetes IN Docker)を選択して、詳細設定はデフォルトのままインストールします。
![docker-desktop-k8s-kind](/images/docker-desktop-k8s-kind.png)

インストールが完了すると以下の画面のようになります。`kubectl create deployment`や`kubectl run`などが記載されていますが、今回の記事では命令的な操作は行わず、yamlファイルを用いて宣言的な操作を行います。
![docker-desktop-k8s-success](/images/docker-desktop-k8s-success.png)
## 動作確認 - `kubectl`でノード一覧を表示
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



# 3. Nginxをデプロイする
まずはコピペでよいのでとりあえず動かしてみましょう。
Nginxで簡単なWebサーバーを立ててみます。
## Deploymentsの作成
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

# 実行結果の例：
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-xxxxxxxx   1/1     Running   0          1m
```
ここで`STATUS`が`Running`になっていればOKです。

## あえてpodを落としてみる
`get pods`でpodを確認できたはずです。NAME列に表示されているpod名を指定して削除してみましょう。
```bash
kubectl delete pod [Pod名]
```

もう一度`get pods`を実行すると、新しいpodが起動しているはずです。
```bash
kubectl get pods

# すると一瞬で別のpodが起動します。
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-xxxxxxxx   1/1     Running   0          1m
```
このようにpodが死んでも自動的に復旧してくれます。

## `deployment.yaml`の解説
先ほど作成した`deployment.yaml`について特に重要な3点だけ解説します。
```yaml:deployment.yaml
apiVersion: apps/v1
kind: Deployment       # ① 何を作りたいか？（今回はDeployment＝コンテナの管理人）
metadata:
  name: nginx-deployment
spec:
  replicas: 1          # ② いくつ動かしたいか？（コンテナを1つ）
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx     # ③ ラベル（後でネットワークを繋ぐための目印）
    spec:
      containers:
      - name: nginx
        image: nginx:latest  # Dockerでお馴染みのイメージ指定
        ports:
        - containerPort: 80  # コンテナが待ち受けるポート番号
```
① `kind: Deployment`  
`kind: ○○`で作成するリソースの種類を指定します。k8sでは、コンテナを直接単体で起動するのではなく、「`Deployment`」というコンテナの管理人を作ります。コンテナイメージなどのコンテナの定義を主に行います。

② `spec.replicas: 1`  
`replicas`はレプリカの数、つまりコンテナの数を指定します。今回は1つですが、2にすればコンテナが2つ起動します。

③ `spec.template.metadata.labels`  
`labels`は後でネットワークを繋ぐための目印として使います。

:::message
yamlはインデントで階層を表現しますが、ドキュメント等では②、③のように階層をドット記法で表現することが多いです。
:::

# 4. ブラウザで見えるようにする
無事にDeploymentを作成し、nginxのコンテナ(pod)が起動しましたね。

しかし、実はこの状態ではまだ外部からアクセスできません。

### なぜまだアクセスできないのか
Dockerを使っていた時、コンテナにアクセスするには起動時に `-p 8080:80` のようにポートフォワードを設定する必要がありました。

k8sでも同様に、ポートフォワードをしたいところですが、PodのIPアドレスは固定ではないため直接アクセスは不可能です。

そこで、Podにアクセスするためのルーターとなる「`Service`」を作成します。

外からのアクセスは一旦この「Service」が受け取り、さきほどDeploymentで設定した`spec.template.metadata.labels`の`app: nginx`という**ラベル（目印）**を探します。そして、見つけたPodへ通信を自動で振り分けてくれます。

このように、**Serviceの `spec.selector.app` と Deploymentの `spec.template.metadata.labels.app` が同じ値（`nginx`）であることで、これら2つが紐付く**という仕組みになっています。この「ラベルを使った紐付け」は初心者の方が一番躓きやすいポイントなので、ぜひ覚えておいてください。

それでは実際にServiceを作成してみましょう。
## serviceを作成
好きな場所に `service.yaml` というファイルを作成して以下の内容を貼り付けてください。
```yaml:service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
作成したら以下のコマンドを実行してServiceを作成します。
```bash
kubectl apply -f service.yaml
```


## 動作確認
### `port-forward`でアクセス
今回は環境依存をなるべくなくすためにDockerコンテナ内でk8sを動かしているので、PCのブラウザからアクセスするために`kubectl port-forward`というコマンドを使います。実はpodに直接アクセスすることもできますが、podはいつ死ぬかわからないので、Serviceを通してアクセスします。

```bash
kubectl port-forward service/nginx-service 8080:80
```

### ブラウザで確認
ブラウザで`http://localhost:8080`にアクセスして、nginxのデフォルトページが表示されれば成功です。

# 5. 後片付け
1. `port-forward`は`Ctrl+C`で終了できます。
2. 作成したリソースを削除するには`kubectl delete`コマンドを使います。  
リソースを消すときも`kubectl apply`で作成したときと同じようにyamlファイルを指定します。
```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
```
これで綺麗に削除できました。確認したい場合は`kubectl get <リソースの種類>`コマンドを打ってみましょう。

3. k8sクラスタ自体の削除  
Docker Desktopのサイドバーにある「Kubernetes」メニューをクリックし、作成したクラスタの右側にあるゴミ箱アイコン（Delete）や「Stop」アクションを選択することで削除できます（環境によって `Settings (歯車アイコン) > Kubernetes > Reset Kubernetes Cluster` を実行する場合もあります）。

# 6. k8sの素晴らしさ
今回は簡単なNginxのデプロイだけでしたが、この記事の体験を通じて、k8sがなぜ世界中で使われているのか、その片鱗を味わうことができたはずです。

## インフラをコードで管理する（IaC）
今回、画面をポチポチしたり、コマンドだけでデプロイすることはなかったですよね。

k8sに適用したものは**設計図**(yamlファイル)だけです。

この理想を書いた設計図さえあれば、世界中のk8sクラスタで全く同じようにコンテナ設定やネットワーク設定を再現できます。これが最大の強みです。

## 勝手に復旧してくれる
記事の途中でわざとPodを削除しましたが、一瞬で新しいPodが立ち上がりました。
k8sは常に「YAMLの設計図（レプリカ数は1）」と「現実のサーバーの状態」を見比べており、ズレが生じた瞬間に自動で修復してくれます。これにより、夜中にコンテナが一つ落ちたくらいでエンジニアが叩き起こされることがなくなります。

## 拡張性が高い
今回は `replicas: 1` でしたが、ここを `100` に書き換えて `kubectl apply` するだけで、一瞬で100台のNginxコンテナが立ち上がり、Serviceが100台に通信を分散してくれます。アクセスが急増した時も、この数文字の変更で対応できてしまいます。

# 7. 次にやること
## 外部公開の仕組みをもっと知る（Serviceの種類とIngress）
今回は`kubectl port-forward`を使って外部からアクセスしましたが、これはあくまで一時的な方法です。本番環境では、Serviceの種類を`LoadBalancer`にしたり、`Ingress`という仕組みを使って外部からアクセスできるようにします。
- Serviceの種類
  - ClusterIP
  - NodePort
  - LoadBalancer
- Ingress

## マニフェスト管理を便利にする (Helm)
yamlファイルを毎回ゼロから手書きするのは大変ですよね。yamlファイルが増えてくると、毎回`kubectl apply`するわけにもいかなくなります。そこで、`Helm`というk8s専用のパッケージマネージャー（Node.jsのnpmのようなもの）がよく使われます。複雑なyaml設定をテンプレート化し、変数を少し書き換えるだけで巨大なシステムを簡単にデプロイできるようになります。


## CDの進化版 (GitOps)
従来のコンテナだとGitHubでコードを更新して、CI/CDツールでデプロイしていましたが、k8sではGitHubでyamlファイルを更新するだけで、`ArgoCD`や`Flux`といったツールが自動で検知してデプロイしてくれます。これをGitOpsと呼びます。