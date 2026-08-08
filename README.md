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

## ブランチ運用（feature → develop → main）

- **`feature/*`（下書き）**: 記事ごとに `develop` から派生（例: `feature/gap-08-cow`）。`published: false` で執筆。
- **`develop`（清書）**: 下書きを統合・推敲する場所。`published: false` のまま。
- **`main`（公開）**: Zenn 連携ブランチ。**`main` への push だけが自動デプロイ**される。

### 公開フロー
1. `git switch develop && git switch -c feature/<topic>` で下書き開始（`published: false`）。
2. `npm run preview` で確認しつつ執筆 → `develop` にマージして清書。
3. 公開OKになったら `develop` → `main` にマージし、その際に **`published: true`** へ変更。
4. `main` を push → Zenn が自動デプロイ＝公開。

> 事前に一度だけ [Zenn Dashboard → Deploys](https://zenn.dev/dashboard/deploys) で連携（連携ブランチは `main`）。
> `develop`/`feature` は Zenn 未連携なので、push しても公開されない（安全に清書できる）。

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
| [closure-capture-escaping](articles/closure-capture-escaping.md) | Swiftクロージャ捕捉の使い分け（変数/値・@escaping・weak self） | ✅ [公開](https://zenn.dev/pg_raizin/articles/closure-capture-escaping) |
| [mutating-copy-on-write](articles/mutating-copy-on-write.md) | Swiftのmutatingとcopy-on-writeを、自作CoWで腹落ちさせる | ✅ [公開](https://zenn.dev/pg_raizin/articles/mutating-copy-on-write) |
| [swift-error-handling](articles/swift-error-handling.md) | Swiftのエラー処理を使い分ける（try?/try!/Result/typed throws） | 下書き（feature/article-errors） |
