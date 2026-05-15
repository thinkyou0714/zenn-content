---
title: "ObsidianをAIの第二の脳にした：7本のn8nワークフローで作ったナレッジ自動管理システム"
emoji: "🧠"
type: "tech"
topics: ["obsidian", "n8n", "automation", "claudecode"]
published: false
publish_scheduled: "2026-05-23T09:00:00+09:00"
hold_reason: ""
next_action: "X先出しツイート 09:30 + 30分後リプライで本文URL"
stage: "SCHEDULED"
---

## TL;DR

- Obsidianの毎日の運用（収集・分類・埋め込み・サマリー生成）を n8n の7本のWFで完全自動化した
- 埋め込みパイプラインで HOT layer（頻繁にアクセスするコンテキスト）を 100KB → 49KB に圧縮（-51%）
- タグ付け・MOC生成・レビュー提示がゼロ作業になった
- 「書いたら終わり」のメモが「AIが参照する知識ベース」に変わった

:::message
技術スタック: **Obsidian + n8n（セルフホスト）+ Supabase（pgvector）+ Anthropic API**。月次コストは数千円以内。
:::

---

## 「第二の脳」が機能しなかった本当の理由

Obsidian を使い始めた頃、俺はひたすらメモを取り続けていた。毎日20〜30ノートが増える。タグも貼る。リンクも張る。でも3ヶ月後に見返すと、ほとんど使っていない。

「書き方が悪いのか」「整理の仕方が足りないのか」とずっと思っていた。

でも本当の問題はそこじゃなかった。**「参照される仕組み」がなかった**のだ。

どんなに丁寧にメモを書いても、それが「必要なタイミングに自動で浮上してくる」設計でなければ、書いた瞬間から忘れていく一方だ。Obsidian は保存の道具として優秀だが、「参照の自動化」は自分で作らないといけない。

それがわかったとき、n8n の出番だと思った。

---

## 7本のワークフロー：何を自動化したか

```
WF-EMBED        毎時:     ノートをベクトル化してSupabaseに格納
WF-TRIAGE       毎時:     未整理ノートを分類・タグ付け
WF-MOC          毎日 2:00: 関連ノートのMOC（目次）を自動生成
WF-REVIEW       毎日 9:00: 7日前のノートをレビュー候補として提示
WF-DAILY-NOTE   毎朝 6:30: デイリーノートを自動生成（昨日の要約付き）
WF-DAILY-BRIEF  毎朝 7:00: RSS + X bookmarks → 当日の情報ブリーフィング
WF-CLEANUP      毎週日曜:  重複ノート・孤立ノートの検出とアーカイブ
```

「ノートを書く」作業だけ人間がやって、あとは全部自動で回る。これが目標だった。

設計するとき、一つ原則を決めた。「俺がやりたくない作業だけを自動化する」。楽しい作業まで自動化すると、何のためにObsidianを使っているかわからなくなる。

---

## WF-EMBED：これがなければ何も始まらない

7本の中で一番重要なのがこれだ。なぜかというと、他のWFが全部「意味検索ができること」を前提に設計されているからだ。

### タグ検索の限界を感じていた

以前はタグで管理していた。でもある日、「先週書いた n8n のエラー解決メモ」を探せなかった。タグが `n8n` でも `エラー` でもなく、「試行錯誤」みたいなタグを付けていたからだ。

ベクトル検索は「意味の近さ」で検索する。「Codeノードが動かない理由」で検索すると、「JavaScript の require が n8n サンドボックスで使えない件」が引っかかる。意味的に近いからだ。これがタグ検索では絶対に出てこない。

### 実装

