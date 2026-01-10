# youkan-zenn

Zenn で技術ブログ記事を執筆・管理するリポジトリです。

## セットアップ

### 前提条件

- Node.js 14以上がインストール済み

### インストール

```bash
# 依存パッケージをインストール
npm install
```

## 使い方

### 記事の作成

```bash
# 基本的な作成
npx zenn new:article

# slug（ファイル名）を指定して作成
npx zenn new:article --slug my-article-slug

# Front Matterもオプションで指定
npx zenn new:article --slug my-article --title "記事のタイトル" --type tech --emoji 📚
```

作成されるファイル構成：

```
articles/
└── my-article-slug.md   # 記事ファイル
```

**Front Matter（ファイルの先頭）**

```yaml
---
title: "記事のタイトル"
emoji: "📚"                    # 1文字の絵文字
type: "tech"                   # tech（技術記事）/ idea（アイデア）
topics: ["kubernetes", "zenn"]  # タグ
published: true                 # 公開設定
---
ここから本文を書きます...
```

**slug の制約**

- 12〜50文字
- 使用可能な文字：a-z, 0-9, ハイフン（-）, アンダースコア（_）

### 本の作成

```bash
npx zenn new:book --slug my-book
```

### ローカルでプレビュー

```bash
# プレビューサーバーを起動（http://localhost:8000）
npx zenn preview

# ポート番号を指定
npx zenn preview --port 3000

# ファイル監視を無効化
npx zenn preview --no-watch
```

ブラウザでリアルタイムにプレビューしながら執筆できます。

## ディレクトリ構造

```text
.
├── articles/          # 技術記事を格納（markdown形式）
│   ├── article1.md    # 各記事は1ファイル
│   └── article2.md
├── books/             # 本形式のコンテンツを格納
├── images/            # 記事で使用する画像を格納
├── .markdownlintrc.json  # Markdownlintの設定
├── .textlintrc.json    # textlintの設定（日本語チェック）
├── .husky/             # Git hookの設定
├── package.json        # プロジェクト設定・npm scripts
├── .gitignore          # Git無視ファイル
└── README.md           # このファイル
```

**ファイル配置ルール**

- 1つの記事 = 1つのmarkdownファイル（`articles/◯◯.md`）
- 画像ファイルは `images/` に配置
- 記事ファイルとイメージはそれぞれ別ディレクトリで管理

## GitHub連携

このリポジトリをZennダッシュボードと連携することで、pushされた内容が自動的にZennに反映されます。

詳細は [Zenn公式ドキュメント](https://zenn.dev/zenn/articles/connect-to-github) を参照してください。

### 記事の公開・管理

#### 公開する

Front Matterの `published` を `true` にして、GitHub にpushします。

```yaml
---
title: "公開する記事"
published: true
---
```

```bash
git add articles/
git commit -m "新しい記事を公開"
git push
```

#### 公開を予約する

`published_at` に未来の日時を指定すると、その時刻に自動公開されます。

```yaml
---
title: "公開予約する記事"
published: true
published_at: 2026-02-01 09:00  # JST（日本時間）
---
```

#### 公開日時を指定する（過去の日時）

他サービスからの移行時に、元の公開日時を指定できます。

```yaml
---
title: "過去に公開した記事"
published: true
published_at: 2024-01-15 08:00
---
```

#### 下書きとして保存

`published: false` の状態でpushすると、Zennダッシュボードのみで確認できます。

```yaml
---
title: "執筆中の記事"
published: false
---
```

#### 記事を更新する

markdownファイルを編集してpushするだけで更新されます。**slug を変更するとは別の記事として作成される**ため注意。

#### 記事を削除する

**注意**：markdownファイルを削除してもZenn上では削除されません。Zennダッシュボードから削除してください。

## Lint・品質チェック

このリポジトリでは、Markdown形式のチェックと日本語表記のチェックを実装しています。

### markdownlint

Markdown記法の統一性をチェックします。

```bash
# Markdownファイル全体をチェック
npm run lint:md
```

### textlint

日本語表記の統一をチェックします（句読点、全角・半角など）。

```bash
# 記事をチェック
npm run lint:text
```

### pre-commit フック

コミット前に自動的にlintを実行します。

```bash
# コミット時に自動実行
git commit -m "記事を追加"
# → lint チェック実行 → OKならコミット
```

lintエラーがある場合はコミットが中断されます。必要に応じて修正してから再度コミットしてください。

## 参考リンク

- [Zenn 公式](https://zenn.dev/)
- [Zenn CLI ドキュメント](https://zenn.dev/zenn/articles/zenn-cli-guide)
- [GitHub連携について](https://zenn.dev/tsukulink/articles/2025-07-how-to-write-techblog-with-zenn)
