# raizin-zenn-articles

Zenn 記事連携用リポジトリ。[Zenn](https://zenn.dev) と GitHub 連携し、`main` への push で自動デプロイする。
記事の元ネタ（検証・学習）は別リポジトリ **iOS-Lab** の `labs/` `Docs/notes/` で作り、ここには公開用の記事だけを置く。

## 構成
```
.
├── articles/        # 記事（1ファイル=1記事、slugはファイル名）
├── books/           # 本（任意・未使用なら無くてよい）
├── package.json     # zenn-cli
└── .gitignore
```

## セットアップ / プレビュー
```bash
npm install          # 初回のみ（zenn-cli を取得）
npm run preview      # http://localhost:8000 でローカルプレビュー
npm run new:article  # articles/ に記事雛形を生成
```

## 公開フロー
1. `articles/*.md` の frontmatter `published: false`（下書き）で書く。
2. ローカルプレビューで確認。
3. 公開時に `published: true` に変更。
4. `main` に push → Zenn が自動デプロイ。

> 事前に一度だけ [Zenn Dashboard → Deploys](https://zenn.dev/dashboard/deploys) でこのリポジトリを連携しておくこと。連携ブランチは `main`。

## frontmatter 例
```yaml
---
title: "記事タイトル"
emoji: "🧩"
type: "tech"        # tech（技術記事） / idea（アイデア）
topics: ["swift", "ios"]
published: false     # true で公開
---
```
slug（＝ファイル名）は半角英数・ハイフン/アンダースコアで 12〜50 文字。

## 記事一覧
| slug | タイトル | 状態 |
|---|---|---|
| [closure-capture-escaping](articles/closure-capture-escaping.md) | Swiftクロージャ捕捉の使い分け（変数/値・@escaping・weak self） | 下書き |
