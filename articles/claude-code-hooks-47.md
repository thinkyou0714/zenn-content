---
title: "Claude Code hooksを47本実装した話：AIへの自動指示を設計するという仕事"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "automation", "bestpractices", "windows"]
published: true
---

## TL;DR

- Claude Code の hooks は「AIが何かをするたびに自動で走るスクリプト」。設定すれば手動指示がほぼ不要になる
- 47本実装してわかったこと：Windows環境では stdout の文字コードが罠。全フックに `sys.stdout.reconfigure(encoding="utf-8")` が必須
- `async: true` と `exit 2` は非互換。lint/guardフックに async を使うと Claude に届かなくなる

---

## なぜ47本も作ったのか

正直に言うと、最初は3本だった。

Claude Code を使い始めた頃、俺がフックとして設定していたのは「セッション開始時にコンテキストを読む」「停止時にログを書く」「危険なコマンドをブロックする」の3つだけだった。それで十分だと思っていた。

変わったのは「毎回同じことを指示している」と気づいた瞬間だ。

```
「テストを実行して確認してから完了報告してください」
「コードを変更したらdiffを見せてください」
「secretをチャットに貼らないでください」
```

毎セッション、これを繰り返していた。Claudeが優秀なほど、同じ指示を何度もすることへの違和感が強くなる。そこで気づいた。**これはフックで自動化できる**。

結果として47本になった。

---

## フックの種類と設計方針

Claude Code の hooks は5種類のイベントに対して設定できる。

| イベント | タイミング | 主な用途 |
|---|---|---|
| `PreToolUse` | ツール実行前 | セキュリティチェック・ブロック |
| `PostToolUse` | ツール実行後 | 検証・ログ記録 |
| `UserPromptSubmit` | ユーザーが送信した瞬間 | コンテキスト注入 |
| `SessionStart` | セッション開始時 | 初期設定・状態確認 |
| `Stop` | Claude が停止するとき | 品質ゲート・ハンドオフ保存 |

俺が実装した47本の内訳はこうだ。

```
PreToolUse:  12本（ブロック系 / セキュリティ系）
PostToolUse: 9本（検証系 / ログ系）
UserPromptSubmit: 11本（コンテキスト注入系）
SessionStart: 8本（初期化系）
Stop: 7本（品質ゲート / ハンドオフ系）
```

---

## Windows 環境で全フックが無音で失敗していた件

実装して最初にハマったのは、**日本語を含むフックが全部 exit 0 で終了するのに何も出力されない**という現象だった。

エラーが出ない。ログも残らない。フックは確かに実行されている。でも何も起きていない。

原因は stdout の文字コードだった。

Windows の Python は、デフォルトの stdout エンコーディングが `cp932`（Shift-JIS）になる。日本語の `print()` が `UnicodeEncodeError` を吐いて、それが `except` 節に吸い込まれ、無言で `exit 0` で終わっていたのだ。

```python
# NG: Windows では cp932 で UnicodeEncodeError が発生する
def main():
    print("セキュリティチェック開始")  # これが無音で失敗する

# OK: 全フックの先頭に必ず書く
import sys
sys.stdout.reconfigure(encoding="utf-8")
sys.stderr.reconfigure(encoding="utf-8")

def main():
    print("セキュリティチェック開始")  # これで正常に出力される
```

この修正を37本のフックに一括で入れた。実装したフックが全部「実は動いていなかった」と発覚するのはなかなか心臓に悪かった。

---

## `async: true` と `exit 2` は非互換だった

もう一つの落とし穴は、フックの `async` 設定と `exit 2` の非互換性だ。

`exit 2` は「フックが Claude の動作を止める」ためのシグナルだ。lint が失敗した、secrets が検出された、といった場合に Claude に「止まれ」を伝える手段として使う。

ところが `async: true` を付けると、フックの終了コードが Claude に届かなくなる。

```json
// NG: async: true にすると exit 2 が Claude に届かない
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "python lint_check.py",
        "async": true
      }]
    }]
  }
}

// OK: lint/guard 系フックには async を使わない
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "python lint_check.py"
      }]
    }]
  }
}
```

`async: true` が有効なのは「ログを別プロセスで書く」「Slack 通知を非同期で飛ばす」など、**Claude の動作を止める必要がない副作用系のフック**だけだ。

---

## 実装して変わったこと

47本のフックを入れた後のセッションは、体感が変わった。

以前は毎セッション「テスト確認して」「diff見せて」「secretは貼らないで」と言っていた。今は何も言わなくてもそれが全部自動で走る。

具体的に自動化できたもの：

```
budget_enforcer.py    → API コスト超過を自動警告
tdd_guard.py          → テスト通過前に「完了」と言わせない
secret_scanner.py     → secrets 検出時に即 exit 2
auto_tester.py        → コード変更後にテスト自動実行
context_budget.py     → セッション開始時に context 使用率を表示
handoff_rotator.py    → セッション終了時にハンドオフを自動保存
failure_capture.py    → 失敗パターンを19カテゴリで自動分類
n8n_skill_inject.py   → n8n 操作前に関連スキルを自動注入
```

**「AIに指示する」から「AIが自動で動く」への転換**。これがフック設計の本質だと思う。

---

## 次にやること

失敗パターンの分類は現在19カテゴリ。これをもとに `/evolve` スキルが学習候補を自動提案する仕組みを作っている。「同じ失敗を繰り返さない Claude」の設計を続けていく。

フックの全コードはTHINK YOU LABのコミュニティで共有予定。

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
