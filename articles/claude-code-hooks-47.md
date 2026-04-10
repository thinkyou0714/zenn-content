---
title: "Claude Code hooksを47本実装した話：AIへの自動指示を設計するという仕事"
emoji: "🪝"
type: "tech"
topics: ["claudecode", "automation", "bestpractices", "windows"]
published: false
---

## TL;DR

- Claude Code の hooks は「AIが何かをするたびに自動で走るスクリプト」。設定すれば手動指示がほぼ不要になる
- 47本実装してわかったこと：Windows環境では stdout の文字コードが罠。全フックに `sys.stdout.reconfigure(encoding="utf-8")` が必須
- `async: true` と `exit 2` は非互換。lint/guardフックに async を使うと Claude に届かなくなる
- Before: 毎セッション同じ指示を手動で10回以上入力 → After: 指示ゼロで全部自動実行

---

## なぜ47本も作ったのか

正直に言うと、最初は3本だった。

Claude Code を使い始めた頃、俺がフックとして設定していたのは「セッション開始時にコンテキストを読む」「停止時にログを書く」「危険なコマンドをブロックする」の3つだけだった。それで十分だと思っていた。

変わったのは「毎回同じことを指示している」と気づいた瞬間だ。

```
「テストを実行して確認してから完了報告してください」
「コードを変更したらdiffを見せてください」
「secretをチャットに貼らないでください」
「API費用が高くなりすぎたら教えてください」
「n8nの操作前にスキルを確認してください」
```

毎セッション、これを繰り返していた。Claudeが優秀なほど、同じ指示を何度もすることへの違和感が強くなる。そこで気づいた。**これはフックで自動化できる**。

結果として47本になった。

---

## フックとは何か：5種類のイベントと設計思想

Claude Code の hooks は5種類のイベントに対して設定できる。

| イベント | タイミング | 主な用途 |
|---|---|---|
| `PreToolUse` | ツール実行前 | セキュリティチェック・ブロック |
| `PostToolUse` | ツール実行後 | 検証・ログ記録 |
| `UserPromptSubmit` | ユーザーが送信した瞬間 | コンテキスト注入 |
| `SessionStart` | セッション開始時 | 初期設定・状態確認 |
| `Stop` | Claude が停止するとき | 品質ゲート・ハンドオフ保存 |

### フックの出力と Claude の動作の関係

フックが何を返すかで Claude の挙動が変わる。

| 終了コード | フックの出力 | Claude の動作 |
|---|---|---|
| `exit 0` | stdout のテキスト | Claude に情報として渡す（続行） |
| `exit 1` | stderr のテキスト | エラーとして Claude に通知（続行） |
| `exit 2` | stderr のテキスト | **Claude を強制停止**（ブロック） |

`exit 2` が「ブロック」できる点が重要だ。これを使えば「secrets が含まれていたら Claude を止める」「テストが通っていない状態で complete を言わせない」といった制御が可能になる。

### 俺が実装した47本の内訳

```
PreToolUse:       12本（ブロック系 / セキュリティ系）
PostToolUse:       9本（検証系 / ログ系）
UserPromptSubmit: 11本（コンテキスト注入系）
SessionStart:      8本（初期化系）
Stop:              7本（品質ゲート / ハンドオフ系）
```

---

## PreToolUse（12本）：「実行させない」設計

最も重要なのが PreToolUse だ。ここは「Claude が何かをやろうとした瞬間に割り込む」場所になる。

### secret_scanner.py：秘密情報の流出防止

一番重要なフックがこれだ。Claude が Bash や Edit を実行しようとした瞬間に、コマンドの内容に secrets が含まれていないかをスキャンする。

