---
title: "自宅サーバーのHDDが物理的に死んだので、MisskeyのデータをProxmox CSIに移行した"
emoji: "💀"
type: "tech"
topics:
  - "Kubernetes"
  - "proxmox"
  - "zfs"
  - "自宅サーバ"
  - "misskey"
published: false
---

# はじめに

前回の記事でMisskeyをk3s上に構築してTrueNASのNFSとMinIOを使う構成を紹介しました。まとめの展望に「NASがHDD一つという限界構成なのでどうにかしたい」と書いていましたが、やはり来てしまった。HDDが物理的に終わってしまったのです。

https://zenn.dev/yokan/articles/kubernetes-misskey-minio

というわけでデータ救出して新しい構成に移行したという記事です。

## 何が起きたか

ある日Misskeyのバックエンドと通信できなくなったことに気づきました。その時（まさか...DBが?）とArgoCDを見てログを確認しました。案の定以下のようになってました
```
QueryFailedError: canceling statement due to statement timeout
ERR 1 [queue inbox] failed(QueryFailedError: canceling statement due to statement timeout)
Unhealthy Liveness probe failed: 127.0.0.1:5432 - no response
Killing Container postgresql failed liveness probe, will be restarted
```
やはり物理的に怪しいと思い、ホスト（monaka）のdmesgを見てみました。

```
[Tue Feb 24 15:18:14 2026] sd 4:0:0:0: [sdc] tag#7 Sense Key : Medium Error [current]
[Tue Feb 24 15:18:14 2026] sd 4:0:0:0: [sdc] tag#7 Add. Sense: Address mark not found for data field
[Tue Feb 24 15:18:14 2026] I/O error, dev sdc, sector 3226508912 op 0x0:(READ) flags 0xa00000 phys_seg 1 prio class 2
[Tue Feb 24 15:18:21 2026] ata5.00: error: { AMNF }
```
これはHDDのセクターが物理的に読めなくなってる状態です。このHDDはたしか高校のときに秋葉原で500円で買ったものでした..みなさんはお金ないからといってジャンクHDDは買わないように。

## TrueNAS + NFS構成が微妙だったこと

今回の構成、改めて振り返るとかなり無駄が多かったです。

`Proxmox上のHDD → TrueNAS（VM） → NFS → k3s（VM）`

という経路になっていて、同じ物理マシン内で通信がネットワーク層を余計に通ってました。



## データ救出

HDDはまだ完全には死んでいなかったので、壊れる前にデータを出しておく必要がありました。

- まずTrueNASから読み書きされてさらに壊れることを防ぐため、TrueNASのvmをシャットダウンしました。
- TrueNASのVMを経由せず、Proxmoxのホスト側でZFSプールを直接マウントします。

```bash
# 救出だけでよいのでreadonly=onをつけること。
zpool import -f -o readonly=on -R /mnt/backup <プール名>
```

あとはrsyncで別のディスクに退避させます。

```bash
# PostgreSQLのデータ
rsync -av --progress \
  /mnt/backup/<プール名>/k3s/misskey-data-misskey-postgresql-0-pvc-52762370-da9a-4170-bc25-cdabe3489ba7/ \
  /root/misskey_backup/k3s/misskey-db/

# MinIOのS3データ
rsync -avP --ignore-errors /mnt/backup/<プール名>/s3/ /root/misskey_backup/s3/
```

I/Oエラーが頻発してたので転送速度が100kb/sくらいまで落ちてたのですが無事に読み込みだけでもできてて助かりました。

## Proxmox CSIに移行する