```javascript:WF-EMBED（n8n Code ノード）
const notes = $input.all();
const results = [];

for (const note of notes) {
  const content = note.json.content;
  const path = note.json.path;
  
  // 差分更新：更新されていないノートはスキップ
  const lastEmbedded = await getLastEmbeddedAt(path);
  if (note.json.updated_at <= lastEmbedded) continue;
  
  // 6000字超のノートはチャンク分割（overlap 200字で文脈を保持）
  const chunks = content.length > 6000
    ? splitIntoChunks(content, 6000, { overlap: 200 })
    : [content];
  
  for (let i = 0; i < chunks.length; i++) {
    const embedding = await generateEmbedding(chunks[i]);
    results.push({
      note_path: path,
      chunk_index: i,
      total_chunks: chunks.length,
      content: chunks[i],
      embedding: embedding,
      updated_at: new Date().toISOString()
    });
  }
}

return results;
```

### Supabase pgvector のスキーマ

```sql:obsidian_embeddings.sql
CREATE TABLE obsidian_embeddings (
  id          BIGSERIAL PRIMARY KEY,
  note_path   TEXT NOT NULL,
  chunk_index INT  NOT NULL DEFAULT 0,
  content     TEXT NOT NULL,
  embedding   VECTOR(1536),  -- text-embedding-3-small
  updated_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(note_path, chunk_index)
);

CREATE INDEX ON obsidian_embeddings
  USING ivfflat (embedding vector_cosine_ops)
  WITH (lists = 100);
```

:::message
埋め込みモデルは `text-embedding-3-small`（次元数1536）を使っている。`ada-002` より精度が高く、`3-large` の1/5以下のコスト。月数十円で収まる。
:::

---

## HOT layer：コンテキスト圧縮が生産性を変えた

Claude Code を使っていて気づいたのは、「セッション開始時のコンテキスト読み込み」が地味にきつい問題だということだ。

俺のプロジェクトでは毎セッション「自分は誰で、何をしているか、どんなルールがあるか」を伝えるためのファイルを渡している。これが膨らむと、セッションを開いた瞬間からコンテキストの35%が消える。

これを解決するために設計したのが HOT layer だ。

```
HOT layer（Claude Code が毎セッション最初に読む最小セット）
  ├── TASKS.md          - 今やること（常に最新）
  ├── session_handoff/  - 直近7日のセッションサマリー
  ├── DECISIONS.md      - 設計上の重要な意思決定
  └── rules/            - brand-voice / offer-boundary ルール
```

WF-DAILY-BRIEF が毎朝この layer を整理する。完了タスクの削除、古いハンドオフのアーカイブ、7日以上前のセッション記録の圧縮。

| 項目 | Before | After |
|---|---|---|
| HOT layer サイズ | 100 KB | 49 KB（**-51%**） |
| コンテキスト使用率（開始直後） | 約35% | 約17% |
| 1セッションで扱える追加情報量 | 基準 | **約2倍** |

この数字を見たとき、「コンテキスト管理はシステムの問題だ」と思った。Claude の能力の問題じゃない。俺がコンテキストを垂れ流していた問題だ。

---

## WF-TRIAGE：書いたら自動でタグが付く

毎時実行される、地味だけど一番恩恵を感じているフローだ。

```javascript:WF-TRIAGE（n8n Code ノード）
const prompt = `
以下のメモに適切なタグを付けてください。

## メモ
${note.content.slice(0, 2000)}

## 出力形式（JSON）
{
  "tags": ["tag1", "tag2"],   // 最大5個
  "category": "tech|idea|log|reference|task",
  "priority": "hot|warm|cold",
  "summary": "1行要約（30字以内）"
}
`;
```

手動タグ付けをやめたのは「タグを付ける判断をするのが面倒」だったからだ。書き終わった直後に「これに何のタグをつけるか」を考えるのは認知負荷が高い。書くことに集中したい。

今は書いて保存すれば、1時間以内にタグが付いている。それでいい。

---

## WF-REVIEW：「書いたら終わり」を根本から解決した

この設計が一番気に入っている。

「スペース繰り返し（Spaced Repetition）」はAnkiが有名な学習法だ。忘却曲線に合わせて適切なタイミングで復習させることで、記憶の定着を高める。

これをメモ管理に応用した。

