---
title: "ObsidianをAIの第二の脳にした：7本のn8nワークフローで作ったナレッジ自動管理システム"
emoji: "🧠"
type: "tech"
topics: ["obsidian", "n8n", "automation", "claudecode"]
published: false
---

## TL;DR

- Obsidianの毎日の運用（収集・分類・埋め込み・サマリー生成）を n8n の7本のWFで完全自動化した
- 埋め込みパイプラインで HOT layer（頻繁にアクセスするコンテキスト）を 100KB → 49KB に圧縮（-51%）
- タグ付け・MOC生成・レビュー提示がゼロ作業になった
- 「書いたら終わり」のメモが「AIが参照する知識ベース」に変わる設計のポイントを解説

---

## 「第二の脳」はメモを取るためじゃない

Obsidian を使い始めた頃、俺はひたすらメモを取り続けていた。毎日20〜30ノートが増える。タグも貼る。リンクも張る。でも3ヶ月後に見返すと、ほとんど使っていない。

問題はメモの量じゃなかった。**メモが「参照される仕組み」になっていなかった**のだ。

第二の脳として機能させるには、2つのことが必要だと気づいた。

1. **収集の自動化**: 毎日手動でメモを整理していたら続かない
2. **参照のしやすさ**: Claude Code がセッション開始時に「昨日の文脈」を自動で引っ張れること

その2つを解決するために作ったのが、n8n の7本のWFだ。

---

## 7本のワークフロー：全体像とトリガー

```
WF-EMBED        毎時: ノートをベクトル化してSupabaseに格納
WF-TRIAGE       毎時: 未整理ノートを分類・タグ付け
WF-MOC          毎日 2:00: 関連ノートのMOC（目次）を自動生成
WF-REVIEW       毎日 9:00: 7日前のノートをレビュー候補として提示
WF-DAILY-NOTE   毎朝 6:30: デイリーノートを自動生成（昨日の要約付き）
WF-DAILY-BRIEF  毎朝 7:00: RSS + X bookmarks → 当日の情報ブリーフィング
WF-CLEANUP      毎週日曜: 重複ノート・孤立ノートの検出とアーカイブ
```

ポイントは「ノートを書く」作業だけ人間がやって、あとは全部自動で回ることだ。

### WFの依存関係

```
情報が入ってくる
  ↓
WF-TRIAGE （分類・タグ付け）
  ↓
WF-EMBED （ベクトル化 → Supabase）
  ↓
WF-MOC （関連ノートのまとめ）
  ↓
WF-REVIEW / WF-DAILY-NOTE / WF-DAILY-BRIEF（参照・出力）
  ↓
WF-CLEANUP （定期清掃）
```

WF-EMBEDがすべての基盤になっている。ここが動かないと「意味検索」ができない。

---

## WF-EMBED：埋め込みパイプラインの設計

最重要なのが WF-EMBED だ。これがなければ「検索できる知識ベース」にならない。

### なぜベクトル化が必要か

タグ検索には限界がある。「昨日書いたn8nのエラー解決メモ」を探すとき、タグが `n8n` でも `エラー` でもない場合、見つからない。

ベクトル検索（semantic search）は「意味の近さ」で検索できる。「Codeノードが動かない理由」で検索すると、「JavaScript の require が n8n サンドボックスで使えない件」というノートが引っかかる。意味的に近いからだ。

### 実装の全貌

```javascript
// WF-EMBED の Code ノード（n8n）
const notes = $input.all();
const results = [];

for (const note of notes) {
  const content = note.json.content;
  const path = note.json.path;
  
  // 更新済みのノートのみ処理
  const lastEmbedded = await getLastEmbeddedAt(path);
  if (note.json.updated_at <= lastEmbedded) continue;
  
  // 6000文字を超えるノートはチャンクに分割
  const chunks = content.length > 6000
    ? splitIntoChunks(content, 6000, { overlap: 200 })
    : [content];
  
  for (let i = 0; i < chunks.length; i++) {
    const chunk = chunks[i];
    const embedding = await generateEmbedding(chunk);
    
    results.push({
      note_path: path,
      chunk_index: i,
      total_chunks: chunks.length,
      content: chunk,
      embedding: embedding,
      updated_at: new Date().toISOString()
    });
  }
}

return results;

// チャンク分割関数（overlap 付き）
function splitIntoChunks(text, maxLen, { overlap = 100 } = {}) {
  const chunks = [];
  let start = 0;
  while (start < text.length) {
    chunks.push(text.slice(start, start + maxLen));
    start += maxLen - overlap;
  }
  return chunks;
}
```

### Supabase pgvector のスキーマ

