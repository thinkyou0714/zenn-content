---
title: "n8n × Claude Code で55本のWFを動かしている：副業自動化システムの全体像【2026年版】"
emoji: "⚙️"
type: "tech"
topics: ["n8n", "claudecode", "automation", "個人開発"]
published: false
---

## TL;DR

- n8n（ワークフロー自動化）と Claude Code（AIコーディング）を組み合わせると「勝手に動く副業システム」が作れる
- 現在55本のWFが動いており、毎朝の情報収集・コンテンツ生成・X投稿管理が自動化されている
- 重要なのはツールではなく「3レイヤー設計（収集/処理/出力）」という構造の考え方
- Make（旧Integromat）・Zapierとの比較、セルフホスト環境の構成も解説

:::message
**環境**: AMD Radeon / RAM 32GB の自宅NUCマシン（NUCBOX-M7）でn8nをセルフホスト。月額コストはほぼゼロ。
:::

---

## この記事でわかること

- n8n をセルフホストして55本のWFを運用する設計方針
- Make / Zapier との違いとn8nを選ぶべき理由
- 3レイヤー設計（収集/処理/出力）の具体的な実装
- Claude Code が n8n API を直接叩いてWFを自動修正するパターン
- DLQ（Dead Letter Queue）パターンによる障害復旧設計

---

## 毎朝Slackに届くもの

俺の毎朝はこんな感じだ。

起きてSlackを開くと、AIが夜中に作業していた結果が届いている。

```
#lab-brief:   昨日のブックマーク記事 5本を要約 + 今日のタスク一覧
#lab-x:       今日投稿できるX下書き 3本（承認待ち）
#lab-n8n:     エラーが出たWFの通知（あれば）
#lab-daily:   昨日のセッションサマリー + 次にやること
#lab-pokemon: ポケモン育成システムの日次進捗（デイリー報酬）
```

手動でやったことはゼロ。全部 n8n と Claude が勝手にやってくれている。

これが「副業自動化システム」の現実だ。劇的じゃない。地味だ。でも毎日確実に動く。

---

## なぜ n8n × Claude Code なのか

よく聞かれる。「Make（旧Integromat）やZapierじゃダメなの？」

### Make / Zapier との比較

| 比較軸 | Make | Zapier | **n8n（セルフホスト）** |
|---|---|---|---|
| 月額コスト | 9〜29ドル/月 | 20〜69ドル/月 | **0円** |
| カスタムコード | 制限あり | 制限あり | **JavaScriptフル実装** |
| データ保持先 | クラウド | クラウド | **ローカル（自宅マシン）** |
| WF実行数上限 | あり | あり | **無制限** |
| セルフホスト | 不可 | 不可 | **可能** |
| Claude Code との連携 | 間接的 | 間接的 | **直接API連携** |

理由は3つある。

**1. セルフホスト可能**

n8n はセルフホストできる。クラウドサービスへのデータ送出を最小限にできるし、月額コストがゼロだ。

```yaml:docker-compose.yml（n8n最小構成）
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5679:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    volumes:
      - n8n_data:/home/node/.n8n
```

**2. Code ノードが使える**

n8n の Code ノード（JavaScript）を使えば、API が存在しないサービスへの操作やカスタムロジックが書ける。Make や Zapier では無理なことが多い。

:::message alert
**n8n Code ノードのサンドボックス制約**: コンテナ内で動くため `process.env.X` は使えない。`$env.X` を使うこと。`new URL()` も使えない（`require('url').URL` を使う）。
:::

```javascript:n8n Code ノード（制約対応版）
// NG: process.env は使えない
const key = process.env.ANTHROPIC_API_KEY;

// OK: $env を使う
const key = $env.ANTHROPIC_API_KEY ?? '';

// NG: new URL() はサンドボックスで動かない
const url = new URL('https://api.example.com/path');

// OK: require で URL を取得
const { URL: _URL } = require('url');
const url = new _URL('https://api.example.com/path');
```

**3. Claude Code との相性**

WFの設計・デバッグを Claude Code に任せられる。「このWFのコードノードを修正して」と言うだけで直してくれる。

```
俺:     「WF-X-ANALYZE の Code ノードが TypeError を出している」
Claude: 「エラーを確認します。new URL() は n8n サンドボックスで使えないため、
         require('url').URL を使う必要があります。修正して n8n API 経由でインポートします」
Claude: [自動でAPIに PUT リクエスト → WFを更新]
```

