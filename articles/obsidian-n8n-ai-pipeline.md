---
title: "ObsidianをAIの第二の脳にした：7本のn8nワークフローで作ったナレッジ自動管理システム"
emoji: "🧠"
type: "tech"
topics: ["obsidian", "n8n", "automation", "claudecode"]
published: true
---

## TL;DR

- Obsidianの毎日の運用（収集・分類・埋め込み・サマリー生成）を n8n の7本のWFで完全自動化した
- 埋め込みパイプラインで HOT layer（頻繁にアクセスするコンテキスト）を 100KB → 49KB に圧縮（-51%）
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

## 7本のワークフロー全体像

```
WF-EMBED        毎時: ノートをベクトル化してSupabaseに格納
WF-TRIAGE       毎時: 未整理ノートを分類・タグ付け
WF-MOC          毎日: 関連ノートのMOC（目次）を自動生成
WF-REVIEW       毎日: 7日前のノートをレビュー候補として提示
WF-DAILY-NOTE   毎朝: デイリーノートを自動生成（昨日の要約付き）
WF-DAILY-BRIEF  毎朝: RSS + X bookmarks → 当日の情報ブリーフィング
WF-CLEANUP      毎週: 重複ノート・孤立ノートの検出とアーカイブ
```

ポイントは「ノートを書く」作業だけ人間がやって、あとは全部自動で回ることだ。

---

## 埋め込みパイプライン（WF-EMBED）の設計

最重要なのが WF-EMBED だ。これがなければ「検索できる知識ベース」にならない。

```javascript
// WF-EMBED の Code ノード（n8n）
const notes = $input.all();
const results = [];

for (const note of notes) {
  const content = note.json.content;
  
  // 6000文字を超えるノートはチャンクに分割
  const chunks = content.length > 6000
    ? splitIntoChunks(content, 6000)
    : [content];
  
  for (const chunk of chunks) {
    const embedding = await generateEmbedding(chunk);  // Claude API
    results.push({
      note_path: note.json.path,
      content: chunk,
      embedding: embedding,
      updated_at: new Date().toISOString()
    });
  }
}

return results;
```

Supabase の pgvector に格納することで、「意味的に近いノートを検索する」ことができる。タグ検索では見つからない「文脈の近さ」で情報を引っ張れるようになった。

---

## HOT layer の設計とコンテキスト圧縮

Claude Code を使うときの最大の課題は **コンテキストウィンドウの圧迫**だ。毎セッション「自分は誰で、何をしているか」を伝えなければならない。

そこで設計したのが HOT layer だ。

```
HOT layer (頻繁に参照されるコンテキスト)
  ├── 現在のタスク状態 (TASKS.md)
  ├── 直近7日のセッションハンドオフ
  ├── プロジェクトの意思決定記録 (DECISIONS.md)
  └── brand-voice / offer-boundary ルール
```

WF-DAILY-BRIEF が毎朝これらをまとめてコンパクトなサマリーを生成する。

結果として HOT layer のサイズが **100KB → 49KB に圧縮（-51%）** できた。同じセッション内で扱える情報量が倍近くになった計算だ。

---

## 実装して変わったこと

7本のWFを動かし始めて1ヶ月後。

毎朝 Slack を開くと WF-DAILY-BRIEF が昨日のブックマーク記事を要約してくれている。セッションを開始すると WF-DAILY-NOTE が生成したデイリーノートに「昨日やったこと」が書いてある。書いたメモには WF-TRIAGE が自動でタグを付けている。

「メモを管理する」という作業がほぼゼロになった。

```
以前: メモを書く → タグ付け (手動) → 整理 (手動) → たまに見返す
今:   メモを書く → あとは全部自動
```

一番効いているのは WF-REVIEW だ。7日前に書いたノートをランダムに持ってきて「これ覚えてる？」と問いかける設計。ノートが「書いたら終わり」ではなく「定期的に再浮上する」ようになった。

---

## 次にやること

次は WF-EMBED の検索精度を上げたい。現状は「ノート全体」を埋め込んでいるが、見出し単位でチャンクを切ることでより精度の高い検索ができるはずだ。

また、Zennへの投稿候補を書いたら自動で品質チェックして Slack に通知する WF-ZENN-TRIAGE も設計中。

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
