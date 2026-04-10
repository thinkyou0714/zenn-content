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

:::message
この記事は **Windows 11 + Claude Code** 環境での実装ログです。Mac/Linux では文字コード問題は発生しません（その点だけ読み替えてください）。
:::

---

## この記事でわかること

- Claude Code の5種類の hooks イベントと使い分け方
- `exit 0 / 1 / 2` の違いと Claude の動作制御
- Windows 環境固有の文字コード問題と完全な解決策
- `async: true` と `exit 2` が非互換な理由
- 47本の内訳と「Layer 0〜2」のアーキテクチャ設計

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

:::details 47本の内訳（カテゴリ別）
```
PreToolUse:       12本（ブロック系 / セキュリティ系）
PostToolUse:       9本（検証系 / ログ系）
UserPromptSubmit: 11本（コンテキスト注入系）
SessionStart:      8本（初期化系）
Stop:              7本（品質ゲート / ハンドオフ系）
合計:             47本
```
:::

---

## PreToolUse（12本）：「実行させない」設計

最も重要なのが PreToolUse だ。ここは「Claude が何かをやろうとした瞬間に割り込む」場所になる。

### secret_scanner.py：秘密情報の流出防止

一番重要なフックがこれだ。Claude が Bash や Edit を実行しようとした瞬間に、コマンドの内容に secrets が含まれていないかをスキャンする。

```python:secret_scanner.py
import sys
import re
import json

sys.stdout.reconfigure(encoding="utf-8")
sys.stderr.reconfigure(encoding="utf-8")

PATTERNS = [
    r'sk-[a-zA-Z0-9]{48}',               # OpenAI API key
    r'sk-ant-[a-zA-Z0-9-]{90,}',          # Anthropic API key
    r'xoxb-[0-9]+-[0-9]+-[a-zA-Z0-9]+',  # Slack Bot Token
    r'ghp_[a-zA-Z0-9]{36}',              # GitHub Personal Access Token
    r'AKIA[A-Z0-9]{16}',                 # AWS Access Key
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

```python:guard.py
DENIED_PATTERNS = [
    r'git push --force(?!-with-lease)',
    r'rm\s+-rf\s+/',
    r'git\s+reset\s+--hard',
    r'git\s+clean\s+-f',
    r'git\s+add\s+-A',   # 全ファイル add は禁止（個別指定必須）
    r'DROP\s+TABLE',
    r'DELETE\s+FROM.*(?!WHERE)',  # WHERE なし DELETE
]
```

### budget_enforcer.py：API コスト超過を自動警告

```python:budget_enforcer.py
DAILY_LIMIT_JPY = int(os.environ.get('CLAUDE_DAILY_BUDGET_JPY', '500'))

if estimated_cost_jpy > DAILY_LIMIT_JPY * 0.8:
    print(f"[WARNING] 本日のAPI使用量が上限の80%を超えました: ¥{estimated_cost_jpy:.0f} / ¥{DAILY_LIMIT_JPY}")
```

---

## UserPromptSubmit（11本）：「文脈を自動注入する」設計

ユーザーがメッセージを送信した瞬間に走るフックだ。ここでやることは「Claude に渡すコンテキストの自動整備」だ。

### context_budget.py：コンテキスト使用率の可視化

```python:context_budget.py
print(f"""
=== Context Budget ===
使用量: {usage_tokens:,} / {total_tokens:,} tokens ({usage_pct:.1f}%)
{"[WARNING] 60% を超えました。/compact を推奨します。" if usage_pct > 60 else "OK"}
HOT layer: {hot_size_kb:.1f} KB（目標 50KB 以下）
""")
```

### n8n_skill_inject.py：n8n 操作前に関連スキルを自動注入

```python:n8n_skill_inject.py
N8N_KEYWORDS = ['n8n', 'WF-', 'ワークフロー', 'n8n_workflows']

prompt = input_data.get('prompt', '')
if any(kw in prompt for kw in N8N_KEYWORDS):
    skill_content = read_skill('n8n-wf-deploy')
    print(f"[Context] n8n スキルを注入しました:\n{skill_content[:500]}")