```python
import sys
import re
import json

sys.stdout.reconfigure(encoding="utf-8")
sys.stderr.reconfigure(encoding="utf-8")

# 検出対象パターン
PATTERNS = [
    r'sk-[a-zA-Z0-9]{48}',              # OpenAI API key
    r'sk-ant-[a-zA-Z0-9-]{90,}',         # Anthropic API key
    r'xoxb-[0-9]+-[0-9]+-[a-zA-Z0-9]+', # Slack Bot Token
    r'ghp_[a-zA-Z0-9]{36}',             # GitHub Personal Access Token
    r'AKIA[A-Z0-9]{16}',                # AWS Access Key
]

def check_secrets(text):
    for pattern in PATTERNS:
        if re.search(pattern, text):
            return True, pattern
    return False, None

input_data = json.loads(sys.stdin.read())
tool_input = str(input_data.get('tool_input', {}))

has_secret, pattern = check_secrets(tool_input)
if has_secret:
    print(f"[BLOCK] secrets detected. Pattern: {pattern}", file=sys.stderr)
    sys.exit(2)

sys.exit(0)
```

`exit 2` でブロックするため、Claude は secrets を含むコマンドを実行できない。

### guard.py：危険なコマンドのブロック

`git push --force`、`rm -rf`、`git reset --hard` などは実行前にブロックする。

```python
DENIED_PATTERNS = [
    r'git push --force(?!-with-lease)',
    r'rm\s+-rf\s+/',
    r'git\s+reset\s+--hard',
    r'git\s+clean\s+-f',
    r'git\s+add\s+-A',  # 全ファイル add は禁止（個別指定必須）
    r'DROP\s+TABLE',
    r'DELETE\s+FROM.*(?!WHERE)',  # WHERE なし DELETE
]
```

### budget_enforcer.py：API コスト超過を自動警告

Bash ツールを実行するたびにコスト状況をチェックする。

```python
# Claude の Usage API を使って今日のコスト計算
# 設定した日次上限（デフォルト 500円）を超えたら警告
DAILY_LIMIT_JPY = int(os.environ.get('CLAUDE_DAILY_BUDGET_JPY', '500'))

if estimated_cost_jpy > DAILY_LIMIT_JPY * 0.8:
    print(f"[WARNING] 本日のAPI使用量が上限の80%を超えました: ¥{estimated_cost_jpy:.0f} / ¥{DAILY_LIMIT_JPY}")
```

---

## UserPromptSubmit（11本）：「文脈を自動注入する」設計

ユーザーがメッセージを送信した瞬間に走るフックだ。ここでやることは「Claude に渡すコンテキストの自動整備」だ。

### context_budget.py：コンテキスト使用率の可視化

セッション開始時に context window の使用状況を表示する。

```python
# セッション開始時に context 使用率を自動表示
print(f"""
=== Context Budget ===
使用量: {usage_tokens:,} / {total_tokens:,} tokens ({usage_pct:.1f}%)
{"⚠️ 60% を超えました。/compact を推奨します。" if usage_pct > 60 else "OK"}
HOT layer: {hot_size_kb:.1f} KB（目標 50KB 以下）
""")
```

コンテキスト使用率が 60% を超えたら `/compact` をリマインドする。これにより「知らないうちに context が詰まって Claude の品質が落ちる」問題を防いでいる。

### n8n_skill_inject.py：n8n 操作前に関連スキルを自動注入

ユーザーが「n8n」「WF」などのキーワードを含むプロンプトを送った瞬間、関連スキルの内容を自動で注入する。

```python
N8N_KEYWORDS = ['n8n', 'WF-', 'ワークフロー', 'n8n_workflows']

prompt = input_data.get('prompt', '')
if any(kw in prompt for kw in N8N_KEYWORDS):
    # n8n-wf-deploy スキルの内容を読み込んで注入
    skill_content = read_skill('n8n-wf-deploy')
    print(f"[Context] n8n スキルを注入しました:\n{skill_content[:500]}")
```

以前は「n8n の操作前にスキルを確認してください」と毎回指示していた。今は何も言わなくてもスキルが自動で渡される。

---

## Stop（7本）：「完了させない」設計

Claude が「完了しました」と言おうとした瞬間に走るフックだ。品質が担保されていない状態で完了報告させないための最終ゲート。

### tdd_guard.py：テスト通過前に「完了」と言わせない

