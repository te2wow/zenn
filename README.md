# zenn

[Zenn](https://zenn.dev) の投稿を [Zenn CLI](https://zenn.dev/zenn/articles/install-zenn-cli) で管理するリポジトリ。

## このリポジトリで管理することのメリット

- `npx zenn preview` でローカルプレビューが可能
- `npx zenn new:article` で frontmatter 込みのテンプレを自動生成
- GitHub 連携を設定すれば `git push` で自動公開
- Zenn のリンターが効く（タイトル長、トピック数、slug 制約 など）

## セットアップ

```bash
npm install
```

## 使い方

```bash
# 新規記事を作成（frontmatter 付きの空ファイルを生成）
npm run new

# 新規本を作成
npm run new:book

# ローカルプレビュー（http://localhost:8000）
npm run preview

# 記事一覧を表示
npm run list
```

## ディレクトリ構成

```
.
├── articles/           # 記事 (1記事1ファイル、ファイル名が slug)
├── books/              # 本
└── images/             # 画像 (記事内では /images/<slug>/<file> で参照)
    └── <slug>/
```

### slug の制約

- 12〜50 文字
- `a-z` / `0-9` / `-` / `_` のみ

### frontmatter

```yaml
---
title: "記事タイトル"
emoji: "😸"             # サムネに使う絵文字 (1文字)
type: "tech"            # "tech" | "idea"
topics: ["nextjs", "typescript"]   # 最大 5 個
published: false        # true で公開、false で下書き
published_at: "2026-04-25 12:00"   # 任意。公開日時 (JST)
---
```

## GitHub 連携

[Zenn のダッシュボード](https://zenn.dev/dashboard/deploys) から本リポジトリを連携することで、`main` ブランチへのプッシュで自動デプロイされる。
