---
title: "n8n × Claude Code で55本のWFを動かしている：副業自動化システムの全体像【2026年版】"
emoji: "⚙️"
type: "tech"
topics: ["n8n", "claudecode", "automation", "個人開発"]
published: false
publish_scheduled: "2026-05-20T09:00:00+09:00"
hold_reason: ""
next_action: "X先出しツイート 09:30 + 30分後リプライで本文URL"
stage: "SCHEDULED"
---

## TL;DR

- n8n（ワークフロー自動化）と Claude Code（AIコーディング）を組み合わせると「勝手に動く副業システム」が作れる
- 現在55本のWFが動いており、毎朝の情報収集・コンテンツ生成・X投稿管理が自動化されている
- 重要なのはツールではなく「3レイヤー設計（収集/処理/出力）」という構造の考え方

:::message
環境: 自宅NUCマシン（AMD Radeon / RAM 32GB）でn8nをセルフホスト。月額コストはほぼ電気代のみ。
:::

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

これが「副業自動化システム」の現実だ。劇的じゃない。地味だ。でも毎日確実に動く。自動化のロマンを語る記事は多いが、実際に動かし続けるのがどんな体験かを書いた記事は少ない。俺の記録はその視点から書いている。

---

## なぜ n8n なのか：Make / Zapier を使わない理由

よく聞かれる。「Make（旧Integromat）やZapierじゃダメなの？」

使ってみたことがある。どちらも悪くない。特に Make は UI が洗練されていて、最初の自動化を作るには快適だ。

でも俺が n8n に移行した理由は3つある。

### 1. コストが積み重なる問題

| 比較軸 | Make | Zapier | **n8n（セルフホスト）** |
|---|---|---|---|
| 月額コスト | 9〜29ドル/月 | 20〜69ドル/月 | **0円** |
| WF実行数上限 | あり | あり | **無制限** |
| データ保持先 | クラウド | クラウド | **ローカル** |
| カスタムコード | 制限あり | 制限あり | **JavaScriptフル** |

55本のWFが動いていたら、Make や Zapier では月数万円になる。セルフホストなら電気代だけだ。

### 2. Code ノードがなければ無理な処理がある

Make や Zapier のロジックだけでは表現できない処理が山ほどある。Obsidian のノートをベクトル化する、スコアリングアルゴリズムを書く、エラーをカテゴリ別に分類する。こういうカスタムロジックは JavaScript で書ける Code ノードが必要だ。

```javascript:n8n Code ノード（サンドボックス制約対応版）
// process.env は使えない → $env を使う
const key = $env.ANTHROPIC_API_KEY ?? '';

// new URL() はサンドボックスで動かない → require で取得
const { URL: _URL } = require('url');
const url = new _URL('https://api.example.com/path');
```

:::message alert
n8n の Code ノードにはサンドボックス制約がある。`process.env.X` は使えない（`$env.X` を使う）。`new URL()` も使えない。知らないとランタイムエラーで詰まる。
:::

### 3. Claude Code と直接連携できる

WFのエラーを Claude Code に直接修正させられる。「このコードノードの TypeError を直して、n8n API 経由でインポートして」と言えば、Claude がコードを修正して API を叩いて更新まで完了する。

```
以前: エラー調査 → コード修正 → n8n にコピペ → 動作確認 → 30分
今:   「このWFのエラーを直して」と言うだけ → 5分
```

---

## 3レイヤー設計という考え方

55本を設計するとき、バラバラに作ると管理できなくなると思った。

ある日、全WFを並べて眺めていたら、自然に3つのグループに分かれることに気づいた。

```
Layer 1: 収集レイヤー（情報を集める）
Layer 2: 処理レイヤー（情報を変換・分析する）
Layer 3: 出力レイヤー（外部に出力する）
```

