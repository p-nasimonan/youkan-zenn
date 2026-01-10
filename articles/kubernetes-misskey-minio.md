---
title: "おうちKubernetes × GitOpsでMisskeyを構築、TrueNASでおうちS3を作り、投稿も画像も家で保管する！"
emoji: "🏠"
type: "tech"
topics:
  - "Kubernetes"
  - "argocd"
  - "minio"
  - "自宅サーバ"
  - "gitops"
published: true
---

# はじめに

私のMisskey
https://mi.youkan.uk/

自分のMisskeyを建てたい！って思ったことありませんか？
でもせっかくだからKubernetesの勉強したいな〜
画像の保存もどうしよう...でもクラウドお金かかるし...そうだ！家のk3sクラスタにMisskey建てて、それからS3互換のオブジェクトストレージも家のNASで作っちゃおう。
冬休み少し時間があったのでKubernetesを勉強するためにMisskeyを建ててみた感じです！

もちろんそのまま参考にはあまりならないとは思いますが、Kubernetesをがっつり触ったのはこれが初めてなのですが、その割には楽でいい感じの構成になったと思います。構成例として見て頂ければいいなと思います。もっとよくなりそうな書き方や構成があれば言ってくれると助かります。

## ちょっとした自宅鯖紹介
現在(2026/01)
以下のコンピュータにProxmoxを入れ、クラスタ上にLXCやVMを建てている

- N100のミニPC
- Intel6世代オフィスpc(ハードオフで1100円)
- HPE ML150 Gen9(ハードオフで2200円)
- 前のゲーミングpc(HDDがたくさん入るケースなのでNASを入れてる)

ゲーミングpcが半分NASで半分k3sのワーカーノード、HPEのML150もワーカーノードとコントロールプレーン、残り二つがコントロールプレーンやDNS、DHCPなどの基盤システムと過去の残骸をLXCで担ってる。KubernetesのノードはそれぞれVMで作っています。

### 前提条件

- Kubernetesクラスターを構築済み
- Argocdが入ってる
- TrueNASがある(minioを入れればいいから別になんでもいい)
- cloudflareにドメインを登録している(cloudflare tunnel使いたいので)

クラスタの構築は前回の記事のようにIaCでbootstrapしています。記事を書いた当初と今のコード自体はだいぶ変更してますが、やってることは大体同じです。

https://zenn.dev/yokan/articles/c452db930f7de6

それと今回はリポジトリを安全に**公開したいのでSealedSecret**を使います。ローカルで公開鍵で暗号化してサーバーの秘密鍵で復号化されてKubernetesSecretになるって感じです。
↓これがリポジトリです。
https://github.com/p-nasimonan/home-manifests

## 構成図

