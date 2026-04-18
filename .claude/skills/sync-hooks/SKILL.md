---
name: sync-hooks
description: Sync hook definitions between Claude Code (.claude/settings.json) and GitHub Copilot (.github/copilot/settings.json) with format conversion
argument-hint: "[--from claude | --from copilot]"
disable-model-invocation: true
---

# sync-hooks

`.claude/settings.json` と `.github/copilot/settings.json` の `hooks` セクションを相互変換・同期するスキル。

両ファイルは同じ `settings.json` という形式だが、hooks のイベント名・フィールド名が異なるため変換が必要。

## Usage

```sh
/sync-hooks [--from claude | --from copilot]
```

- `--from claude`: `.claude/settings.json` の hooks を変換して `.github/copilot/settings.json` へ書き出す
- `--from copilot`: `.github/copilot/settings.json` の hooks を変換して `.claude/settings.json` へ書き出す
- 引数なし: 両ファイルを比較して差分を報告し、変換方向を確認する

## フォーマット仕様

### Claude Code（`.claude/settings.json`）

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/lint.sh",
            "timeout": 30,
            "statusMessage": "Linting..."
          }
        ]
      }
    ]
  }
}
```

- イベント名: PascalCase
- `matcher`: ツール名を正規表現でフィルタ（省略時は全ツール）
- `hooks[]`: ハンドラの配列（`matcher` グループにネスト）
- `command`: 実行コマンド文字列
- `timeout`: 秒単位
- `type`: `command` / `http` / `prompt` / `agent`

### GitHub Copilot（`.github/copilot/settings.json`）

```json
{
  "hooks": {
    "postToolUse": [
      {
        "type": "command",
        "bash": ".github/hooks/lint.sh",
        "powershell": ".github/hooks/lint.ps1",
        "cwd": ".",
        "timeoutSec": 30,
        "env": {}
      }
    ]
  }
}
```

- イベント名: camelCase
- ツールフィルタ機能なし（`matcher` 相当なし）
- `bash` / `powershell`: OS 別コマンド（別フィールド）
- `timeoutSec`: 秒単位
- `cwd`: 作業ディレクトリ（リポジトリルート相対）
- `env`: 追加環境変数
- `type: "command"` のみ対応

## イベント名マッピング

| Claude Code | Copilot | 備考 |
|---|---|---|
| `SessionStart` | `sessionStart` | |
| `SessionEnd` | `sessionEnd` | |
| `UserPromptSubmit` | `userPromptSubmitted` | 末尾も異なる |
| `PreToolUse` | `preToolUse` | |
| `PostToolUse` | `postToolUse` | |
| `Stop` | `agentStop` | 名前も異なる |
| `SubagentStop` | `subagentStop` | |
| `StopFailure` | `errorOccurred` | 部分対応 |
| `PostToolUseFailure` | `errorOccurred` | 部分対応・統合 |
| その他多数 | 対応なし | 変換時にスキップ・警告 |

## フィールドマッピング

### Claude Code → Copilot

| Claude Code | Copilot | 備考 |
|---|---|---|
| `command: "..."` | `bash: "..."` | そのまま移植 |
| `timeout: N` | `timeoutSec: N` | 値はそのまま |
| `matcher: "..."` | （なし） | 削除・警告を出す |
| `async`, `statusMessage`, `if`, `shell` | （なし） | 削除（サイレント） |
| `type: "http"/"prompt"/"agent"` | （なし） | ハンドラごとスキップ・警告 |
| （なし） | `powershell` | 付与しない（ユーザーが手動追加） |

### Copilot → Claude Code

| Copilot | Claude Code | 備考 |
|---|---|---|
| `bash: "..."` | `command: "..."` | そのまま移植 |
| `timeoutSec: N` | `timeout: N` | 値はそのまま |
| `cwd`, `env`, `powershell` | （なし） | 削除・警告を出す |
| （なし） | `matcher: ""` | 空文字（全ツールマッチ）を付与 |
| `type: "command"` | `type: "command"` | そのまま |

## 手順

### 1. 引数を解析する

- `--from claude` / `--from copilot` で方向を決定
- 引数がない場合:
  - 両ファイルの `hooks` キーを読み込んで差分を報告
  - どちらをソースにするかユーザーに確認する

### 2. ソースを読み込む

- ソース側 `settings.json` を読み込み `hooks` キーを取得する
- `hooks` キーが存在しない場合は処理を終了する

### 3. 変換して書き出す

**`--from claude` の場合:**
1. 各イベント名をイベント名マッピングで変換（対応なしはスキップ・警告）
2. 各 `matcher` グループのハンドラ配列をフラット化（`matcher` は削除・警告）
3. `type: "command"` 以外のハンドラをスキップ・警告
4. フィールドマッピングに従って各フィールドを変換
5. `.github/copilot/settings.json` の `hooks` キーを上書き（他のキーは維持）

**`--from copilot` の場合:**
1. 各イベント名をイベント名マッピングで変換（対応なしはスキップ・警告）
2. 各ハンドラにフィールドマッピングを適用
3. `matcher: ""` でラップ
4. `.claude/settings.json` の `hooks` キーを上書き（他のキーは維持）

### 4. 結果を報告する

- 変換したイベント名・件数を表示する
- 情報が失われたフィールド（`matcher`, `cwd`, `env`, `powershell` 等）を警告として表示する
- スキップしたハンドラ（非対応 `type` 等）を警告として表示する

## 注意事項

- `hooks` 以外のキー（`env`, `permissions`, `enabledPlugins` 等）は変更しない
- `matcher` は Copilot に対応機能がないため Claude Code → Copilot 変換時に失われる
- `cwd`, `env`, `powershell` は Claude Code に対応機能がないため Copilot → Claude Code 変換時に失われる
