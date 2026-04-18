---
paths:
  - ".vscode/extensions.json"
  - "**/settings.json"
  - "**/settings.local.json"
applyTo: ".vscode/extensions.json,**/settings.json,**/settings.local.json"
description: テスト用
---

# VSCode 設定ファイルのルール

- `settings.json` は JSON 形式を維持し、末尾カンマを含めない
- `extensions.json` の `recommendations` はアルファベット順に並べる
- エージェント固有の設定（`.claude/`, `.github/copilot/` など）は対応する設定ファイルにのみ記述する