データが取れたのでストレージ構成を作り直します。NFS + TrueNASをやめて、k3sからProxmoxのストレージを直接使える[Proxmox CSI Plugin](https://github.com/sergelogvinov/proxmox-csi-plugin)を導入することにしました。これを使うとPVCをProxmox上の仮想ディスクとして作成してくれるので、k3sのPodがNVMe上のブロックストレージを直接使えます。

### Proxmox側の準備

CSI用のAPIトークンを作成します。

```bash
pveum role add CSI -privs "VM.Audit VM.Config.Disk Datastore.Allocate Datastore.AllocateSpace Datastore.Audit"
pveum user add kubernetes-csi@pve
pveum aclmod / -user kubernetes-csi@pve -role CSI
pveum user token add kubernetes-csi@pve mytoken
```

k3sノードにtopologyのラベルを貼ります。Proxmoxのどのノードにいるかを示すためのものです。

```bash
kubectl label node <node-name> \
  topology.kubernetes.io/region=homelab \
  topology.kubernetes.io/zone=<proxmoxノード名>
```

### Secretの作成

config.yamlを作ってSealedSecretにします。管理しているProxmoxノード分だけ書きます。

```yaml:proxmox-csi-config.yaml
clusters:
  - url: https://192.168.0.13:8006/api2/json
    insecure: true
    token_id: "kubernetes-csi@pve!mytoken"
    token_secret: "実際のトークン値"
    region: homelab
  - url: https://192.168.0.15:8006/api2/json
    insecure: true
    token_id: "kubernetes-csi@pve!mytoken"
    token_secret: "実際のトークン値"
    region: homelab
```

これをSealedSecretにしてGitにコミットします。

### CSIのデプロイ

```yaml:proxmox-csi.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: proxmox-csi
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://github.com/p-nasimonan/home-manifests.git
      targetRevision: main
      path: apps/proxmox-csi
    - repoURL: https://sergelogvinov.github.io/helm-charts
      chart: proxmox-csi-plugin
      targetRevision: 0.2.x
      helm:
        valuesObject:
          config:
            clusters: []
          existingConfigSecret: proxmox-csi-config
          existingConfigSecretKey: config.yaml
          storageClass:
            - name: proxmox-nvme
              storage: local-nvme  # Proxmox側のストレージ名
              reclaimPolicy: Retain
              fstype: ext4
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### データのリストア

**PostgreSQL**

pg_dumpでダンプしたわけではなく生のデータディレクトリをrsyncしたので、リストアも生データをそのままPodに入れます。

```bash
# 新しいPVCにデータを書き戻す
kubectl cp /root/misskey_backup/k3s/misskey-db/. \
  misskey/misskey-postgresql-0:/bitnami/postgresql/data/
```

**MinIO**

S3のデータはMinIOのAPIを通さないとメタデータの整合性が取れないため、そのまま新しいPVCに書き戻すことができませんでした。そこで救出したデータを一度Dockerで古いMinIOとして起動して、新しいMinIOにmirrorするという方法をとりました。

```bash
# バックアップデータをマウントして一時的にMinIOを起動
docker run -d \
  -p 19000:9000 \
  -v /root/misskey_backup/s3:/data \
  -e MINIO_ROOT_USER=<OLD_ACCESS_KEY> \
  -e MINIO_ROOT_PASSWORD=<OLD_SECRET_KEY> \
  quay.io/minio/minio:RELEASE.2025-04-22T22-12-26Z \
  server /data

# 一時MinIOと新MinIOをエイリアス登録
mc alias set old http://localhost:19000 <OLD_ACCESS_KEY> <OLD_SECRET_KEY>
mc alias set new http://<新MinIOのIP>:9000 <NEW_ACCESS_KEY> <NEW_SECRET_KEY>

# mirror
mc mirror old/misskey new/misskey
```

## 復旧後に管理者権限がなくなっていた

DBのリストア後にMisskeyにログインしたら、管理者メニューが表示されなくなっていました。

原因は、前回の記事で書いた構成では**最初にアカウントを作った人が管理者**になる仕組みだったのですが、そのユーザーはロールとして管理者権限が付与されているわけではなく、「初期ログインユーザー」というフラグで管理者として扱われていたためです。

さらに、初期セットアップ画面が再度有効になっていました。つまり今誰かがアクセスしてアカウントを作ると、その人が管理者になってしまいます。前回の記事で設定した`setupPassword`があったので外部から勝手に管理者を作られることはなかったですが、気づかずにいたら危なかった。

対処はシンプルで、初期セットアップで新しくadminアカウントを作ってロールを作成しました。

1. `setupPassword`を使って初期セットアップ画面からadminアカウントを新規作成
2. adminアカウントの管理画面でロールを作成し、管理者権限を付与
3. 自分のメインアカウントにそのロールを割り当てる

これだけで管理者権限が復旧します。**初期ログインでadminアカウントを別途作っておくのが重要**で、DBが復旧しても管理者に戻る手段がなくなるところでした。

## まとめ

NASがHDD一本という構成のまま放置してたら案の定死にました。ただrsyncで抜き出せたのでデータは全部助かりました。
Proxmox CSIにしてNFSの余計な層がなくなったのでシンプルになりました。PVCがProxmoxの仮想ディスクとして管理されるので、次はバックアップ戦略を考えたいところです。
