---
name: sync-agents
description: Sync sub-agent definitions across multiple AI agents (Claude Code, GitHub Copilot) with format conversion
argument-hint: "[--from claude | --from copilot] [agent-name]"
---

# sync-agents

複数のAIエージェントが持つサブエージェント定義を相互変換・同期するスキル。

canonical 形式を中間表現として使い、各エージェントのフォーマットと相互変換する。新しいエージェントを追加する場合は parse と serialize のマッピングを追加するだけでよい。

## Usage

```sh
/sync-agents [--from claude | --from copilot] [agent-name]
```

- `--from claude`: `.claude/agents/` のエージェントを変換して `.github/agents/` へ書き出す
- `--from copilot`: `.github/agents/` のエージェントを変換して `.claude/agents/` へ書き出す
- `[agent-name]`: 対象エージェント名（例: `code-reviewer`）。省略時は全ファイルを処理
- 引数なし: 両ディレクトリを比較して差分を報告し、変換方向を確認する

## フォーマット仕様

### Claude Code（`.claude/agents/<name>.md`）

```markdown
---
name: code-reviewer
description: Reviews code for quality and best practices. Use proactively after code changes.
tools: Read, Grep, Glob
model: sonnet
argument-hint: "[file-or-dir]"
---

You are a senior code reviewer...
```

| フィールド | 必須 | 型 | canonical 対応 |
|---|---|---|---|
| `name` | ✅ | 文字列 | ✅ `name` |
| `description` | ✅ | 文字列 | ✅ `description` |
| Body | — | Markdown | ✅ `prompt` |
| `tools` | — | カンマ区切り文字列 | ✅ `tools` |
| `model` | — | 文字列 | ✅ `model` |
| `argument-hint` | — | 文字列 | ✅ `argument-hint` |
| `mcpServers` | — | オブジェクト | ✅ `mcp-servers` |
| `disallowedTools` | — | カンマ区切り文字列 | ❌ Claude Code 固有 |
| `permissionMode` | — | 文字列 | ❌ Claude Code 固有 |
| `maxTurns` | — | 数値 | ❌ Claude Code 固有 |
| `memory` | — | 文字列 | ❌ Claude Code 固有 |
| `skills` | — | リスト | ❌ Claude Code 固有 |
| `hooks` | — | オブジェクト | ❌ Claude Code 固有 |
| `color` | — | 文字列 | ❌ Claude Code 固有 |
| `background` | — | 真偽値 | ❌ Claude Code 固有 |
| `effort` | — | 文字列 | ❌ Claude Code 固有 |
| `isolation` | — | 文字列 | ❌ Claude Code 固有 |
| `initialPrompt` | — | 文字列 | ❌ Claude Code 固有 |

### GitHub Copilot（`.github/agents/<name>.agent.md`）

```markdown
---
description: Reviews code for quality and best practices.
argument-hint: "[file-or-dir]"
tools:
  - read
  - search
model: gpt-4o
---

You are a senior code reviewer...
```

**命名規則**: ファイル名の `<name>` 部分がエージェント名となる（`name` はオプションの表示名）。

| フィールド | 必須 | 型 | canonical 対応 |
|---|---|---|---|
| （ファイル名） | ✅ | — | ✅ `name` |
| `description` | ✅ | 文字列 | ✅ `description` |
| Body | — | Markdown | ✅ `prompt` |
| `tools` | — | YAML 配列 | ✅ `tools` |
| `model` | — | 文字列 | ✅ `model` |
| `argument-hint` | — | 文字列 | ✅ `argument-hint` |
| `mcp-servers` | — | オブジェクト | ✅ `mcp-servers` |
| `name` | — | 文字列 | ❌ Copilot 固有（表示名） |
| `target` | — | 文字列 | ❌ Copilot 固有 |
| `disable-model-invocation` | — | 真偽値 | ❌ Copilot 固有 |
| `user-invocable` | — | 真偽値 | ❌ Copilot 固有 |
| `metadata` | — | オブジェクト | ❌ Copilot 固有 |
| `infer` | — | 真偽値 | ❌ 廃止済み（使用しない） |

## 変換マッピング（canonical ↔ 各フォーマット）

### canonical 形式

エージェント間で共有可能なフィールドの中間表現。

```yaml
name: code-reviewer        # エージェント識別子（ファイル名ベース）
description: "..."         # エージェントの説明
prompt: |                  # システムプロンプト（Markdown 本体）
  ...
tools: [...]               # ツール名のリスト（canonical ツール名）
model: "..."               # 使用モデル
argument-hint: "..."       # ユーザーへの入力ガイダンス
mcp-servers: {}            # MCP サーバー設定
```