```sql
CREATE TABLE obsidian_embeddings (
  id          BIGSERIAL PRIMARY KEY,
  note_path   TEXT NOT NULL,
  chunk_index INT  NOT NULL DEFAULT 0,
  content     TEXT NOT NULL,
  embedding   VECTOR(1536),  -- text-embedding-3-small
  updated_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(note_path, chunk_index)
);

-- コサイン類似度インデックス
CREATE INDEX ON obsidian_embeddings
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
```

### 意味検索のクエリ

```sql
-- 「コンテキストを渡す方法」に意味的に近いノートを検索
SELECT note_path, content,
       1 - (embedding <=> query_embedding) AS similarity
FROM obsidian_embeddings
WHERE 1 - (embedding <=> query_embedding) > 0.7
ORDER BY similarity DESC
LIMIT 5;
```

---

## WF-TRIAGE：自動タグ付けと分類

未整理ノートを自動で分類する。毎時実行で「書いたら自動でタグが付く」状態になる。

### 分類ロジック

```javascript
// WF-TRIAGE の Code ノード
const note = $input.first().json;
const content = note.content;

// Claude API でタグ推定
const prompt = `
以下のメモに適切なタグを付けてください。

## メモ
${content.slice(0, 2000)}

## 出力形式（JSON）
{
  "tags": ["tag1", "tag2", "tag3"],  // 最大5個、英小文字またはカタカナ
  "category": "tech|idea|log|reference|task",
  "priority": "hot|warm|cold",  // hot=今すぐ参照したい, warm=近日中, cold=アーカイブ候補
  "summary": "1行要約"
}
`;

const response = await callClaude(prompt);
const { tags, category, priority, summary } = JSON.parse(response);

// Obsidian ノートのフロントマターを更新
await updateFrontmatter(note.path, {
  tags: [...(note.frontmatter.tags || []), ...tags],
  category,
  priority,
  ai_summary: summary,
  triage_at: new Date().toISOString()
});
```

### 分類カテゴリと優先度

| カテゴリ | 説明 | 保管先 |
|---|---|---|
| `tech` | 技術実装・コード | `10_TECH/` |
| `idea` | アイデア・着想 | `30_IDEAS/` |
| `log` | 作業ログ・日記 | `50_LOG/` |
| `reference` | 参考資料・リンク | `40_REFS/` |
| `task` | タスク・TODO | `60_TASKS/` |

---

## HOT layer の設計：コンテキスト圧縮の核心

Claude Code を使うときの最大の課題は **コンテキストウィンドウの圧迫**だ。毎セッション「自分は誰で、何をしているか」を伝えなければならない。

そこで設計したのが HOT layer だ。

### HOT layer の構成

```
HOT layer（Claude Code が毎セッション最初に読む）
  ├── TASKS.md          - 現在の作業状態（常に最新）
  ├── session_handoff/  - 直近7日のセッションサマリー
  ├── DECISIONS.md      - 設計上の重要な意思決定記録
  └── rules/            - brand-voice / offer-boundary ルール
```

### 圧縮前と圧縮後の比較

| 項目 | Before | After | 削減率 |
|---|---|---|---|
| HOT layer サイズ | 100 KB | 49 KB | **-51%** |
| セッション開始時の読み込み時間 | 約8秒 | 約4秒 | **-50%** |
| コンテキスト使用率（セッション開始直後） | 約35% | 約17% | **-51%** |
| 1セッションで扱える追加情報量 | 基準 | **約2倍** | +100% |

### WF-DAILY-BRIEF による自動圧縮

毎朝7:00に WF-DAILY-BRIEF が以下を実行する。

1. 昨日の session_handoff ファイルを読み込む
2. 完了済みタスクを TASKS.md から削除
3. 7日以上前の古いハンドオフをアーカイブ
4. 当日のブリーフィングを生成してSlackに送信

```javascript
// WF-DAILY-BRIEF の圧縮ロジック
const handoffs = await readRecentHandoffs(7); // 直近7日分
const completedTasks = await getCompletedTasks();

// 完了タスクは git 履歴に残るので削除
await removeCompletedTasks(completedTasks);

// 古いハンドオフをアーカイブ
for (const handoff of handoffs) {
  if (handoff.age_days > 7) {
    await archiveHandoff(handoff);
  }
}

// 当日ブリーフを生成
const brief = await generateDailyBrief({
  bookmarks: await getYesterdaysBookmarks(),
  tasks: await getPendingTasks(),
  sessions: handoffs.slice(0, 3) // 直近3セッションのみ
});

await sendToSlack('#lab-daily', brief);
```

---

## WF-REVIEW：スペース繰り返しで「書いたら終わり」を防ぐ

一番効いているのが WF-REVIEW だ。

「スペース繰り返し（Spaced Repetition）」は、記憶の忘却曲線に合わせて適切なタイミングで復習させる学習法だ。Anki が有名だが、同じ原理をメモ管理に応用した。

### 実装

