---
name: sync-rules
description: Add missing agent-specific frontmatter fields to .claude/rules/*.md files so each agent can read its own field from the same file
argument-hint: "[file-name]"
---

# sync-rules

`.claude/rules/*.md` ファイルに、不足しているエージェント向けの frontmatter フィールドを追記するスキル。

## 背景

このプロジェクトでは `.github/instructions/*.instructions.md` 等が `.claude/rules/*.md` へのシンボリックリンクになっており、1つのファイルに複数エージェントの frontmatter フィールドを併記する設計を取っている。各エージェントは自分に対応するフィールドだけを読む。

```yaml
---
paths:          # Claude Code が読む
  - "src/**/*.ts"
applyTo: "src/**/*.ts"  # GitHub Copilot が読む
---

ルール本文（共通）
```

このスキルは既存のフィールドから glob パターンを読み取り、不足しているエージェントのフィールドを同じ frontmatter に追記する。

## Usage

```sh
/sync-rules [file-name]
```

- `[file-name]`: 対象ファイルのベース名（例: `backend`）。省略時は全ファイルを処理

## エージェント仕様

### Claude Code

| 項目         | 値               |
| ------------ | ---------------- |
| フィールド名 | `paths`          |
| 形式         | YAML 配列        |
| 全体適用     | フィールドを省略 |

```yaml
paths:
  - "src/**/*.ts"
  - "lib/**/*.ts"
```

### GitHub Copilot

| 項目           | 値                           |
| -------------- | ---------------------------- |
| フィールド名   | `applyTo`                    |
| 形式           | カンマ区切り文字列           |
| 全体適用       | `"**"`                       |
| 固有フィールド | `excludeAgent`（オプション） |

```yaml
applyTo: "src/**/*.ts,lib/**/*.ts"
```

## 変換マッピング（canonical ↔ 各フォーマット）

glob パターンの配列を canonical 形式として扱い、各エージェントのフィールドに変換する。

### parse（既存フィールド → canonical）

全エージェントのフィールドをそれぞれ parse し、得られたパターンの **union** を canonical とする。

| エージェント | フィールド       | canonical 変換             |
| ------------ | ---------------- | -------------------------- |
| Claude Code  | `paths: [...]`   | そのまま配列として取得     |
| Claude Code  | `paths` なし     | `[]`（全体適用）として扱う |
| Copilot      | `applyTo: "a,b"` | カンマで分割して配列に変換 |
| Copilot      | `applyTo: "**"`  | `[]`（全体適用）として扱う |

> どれか1つのフィールドにだけ存在するパターンも canonical に含める。

### serialize（canonical → 各フォーマット）

| エージェント | canonical        | 出力フィールド             |
| ------------ | ---------------- | -------------------------- |
| Claude Code  | `["a", "b"]`     | `paths:\n  - "a"\n  - "b"` |
| Claude Code  | `[]`（全体適用） | `paths` フィールドを省略   |
| Copilot      | `["a", "b"]`     | `applyTo: "a,b"`           |
| Copilot      | `[]`（全体適用） | `applyTo: "**"`            |

> `excludeAgent` などの固有フィールドは変換対象外。既存の値があればそのまま維持する。

## 手順

### 1. **対象ファイルを列挙する**

- `.claude/rules/` 内の `*.md` ファイル（`.gitkeep` を除く）
- `[file-name]` が指定された場合はそのファイルのみ処理

### 2. **各ファイルを処理する**

1. ファイルを読み込み、frontmatter とボディを分離する
2. 全エージェントのフィールドをそれぞれ parse し、union を canonical とする
3. canonical と一致しないフィールド（存在しない、または内容が canonical と異なる）を検出する
4. canonical パターンから不一致フィールドを serialize して frontmatter に追記・更新する
5. ファイルを上書き保存する

### 3. **結果を報告する**

- 追記・更新したファイルとフィールド名を表示する
- 全フィールドが揃っていて canonical と完全に一致しているファイルはスキップとして報告する

## 注意事項

- ルール本文（Markdown 本体）や既存フィールドの値は変更しない
- frontmatter が存在しないファイルは全体適用ルールとして扱い、各エージェントの全体適用形式を追加する