### ツール名マッピング

Copilot のツール名は Claude Code のツール名をエイリアスとして受け付けるが、canonical では Copilot の正規名（小文字）を使用する。

#### parse（各フォーマット → canonical ツール名）

| canonical | Claude Code | Copilot |
|---|---|---|
| `read` | `Read` | `read` |
| `edit` | `Edit`, `Write` | `edit` |
| `search` | `Grep`, `Glob` | `search` |
| `execute` | `Bash` | `execute` |
| `agent` | `Agent` | `agent` |
| `web` | `WebSearch`, `WebFetch` | `web` |
| `todo` | `TodoWrite` | `todo` |
| （そのまま）| その他 | その他（MCP ツール等） |

#### serialize（canonical ツール名 → 各フォーマット）

| canonical | Claude Code | Copilot |
|---|---|---|
| `read` | `Read` | `read` |
| `edit` | `Edit` | `edit` |
| `search` | `Grep` | `search` |
| `execute` | `Bash` | `execute` |
| `agent` | `Agent` | `agent` |
| `web` | `WebSearch` | `web` |
| `todo` | `TodoWrite` | `todo` |
| その他 | そのまま・警告 | そのまま・警告 |

### parse（各フォーマット → canonical）

各エージェントの全フィールドを parse し、canonical フィールドを抽出する。canonical に対応しないフィールドは破棄する（警告 or サイレント）。

| canonical | Claude Code | Copilot |
|---|---|---|
| `name` | `name` フィールド | ファイル名（`.agent.md` / `.md` を除く） |
| `description` | `description` フィールド | `description` フィールド |
| `prompt` | Markdown 本体 | Markdown 本体 |
| `tools` | `tools` をカンマで分割 → ツール名マッピングで変換 | `tools` 配列 → ツール名マッピングで変換 |
| `model` | `model` フィールド | `model` フィールド |
| `argument-hint` | `argument-hint` フィールド | `argument-hint` フィールド |
| `mcp-servers` | `mcpServers` フィールド | `mcp-servers` フィールド |

### serialize（canonical → 各フォーマット）

canonical から各エージェントのフォーマットへ変換する。

| canonical | Claude Code | Copilot |
|---|---|---|
| `name` | `name: <name>` フィールド | ファイル名 `<name>.agent.md` |
| `description` | `description: "..."` フィールド | `description: "..."` フィールド |
| `prompt` | Markdown 本体 | Markdown 本体 |
| `tools` | ツール名マッピング → カンマ区切り文字列 | ツール名マッピング → YAML 配列 |
| `model` | `model: "..."` フィールド | `model: "..."` フィールド |
| `argument-hint` | `argument-hint: "..."` フィールド | `argument-hint: "..."` フィールド |
| `mcp-servers` | `mcpServers: ...` フィールド | `mcp-servers: ...` フィールド |

serialize 時にエージェント固有フィールドは付与しない（`infer`, `target`, `disable-model-invocation` 等）。

## 手順

### 1. 引数を解析する

- `--from claude` / `--from copilot` で変換方向を決定する
- `[agent-name]` が指定された場合は対象ファイルを絞る
- 引数がない場合:
  - `.claude/agents/` と `.github/agents/` を双方列挙して差分を報告する
    - Claude Code のみに存在するファイル
    - Copilot のみに存在するファイル
    - 両方に存在するファイル
  - 変換方向をユーザーに確認する

### 2. ソースファイルを列挙する

- `--from claude`: `.claude/agents/` の `*.md` ファイル（`.gitkeep` 除く）
- `--from copilot`: `.github/agents/` の `*.agent.md` ファイル
- `[agent-name]` 指定時はそのファイルのみ

### 3. 各ファイルを変換して書き出す

1. frontmatter とボディを分離する
2. parse マッピングに従って canonical を抽出する（ツール名も変換する）
3. serialize マッピングに従って出力フォーマットに変換する（ツール名も変換する）
4. 出力先ファイルを書き出す

### 4. 結果を報告する

- 書き出したファイルの一覧を表示する
- 情報が失われたフィールド（エージェント固有フィールド）を警告として表示する
- ツール名変換で対応できなかったものを警告として表示する

## 注意事項

- 既存ファイルは上書きされる
- `model` の値はエージェント間で有効な値が異なる場合がある（例: Claude Code の `sonnet` ↔ Copilot の `gpt-4o`）。変換後に手動確認を推奨
- `mcp-servers` の構造はエージェント間で異なる場合がある。変換後に手動確認を推奨
- hooks の変換は `/sync-hooks` スキルが担当する
- Copilot の `infer` は廃止済み。出力に含めない
