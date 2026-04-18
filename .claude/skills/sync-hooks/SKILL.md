---
name: sync-hooks
description: Sync hook definitions across multiple AI agents (Claude Code, GitHub Copilot) with format conversion
argument-hint: "[--from claude | --from copilot]"
---

# sync-hooks

複数のAIエージェントが持つhook定義を相互変換・同期するスキル。

canonical 形式を中間表現として使い、各エージェントのフォーマットと相互変換する。新しいエージェントを追加する場合は parse と serialize のマッピングを追加するだけでよい。

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

| 項目 | 仕様 | canonical 対応 |
|---|---|---|
| イベント名 | PascalCase | ✅ `event` |
| `matcher` | ツール名の正規表現フィルタ（省略時は全ツール） | ✅ `matcher` |
| `command` | 実行コマンド文字列 | ✅ `command` |
| `timeout` | 秒単位 | ✅ `timeout` |
| `type` | `command` / `http` / `prompt` / `agent` | ✅ `type`（`command` のみ） |
| `async`, `statusMessage`, `if`, `shell` | 各種オプション | ❌ Claude Code 固有 |

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

| 項目 | 仕様 | canonical 対応 |
|---|---|---|
| イベント名 | camelCase | ✅ `event` |
| `bash` | Unix 系コマンド | ✅ `command` |
| `timeoutSec` | 秒単位 | ✅ `timeout` |
| `type` | `command` のみ対応 | ✅ `type` |
| `matcher` 相当 | なし | ❌ なし |
| `powershell` | Windows 用コマンド | ❌ Copilot 固有 |
| `cwd` | 作業ディレクトリ | ❌ Copilot 固有 |
| `env` | 追加環境変数 | ❌ Copilot 固有 |

## 変換マッピング（canonical ↔ 各フォーマット）

### canonical 形式

エージェント間で共有可能なフィールドの中間表現。

```yaml
event: "post-tool-use"          # イベント名（kebab-case）
matcher: "Write|Edit"           # ツールフィルタ（null = 全ツール）
command: ".claude/hooks/lint.sh" # 実行コマンド
timeout: 30                     # タイムアウト（秒）
type: "command"                 # ハンドラ種別
```

canonical イベント名は kebab-case を使用する。

### parse（各フォーマット → canonical）

各エージェントの全フィールドを parse し、canonical フィールドを抽出する。canonical に対応しないフィールドは破棄する（警告 or サイレント）。

#### イベント名

| canonical | Claude Code | Copilot | 備考 |
|---|---|---|---|
| `session-start` | `SessionStart` | `sessionStart` | |
| `session-end` | `SessionEnd` | `sessionEnd` | |
| `user-prompt-submit` | `UserPromptSubmit` | `userPromptSubmitted` | 末尾も異なる |
| `pre-tool-use` | `PreToolUse` | `preToolUse` | |
| `post-tool-use` | `PostToolUse` | `postToolUse` | |
| `stop` | `Stop` | `agentStop` | 名前も異なる |
| `subagent-stop` | `SubagentStop` | `subagentStop` | |
| `stop-failure` | `StopFailure` | `errorOccurred` | 部分対応・統合 |
| `post-tool-use-failure` | `PostToolUseFailure` | `errorOccurred` | 部分対応・統合 |
| （なし） | その他多数 | — | スキップ・警告 |

#### フィールド

| canonical | Claude Code | Copilot | 備考 |
|---|---|---|---|
| `matcher` | `matcher` | （なし） | Copilot parse 時は `null` |
| `command` | `command` | `bash` | |
| `timeout` | `timeout` | `timeoutSec` | |
| `type` | `type` | `type` | `command` のみ canonical 対象 |
| （破棄・サイレント） | `async`, `statusMessage`, `if`, `shell` | — | |
| （破棄・警告） | — | `cwd`, `env`, `powershell` | |
| （スキップ・警告） | `type: "http"/"prompt"/"agent"` | — | ハンドラごとスキップ |

### serialize（canonical → 各フォーマット）

canonical から各エージェントのフォーマットへ変換する。

#### イベント名

parse テーブルの逆変換を使用する（canonical → 各エージェントの列）。

#### フィールド

| canonical | Claude Code | Copilot | 備考 |
|---|---|---|---|
| `matcher` | `matcher: "..."` でラップ | （なし・警告） | |
| `matcher: null` | `matcher: ""` （全ツールマッチ）でラップ | （省略） | |
| `command` | `command: "..."` | `bash: "..."` | |
| `timeout` | `timeout: N` | `timeoutSec: N` | |
| `type: "command"` | `type: "command"` | `type: "command"` | |
| — | — | `powershell` | 付与しない（ユーザーが手動追加） |

serialize 時にエージェント固有フィールドは付与しない（`cwd`, `env` 等）。

## 手順

### 1. 引数を解析する

- `--from claude` / `--from copilot` で変換方向を決定する
- 引数がない場合:
  - 両ファイルの `hooks` キーを読み込んで差分を報告する
  - どちらをソースにするかユーザーに確認する

### 2. ソースを読み込む

- ソース側 `settings.json` を読み込み `hooks` キーを取得する
- `hooks` キーが存在しない場合は処理を終了する

### 3. 変換して書き出す

1. parse マッピングに従って canonical を抽出する
   - イベント名を canonical に変換（対応なしはスキップ・警告）
   - 各ハンドラのフィールドを canonical に変換
   - `type: "command"` 以外のハンドラはスキップ・警告
2. serialize マッピングに従って出力フォーマットに変換する
   - canonical イベント名を出力先フォーマットに変換
   - Claude Code 向け: `matcher` グループにネストして書き出す
   - Copilot 向け: フラットなハンドラ配列として書き出す
3. 出力先 `settings.json` の `hooks` キーを上書きする（他のキーは維持）

### 4. 結果を報告する

- 変換したイベント名・件数を表示する
- 情報が失われたフィールド（`matcher`, `cwd`, `env`, `powershell` 等）を警告として表示する
- スキップしたハンドラ（非対応 `type`、対応なしイベント等）を警告として表示する

## 注意事項

- `hooks` 以外のキー（`env`, `permissions`, `enabledPlugins` 等）は変更しない
- `matcher` は Copilot に対応機能がないため Claude Code → Copilot 変換時に失われる
- `cwd`, `env`, `powershell` は Claude Code に対応機能がないため Copilot → Claude Code 変換時に失われる