以前は「エラーを調べる → コードを直す → n8n にコピペ → 動作確認」に30分かかっていた作業が、5分になった。

---

## 3レイヤー設計という考え方

55本のWFを分類すると、3つのレイヤーに収まる。

:::message
**設計の核心**: 「収集だけあっても出力がなければ機能しない。処理が弱ければ情報が溜まるだけになる。」この3レイヤーをすべて揃えて初めてシステムが動く。
:::

```
Layer 1: 収集レイヤー（情報を集める）
Layer 2: 処理レイヤー（情報を変換・分析する）
Layer 3: 出力レイヤー（外部に出力する）
```

---

## Layer 1（収集）：情報が自動で入ってくる設計

収集レイヤーは「人間が何もしなくても情報が溜まる」ことが目標だ。

### 主要WF一覧

| WF名 | トリガー | 収集元 | 収集内容 |
|---|---|---|---|
| WF-BOOKMARK | 毎時 | X（Twitter） | ブックマークした投稿 |
| WF-AI-NEWS | 毎時 | RSSフィード×30媒体 | AIニュース |
| WF-X-CANDIDATE | 毎日 | X検索 | フォロー候補アカウント |
| WF-EMBED | 毎時 | Obsidian vault | ノート全件（差分のみ） |
| WF-TRIAGE | 毎時 | Obsidian inbox | 未整理ノート |
| WF-SS-1 | 毎日 | スクリーンショット | 画像 → Obsidian保存 |
| WF-CLIP-DIGEST | Webhook | Chrome拡張 | Webページのクリップ |

### 収集量（1日あたりの実績）

```
Xブックマーク:      10〜30件/日
AIニュース:         50〜100件/日（フィルタリング後 5〜10件）
Obsidianノート更新: 20〜40件/日
スクリーンショット: 5〜15件/日
```

### 収集レイヤーの設計原則

```javascript:収集時のエラーハンドリング
// 1つのフィードが死んでも他の収集は続く設計
for (const feed of feeds) {
  try {
    const items = await fetchFeed(feed.url);
    results.push(...items);
  } catch (err) {
    // エラーをログに記録して続行（全体を止めない）
    await logError('WF-AI-NEWS', feed.url, err.message);
    continue;
  }
}
```

---

## Layer 2（処理）：情報を使える形に変換する

処理レイヤーは「生の情報を価値に変える」場所だ。ここが弱いと情報が溜まるだけで終わる。

### 主要WF一覧

| WF名 | トリガー | 処理内容 | 出力 |
|---|---|---|---|
| WF-DAILY-BRIEF | 毎朝 7:00 | 昨日のブックマーク→要約 | Slack通知 |
| WF-X-ANALYZE | 毎日 | X投稿パフォーマンス分析 | Supabaseに蓄積 |
| WF-MOC | 毎夜 2:00 | 関連ノートの目次生成 | Obsidianに書き戻し |
| WF-REVIEW | 毎朝 9:00 | 7日前のノート再浮上 | Slack通知 |
| WF-BRIEF-TRIAGE | 毎時 | Briefの品質スコアリング | 優先度付き一覧 |
| WF-X-DRAFT | Webhook | ブックマーク→X投稿生成 | Supabaseにdraft保存 |

### スコアリングロジックの例

```javascript:WF-BRIEF-TRIAGE（スコアリング）
function scoreNote(note) {
  let score = 0;
  
  // 基本スコア: いいね数・引用数
  score += Math.min(note.likes / 10, 50);
  score += Math.min(note.retweets * 3, 30);
  
  // TYLとの関連性ボーナス
  const keywords = ['claude', 'n8n', 'obsidian', '副業', '自動化'];
  const relevance = keywords.filter(kw => 
    note.content.toLowerCase().includes(kw)
  ).length;
  score += relevance * 5;
  
  // 著者フォロワー数ボーナス（対数スケール）
  score += Math.log10(note.author_followers + 1) * 2;
  
  return Math.min(score, 100);
}
// スコア 75+ → 自動でBrief候補に昇格
```

---

## Layer 3（出力）：人間の意思決定だけで外に出る

出力レイヤーの設計思想は「AIが準備して、人間が承認する」だ。

### 主要WF一覧