```python
# テストファイルが変更されたら自動でテストを実行
# テストが失敗したら exit 2 で停止

if has_changed_files and test_files_exist:
    result = subprocess.run(['npm', 'test', '--passWithNoTests'],
                           capture_output=True, text=True, timeout=60)
    if result.returncode != 0:
        print(f"[BLOCK] テストが失敗しています。修正してから完了報告してください。\n{result.stdout[-500:]}", 
              file=sys.stderr)
        sys.exit(2)
```

### handoff_rotator.py：セッション終了時にハンドオフを自動保存

Claude がセッションを終了しようとするたびに、そのセッションの状態を自動で保存する。

```python
# セッションの状態を自動でファイルに書き出す
handoff = {
    'timestamp': datetime.now().isoformat(),
    'touched_files': get_touched_files(),
    'last_task': get_last_task(),
    'next_step': extract_next_step(),
}

# .claude/sessions/YYYY-MM-DD-HHmm.md に自動保存
save_handoff(handoff)
print(f"[Handoff] セッション状態を保存しました: {handoff_path}")
```

### failure_capture.py：失敗パターンを19カテゴリで自動分類

セッション中に発生したエラーや失敗を自動で分類・記録する。

```
失敗カテゴリ（一部）:
1. encoding_error        - 文字コード関連
2. import_error          - モジュール import 失敗
3. api_rate_limit        - API レート制限超過
4. file_not_found        - ファイルパス誤り
5. git_conflict          - git マージ競合
6. test_failure          - テスト失敗
7. type_error            - 型エラー
8. permission_denied     - アクセス権限エラー
9. timeout               - タイムアウト
10. n8n_code_node_error  - n8n Code ノードエラー
...（残り9カテゴリ）
```

蓄積した失敗パターンをもとに、次のセッションでは「これは前回の失敗パターン5番に似ている。対処法は…」と自動でヒントを出せるようになった。

---

## Windows 環境で全フックが無音で失敗していた件

実装して最初にハマったのは、**日本語を含むフックが全部 exit 0 で終了するのに何も出力されない**という現象だった。

エラーが出ない。ログも残らない。フックは確かに実行されている。でも何も起きていない。

### 原因：Python の stdout エンコーディング

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

この修正を **37本のフックに一括で入れた**。実装したフックが全部「実は動いていなかった」と発覚するのはなかなか心臓に悪かった。

### subprocess も別途設定が必要

さらに罠があった。Python から別のプロセスを呼ぶ `subprocess` も、環境変数で文字コードを指定しないと同じ問題が起きる。

```python
import subprocess
import os

# NG: subprocess も cp932 になる
result = subprocess.run(['python', 'other_script.py'], capture_output=True)

# OK: env で明示的に UTF-8 を指定
env = os.environ.copy()
env['PYTHONIOENCODING'] = 'utf-8'
result = subprocess.run(['python', 'other_script.py'], 
                       capture_output=True, 
                       env=env,
                       encoding='utf-8')
```

Windows でフックを書くなら、最初の3行は呪文として必ず書くと決めてしまうのが一番だ。

---

## `async: true` と `exit 2` は非互換だった

もう一つの落とし穴は、フックの `async` 設定と `exit 2` の非互換性だ。

### 非互換の仕組み

`exit 2` は「フックが Claude の動作を止める」ためのシグナルだ。Claude は、フックの終了コードを同期的に受け取って動作を制御する。

ところが `async: true` を付けると、フックはバックグラウンドで実行される。バックグラウンドプロセスの終了コードは Claude に届かない。

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

### async を使うべき場面

`async: true` が有効なのは「ログを別プロセスで書く」「Slack 通知を非同期で飛ばす」など、**Claude の動作を止める必要がない副作用系のフック**だけだ。

| フック種別 | async | 理由 |
|---|---|---|
| lint/guard（exit 2 使用） | NG | 終了コードが Claude に届かない |
| ログ記録 | OK | 書き込みを待つ必要がない |
| Slack 通知 | OK | 通知を待つ必要がない |
| secret スキャン | NG | ブロックする必要がある |
| コンテキスト注入 | NG | Claude が情報を受け取る前に完了する必要がある |

