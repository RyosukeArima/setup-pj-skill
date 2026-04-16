## ステップ11: 完了報告

作成したファイル一覧を表示し、以下を案内:

```
## セットアップ完了

### 作成ファイル
- .claude/CLAUDE.md
- .claude/active-task.md
- .claude/settings.json（hooks含む）
- .claude/hooks/pre-compact.sh
- .claude/skills/anchor/SKILL.md
- .claude/skills/anchor/reference/doc-template.md
- （その他作成したファイル）

### Compaction耐性
- PreCompact hook: compaction 直前の dialogue + tool_result を `.claude/snapshots/` に保存
  （次セッション Claude が LAST を Read して性能劣化を防ぐ）
- `/anchor` スキル: フィードバック・合意をドキュメントに固定

### ハーネス協働進化（中長期PJのみ）
- Stop hook: 有効（タスク完了時にactive-task.md更新チェック）
- 違反ログ: .claude/harness-log.jsonl
- 初期チェック項目: active-task.md更新のみ（追加はユーザー承認制）
- 進化の仕方: 違反が蓄積するとエスカレーションを提案。ユーザー承認後にrules/またはhooksに追加

### 品質ゲート（全PJ）
- rules/quality-gates.md: 成果物に応じた品質チェックの使用ルール

### PJ固有エージェント（作成した場合）
- .claude/agents/{name}.md — {用途の説明}

### このPJで使えるグローバルエージェント・スキル
（reference/agent-table.md の該当テーブルを表示）

### 推奨ワークフロー
1. 作業開始時: active-task.md を更新
2. フィードバック受領時: `/anchor update` で文書に固定 → active-task.md 更新 → 作業
3. 作業完了時: active-task.md のチェックを更新
4. Stop hookが問題を検知した場合: ユーザーに報告→確認→対応

### 追加セットアップ（必要な場合）
- 開発PJ: 推奨プラグインのインストールコマンド
- データPJ: Python環境のセットアップコマンド
```