![Kubernetes + Misskey + MinIO構成図](https://storage.googleapis.com/zenn-user-upload/9b495cf2c1ec-20260109.png)
*Kubernetes + Misskey + MinIO構成図*

Misskey 自体基本は3つの Deployment で構成されている

- Misskey (本体)
- Postgres (DB)
- Redis (Cache)
それに加えてノートの検索を行うためにMeilisearchをデプロイしてます。

Misskeyにはvolumeをマウントしてデータを永続化する必要があります。例えばサーバーの名前などの設定や投稿データはpostgresで管理されますが、そのデータをどこかに保存しないといけないのです。
今回はNASに保存することにしました。KubernetesにはPVとPVCがありますがnfs-subdir-external-provisionerを使っているためそれぞれを書く必要はなくなりました。動的にディレクトリを作ってくれます。ローカルストレージでもLonghornを作ってみてもよかったのですが、VMを壊しにくくなってしまうのでnfsにしました。
さらにMisskeyでは画像などのファイルをオブジェクトストレージに保存することができます。これはNASのMinIOに保存します。初期はオブジェクトストレージの設定方法が管理画面にしかないと知らずに画像がpodに保存されていました。
https://mi.youkan.uk/notes/ah3duf0syf630021

例えばこれでデプロイできました。それではMisskeyを実際に家の外に公開したい場合どうしたらよいでしょう。IPをルーターでポート開放して公開することもできますが、家庭でサーバーを公開するときはポート開放が難しかったり、IPが固定されてなかったりします。そこでCloudflare Tunnelを使うことでそれらの障壁を無視して公開することができます。しかも証明書も自動で管理してくれます。さらにcloudflaredをそのまま入れるだけだと毎回Cloudflareのコンソールを開く必要があるため、`cloudflare-tunnel-ingress-controller`を使ってます。

実家暮らしの学生（電気代はまだ許されている）でお金がないけどサーバーはあるのでこんな構成になってます。ドメイン代以外実質無料で構築できてます。

- Cloudflare Tunnel（超絶便利！IP固定不要、ポート開放不要で公開できちゃうやつ）
- MinIO（ちゃんと運用するならCloudflare R2のほうがデータの心配はないかな...）



## GitOps、SealedSecretの準備

### ArgoCDの何がいいのか
**PCのエディタでマニフェストを書いてGitにPushするだけでよい！**

**旧来の手順:** PCでPush → サーバーにSSH → git pull → 再起動 (結構めんどくさい)
**GitOps:** PCでPush → (Argo CDが検知) → 勝手に同期  (楽!)
後UIが見やすかったり、同期方法変えたりできる！これのおかげでだいぶkubernetesの理解が深まった
### リポジトリを登録する
ArgoCDのSetting -> Repositories ->CONNECT REPOから登録する
プライベートリポジトリならssh鍵を追加しないといけないが今回は公開リポジトリなので、httpsを選択し、URLを貼り付けるだけ。
![ArgoCDリポジトリ登録画面](https://storage.googleapis.com/zenn-user-upload/e84dcda320d6-20260108.png)
### SealedSecret
ArgoCDはPull型なのでGitHub ActionsのようにSecretを設定することはできない。GitOpsの考え方としてはgitを見に行けばすべて書いてありそれが正解なのでGitだけで動くのがいいかなと考えました。またExternal Secretsなどもありますが追加でインフラが必要でお金がかかる可能性もあるためSealedSecretにしました。
登録したリポジトリにマニフェストを書くApplicationを適用する

```yml:sealed-secrets.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    chart: sealed-secrets
    targetRevision: 2.13.2
    helm:
      values: |
        commandArgs:
          - "--update-status=true"
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

New Appからも登録できますがCLIのほうが楽なのでkubectlでできます。

```bash
kubectl apply -f https://raw.githubusercontent.com/p-nasimonan/home-manifests/main/argocd-apps/sealed-secrets.yaml
```

#### 使い方

1. kubeseal をインストール
2. .envを作成し、シークレットを書く(.envはgitignoreで除く)
3. サーバーから公開鍵を取得
`~/.kube/config`が設定されている状態で`kubeseal --fetch-cert > cert.pem`で取得する
4. 公開鍵を使って.envを暗号化。namespaceやnameを指定し忘れないように！生成後に編集はできないはず

```bash
kubectl create secret generic "$SECRET_NAME" \
  --namespace "$NAMESPACE" \
  --from-env-file="$ENV_FILE" \
  --dry-run=client -o yaml | \
  kubeseal --cert "$CERT_PATH" -o yaml \
```

私は毎回これを実行するのがめんどくさい上、k3sのAPIは外部に公開してないためSSH経由で操作する必要があり手順が複雑でした。そこで、運用コストを下げるためにローカルからコマンド一発で暗号化できるラッパースクリプトを作成してます。（流れは上記と同じ感じ。3だけ違うくらい）

## ボリューム・ネットワーク周り

### cloudflare-tunnel-ingress-controller
元々家のルーターでポート解放するのめんどくさいしセキュアではないと感じてcloudflaretunnelを使っていたが、なんとKubernetesでingressとして設定してあげるだけで超手軽に外部に公開できるらしい！コンソールを開かなくても自動で設定される!

**参考にした記事**

https://zenn.dev/yh/articles/11823e77bd4379

参考記事ではSecretを直接書いていましたが、私はSealedSecretを使うために以下のように書きました。

```yml:cloudflare-tunnel-ingress-controller.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cloudflare-tunnel-ingress-controller
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    # Secret ファイルの管理
    - repoURL: https://github.com/p-nasimonan/home-manifests.git
      targetRevision: main
      path: argocd-apps/secrets
    # Helm chart の管理
    - repoURL: https://helm.strrl.dev
      chart: cloudflare-tunnel-ingress-controller
      targetRevision: 0.0.12
      helm:
        valuesObject:
          cloudflare:
            secretRef:
              name: "cloudflare-secrets"
              apiTokenKey: "api-token"
              accountIDKey: "cloudflare-account-id"
              tunnelNameKey: "cloudflare-tunnel-name"
          ingressClass:
            name: cloudflare
            isDefaultClass: false
          replicaCount: 1
  destination:
    server: https://kubernetes.default.svc
    namespace: cloudflare-tunnel-ingress-controller
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

secretRef.name(ここでは`cloudflare-secrets`)はSealedSecretで設定したnameと同じであることを確認しましょう

APIについては以下の権限を持たせる必要があります
`ゾーン(Zone)`:`ゾーン(Zone)`:`読み取り(Read)`
`ゾーン(Zone)`:`DNS`:`編集(Edit)`
`アカウント(Account)`:`Cloudflare Tunnel`:`編集(Edit)`


## TrueNASでNFS / MinIO
TrueNASでデータセットを作成する。今回は`k3s`がnfsで共有され、`s3`をminioにマウントする形にする
![TrueNASデータセット構成](https://storage.googleapis.com/zenn-user-upload/754a1654880a-20260109.png)
*TrueNASデータセット構成*
### NFS
ShereでNFSを有効にして、マウントしたいpathを選択して、ネットワーク制限をつけて作る
![NFS設定](https://storage.googleapis.com/zenn-user-upload/ab33bc7b2c29-20260109.png)
*NFS共有設定画面*
### MinIO
``RELEASE.2025-04-22T22-12-26Z`より後のイメージはGUIがほとんどなくなっているため、少し古いが`RELEASE.2025-04-22T22-12-26Z`のバージョンを使う。Dockerイメージが配布されなくなったらしいので今後代替を調べたいがなかなか見つからないところ...
#### 1. TrueNASのApps -> Discover Apps -> customAppへ移動する
Discover内で検索してすぐにインストールできるが、それだと最新バージョンになってしまうので、customAppで手動で設定してインストールする
#### 2. customAppを設定する
特に重要なところのみ書いた。他はそれぞれの環境に合わせて書くと良い

**Image Configuration**
![MinIOイメージ設定](https://storage.googleapis.com/zenn-user-upload/f1e817048168-20260109.png)
*MinIOコンテナイメージ設定*
Repository: `quay.io/minio/minio`
Tag: `RELEASE.2025-04-22T22-12-26Z`
Pull Policy: `Pull the image if it is not already present on the host.`

**command**
![MinIOコマンド設定](https://storage.googleapis.com/zenn-user-upload/14122a3742d7-20260109.png)
*MinIOサーバー起動コマンド設定*
`[server, /data, --console-address, :9001]`念の為それぞれ分けて書いた

**Environment Variables**

:::message alert
MINIO_ROOT_USER と MINIO_ROOT_PASSWORD を設定する。これがrootユーザーの認証情報になるため、必ず強力なパスワードを設定してください
:::

**Ports**
9000と9001の二つを追加した。9000が公開するBucketのポートで、9001が管理画面

**Storage**
Host Pathを選択する。

#### 3. MinIOでバケットとアクセスキーを作成する
![MinIOバケット作成画面](https://storage.googleapis.com/zenn-user-upload/6b0ecf20f418-20260109.png)
*MinIOバケット作成画面*
特に特別なことはせず、デフォルトのままバケットを作ってアクセスキーを作る。
そしてアクセスキーのポリシーを設定する。作ったバケットにアクセスは限定しよう。

```yml
{
 "Version": "2012-10-17",
 "Statement": [
  {
   "Effect": "Allow",
   "Action": [
    "s3:PutObject",
    "s3:DeleteObject",
    "s3:GetBucketLocation",
    "s3:GetObject",
    "s3:ListBucket"
   ],
   "Resource": [
    "arn:aws:s3:::misskey",
    "arn:aws:s3:::misskey/*"
   ]
  }
 ]
}
```

その後9000ポートを外部に公開する。↓このようになります。misskeyで使う画像をこのurlで使われるようにします。
https://s3.youkan.uk/misskey

## `nfs-subdir-external-provisioner`を使う
毎回PVとPVCのことを書いてパスを設定するのは管理コストがかかるためStorageClass`nfs-client`を作ってそれさえ設定することで動的にやってくれるようにしました。さっき設定したNFSをあまり意識せずに使えるようになります。

```yml:storage.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nfs-storage
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
    chart: nfs-subdir-external-provisioner
    targetRevision: 4.0.18
    helm:
      values: |
        nfs:
          server: 192.168.0.9
          path: /mnt/public/k3s
        storageClass:
          name: nfs-client
          defaultClass: true
          archiveOnDelete: true
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

# Misskeyを建てる
まずArgoCDに登録するApplicationを作成します。Namespaceも作ってくれるようにしてます。同期に関してはgitに変更があった場合自動で適用されるようにしてます。ちなみにミスると自動で同期されて消えます。
## Application
### Misskey本体
ここで書いたリポジトリのパス(`apps/misskey`)にDeploymentなどを書いていきます。あとでカスタムマニフェストは解説します)

```yml:misskey.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: misskey
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/p-nasimonan/home-manifests.git
    targetRevision: main
    path: apps/misskey
    directory:
      recurse: false
      include: misskey.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: misskey
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Postgresql
これはHelmChartを使っているのでここで設定も書いています。storageClassを設定してます。わざわざPV/PVCを書かなくてもいいので楽ですね

```yml:misskey-postgresql.yaml=1
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: misskey-postgresql
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: postgresql
    targetRevision: 16.4.x 
    helm:
      valuesObject:
        global:
          storageClass: nfs-client
          security:
            allowInsecureImages: true
        image:
          repository: bitnami/postgresql
          tag: latest
        auth:
          username: misskey
          database: misskey
          existingSecret: misskey-secrets
          secretKeys:
            adminPasswordKey: POSTGRES_PASSWORD
            userPasswordKey: POSTGRES_PASSWORD
        primary:
          persistence:
            enabled: true
            size: 20Gi
        fullnameOverride: misskey-postgresql
  destination:
    server: https://kubernetes.default.svc
    namespace: misskey
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Redis
ここも同様です。バックアップ用にPVCを追加してますがあんまりいらないかも？

```yml:misskey-redis.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: misskey-redis
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: redis
    targetRevision: 19.5.2
    helm:
      valuesObject:
        architecture: standalone
        fullnameOverride: misskey-redis
        global:
          storageClass: nfs-client
          security:
            allowInsecureImages: true
        image:
          repository: bitnami/redis
          tag: latest
        auth:
          enabled: true
          existingSecret: misskey-secrets
          existingSecretPasswordKey: REDIS_PASSWORD
        master:
          persistence:
            enabled: true
            size: 5Gi
            storageClass: nfs-client
  destination:
    server: https://kubernetes.default.svc
    namespace: misskey
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Meilisearch
これがなくてもMisskey自体は動くが、これがないとノート検索ができないため

```yml:misskey-meilisearch.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: misskey-meilisearch
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/p-nasimonan/home-manifests.git
    targetRevision: main
    path: apps/misskey
    directory:
      recurse: false
      include: meilisearch.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: misskey
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## カスタムマニフェスト
### Misskey Server
イメージは`misskey/misskey`を使いました。今後変えてみても面白そうですね!

ここで参考にした記事では`/misskey/.config/default.yaml`を設定するために`ExternalSecrets`を使ってそれをマウントしていましたが、ここでは`SealedSecret`を使うために少し工夫が必要です。
initContainersを使い、毎回コンテナ起動時に設定ファイルを書き換えています。設定の可読性が下がるので別の方法を考えたいところですが、`SealedSecret`を使うため環境変数で書き換えられるこの方法が確実なためこうしています。

:::message
設定は公式リポジトリの`example.yml`を参考にしました。S3は後で管理画面から有効にします。つまり初期だと画像が永続化されません!管理画面から有効にする必要があります。
:::

:::message
**最初にアカウントを作った人が管理者**になります。`cloudflare-tunnel-ingress-controller`は欠点としてローカルで確認する前に公開してしまいます。そのため何かしら初期ではアクセス制御する必要があります。そのため念のため`setupPassword`を設定しています。
:::

https://github.com/misskey-dev/misskey/blob/develop/.config/example.yml

詳細を知りたい人は直接設定ファイルを定義しているところを見てみましょう

https://github.com/misskey-dev/misskey/blob/develop/packages/backend/src/config.ts

```yml:misskey.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: misskey
  namespace: misskey
spec:
  replicas: 1
  selector:
    matchLabels:
      app: misskey
  template:
    metadata:
      labels:
        app: misskey
    spec:
      initContainers:
        - name: config-generator
          image: busybox:1.35
          command:
            - sh
            - -c
            - |
              mkdir -p /misskey/.config
              cat > /misskey/.config/default.yml <<EOF
              url: https://mi.youkan.uk
              port: 3000
              setupPassword: $SETUP_PASSWORD
              db:
                host: $DB_HOST
                port: $DB_PORT
                db: $DB_NAME
                user: $DB_USER
                pass: $DB_PASS
              redis:
                host: $REDIS_HOST
                port: $REDIS_PORT
                pass: $REDIS_PASS
              meilisearch:
                host: $MEILISEARCH_HOST
                port: $MEILISEARCH_PORT
                apiKey: $MEILISEARCH_API_KEY
                ssl: false
                index: mi_youkan_uk
              fulltextSearch:
                provider: meilisearch
              id: $MISSKEY_ID
              allowedPrivateNetworks:
                - 192.168.0.0/24
              EOF
          env:
            - name: DB_HOST
              value: misskey-postgresql
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: misskey
            - name: DB_USER
              value: misskey
            - name: DB_PASS
              valueFrom:
                secretKeyRef:
                  name: misskey-secrets
                  key: POSTGRES_PASSWORD
            - name: REDIS_HOST
              value: misskey-redis-master
            - name: REDIS_PORT
              value: "6379"
            - name: REDIS_PASS
              valueFrom:
                secretKeyRef:
                  name: misskey-secrets
                  key: REDIS_PASSWORD
            - name: MEILISEARCH_HOST
              value: meilisearch
            - name: MEILISEARCH_PORT
              value: "7700"
            - name: MEILISEARCH_API_KEY
              valueFrom:
                secretKeyRef:
                  name: misskey-secrets
                  key: MEILISEARCH_MASTER_KEY
            - name: MISSKEY_ID
              value: aidx
            - name: SETUP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: misskey-secrets
                  key: SETUP_PASSWORD
          volumeMounts:
            - name: config-volume
              mountPath: /misskey/.config
      containers:
        - name: misskey
          image: misskey/misskey:2026.1.0-alpha.0
          env:
            - name: NODE_ENV
              value: production
          ports:
            - name: http
              containerPort: 3000
          volumeMounts:
            - name: config-volume
              mountPath: /misskey/.config
          resources:
            requests:
              cpu: 500m
              memory: 2Gi
            limits:
              cpu: 2000m
              memory: 4Gi
      volumes:
        - name: config-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: misskey
  namespace: misskey
spec:
  type: ClusterIP
  selector:
    app: misskey
  ports:
    - name: http
      port: 3000
      targetPort: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: misskey
  namespace: misskey
  annotations:
    external-dns.alpha.kubernetes.io/hostname: mi.youkan.uk
spec:
  ingressClassName: cloudflare
  rules:
    - host: mi.youkan.uk
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: misskey
                port:
                  number: 3000
```


## オブジェクトストレージを有効にする
これは設定ファイルからはできないため管理画面から行う必要があります。MinIOで取得したAPIキーなどを準備しましょう。

MinIOの場合以下を設定する必要があるみたい

- Regionは`us-east-1`にする
- S3ForePathStyleを有効にする

Endpointは実際にmisskey-serverがMinIOと通信するときに使うものでhttpsとなっているがあまり気にしなくてよいSSLを使用するをオフにしたら実際はhttpで通信されてるはず、Proxyは利用しないのでオフ。これによって外部から画像を見たいときはMinIOを見ることができて無駄がない。
![Misskeyオブジェクトストレージ設定](https://storage.googleapis.com/zenn-user-upload/5fab2a0b894a-20260111.png)
*Misskeyオブジェクトストレージ設定画面*


## まとめ

Kubernetes + nfs+subdir-external-provisioner + sealed-secrets + cloudflare-tunnel-ingress-controller + MinIOでMisskeyを構築するという盛りだくさんな記事になってしまいました。何か参考になったらいいな

展望としては別のフォークのMisskeyにしたりそれ自体も少しいじっていけたら楽しそうというのと、NASがHDD一つという限界構成なのでどうにかしたい、DBが飛んだら悲しいのでバックアップは取りたい（`democratic-csi`を使ってみるとか？）。Kustomizeを入門したいといったところです。

いまのところ結構安定して稼働できてるのでよさそうです！
