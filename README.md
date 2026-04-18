# AIエージェント用のファイルセット

このリポジトリは、AIエージェントを活用した開発をスムーズに行うためのファイルセットを提供する。

## サポート対象

- [Claude Code](https://claude.ai/code)
- [GitHub Copilot](https://copilot.github.com/)

## ディレクトリ構成

### 共通

| パス        | 説明                                   |
| ----------- | -------------------------------------- |
| `AGENTS.md` | プロジェクト概要、大原則などの共通知識 |

### Claude Code

| パス                          | 説明                                                                                 |
| ----------------------------- | ------------------------------------------------------------------------------------ |
| `CLAUDE.md`                   | Claude Code 用の知識。`@AGENTS.md` で共通知識をロード                                |
| `.claude/settings.json`       | Claude Code 設定ファイル。フックの定義                                               |
| `.claude/settings.local.json` | ローカル固有設定（gitignore対象）                                                    |
| `.claude/rules/`              | ファイルタイプ別のルール。`.github/instructions/` からシンボリックリンクで参照される |
| `.claude/skills/`             | スキル。`.github/skills/` からシンボリックリンクで参照される                         |
| `.claude/agents/`             | エージェント定義                                                                     |
| `.claude/hooks/`              | フックのスクリプトやアセット                                                         |

### GitHub Copilot

| パス                                  | 説明                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| `.github/copilot-instructions.md`     | Copilot 用の知識。Copilotが自動でAGENTS.mdの共通知識をロード |
| `.github/copilot/settings.json`       | Copilot 設定ファイル                                         |
| `.github/copilot/settings.local.json` | ローカル固有設定（gitignore対象）                            |
| `.github/instructions/`               | `.claude/rules/` へのシンボリックリンク                      |
| `.github/skills/`                     | `.claude/skills/` へのシンボリックリンク                     |
| `.github/agents/`                     | エージェント定義                                             |
| `.github/hooks/`                      | フックの定義。スクリプトやアセット                           |

## 設計決定事項

### シンボリックリンク

#### .claude/rules/ と .github/instructions/

ファイルタイプ別のルールを共有するため。  
ルールファイルには [`applyTo`（GitHub Copilot）](https://docs.github.com/ja/copilot/how-tos/configure-custom-instructions/add-repository-instructions#creating-path-specific-custom-instructions)と [`paths`（Claude Code）](https://code.claude.com/docs/ja/memory#%E3%83%91%E3%82%B9%E5%9B%BA%E6%9C%89%E3%81%AE%E3%83%AB%E3%83%BC%E3%83%AB)の両フィールドを併記することで、各ツールが自分の対応フィールドだけ解釈する形にしてツール間の非互換を吸収する。

#### .claude/skills/ と .github/skills/

Agent Skills で標準化されているスキルをそのまま共有するため。

### 個別管理

#### エージェント定義・フック

書き方がツール固有のため個別管理する。

TODO: ツール固有の部分を置き換えるスキルを作成する

#### 設定まわり

プラグイン、MCP、権限関連の設定はツール固有のため個別管理する。