```

以前は「n8n の操作前にスキルを確認してください」と毎回指示していた。今は何も言わなくてもスキルが自動で渡される。

---

## Stop（7本）：「完了させない」設計

Claude が「完了しました」と言おうとした瞬間に走るフックだ。品質が担保されていない状態で完了報告させないための最終ゲート。

### tdd_guard.py：テスト通過前に「完了」と言わせない

```python:tdd_guard.py
if has_changed_files and test_files_exist:
    result = subprocess.run(['npm', 'test', '--passWithNoTests'],
                           capture_output=True, text=True, timeout=60)
    if result.returncode != 0:
        print(f"[BLOCK] テストが失敗しています。修正してから完了報告してください。\n{result.stdout[-500:]}", 
              file=sys.stderr)
        sys.exit(2)
```

### handoff_rotator.py：セッション終了時にハンドオフを自動保存

```python:handoff_rotator.py
handoff = {
    'timestamp': datetime.now().isoformat(),
    'touched_files': get_touched_files(),
    'last_task': get_last_task(),
    'next_step': extract_next_step(),
}
save_handoff(handoff)  # .claude/sessions/YYYY-MM-DD-HHmm.md に自動保存
print(f"[Handoff] セッション状態を保存しました: {handoff_path}")
```

### failure_capture.py：失敗パターンを19カテゴリで自動分類

:::details 19カテゴリの一覧
```
1.  encoding_error        - 文字コード関連
2.  import_error          - モジュール import 失敗
3.  api_rate_limit        - API レート制限超過
4.  file_not_found        - ファイルパス誤り
5.  git_conflict          - git マージ競合
6.  test_failure          - テスト失敗
7.  type_error            - 型エラー
8.  permission_denied     - アクセス権限エラー
9.  timeout               - タイムアウト
10. n8n_code_node_error   - n8n Code ノードエラー
11. context_overflow      - コンテキストウィンドウ超過
12. secret_leak_attempt   - secrets 漏洩の試み（ブロック）
13. hallucination         - 存在しないファイル/APIの参照
14. scope_creep           - スコープ外の変更
15. regression            - 既存機能の破壊
16. lint_failure          - lint/typecheck エラー
17. deploy_failure        - デプロイエラー
18. migration_error       - DB マイグレーション失敗
19. dependency_conflict   - パッケージ依存関係の競合
```
:::

---

## Windows 環境で全フックが無音で失敗していた件

:::message alert
**Windows ユーザー必読。** この問題を知らないと、フックを実装しても「全部正常に動いているように見えて、実は何もしていない」状態になる。俺は37本のフックが無音で失敗し続けていた。
:::

実装して最初にハマったのは、**日本語を含むフックが全部 exit 0 で終了するのに何も出力されない**という現象だった。

エラーが出ない。ログも残らない。フックは確かに実行されている。でも何も起きていない。

### 原因：Python の stdout エンコーディング

Windows の Python は、デフォルトの stdout エンコーディングが `cp932`（Shift-JIS）になる。日本語の `print()` が `UnicodeEncodeError` を吐いて、それが `except` 節に吸い込まれ、無言で `exit 0` で終わっていたのだ。

```python
# NG: Windows では cp932 で UnicodeEncodeError が発生する
def main():
    print("セキュリティチェック開始")  # 無音で失敗する

# OK: 全フックの先頭に必ず書く（呪文として固定）
import sys
sys.stdout.reconfigure(encoding="utf-8")
sys.stderr.reconfigure(encoding="utf-8")

def main():
    print("セキュリティチェック開始")  # 正常に出力される
```

この修正を **37本のフックに一括で入れた**。

### subprocess も別途設定が必要

```python
import subprocess, os

# NG: subprocess も cp932 になる
result = subprocess.run(['python', 'other_script.py'], capture_output=True)

# OK: env で明示的に UTF-8 を指定
env = os.environ.copy()
env['PYTHONIOENCODING'] = 'utf-8'
result = subprocess.run(
    ['python', 'other_script.py'],
    capture_output=True,
    env=env,
    encoding='utf-8'
)
```

:::message
Windows でフックを書くなら、**最初の2行（`sys.stdout/stderr.reconfigure`）を全ファイルのテンプレートに入れておく**のがベスト。書き忘れた瞬間に無音で失敗する。
:::

---

## `async: true` と `exit 2` は非互換だった

:::message alert
**lint/guardフックに `async: true` を付けてはいけない。** Claude の動作をブロックできなくなる。これを知らずに `async: true` を付けると、guards が全部「ブロックしているつもりで何もしていない」状態になる。
:::

### 非互換の仕組み

`exit 2` は Claude の動作を同期的にブロックするシグナルだ。`async: true` を付けるとフックはバックグラウンドで実行され、終了コードが Claude に届かない。

```json:settings.json（NG例）
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
```

```json:settings.json（OK例）
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