:::message
この3レイヤーを意識することが一番重要だ。収集だけあっても出力がなければ機能しない。処理が弱ければ情報が溜まるだけになる。どこかが欠けると全体が死ぬ。
:::

---

## Layer 1（収集）：情報が自動で入ってくる

| WF名 | トリガー | 収集元 |
|---|---|---|
| WF-BOOKMARK | 毎時 | X（Twitter）ブックマーク |
| WF-AI-NEWS | 毎時 | RSSフィード×30媒体 |
| WF-X-CANDIDATE | 毎日 | X検索（フォロー候補） |
| WF-EMBED | 毎時 | Obsidian vault（差分） |
| WF-TRIAGE | 毎時 | Obsidian inbox（未整理） |
| WF-CLIP-DIGEST | Webhook | Chrome拡張からのクリップ |

1日あたり、ブックマーク10〜30件、AIニュース50〜100件が自動収集される。俺は何もしていない。

**設計の核心**: 収集時にフィルタリングしない。全部集めて、処理レイヤーで仕分ける。収集時に判断を入れると「重要かもしれないものを捨てる」リスクがある。

```javascript:収集エラーをハンドリングするパターン
// 1つのフィードが死んでも全体を止めない
for (const feed of feeds) {
  try {
    results.push(...await fetchFeed(feed.url));
  } catch (err) {
    await logError('WF-AI-NEWS', feed.url, err.message);
    continue;  // ここが重要
  }
}
```

---

## Layer 2（処理）：情報を使える形に変える

ここが一番設計に時間をかけた。収集した情報をどう「価値」に変えるか。

| WF名 | 処理内容 |
|---|---|
| WF-DAILY-BRIEF | 昨日のブックマーク → 要約5件 + 重要度スコア |
| WF-X-ANALYZE | X投稿のパフォーマンス分析 → Supabase蓄積 |
| WF-MOC | 関連ノートのクラスタリング → MOC自動生成 |
| WF-REVIEW | 7日前ノートの再浮上 → Slack通知 |
| WF-BRIEF-TRIAGE | BriefのスコアリングでTop候補を昇格 |

### スコアリングロジック

```javascript:WF-BRIEF-TRIAGE（スコアリング）
function scoreNote(note) {
  let score = 0;
  
  // いいね・引用数
  score += Math.min(note.likes / 10, 50);
  score += Math.min(note.retweets * 3, 30);
  
  // TYLとの関連性ボーナス
  const keywords = ['claude', 'n8n', 'obsidian', '副業', '自動化'];
  score += keywords.filter(kw => note.content.includes(kw)).length * 5;
  
  // 著者フォロワー数（対数）
  score += Math.log10(note.author_followers + 1) * 2;
  
  return Math.min(score, 100);
}
// 75点以上 → 自動でBrief候補に昇格
```

スコア75以上のものだけ次のフローに進む。これにより「全部見ないといけない感覚」がなくなった。

---

## Layer 3（出力）：人間の承認だけで外に出る

ここで俺が強くこだわっているのは、**外部への出力は必ず人間の確認を挟む**ことだ。

| WF名 | 出力先 | 承認フロー |
|---|---|---|
| WF-X-SCHEDULER | X（自動投稿） | Slack #lab-x で承認後 |
| WF-ZENN-TRIAGE | GitHub→Zenn | Slack #lab-zenn で承認後 |
| WF-WEEKLY-RPT | Slack | 自動（レポートのみ） |
| WF-DAILY-NOTE | Obsidian | 自動（外部投稿ではないため） |

全自動投稿にしない理由は単純だ。**1回のミスが取り返しつかない**。AIが生成したコンテンツを確認なしで出すリスクは、承認の手間に見合わない。

「自動化する」と「全自動にする」は別のことだ。俺は前者を目指している。

---

## Claude Code との連携：WFをAIに修正させる

これが n8n × Claude Code の一番の価値だと思っている。

