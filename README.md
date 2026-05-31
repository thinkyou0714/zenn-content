# zenn-content

[![ci](https://github.com/thinkyou0714/zenn-content/actions/workflows/ci.yml/badge.svg)](https://github.com/thinkyou0714/zenn-content/actions/workflows/ci.yml)

[Zenn](https://zenn.dev/) で公開している技術記事のソースリポジトリです。
Claude Code / n8n / Obsidian を使った副業自動化システムの実践記録を中心にまとめています。

## 記事一覧

| 記事 | テーマ | slug |
|---|---|---|
| [Claude Code hooksを47本実装した話](articles/claude-code-hooks-47.md) | AIへの自動指示の設計 | `claude-code-hooks-47` |
| [n8n × Claude Code で55本のWFを動かしている](articles/n8n-claudecode-automation-overview.md) | 副業自動化システムの全体像 | `n8n-claudecode-automation-overview` |
| [ObsidianをAIの第二の脳にした](articles/obsidian-n8n-ai-pipeline.md) | ナレッジ自動管理システム | `obsidian-n8n-ai-pipeline` |

## ディレクトリ構成

```
.
├── articles/              # Zenn 記事（Markdown + frontmatter）
├── .github/
│   ├── workflows/
│   │   └── zenn-auto-publish.yml   # 予約公開の自動化
│   └── CODEOWNERS
├── package.json           # zenn-cli
└── LICENSE                # MIT
```

## ローカルプレビュー

```bash
npm install
npm run preview   # zenn preview（http://localhost:8000）
```

## 品質チェック（Lint）

PR では CI（[`ci.yml`](.github/workflows/ci.yml)）が以下を自動実行します。ローカルでも同じチェックを実行できます。

```bash
npm run lint        # 下記すべてをまとめて実行
npm run lint:md     # markdownlint（Markdown 整形）
npm run lint:text   # textlint（日本語の客観的な誤り検出）
npm run lint:zenn   # zenn list:articles（frontmatter 検証）
```

textlint は一人称・だ体・口語の文体を尊重し、スタイル指摘ではなく「ら抜き・二重否定・半角カナ・冗長表現・誤用」など客観的な誤りのみを検出する設定です（[`.textlintrc.json`](.textlintrc.json)）。

## 予約公開の仕組み

各記事の frontmatter に `publish_scheduled`（公開予定日時）を設定すると、
GitHub Actions（[`x-color/zenn-post-scheduler`](https://github.com/x-color/zenn-post-scheduler)）が
その時刻を過ぎた記事の `published` を自動で `true` に切り替え、コミット・プッシュします。

- 実行枠は **09:00 JST 公開運用**に合わせ、`00:00-00:45 UTC`（= `09:00-09:45 JST`）の 15 分間隔のみ。
- 即時公開・手動再実行が必要な場合は Actions タブから `workflow_dispatch` で起動できます。

### frontmatter の例

```yaml
---
title: 記事タイトル
emoji: 🪝
type: tech            # tech / idea
topics: [claudecode, automation]   # 最大 5 個
published: false      # 公開時に true（スケジューラが自動で切替）
publish_scheduled: '2026-05-17T09:00:00+09:00'   # 予約公開日時
---
```

## ライセンス

[MIT](LICENSE) © 2026 thinkyou0714