| フック種別 | async | 理由 |
|---|---|---|
| lint/guard（exit 2 使用） | **NG** | 終了コードが Claude に届かない |
| ログ記録 | OK | 書き込みを待つ必要がない |
| Slack 通知 | OK | 通知を待つ必要がない |
| secret スキャン | **NG** | ブロックする必要がある |
| コンテキスト注入 | **NG** | Claude が情報を受け取る前に完了する必要がある |

---

## フック設計のアーキテクチャ：Layer 0 → Layer 2

47本のフックは、設計的に3つのレイヤーに分かれている。

```
Layer 0: 安全ガード（常に同期実行、ブロック系）
  PreToolUse:  secret_scanner.py, guard.py, budget_enforcer.py
  Stop:        tdd_guard.py

Layer 1: 状態管理（情報の収集と注入）
  UserPromptSubmit: context_budget.py, n8n_skill_inject.py, brand_voice_inject.py
  SessionStart:     foundation_context.py, handoff_reader.py
  PostToolUse:      auto_tester.py, diff_logger.py

Layer 2: 観察と学習（非同期・副作用系）
  PostToolUse:  failure_capture.py (async: true)
  Stop:         handoff_rotator.py, session_logger.py (async: true)
```

Layer 0 は失敗したら即ブロック。Layer 1 は情報の入出力を制御。Layer 2 は記録と学習のための非同期フック。

---

## 実装前後の変化：定量的に

| 指示の種類 | Before | After |
|---|---|---|
| テスト確認 | 毎セッション手動 | `auto_tester.py` が自動実行 |
| diff 確認 | 毎セッション手動 | `PostToolUse` で自動表示 |
| secret 非貼付 | 毎セッション手動 | `secret_scanner.py` が自動ブロック |
| コスト確認 | 毎セッション手動 | `budget_enforcer.py` が自動警告 |
| n8n スキル確認 | n8n 操作前に毎回 | `n8n_skill_inject.py` が自動注入 |
| context 圧縮 | 忘れると圧迫 | `context_budget.py` が自動リマインド |

一番体感が変わったのは「セッション開始直後の密度」だ。以前は最初の数ターンをセットアップに使っていた。今はセッション開始と同時に本質的な作業に入れる。

---

## よくある失敗パターンと対策

### 失敗1: matcher の設定ミス

```json:settings.json
// NG: Bash ツールしかマッチしない
{ "matcher": "Bash" }

// OK: 複数ツールに適用
{ "matcher": "Bash|Edit|Write" }

// OK: 全ツールに適用（matcher を省略）
{}
```

### 失敗2: stdin の読み取り忘れ

```python
import json, sys

# 必須: stdin からツール情報を読む
input_data = json.loads(sys.stdin.read())
tool_name = input_data.get('tool_name')
tool_input = input_data.get('tool_input', {})
```

### 失敗3: フックが重くなりすぎる

:::message
同期フック1本の目安は **200ms 以内**。重い処理（外部 API 呼び出し・大きなファイルのスキャン）は `async: true` にするか、軽量な判定ロジックに絞ること。Claude の応答体感に直接影響する。
:::

---

## 次にやること

失敗パターンの分類は現在19カテゴリ。これをもとに `/evolve` スキルが学習候補を自動提案する仕組みを作っている。「同じ失敗を繰り返さない Claude」の設計を続けていく。

フックの全コードはTHINK YOU LABのコミュニティで共有予定。

---

## 関連記事

このシステム全体の設計については以下を参照してほしい。

- [n8n × Claude Code で55本のWFを動かしている：副業自動化システムの全体像](/articles/n8n-claudecode-automation-overview)
- [ObsidianをAIの第二の脳にした：7本のn8nワークフローで作ったナレッジ自動管理システム](/articles/obsidian-n8n-ai-pipeline)

---

## THINK YOU LABについて

AIと自動化で副業システムを構築するコミュニティ「THINK YOU LAB」を運営しています。
Claude Code / n8n / Obsidianを活用した実践的なワークフローをブログで公開中。

→ **[think-you-lab.vercel.app](https://think-you-lab.vercel.app)**