```python:n8n_deploy.py（Claude Code が使う）
import json, urllib.request

def update_workflow(wf_id, payload, api_key, base_url="http://localhost:5679"):
    # PUT で送れるフィールドは name/nodes/connections/settings のみ
    # id/active/tags は read-only → 送ると400エラー
    safe_payload = {k: v for k, v in payload.items()
                    if k in ('name', 'nodes', 'connections', 'settings', 'staticData')}
    
    data = json.dumps(safe_payload, ensure_ascii=False).encode('utf-8')
    req = urllib.request.Request(
        f"{base_url}/api/v1/workflows/{wf_id}",
        data=data,
        headers={"X-N8N-API-KEY": api_key, "Content-Type": "application/json; charset=utf-8"},
        method="PUT"
    )
    with urllib.request.urlopen(req) as r:
        return json.load(r)
```

「WF-X-ANALYZE が TypeError を出している」と言えば、Claude がコードを直してこのスクリプトで更新まで完了する。俺は n8n の UI を一切操作しない。

---

## エラーハンドリング：DLQパターンで障害復旧

55本が動いていると、どこかが必ず失敗する。Slack の API が一時的に落ちる、Obsidian API がタイムアウトする、レート制限に引っかかる。

これを「その都度手動対処」にしたら運用できない。

:::details DLQ（Dead Letter Queue）パターンの全体設計
```
WF実行 → エラー発生
  ↓
WF-ERROR-HANDLER（全WFの errorWorkflow に設定）
  ↓
Supabase n8n_failures テーブルに upsert
  ↓
Slack #lab-n8n に通知
  ↓
WF-DLQ-RETRY（毎時、status=open を自動再実行）
  ↓
3回失敗で abandoned → 毎朝 WF-OPS-FAILURE-REPORT に記載
```
:::

全WFの設定に `"errorWorkflow": "IAARcITmwrAsHjpY"` を入れている。これが入っていないと、エラーが起きても誰も気づかない。

---

## システム全体のコスト

| 項目 | 月次コスト |
|---|---|
| n8n（セルフホスト） | 0円 |
| Anthropic API | 2,000〜5,000円 |
| Supabase | 0円（Free tier） |
| 電気代（NUC 24時間稼働） | 約800円/月 |
| **合計** | **3,000〜6,000円/月** |

Make + Zapier の同等機能で月数万円かかるところを、3〜6千円に抑えられている。これが55本のWFを躊躇なく増やせる理由だ。コストが気になったらWFを削るという思考が必要なくなった。

---

## 「副業」との接続：自動化がお金になる仕組み

「自動化=副収入」という単純な図式は成り立たない。

正しくは「自動化 → コンテンツ発信の速度・質の向上 → 収益」という間接的な構造だ。

```
X投稿（自動生成→承認→投稿）
  → フォロワー獲得 → note / TYL への誘導

Zenn記事（Obsidian → WF-ZENN-TRIAGE → 承認 → 公開）
  → Google検索流入 → think-you-lab.vercel.app → TYL月額会員

Obsidianナレッジ → Claude Code セッション効率化
  → 実装速度向上 → 教材化
```

俺が自動化に投資するのは、「楽をする」ためではなく「発信の質と量を上げる」ためだ。同じ時間でより多くのアウトプットを出せるなら、それが副業の競争力になる。

---

## 次にやること

現在設計中のWF-ZENN-TRIAGE が完成すれば、Obsidianで書いた記事が Slack 承認 → Zenn 自動公開というフローになる。コンテンツの出力速度がさらに上がる予定。

n8n の WF 設計ノウハウは THINK YOU LAB で詳しく共有している。

---

## 関連記事

- [ObsidianをAIの第二の脳にした：7本のn8nワークフローで作ったナレッジ自動管理システム](/articles/obsidian-n8n-ai-pipeline)
- [Claude Code hooksを47本実装した話：AIへの自動指示を設計するという仕事](/articles/claude-code-hooks-47)

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