---

## フック設計のアーキテクチャ：Layer 0 → Layer 2

47本のフックは、設計的に3つのレイヤーに分かれている。

```
Layer 0: 安全ガード（常に実行、ブロック系）
  PreToolUse:  secret_scanner.py, guard.py, budget_enforcer.py
  Stop:        tdd_guard.py

Layer 1: 状態管理（情報の収集と注入）
  UserPromptSubmit: context_budget.py, n8n_skill_inject.py, brand_voice_inject.py
  SessionStart:     foundation_context.py, handoff_reader.py
  PostToolUse:      auto_tester.py, diff_logger.py

Layer 2: 観察と学習（非同期・副作用系）
  PostToolUse:  failure_capture.py (async)
  Stop:         handoff_rotator.py, session_logger.py (async)
```

Layer 0 は常に同期実行。失敗したら即ブロック。
Layer 1 は Claude の入出力に情報を付け加える。
Layer 2 は記録と学習のための非同期フック。Claude の動作を止めない。

---

## 実装前後の変化：定量的に

フック設計前後で何が変わったか。数値で並べると効果がよくわかる。

### 手動指示の削減

| 指示の種類 | Before | After |
|---|---|---|
| テスト確認の指示 | 毎セッション手動 | auto_tester.py が自動実行 |
| diff 確認の指示 | 毎セッション手動 | PostToolUse で自動表示 |
| secret 非貼付の指示 | 毎セッション手動 | secret_scanner.py が自動ブロック |
| コスト確認の指示 | 毎セッション手動 | budget_enforcer.py が自動警告 |
| n8n スキル確認の指示 | n8n 操作前に毎回 | n8n_skill_inject.py が自動注入 |
| コンテキスト圧縮の指示 | 忘れると圧迫 | context_budget.py が自動リマインド |

### セッション品質の変化

```
Before:
  「テスト確認して」「diff見せて」「secretは貼らないで」と毎回言っていた
  → 忘れると品質が落ちる
  → 指示を忘れた時のコストが高い

After:
  何も言わなくても全部自動で走る
  → 指示ミスゼロ
  → セッション開始のオーバーヘッドがほぼゼロ
```

一番体感が変わったのは「セッション開始直後の密度」だ。以前は最初の数ターンをセットアップに使っていた。今はセッション開始と同時に本質的な作業に入れる。

---

## よくある失敗パターンと対策

47本を実装する中で、フック設計でよく失敗したパターンをまとめる。

### 失敗1: matcher の設定ミス

```json
// NG: Bash ツールしかマッチしない
{ "matcher": "Bash" }

// OK: 複数ツールに適用したい場合
{ "matcher": "Bash|Edit|Write" }

// OK: 全ツールに適用
{}  // matcher を省略すると全ツールにマッチ
```

### 失敗2: stdin の読み取り忘れ

Claude はフックに対して `stdin` でツール情報を JSON として渡す。読み取らないとデータにアクセスできない。

```python
import json
import sys

# 必須: stdin からツール情報を読む
input_data = json.loads(sys.stdin.read())
tool_name = input_data.get('tool_name')
tool_input = input_data.get('tool_input', {})
```

### 失敗3: フックが重くなりすぎる

フックは Claude の応答をブロックする。重い処理（外部 API 呼び出し、大きなファイルのスキャン）を同期フックに入れると、Claude の応答が遅くなる。

```
対策: 重い処理は async: true にするか、軽量な判定ロジックに絞る
目安: 同期フック1本あたり 200ms 以内に完了させる
```

---

## 次にやること

失敗パターンの分類は現在19カテゴリ。これをもとに `/evolve` スキルが学習候補を自動提案する仕組みを作っている。「同じ失敗を繰り返さない Claude」の設計を続けていく。

フックの全コードはTHINK YOU LABのコミュニティで共有予定。

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
