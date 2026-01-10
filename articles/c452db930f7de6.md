---
title: "【自宅鯖×IaC】ProxmoxをTerraformで管理し、k3sクラスタをAnsibleでセットアップしてみた"
emoji: "⚙️"
type: "tech"
topics:
  - "ansible"
  - "terraform"
  - "k3s"
  - "proxmox"
  - "自宅サーバ"
published: true
published_at: "2025-12-03 16:07"
---

:::message
普通に手動でVM作ってsshしてインストールしたほうが早いかもだけど、IaCに挑戦したい！っていう人向けです。あと初投稿なのでお手柔らかに。間違っていたりアドバイスがあれば書いてほしい！
:::
# 何をしたいの？
ボタンひとつで自宅サーバー上にk3sクラスタをセットアップできるようにしたい。
![](https://storage.googleapis.com/zenn-user-upload/16166409520f-20251124.png)

具体的には、Terraformでproxmoxを操作・管理し、自動でテンプレートをクローンしてVMを作成。その後、**VMがSSHを受け付ける準備ができた段階で、** Ansibleを使ってk3sをインストールし、クラスタを構成する
![](https://storage.googleapis.com/zenn-user-upload/2b759c68411f-20251124.png)
:::message
インストール自体はAnsibleがいいが、kubernetesの状態自体を管理するなら実はTerraformがいいかも
:::
# なんでコードで管理するの？
ProxmoxはGUIを使って簡単にVMをブラウザ上でポチポチ作成できるので非常に便利です。しかし今回はあえてコードで管理することに拘ることにしました。その理由は以下のとおりです。

- インフラが壊れてもコードで保存している安心感
- 特に自宅鯖は作って壊してを繰り返すのでコードで管理できたら作り直しやすそうなので
- 毎回同じ設定をするのがやだ(今回は3ノードだがより多くのノードがある場合もっとめんどくさい）
- 勉強のため。TerraformもAnsibleも一応ghaも勉強できてお得!
- **単純にGitHubActionのワークフローを実行するだけで自動でインフラが作られていくのを見たい！**



# Terraform? cloud-init? Ansible?

- **Terraform**は**VM自体の有無や設定**をコードで管理できる。軽く設定を書くだけでproviderがAPIを叩いて設定通りに構築してくれる。AWSなどのクラウドを扱うことが多いがProxmoxにも対応していた（providerがあった)自宅鯖で役立つか分からないが、プロビジョニングに長けている
![](https://storage.googleapis.com/zenn-user-upload/14cf76ad1c91-20251124.png)
- **cloud-init**はVM初回起動時限定で実行され、素早く設定を行う。今回はAnsibleがsshできるようになる準備が目的
- **Ansible**はVMが起動した後、SSH経由でアクセスし、**VMの中身**を設定する。実行前の検証や、状態確認（冪等性）が可能。


# Part 1: VMのテンプレートの作成
Terraformがクローン元として参照するVM IDを `9000` と固定し、VMテンプレートを作成します。ほとんどの工程をCLIで行います。一度作成するだけなので、デバッグもしやすいですし、全てこのVMをクローンするので最も重要であるため手動で行います。
### 1. cloud-imageダウンロードとVMの作成
今回使うOSはUbuntu Server 24.04 LTS (Noble Numbat) にする。以下のurlをproxmoxの`local`の`ISO images`にある`Download from URL`に貼り付けてイメージをダウンロードする

```
 https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
```

![](https://storage.googleapis.com/zenn-user-upload/7ca58990fba3-20251124.png)

VMのテンプレートを入れたいノードのホストで以下のコマンドを実行し、仮のVMを作成します。

```bash
qm create 9000 --name "ubuntu-2404-cloudinit-template" \
  --memory 4096 --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --serial0 socket --vga serial0 \
  --agent 1 # ★ qemu-guest-agent を有効化
```

### 2. ディスクのインポートと接続
今回はlocal-lvmに作りますが、環境によって変えてもいいと思います。ceqhやnfsならmigrateする必要がなくなりますし。

```bash
qm set 9000 --scsihw virtio-scsi-pci
qm set 9000 --scsi0 local-lvm:0,import-from=/var/lib/vz/template/iso/noble-server-cloudimg-amd64.img

# 任意でサイズ変更
qm resize 9000 scsi0 32G
```

ダウンロードしたCloud-InitイメージをVMのディスクとしてインポートし、SCSIコントローラーに接続します。ここでは **`local-lvm`** ストレージ(0でイメージと同じサイズ）を使用します。後でサイズ変更を推奨します。

### 3. Cloud-Init用ドライブの追加

VMが起動時にCloud-Init設定を読み込むための特別なドライブを追加します。

```bash
qm set 9000 --ide2 local-lvm:cloudinit
```


### 4. その他の基本設定

起動順の設定

```bash
qm set 9000 --boot order=scsi0
```


### 5. テンプレートへの変換

最後に、このVMをテンプレート化します。テンプレート化されたVMは起動できなくなりますが、Terraformがクローン元として使用できるようになります。

```bash
qm template 9000
```

:::message
NFSストレージ上にあるVMをテンプレート化するとエラーになる場合があるため、`local-lvm` のようなローカルストレージに保存するのが確実です。
:::

### まとめ

これで、Terraformが参照できるマスターテンプレート **`VM ID: 9000`** が作成されました。
一度手動でテンプレートをクローンし、VMを作成し、別のノードがあるのであればそこへマイグレートして起動するのか試すと良いでしょう。デバックがしやすいため、自動化前にテストしておくとスムーズでしょう。

# Part 2: 【Terraform x Proxmox】 操作用ユーザーの作成とAPIトークン作成

今回の記事では書かないがGitHubActionsを使いました。セットアップして手動実行のワークフローを作成しましょう。Ansibleの実行のためにこれは事前に入れておいた方がいいです。(sshを公開なんてしたくないですし)
以下の記事にGithubActionのセルフホストランナーのセットアップをまとめました
https://scrapbox.io/pnasi/infra-runner(lxc)%E3%81%A8proxmox%E5%85%AC%E9%96%8B%E7%94%A8%E3%81%AEtunnel(lxc)%E3%81%A7%E5%A4%96%E9%83%A8%E3%82%A2%E3%82%AF%E3%82%BB%E3%82%B9%E3%82%92%E4%BE%BF%E5%88%A9%E3%81%AB%E3%81%99%E3%82%8B

### ProxmoxのwebUIを開き、ロール、ユーザー、APIトークンの設定をします。
**Datacenter->Permissionsに移動します**
#### ロールを作成
Rolesで作成。権限は特にVM・Data関係は全て許可しておく。
![](https://storage.googleapis.com/zenn-user-upload/66f0878e3f31-20251127.png)

#### グループを作成
Groupsで適切な名前をつける。今回は`infra-bot`にした
![](https://storage.googleapis.com/zenn-user-upload/e28447a744e2-20251127.png)

#### パーミッションでグループパーミッションを追加、ロールとグループを紐づける
PermissionsでGloup Permissionを追加する
![](https://storage.googleapis.com/zenn-user-upload/0ef78241f364-20251127.png)

Pathは`/`で、設定したGroupに設定したRoleを紐付ける
![](https://storage.googleapis.com/zenn-user-upload/59b36dd879b6-20251127.png)

#### Terraformユーザーを追加し、グループに所属させる
Usersでユーザーを追加する
![](https://storage.googleapis.com/zenn-user-upload/45fc0e3ad439-20251127.png)

#### APIトークンを作成する
API Tokensでトークンを作成する。
![](https://storage.googleapis.com/zenn-user-upload/98e8e914d3fd-20251127.png)
Privilege Separationをオフにしないと、権限がユーザーと同じにならないのでユーザーで権限を管理するならここをオフにすること

その後APIのトークンIDとシークレットをコピーして、GitHubActionのSecretsに保存して、providerで使います。

```yml:deploy_to_runner.yml
    env:
      TF_VAR_proxmox_token_id: ${{ secrets.PROXMOX_TOKEN_ID }}
      TF_VAR_proxmox_token_secret: ${{ secrets.PROXMOX_TOKEN_SECRET }}
```

# Part 3: インフラ管理リポジトリでTerraformを記述・ワークフロー作成
Terraformは今回以下のファイル構成にしました。

```yml
├── providers.tf
├── terraform.tfvars # ローカル実行用、これはgitに含めない
├── variables.tf
└── vms.tf
```

providers.tfではversion管理や設定を行う。proxmoxのプロバイダーは`bpg/proxmox`にしました。`telmate/proxmox`はLXCも管理できるけど今回はVMで少し古いのでこっちにしました。
state管理をTerraformCLoudにやってもらっているが、外部依存が増えるので自宅サーバー内でのみ実行することを前提に作っているので使わなくてもよかったかも。

```t:providers.tf
terraform {
  required_version = ">= 1.0"

  # Terraform Cloud バックエンド(state管理のために使った)
  cloud {
    organization = "xxxx"

    workspaces {
      name = "xxxxx"
    }
  }

  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.86"
    }

  }
}


# Proxmox Provider (bpg/proxmox for Proxmox VE 9.x support)
provider "proxmox" {
  endpoint  = var.proxmox_api_url
  api_token = "${var.proxmox_token_id}=${var.proxmox_token_secret}"
  insecure  = true

# 今回の実装ではsshは不要かもだがcloudinitを詳細に書くには必要
  ssh {
    agent    = true
    username = "terraform"
  }
}
```



terraform.tfvarsはgitに保存しません(GitHubActionのSecretsを使う)。terraform内で使う値についてはvariables.tfで管理します。

```t:variables.tf
variable "proxmox_api_url" {
  description = "Proxmox API URL"
  type        = string
  sensitive   = true
}

variable "proxmox_token_id" {
  description = "Proxmox API Token ID (format: user@realm!tokenname)"
  type        = string
  sensitive   = true
}

variable "proxmox_token_secret" {
  description = "Proxmox API Token Secret"
  type        = string
  sensitive   = true
}

```

VMの設定は以下のように記述した。

```t:vms.tf
resource "proxmox_virtual_environment_vm" "k3s_server_1" {
  name        = "k3s-server-1"
  description = "K3s Server 1 (etcd, Control Plane, Worker - HA)"
  node_name   = "aduki"
  vm_id       = 201
  migrate     = true  # VMテンプレートと別ノードに作成する場合、migrateが必要

  clone {
    vm_id     = 9000
    full      = true
    node_name = "monaka"
  }

  agent {
    enabled = true
  }

  cpu {
    cores   = 2
    sockets = 1
  }

  memory {
    dedicated = 6144
  }

  disk {
    interface    = "scsi0"
    datastore_id = "local-lvm"
    size         = 32
  }

  network_device {
    bridge = "vmbr0"
  }

  initialization {
    datastore_id = "local-lvm"
    dns {
      servers = ["8.8.8.8", "8.8.4.4"]
    }
    ip_config {
      ipv4 {
        address = "192.168.0.xx/24"
        gateway = "192.168.0.1"
      }
    }
    user_account {
      username = "xxxxx"
      password = var.ubuntu_password
      keys     = [var.ssh_public_key]
    }
  }

  tags = ["k3s", "server", "etcd", "control-plane", "worker", "ha"]
}
```

テンプレートはmonakaノードで作成したのでそこをcloneし、作成するように設定した。実は` initialization`のブロックで設定したところはcloud-initの設定である程度は設定できるが、初期に実行するコマンドなどは別で`user_data_file`などを作る必要もあり、sshも必要となる。今回はAnsibleも使うため、詳細なコマンドは実行せず、最低限sshができるように設定した。
（他のk3sノードはIPアドレスやホスト名などのみ変更しているのでほぼ全てこれと同様で今回は３つ別のノードに作成することにしています。`for_each`を使うともっと綺麗に書けるはず?）

### 実行について
#### 前提

- providerにAPIトークンを読み込ませられる
- APIを叩けるネットワークで実行する
- state(状態)を保存するところがある。

何を変更・追加・削除するのかがわかるため、デプロイ前に実行し、確認する。サーバーには変更は加えない。

```
terraform plan
```

planを確認し、実際にインフラに適用する(`-auto-approve`をつけると確認なしで実行ができるのでCDで使うのであればこの引数をつける)

```
terraform apply
```

# Part 4: Ansibleでk3sをセットアップ
本来PythonはAnsibleを動かすために必要だが、それを入れることすらコードで書きたいので、rawで書き、pythonをインストールする。

k3sクラスタの構成は全てマスタ・ワーカーノードで３ノードで最低限の可用性を確保する構成です。

今回は`xanmanning.k3s`というロールを使うことで簡単にセットアップできるようにしました。

また、家ではcloudflare tunnelを使っていて、Rancherでk3sを操作する形にしたいが、自宅鯖内に内向けDNSがないので外のDNSを参照されては困るので、`/etc/hosts`にIPを設定するようにした。（後でDNSも作りたいところ）

```yml:playbook-k3s-setup.yml
- name: K3s クラスタセットアップ
  hosts: k3s_servers
  tags:
    - setup
    - k3s
  vars:
    # K3s リリース版（false で最新安定版を使用。本当は指定したほうがいい）
    k3s_release_version: false
    # HA クラスタ用 embedded etcd 有効化
    k3s_etcd_datastore: true
    # K3s サーバー設定（制御プレーン）
    k3s_server:
      # Flannel バックエンド設定
      flannel-backend: vxlan
      # API サーバー監査ログ設定
      audit-log-maxage: 30
      audit-log-maxbackup: 10
      audit-log-maxsize: 100
    # K3s エージェント設定（ワーカー）
    k3s_agent:
      # ノードラベル
      node-label:
        - "node-role=worker"
    # バリデーション処理をスキップ（必要に応じて）
    k3s_skip_validation: false
    # Python 3 インタープリタパス
    ansible_python_interpreter: /usr/bin/python3
  
  pre_tasks:
    # Python がない場合はインストール
    - name: Python がない場合はブートストラップ
      raw: |
        if command -v python3 >/dev/null 2>&1; then
          exit 0
        fi
        sudo apt-get update -y
        sudo apt-get install -y python3 python3-apt python3-pip
      changed_when: false

    # APT キャッシュ更新とパッケージ更新
    - name: APT キャッシュ更新とディストリビューション更新
      apt:
        update_cache: yes
        upgrade: dist
        autoremove: yes
        autoclean: yes
      become: yes

    # 必要なパッケージをインストール
    - name: 必要なパッケージをインストール
      apt:
        name:
          - curl
          - wget
          - git
          - vim
          - htop
          - net-tools
          - ntp
        state: present
      become: yes

    # タイムゾーンを Asia/Tokyo に設定
    - name: タイムゾーンを Asia/Tokyo に設定
      timezone:
        name: Asia/Tokyo
      become: yes

    # IP フォワーディング有効化
    - name: IP フォワーディング有効化
      sysctl:
        name: net.ipv4.ip_forward
        value: "1"
        sysctl_set: yes
        state: present
      become: yes

    # ブリッジ iptables サポート用カーネルモジュールロード
    - name: br_netfilter モジュールをロード
      modprobe:
        name: br_netfilter
        state: present
      become: yes
      ignore_errors: yes

    # ブリッジネットワーク設定有効化
    - name: ブリッジネットワーク設定有効化
      sysctl:
        name: net.bridge.bridge-nf-call-iptables
        value: "1"
        sysctl_set: yes
        state: present
      become: yes
      ignore_errors: yes

    # /etc/hosts に Rancher ドメイン設定（LAN アクセス用）
    - name: /etc/hosts に Rancherドメイン を追加
      lineinfile:
        path: /etc/hosts
        line: "RancherのIP Rancherドメイン"
        state: present
      become: yes

    # k3s-server-1 を制御ノードに設定
    - name: k3s-server-1 を制御ノードに設定
      set_fact:
        k3s_control_node: true
      when: inventory_hostname == "k3s-server-1"

  roles:
    - role: xanmanning.k3s
      become: yes

  post_tasks:
    # K3s が起動完了するまで待機
    - name: K3s が起動完了するまで待機
      shell: |
        k3s kubectl get nodes
      register: k3s_nodes_ready
      until: k3s_nodes_ready.rc == 0 and k3s_nodes_ready.stdout_lines | length > 0
      retries: 30
      delay: 10

      when: k3s_control_node is defined and k3s_control_node

    # 制御ノードから kubeconfig を取得
    - name: 制御ノードから kubeconfig を取得
      fetch:
        src: /etc/rancher/k3s/k3s.yaml
        dest: ./kubeconfig.yaml
        flat: yes
      become: yes

      when: inventory_hostname == "k3s-server-1"
```

インベントリ(接続先のサーバーについて書くところ）

```yml:inventory.yml
all:
  hosts:
    localhost:
      ansible_connection: local
      ansible_python_interpreter: /usr/bin/python3

  children:
    k3s_servers:
      hosts:
        k3s-server-1:
          ansible_host: 192.168.0.xx
          ansible_user: xxxxx
          k3s_hostname: k3s-server-1
          k3s_control_node: true
        k3s-server-2:
          ansible_host: 192.168.0.xx
          ansible_user: xxxxx
          k3s_hostname: k3s-server-2
          k3s_control_node: true
        k3s-server-3:
          ansible_host: 192.168.0.xx
          ansible_user: xxxxx
          k3s_hostname: k3s-server-3
          k3s_control_node: true

  vars:
    # Python interpreter
    ansible_python_interpreter: /usr/bin/python3

    # SSH Configuration
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    ansible_ssh_common_args: -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=10

    # Connection settings
    ansible_connection: ssh
    ansible_port: 22

    # Become settings
    ansible_become: true
    ansible_become_method: sudo
    ansible_become_user: root

```

### 実行について
#### 前提

- サーバーにsshが実行環境からできる
- 実行環境にpythonとansibleに必要なパッケージがある
- 実行環境にロールをインストール

チェックについて

```bash
# 構文チェック
ansible-playbook -i inventory.yml playbook-k3s-setup.yml --syntax-check

# インベントリ（接続先の設定）一覧
ansible-inventory -i inventory.yml --list

# ドライラン(サーバーでは実行せずにシミュレートしてくれる）
ansible-playbook -i inventory.yml playbook-k3s-setup.yml $TARGET --check -v

```

これで完了したら私の場合はRancherにクラスタを登録したり、kubectlに登録したりしました。

# まとめ
これでリポジトリのコードを書き換えるとTerraformだとplanで実際の自宅鯖の状態との差分から何が実行されるかがわかり、その後適用(apply)すると自動的にproxmoxをコードと同じ状態にしてくれる。AnsibleだとVMの中身を自動で設定してくれる。
その結果コードで全てを管理でき、設定ミスもなくなり、ghaを動かすだけでインフラからk3sクラスタまで作れるようにできました。本当は手動でやってほうが楽かもですが、皆さんも敢えて自宅鯖でIaCにも挑戦してみては？

インフラリポジトリを公開すること自体、セキュリティ上のリスクがあるかもしれないが、実際にセルフホストランナーが動いているプライベートリポジトリとこのリポジトリとは分離して公開しています。
https://github.com/p-nasimonan/home-infra-public


# 振り返り
人生初Zennがこんなマイナーで挑戦的な記事になってしまった...!(うちの大学でこんなことをやってるのは私だけだと思う！）そもそもVMのテンプレートで挫折したり、Proxmoxを操作できるproviderのドキュメントを見ないと書けなかったり、いろんなことがあってこの記事や今の自宅鯖ができました。これからもっとインフラ関係、web関係の記事がかけたらいいなって思いました。