# zenn-content

THINK YOU LAB の [Zenn](https://zenn.dev) 記事ソース。AI 自動化 / Claude Code / n8n / Obsidian / LAB infrastructure をテーマに発信 (JP/EN)。

## 構成
- `articles/` — Zenn 記事 (Markdown + front-matter)
- `package.json` — `zenn-cli` (プレビュー / 新規作成)
- `.github/` — CI

## ローカルプレビュー
```bash
npm install
npx zenn preview        # http://localhost:8000
npx zenn new:article    # 新規記事を作成
```

## ライセンス
記事本文は [CC-BY-4.0](LICENSE)。再利用時はクレジット表記をお願いします。