| WF名 | トリガー | 出力先 | 承認フロー |
|---|---|---|---|
| WF-X-SCHEDULER | Slack承認後 | X（自動投稿） | Slack #lab-x で承認 |
| WF-DAILY-NOTE | 毎朝 6:30 | Obsidian | 自動（承認不要） |
| WF-WEEKLY-RPT | 毎週月曜 | Slack | 自動（レポートのみ） |
| WF-ZENN-TRIAGE | 毎日 9:00 | GitHub→Zenn | Slack #lab-zenn で承認 |
| WF-POKEMON-DAILY | 毎日 | Supabase+Slack | 自動 |

:::message
**外部投稿は必ず人間承認を経由する**。全自動投稿にしない理由は「AIが生成したコンテンツをノーチェックで出すリスク」。1回のミスが信頼を大きく損なう。
:::

---

## Claude Code との連携：WFをAIに任せる実装

### Claude が n8n API を直接叩く

```python:n8n_deploy.py（Claude Code が使う）
import json, urllib.request

def update_workflow(wf_id, payload, api_key, base_url="http://localhost:5679"):
    # PUT で送れるフィールドは name/nodes/connections/settings のみ
    # id/active/tags/createdAt は read-only → 送ると400エラー
    safe_payload = {k: v for k, v in payload.items()
                    if k in ('name', 'nodes', 'connections', 'settings', 'staticData')}
    
    data = json.dumps(safe_payload, ensure_ascii=False).encode('utf-8')
    req = urllib.request.Request(
        f"{base_url}/api/v1/workflows/{wf_id}",
        data=data,
        headers={
            "X-N8N-API-KEY": api_key,
            "Content-Type": "application/json; charset=utf-8"
        },
        method="PUT"
    )
    with urllib.request.urlopen(req) as r:
        return json.load(r)
```

---

## エラーハンドリング：DLQパターンで障害復旧

:::details DLQ（Dead Letter Queue）パターンの全体設計
```
WF実行 → エラー発生
  ↓
WF-ERROR-HANDLER（全WF共通のerrorWorkflow）
  ↓
Supabase n8n_failures テーブルに upsert
  ↓
Slack #lab-n8n に通知
  ↓
WF-DLQ-RETRY（hourlyで自動リトライ）
  ↓
3回失敗で abandoned → 毎朝 WF-OPS-FAILURE-REPORT に記載
```

全WFの設定：
```json
{
  "settings": {
    "errorWorkflow": "IAARcITmwrAsHjpY"  // WF-ERROR-HANDLER の ID
  }
}
```
:::

一時的な外部APIのダウンや rate limit による失敗を自動でリカバリーできる。

---

## システム全体のコスト

| 項目 | 月次コスト | 備考 |
|---|---|---|
| n8n（セルフホスト） | 0円 | NUCマシンの電気代のみ |
| Anthropic API | 2,000〜5,000円 | WF内のClaude呼び出し |
| Supabase | 0円 | Free tier（pgvector含む） |
| GitHub Actions | 0円 | Free tier内 |
| Vercel | 0円 | Free tier |
| 電気代（NUC 24時間稼働） | 約800円/月 | 15W × 24h × 30日 × 27円/kWh |

**合計: 3,000〜6,000円/月** で55本のWFが24時間365日稼働する。

---

## 「副業」との接続

このシステムを副業に使うには、**出力レイヤーが収益に繋がっている必要がある**。

```
X投稿（自動生成→承認→投稿）→ フォロワー獲得 → note / TYL への誘導

Zenn記事（Obsidian → WF-ZENN-TRIAGE → 自動公開）
  → Google検索流入 → think-you-lab.vercel.app → TYL月額会員

Obsidianナレッジ → Claude Code セッション効率化
  → 実装速度向上 → 教材化
```

「自動化したこと」が直接お金になるのではなく、**自動化がコンテンツ発信の速度と質を上げ、それが収益に繋がる**という構造だ。

---

## 次にやること

現在設計中のWF-ZENN-TRIAGE が完成すれば、Obsidianで書いた記事が Slack 承認 → Zenn 自動公開というフローになる。コンテンツの出力速度がさらに上がる予定。

---

## 関連記事

このシステムのObsidian側の設計と、Claude Code の hooks 設計については以下を参照してほしい。

- [ObsidianをAIの第二の脳にした：7本のn8nワークフローで作ったナレッジ自動管理システム](/articles/obsidian-n8n-ai-pipeline)
- [Claude Code hooksを47本実装した話：AIへの自動指示を設計するという仕事](/articles/claude-code-hooks-47)

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
