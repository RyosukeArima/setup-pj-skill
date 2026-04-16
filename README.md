# setup-project skill for Claude Code

Claude Code のプロジェクト初期セットアップを自動化するスキル。

`/setup-project` を実行すると、以下の `.claude/` ディレクトリ構造を生成します。

```
.claude/
├── CLAUDE.md              # PJ概要と作業ルール（50行目安）
├── state/                 # セッション引き継ぎファイル
│   ├── roadmap.md         # PJ全体のフェーズ構成と進捗
│   ├── active-task.md     # 現在の作業状態（compaction復元用）
│   └── decisions.md       # 確定事項ログ
├── settings.json          # パーミッション + hooks
├── rules/                 # 常時ロードされるルール
├── skills/anchor/         # コンテキスト固定スキル
├── hooks/                 # compaction耐性 + 完了検証
└── agents/                # PJ固有エージェント
```

## 解決する問題

- **忘れる**: セッションが長くなると compaction で指示やルールが消える
- **守らない**: CLAUDE.md のルールがコンテキスト膨張で参照されなくなる
- **暴走する**: スコープが固定されていないと作業範囲が勝手に広がる

## 使い方

### 1. スキルを配置

```bash
git clone https://github.com/RyosukeArima/setup-pj-skill.git
cp -r setup-pj-skill/setup-project/ ~/.claude/skills/setup-project/
```

### 2. プロジェクトで実行

```bash
cd your-project/
claude
# Claude Code 内で:
/setup-project your-project-name
```

PJ種別（開発/文書作成/リサーチ/戦略立案/データ分析）を自動判定し、適切な構成を生成します。

## 設計思想

**セッションを細かく切る。切っても文脈が戻るようにしておく。**

- **L1 引き継ぎの基盤**: `state/roadmap.md` + `state/active-task.md` + `state/decisions.md` + spec でセッション間の文脈を維持
- **L2 保険**: pre-compact hook（compaction 前のスナップショット）、stop-guardian hook（完了検証）
- **L3 育成**: 違反ログ → ルール明文化 → hook 昇格。使い込むほどルールが実態に合ってくる

## ファイル構成

| ファイル | 内容 |
|---------|------|
| `SKILL.md` | スキル本体（実行手順） |
| `phase-compaction.md` | compaction 耐性の構築（テンプレート一式） |
| `phase-harness-evolution.md` | hook・ルール育成の仕組み |
| `phase-completion.md` | セットアップ完了報告 |
| `phase-agent-table.md` | 活動タイプ別エージェント対応表 |
| `phase-agent-template.md` | PJ固有エージェント作成ガイド |

## 依存

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
- `jq`（pre-compact hook が使用。`brew install jq`）

## License

MIT
