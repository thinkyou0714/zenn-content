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

n8n はセルフホストできる。俺の場合 Docker で自宅の NUC マシン（AMD Radeon / RAM 32GB）で動いている。クラウドサービスへのデータ送出を最小限にできるし、月額コストがゼロだ。

```yaml
# docker-compose.yml（最小構成）
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

```javascript
// n8n Code ノード: Obsidian の未整理ノートを取得して分類する
// ← こういうカスタムロジックが書けるのが n8n の強み

const notes = $input.all();
const untagged = notes.filter(n => !n.json.frontmatter?.tags?.length);
const result = await classifyNotes(untagged);
return result;
```

**3. Claude Code との相性**

WFの設計・デバッグを Claude Code に任せられる。「このWFのコードノードを修正して」と言うだけで直してくれる。AI でWFを作る時代になった。

```
俺:     「WF-X-ANALYZE の Code ノードが TypeError を出している」
Claude: 「エラーを確認します。n8n の Code ノードでは new URL() が使えないため、
         require('url').URL を使う必要があります。以下に修正します」
俺:     「じゃあそのまま n8n API 経由でインポートして」
Claude: [自動でAPIに PUT リクエスト → WFを更新]
```

以前は「エラーを調べる → コードを直す → n8n にコピペ → 動作確認」に30分かかっていた作業が、5分になった。

---

## 3レイヤー設計という考え方

55本のWFを分類すると、3つのレイヤーに収まる。

```
Layer 1: 収集レイヤー（情報を集める）
Layer 2: 処理レイヤー（情報を変換・分析する）
Layer 3: 出力レイヤー（外部に出力する）
```

重要なのは「ツール単体を使う」のではなく、**この3レイヤーを意識して設計する**ことだ。

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
| WF-LIE-01 | 毎日 | RSSフィード | SNS最適化情報 |
| WF-CLIP-DIGEST | Webhook | Chrome拡張 | Webページのクリップ |

### 収集量（1日あたりの実績）

```
Xブックマーク:  10〜30件/日
AIニュース:     50〜100件/日（フィルタリング後 5〜10件）
Obsidianノート: 20〜40件更新/日
スクリーンショット: 5〜15件/日
```

### 収集レイヤーの設計原則

**原則1: 収集は徹底的に、処理は後で**

収集時に「これは重要か？」を判断しない。全部収集して、Layer 2 で処理する。

```javascript
// NG: 収集時にフィルタリング → 重要な情報を見落とす可能性
const notes = await getBookmarks({ minLikes: 100 });

