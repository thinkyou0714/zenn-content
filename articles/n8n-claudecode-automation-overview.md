---
title: "n8n × Claude Code で55本のWFを動かしている：副業自動化システムの全体像【2026年版】"
emoji: "⚙️"
type: "tech"
topics: ["n8n", "claudecode", "automation", "個人開発"]
published: true
---

## TL;DR

- n8n（ワークフロー自動化）と Claude Code（AIコーディング）を組み合わせると「勝手に動く副業システム」が作れる
- 現在55本のWFが動いており、毎朝の情報収集・コンテンツ生成・X投稿管理が自動化されている
- 重要なのはツールではなく「3レイヤー設計（収集/処理/出力）」という構造の考え方

---

## 毎朝Slackに届くもの

俺の毎朝はこんな感じだ。

起きてSlackを開くと、AIが夜中に作業していた結果が届いている。

```
#lab-brief:   昨日のブックマーク記事 5本を要約 + 今日のタスク一覧
#lab-x:       今日投稿できるX下書き 3本（承認待ち）
#lab-n8n:     エラーが出たWFの通知（あれば）
#lab-daily:   昨日のセッションサマリー + 次にやること
```

手動でやったことはゼロ。全部 n8n と Claude が勝手にやってくれている。

これが「副業自動化システム」の現実だ。劇的じゃない。地味だ。でも毎日確実に動く。

---

## なぜ n8n × Claude Code なのか

よく聞かれる。「Make（旧Integromat）やZapierじゃダメなの？」

理由は3つある。

**1. セルフホスト可能**
n8n はセルフホストできる。俺の場合 Docker で自宅の NUC マシン（AMD Radeon / RAM 32GB）で動いている。クラウドサービスへのデータ送出を最小限にできるし、月額コストがゼロだ。

**2. Code ノードが使える**
n8n の Code ノード（JavaScript / Python）を使えば、API が存在しないサービスへの操作やカスタムロジックが書ける。Make や Zapier では無理なことが多い。

**3. Claude Code との相性**
WFの設計・デバッグを Claude Code に任せられる。「このWFのコードノードを修正して」と言うだけで直してくれる。AI でWFを作る時代になった。

---

## 3レイヤー設計という考え方

55本のWFを分類すると、3つのレイヤーに収まる。

```
Layer 1: 収集レイヤー（情報を集める）
  WF-EMBED        Obsidianノートをベクトル化
  WF-TRIAGE       未整理ノートの自動分類
  WF-X-CANDIDATE  X上の有望アカウントを自動発掘
  WF-AI-NEWS      RSS + X bookmarks から情報収集
  WF-BOOKMARK     Xのブックマークを Obsidian に保存
  ...

Layer 2: 処理レイヤー（情報を変換・分析する）
  WF-DAILY-BRIEF  情報をサマリーに圧縮
  WF-X-ANALYZE    投稿パフォーマンスを分析
  WF-MOC          関連ノートのMOCを自動生成
  WF-REVIEW       7日前のノートを再浮上させる
  ...

Layer 3: 出力レイヤー（外部に出力する）
  WF-X-SCHEDULER  X投稿を承認後に自動スケジュール
  WF-DAILY-NOTE   デイリーノートを自動生成
  WF-WEEKLY-RPT   週次レポートをSlackに送信
  WF-ZENN-TRIAGE  Zenn記事をGitHubにpush（設計中）
  ...
```

重要なのは「ツール単体を使う」のではなく、**この3レイヤーを意識して設計する**ことだ。収集だけあっても出力がなければ機能しない。処理が弱ければ情報が溜まるだけになる。

---

## Claude Code との連携パターン

n8n のWFを作る・直す作業を Claude Code に任せると、効率が跳ね上がった。

典型的な使い方はこうだ。

```
俺:     「WF-X-ANALYZE の Code ノードが TypeError を出している」
Claude: 「エラーを確認します。n8n の Code ノードでは new URL() が使えないため、
         require('url').URL を使う必要があります。以下に修正します」
俺:     「じゃあそのまま n8n API 経由でインポートして」
Claude: [自動でAPIに PUT リクエスト → WFを更新]
```

以前は「エラーを調べる → コードを直す → n8n にコピペ → 動作確認」に30分かかっていた作業が、5分になった。

---

## 「副業」との接続

このシステムを副業に使うには、**出力レイヤーが収益に繋がっている必要がある**。

俺の場合：

```
X投稿 → フォロワー獲得 → note / TYL への誘導
Zenn記事 → Google検索流入 → blog → TYL月額会員
Obsidianナレッジ → Claude Code セッション効率化 → 実装速度向上 → 教材化
```

「自動化したこと」が直接お金になるのではなく、**自動化がコンテンツ発信の速度と質を上げ、それが収益に繋がる**という構造だ。

---

## 次にやること

現在設計中のWF-ZENN-TRIAGE が完成すれば、Obsidianで書いた記事が Slack 承認 → Zenn 自動公開というフローになる。コンテンツの出力速度がさらに上がる予定。

n8n の WF 設計ノウハウは THINK YOU LAB で詳しく共有している。

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