```javascript
// WF-REVIEW: 7日前に書いたノートをランダムに3〜5件取得
const reviewCandidates = await supabase
  .from('obsidian_notes')
  .select('path, title, summary')
  .gte('created_at', sevenDaysAgo)
  .lt('created_at', sixDaysAgo)
  .eq('reviewed', false)
  .limit(5);

// Slack に「覚えてる？」として通知
const message = reviewCandidates.map(note => 
  `- *${note.title}*\n  ${note.summary}`
).join('\n');

await sendToSlack('#lab-review', 
  `今日の復習候補（7日前）:\n${message}`
);
```

### 効果

```
Before:
  書いたメモ → そのまま埋もれる
  3ヶ月後に見返しても記憶に結びついていない

After:
  書いたメモ → 7日後に再浮上 → 「そういえばこれだ」と記憶に定着
  同じ問題で詰まったとき「以前書いたメモがある」と気づける
```

---

## WF-MOC：関連ノートのMOCを自動生成

MOC（Map of Content）は、関連するノートを一覧化した目次ノートだ。手動で作ると時間がかかるが、WF-MOCが毎日2:00に自動生成する。

```javascript
// WF-MOC: 関連ノートをクラスタリングしてMOCを生成
const clusters = await clusterByEmbedding(allNotes, {
  minClusterSize: 3,
  similarityThreshold: 0.75
});

for (const cluster of clusters) {
  const topicName = await inferTopicName(cluster);
  const mocContent = generateMOC(topicName, cluster);
  
  // 既存のMOCがあれば更新、なければ新規作成
  await upsertNote(`00_MOC/${topicName}.md`, mocContent);
}
```

### 生成されたMOCの例

```markdown
# MOC: n8n × Claude Code 統合

関連ノート（自動生成 2026-04-10）

## WF設計
- [[WF-EMBED設計ログ]]
- [[n8nのCodeノードでfetchを使う方法]]
- [[n8n Webhook v2 bodyネスト問題]]

## デプロイ手順
- [[n8n REST API でWFをインポートする]]
- [[PUT APIで送れるフィールドの制限]]

## エラー解決
- [[UnicodeEncodeError cp932 解決記録]]
- [[async:true と exit 2 非互換]]
```

---

## 実装前後の変化

7本のWFを動かし始めて1ヶ月後の変化を整理する。

### 毎日の作業フロー

| 作業 | Before | After |
|---|---|---|
| タグ付け | 毎回手動（5〜10分） | WF-TRIAGE が自動 |
| MOC更新 | 週1回 手動（30分） | WF-MOC が毎夜自動 |
| 情報収集整理 | 毎朝30分 | WF-DAILY-BRIEF が自動要約 |
| デイリーノート作成 | 毎朝5分 | WF-DAILY-NOTE が自動生成 |
| レビュー（どのメモ見返すか決める） | 不定期（忘れがち） | WF-REVIEW が毎日提案 |
| 重複・孤立ノート整理 | 月1回（1時間） | WF-CLEANUP が毎週自動 |

**合計節約時間: 毎日 40〜50分、毎週 5〜6時間**

### Claude Code セッションへの影響

コンテキスト圧縮の効果が一番大きい。

```
以前: セッション開始 → コンテキスト読み込みに8秒 → 35%消費 → 実作業は65%分だけ
今:   セッション開始 → コンテキスト読み込みに4秒 → 17%消費 → 実作業は83%分使える
```

1セッションで扱える情報量が実質2倍になった。長い作業を1セッションで完了できる頻度が上がった。

---

## 実装する上での注意点

### Obsidian Local REST API の設定

n8n から Obsidian のノートを読み書きするには、[Obsidian Local REST API](https://github.com/coddingtonbear/obsidian-local-rest-api) プラグインが必要だ。

```
設定:
  ポート: 27123（デフォルト）
  API Key: .env で管理
  
n8n（Docker）から接続する場合:
  NG: http://localhost:27123
  OK: http://host.docker.internal:27123
  
理由: n8nはコンテナ内で動いており、localhostはコンテナ自身を指す
```

### pgvector の次元数と埋め込みモデルの選択

| モデル | 次元数 | 月次コスト目安 | 品質 |
|---|---|---|---|
| text-embedding-3-small | 1536 | 数十円/月 | ○ |
| text-embedding-3-large | 3072 | 数百円/月 | ◎ |
| text-embedding-ada-002 | 1536 | 数十円/月 | △（旧世代） |

俺は `text-embedding-3-small` を使っている。コスト対効果が高い。

---

## 次にやること

次は WF-EMBED の検索精度を上げたい。現状は「ノート全体」を埋め込んでいるが、見出し単位でチャンクを切ることでより精度の高い検索ができるはずだ。

また、Zennへの投稿候補を書いたら自動で品質チェックして Slack に通知する WF-ZENN-TRIAGE も設計中。

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