// OK: 全件収集 → スコアリングは処理レイヤーで
const notes = await getBookmarks({ limit: 100 });
```

**原則2: 収集エラーは全体を止めない**

1つのフィードが死んでも他の収集は続く設計にする。

```javascript
for (const feed of feeds) {
  try {
    const items = await fetchFeed(feed.url);
    results.push(...items);
  } catch (err) {
    // エラーをログに記録して続行
    await logError('WF-AI-NEWS', feed.url, err.message);
    continue;  // ← これが重要
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

### WF-DAILY-BRIEF の処理フロー

```
入力: 昨日のXブックマーク（生データ）
  ↓
Step 1: 重複排除（同じURLが複数あれば1件に）
  ↓
Step 2: スコアリング（いいね数・引用数・著者フォロワー数で重み付け）
  ↓
Step 3: Top 5 を Claude API で要約（重要度・TYLとの関連性を含む）
  ↓
Step 4: 今日のタスク一覧と合わせてブリーフィングを生成
  ↓
出力: Slack #lab-daily に送信
```

### スコアリングロジックの例

```javascript
// WF-BRIEF-TRIAGE のスコアリング
function scoreNote(note) {
  let score = 0;
  
  // 基本スコア: いいね数・引用数
  score += Math.min(note.likes / 10, 50);
  score += Math.min(note.retweets * 3, 30);
  
  // TYLとの関連性ボーナス
  const keywords = ['claude', 'n8n', 'obsidian', '副業', '自動化', 'automation'];
  const relevance = keywords.filter(kw => 
    note.content.toLowerCase().includes(kw)
  ).length;
  score += relevance * 5;
  
  // 著者のフォロワー数ボーナス（対数スケール）
  score += Math.log10(note.author_followers + 1) * 2;
  
  return Math.min(score, 100);
}
```

スコアが 75 以上は「今日の有望コンテンツ」として自動でBrief候補に昇格する。

---

## Layer 3（出力）：人間の意思決定だけで外に出る

出力レイヤーの設計思想は「AIが準備して、人間が承認する」だ。

### 主要WF一覧

| WF名 | トリガー | 出力先 | 承認フロー |
|---|---|---|---|
| WF-X-SCHEDULER | Slack承認後 | X（自動投稿） | Slack #lab-xで承認 |
| WF-DAILY-NOTE | 毎朝 6:30 | Obsidian | 自動（承認不要） |
| WF-WEEKLY-RPT | 毎週月曜 | Slack | 自動（レポートのみ） |
| WF-ZENN-TRIAGE | 毎日 9:00 | GitHub→Zenn | Slack #lab-zennで承認 |
| WF-NOTE-PUBLISH | Webhook | note.com | Slack承認後 |
| WF-POKEMON-DAILY | 毎日 | Supabase+Slack | 自動 |

### 出力レイヤーの設計原則

**原則: 外部への出力は必ず人間承認を経由する**

X・note・Zenn への公開は、AIが準備した下書きを Slack で確認してから承認する。承認ボタンを押すと Webhook が叩かれて実際の投稿が実行される。

```
AI が下書き生成 → Slack に通知 → 人間が確認・承認 → 自動投稿
```

全自動投稿にしない理由は「AIが生成したコンテンツをノーチェックで出すリスク」だ。1回のミスが信頼を大きく損なう。

---

## Claude Code との連携：WFをAIに任せる実装

n8n のWFを作る・直す作業を Claude Code に任せると、効率が跳ね上がった。

### Claude が n8n API を直接叩く

```python
# Claude Code が n8n REST API 経由でWFを更新
import json, urllib.request

def update_workflow(wf_id, payload, api_key, base_url="http://localhost:5679"):
    data = json.dumps(payload, ensure_ascii=False).encode('utf-8')
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

# PUT で送れるフィールドは name/nodes/connections/settings のみ
# id/active/tags/createdAt 等は read-only → 送ると400エラー
payload = {
    "name": wf["name"],
    "nodes": wf["nodes"],
    "connections": wf["connections"],
    "settings": wf.get("settings", {})
}
```

### WF設計のセッション例

```
# セッション1: エラー修正
俺:     「WF-X-ANALYZE の Code ノードが TypeError: Cannot read property 'data' of undefined」
Claude: 「入力データが undefined の場合のガードが必要です。修正します」
        [n8n API で PUT → 自動更新]
俺:     「確認して」
Claude: [n8n UI で実行 → ログ確認] 「正常に実行されました」

# セッション2: 新規WF作成
俺:     「毎朝のnote記事候補をObsidianから取得してSlackに送るWFを作りたい」
Claude: [SPEC確認 → JSON生成 → n8n API で POST → 新規WF作成]
        「WF-NOTE-TRIAGE を作成しました。ID: xxxxxxxx。動作確認しますか？」
```

---

## システム全体のコスト

55本のWFを動かすための月次コスト。

| 項目 | 月次コスト | 備考 |
|---|---|---|
| n8n（セルフホスト） | 0円 | NUCマシンの電気代のみ |
| Anthropic API | 2,000〜5,000円 | WF内のClaude呼び出し |
| Supabase | 0円 | Free tier（pgvector含む） |
| GitHub Actions | 0円 | Free tier内 |
| Vercel | 0円 | Free tier |
| 電気代（NUC 24時間稼働） | 約800円/月 | 15W × 24h × 30日 × 27円/kWh |

**合計: 3,000〜6,000円/月**

これで55本のWFが24時間365日稼働する。同等の機能を Make + Zapier + API サービスで実現しようとすると、月数万円になる。

---

## 「副業」との接続

このシステムを副業に使うには、**出力レイヤーが収益に繋がっている必要がある**。

俺の場合：

```
X投稿（自動生成→承認→投稿）
  ↓
フォロワー獲得（現在 1,400+）
  ↓
note / TYL への誘導

Zenn記事（Obsidian → WF-ZENN-TRIAGE → 自動公開）
  ↓
Google検索流入
  ↓
think-you-lab.vercel.app → TYL月額会員

Obsidianナレッジ
  ↓
Claude Code セッション効率化
  ↓
実装速度向上 → 教材化
```

「自動化したこと」が直接お金になるのではなく、**自動化がコンテンツ発信の速度と質を上げ、それが収益に繋がる**という構造だ。

---

## 55本のWFを管理するためのインフラ

WFが増えるにつれて「WFの管理」が問題になった。

### WF一元管理の仕組み

```
n8n_workflows/    ← WF JSONの正本（git管理）
  WF-EMBED.json
  WF-TRIAGE.json
  WF-MOC.json
  ...（55本）

docs/
  SPEC-n8n-skill.md  ← 全WFの仕様・既知エラー・デプロイ手順

tools/n8n-ci/
  n8n-wf-lint.mjs    ← 全WFの自動lint（pre-push hookで必須実行）
```

### エラーハンドリングの仕組み

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
3回失敗で abandoned → 毎朝の WF-OPS-FAILURE-REPORT に記載
```

この「DLQ（Dead Letter Queue）パターン」で、一時的な外部APIのダウンや rate limit による失敗を自動でリカバリーできる。

---

## 次にやること

現在設計中のWF-ZENN-TRIAGE が完成すれば、Obsidianで書いた記事が Slack 承認 → Zenn 自動公開というフローになる。コンテンツの出力速度がさらに上がる予定。

n8n の WF 設計ノウハウは THINK YOU LAB で詳しく共有している。

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