```javascript:WF-REVIEW（n8n Code ノード）
// 7日前に書いたノートをランダムに3〜5件取得してSlackに通知
const reviewCandidates = await supabase
  .from('obsidian_notes')
  .select('path, title, summary')
  .gte('created_at', sevenDaysAgo)
  .lt('created_at', sixDaysAgo)
  .eq('reviewed', false)
  .limit(5);

await sendToSlack('#lab-review', 
  `今日の復習候補（7日前に書いたノート）:\n` +
  reviewCandidates.map(n => `- *${n.title}*\n  ${n.summary}`).join('\n')
);
```

7日前に書いたメモが「覚えてる？」と聞いてくる。この設計のポイントは、「どのメモを見返すか」の選択を自分でしなくていいことだ。選択疲れがない。

実際、WF-REVIEW が送ってくるメモを見て「あ、これだ」と思うことが週に2〜3回ある。同じ問題に別の角度で再び当たったとき、過去の自分のメモが役に立つ。

---

## WF-MOC：使うまで価値がわからなかった

MOC（Map of Content）とは、関連するノートをまとめた目次ノートのことだ。正直、最初は「そんなの手動で作ればいいじゃないか」と思っていた。

でも自動生成してみてわかった。**手動では絶対に気づかない関連性が出てくる**。

```javascript:WF-MOC（自動生成部分）
// ベクトル類似度でノートをクラスタリング
const clusters = await clusterByEmbedding(allNotes, {
  minClusterSize: 3,
  similarityThreshold: 0.75
});

for (const cluster of clusters) {
  const topicName = await inferTopicName(cluster);
  await upsertNote(`00_MOC/${topicName}.md`, generateMOC(topicName, cluster));
}
```

:::details 自動生成されたMOCの例
```markdown
# MOC: n8n × Claude Code 統合

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
:::

「自分が書いたのに知らなかった関連性」に気づかされる感覚は新鮮だった。

---

## 実装前後の変化

### 毎日の作業時間

| 作業 | Before | After |
|---|---|---|
| タグ付け | 5〜10分/日（手動） | 0分（WF-TRIAGE が自動） |
| MOC更新 | 30分/週（手動） | 0分（WF-MOC が自動） |
| 情報収集整理 | 30分/朝 | 0分（WF-DAILY-BRIEF が要約） |
| デイリーノート作成 | 5分/朝 | 0分（WF-DAILY-NOTE が自動生成） |
| レビュー候補の選定 | 不定期（忘れがち） | 0分（WF-REVIEW が毎日提案） |

毎日 40〜50分、毎週 5〜6時間の節約になった。でも数字より大事なことがある。「やらなければいけない作業」から解放されることで、Obsidianを使うこと自体が楽になった。

---

## ハマったポイント

:::message alert
**Dockerネットワークの罠。** n8n はコンテナ内で動いている。`localhost:27123` で Obsidian に接続しようとすると、コンテナ自身を指すため接続できない。
:::

```javascript
// NG: コンテナ内の localhost はコンテナ自身を指す
const obsidianUrl = 'http://localhost:27123';

// OK: host.docker.internal がホストマシンを指す
const obsidianUrl = 'http://host.docker.internal:27123';
```

最初にこれで1時間溶かした。n8n の Docker 環境あるあるだが、知らないと詰まる。

---

## 次にやること

WF-EMBED の検索精度をさらに上げたい。今は「ノート全体」を1つの単位として埋め込んでいるが、見出し単位でチャンクを切ることで、より細かい粒度の検索ができるはずだ。

Obsidianの知識ベースが育つほど、Claude Code のセッション品質が上がる。これが「投資が蓄積していく」感覚の正体だと思っている。

---

## 関連記事

- [n8n × Claude Code で55本のWFを動かしている：副業自動化システムの全体像](/articles/n8n-claudecode-automation-overview)
- [Claude Code hooksを47本実装した話：AIへの自動指示を設計するという仕事](/articles/claude-code-hooks-47)

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
