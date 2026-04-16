---
name: setup-project
description: プロジェクトの .claude/ ディレクトリ構造を自動生成し、ハーネスエンジニアリングの原則でClaude Codeのパフォーマンスを最大化する。開発PJだけでなく、BizDev、リサーチ、文書作成、データ分析、戦略立案など幅広い業務に対応。
---

プロジェクト $ARGUMENTS のClaude Code環境をセットアップする。

## 手順

### 0. PJヒアリング

以下のいずれかに該当する場合、AskQuestionでユーザーにPJの目的と規模を確認する:

- ディレクトリが空 or ファイルが5個以下
- $ARGUMENTS にPJ名のみで目的の記述がない
- 検出シグナルで種別が一意に定まらない（複数候補）

```
AskQuestion:
「このPJについて教えてください。

1. 主な目的:
   (a) ソフトウェア開発
   (b) 文書・資料作成（提案書、レポート等）
   (c) リサーチ・調査
   (d) 戦略・企画立案
   (e) データ分析
   (f) その他（説明してください）

2. 想定規模:
   (1) 短期（1-2セッションで完了）
   (2) 中長期（継続的に作業）」
```

ファイルから種別が明確に判定できる場合（package.json等）でも、規模のみ確認する。

**既存PJの後付け適用:**
`.claude/CLAUDE.md` が既に存在する場合はAskQuestionで確認:
- (a) ハーネス進化層を追加する → 既存ファイルを保持したまま新規ファイルのみ追加
- (b) 既存構成のまま使う → 何もしない
- (c) 全体を再セットアップ → 既存を `.claude/backup-YYYYMMDD/` にバックアップしてから再生成

**既存のCLAUDE.mdは絶対に上書きしない。** settings.jsonはマージ（既存のpermissions/hooksを保持しつつ新規hookを追加）。

### 1. プロジェクト種別の判定

ステップ0の回答で種別が確定していればそれを使用。未確定ならディレクトリ内のファイルを調査して判定する。複数に該当する場合もある。

| 種別 | 検出シグナル |
|------|------------|
| 開発PJ | `package.json`, `tsconfig.json`, `requirements.txt`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, `build.gradle`, `pom.xml`, `Dockerfile` 等 |
| ドキュメント・執筆PJ | `.md`/`.docx`/`.tex` が多数、`docs/`/`content/`/`articles/` ディレクトリ |
| データ分析PJ | `.ipynb`/`.csv`/`.parquet`、`notebooks/`/`data/` ディレクトリ |
| 戦略・企画PJ | 事業計画・施策立案・提案書が目的（ステップ0で確認） |
| リサーチPJ | リサーチが主目的（ステップ0で確認）、または上記に該当しない |

### 2. 活動タイプの判定

PJ種別とは別に、このPJで発生する「活動」を判定する。1つのPJで複数該当しうる。

| 活動タイプ | 検出シグナル |
|---|---|
| つくる（実装） | ソースコード、ビルド設定が存在 |
| つくる（文書） | .md/.docx/.tex が主要成果物 |
| 調べる | リサーチが主目的 |
| 考える（戦略） | 戦略立案、施策検討、意思決定が必要 |
| 検証する（コード） | テスト・レビュー対象のコードが存在 |
| 検証する（文書） | レビュー・校正対象の文書が存在 |
| 検証する（UX） | UIを持つアプリケーション |
| 分析する | データファイル、分析スクリプトが存在 |

### 3. .claude/ ディレクトリ生成

```
.claude/
├── CLAUDE.md              # PJ概要と作業ルール（50行目安、最大100行）
├── state/                 # セッション引き継ぎファイル
│   ├── roadmap.md         # PJ全体のフェーズ構成と進捗
│   ├── active-task.md     # 現在の作業状態（compaction復元用）
│   ├── decisions.md       # 確定事項ログ（specをまたいで残る決定記録）
│   └── harness-log.jsonl  # 違反ログ（中長期PJのみ）
├── settings.json          # パーミッション + hooks
├── rules/                 # 常時ロードされるルール
│   ├── quality-gates.md   # 品質チェック使用ルール（全PJ）
│   └── harness-evolution.md  # 協働進化ルール（中長期PJのみ）
├── skills/
│   └── anchor/            # コンテキスト固定スキル（全PJ共通）
├── hooks/                 # compaction耐性 + ハーネス進化
│   ├── pre-compact.sh     # compaction 直前の dialogue+tool_result を .claude/snapshots/ に保存
│   └── stop-guardian.sh   # Stop hook（中長期PJのみ）
├── snapshots/             # PreCompact hook が出力（.gitignore 対象）
└── agents/                # PJ固有エージェント
```

### 4. CLAUDE.md の内容（50行目安、最大100行）

**原則: CLAUDE.mdは骨子のみ。詳細はrules/やskills/reference/に退避する。**

全PJ共通:
- PJの目的・概要（1-2行）
- コマンド（開発PJのみ）
- 厳守ルール（Git、セキュリティ等、省略不可のもののみ）
- Spec-Driven ワークフロー（下記）
- ハーネス協働進化ポインタ（2行のみ、中長期PJ）
- リファレンステーブル（「詳細は /xxx 参照」形式のポインタ）

**Spec-Driven ワークフロー（全PJ必須）:**

**計画を先に書き、ユーザーと合意してからアクションする。** コードでも文書でも分析でも同じ原則。

```
### Spec-Driven ワークフロー

本PJは Spec-Driven で進める。
**計画を先に書き、ユーザーと合意してからアクションする。** コードでも文書でも分析でも同じ。

#### ロードマップとspecの2段構造

1. **ロードマップ**: PJ全体のフェーズ分け。`state/roadmap.md` で管理する
2. **spec**: 各フェーズの詳細計画。`/anchor create <name>` で作成する

ロードマップはPJ開始時にユーザーと合意し、`state/roadmap.md` に書く。
各フェーズに着手する時にspecを書き、**1つのspecを終えるごとにセッションを切る。**

#### specの規模判定

| 規模 | 判定基準 | 必要なspec |
|------|---------|------------------|
| 小 | バグ修正、1ファイルの微調整、typo修正 | 不要。state/active-task.md の更新のみ |
| 中 | 機能追加、調査タスク、複数ファイル変更 | 軽量spec（目的/スコープ/成功条件/進め方）→ 着手 |
| 大 | 新機能設計、アーキテクチャ変更、戦略立案 | フルspec → **ユーザーレビュー・合意** → 着手 |

spec の置き場: `docs/specs/<name>.md`
必須セクション: 目的 / 背景 / スコープ / 非スコープ / 成功条件 / 進め方 / 検証方法 / リスク

**spec なしでのアクション着手は禁止。** 会話中のフィードバック・方針変更も、spec を更新してからアクションする。
```

**ハーネス協働進化ポインタ（中長期PJのみ、2行）:**
```
### ハーネス協働進化
Stop hookがタスク完了時にチェックを行い、問題があればユーザーに報告する。詳細は rules/harness-evolution.md 参照。
```

種別固有の**骨子のみ**:
- **開発PJ:** ビルド/テストコマンド、ブランチ戦略、型システム注意点
- **ドキュメントPJ:** 文体・トーン、対象読者
- **データ分析PJ:** データソース場所、分析環境
- **戦略・企画PJ:** 意思決定ゴール、判断基準
- **リサーチPJ:** リサーチ目的・問い

**冗長化しがちな情報の退避先:**
| 情報 | 退避先 |
|------|--------|
| URL体系・アーキテクチャ | skills/のreference/ |
| テスト項目・チェックリスト | skills/のreference/ |
| データ整合性ルール | skills/のreference/ |
| よくあるミス・注意事項 | skills/のreference/ |
| 品質チェック使用ルール | rules/quality-gates.md |
| 協働進化ルール | rules/harness-evolution.md |

### 5. settings.json

PJ種別に応じたパーミッション + hooks を設定:
- 開発PJ: テスト、ビルド、lintコマンドを allow
- データPJ: jupyter、pythonスクリプト実行を allow
- 全PJ: git基本操作を allow

**hooks（全PJ共通 — compaction耐性）:**

目的は「次セッション Claude のパフォーマンス維持」。PreCompact で transcript から
dialogue + tool_result を抽出して `.claude/snapshots/` に保存。PostCompact hook は使わない
（state/active-task.md / state/decisions.md は compaction で消えないので注入不要）。

```json
{
  "hooks": {
    "PreCompact": [
      {
        "matcher": "auto",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/pre-compact.sh\"",
            "timeout": 30
          }
        ]
      },
      {
        "matcher": "manual",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/pre-compact.sh\"",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

PreCompact hook の中身と依存（`jq`）、`.gitignore` に `.claude/snapshots/` を追加する件は
`phase-compaction.md` の「生成物3〜5」を参照。

**hooks（中長期PJのみ — ハーネス協働進化）:**

上記に加え、Stop hookをマージする。詳細は `phase-harness-evolution.md` の生成物5を参照。

### 6. .claudeignore

**全PJ共通:** `*.key`, `*.pem`, `*.p12`, `*.pfx`, `*.jks`, `*.keystore`（バイナリ鍵ファイル）。`.env` は除外しない（存在認識が必要。中身はグローバルのdeny+hookでブロック済み）。

**開発PJ追加:** `node_modules/`, `dist/`, `build/`, `.venv/`, `__pycache__/`

**データ分析PJ追加:** 100MB超の `.csv`/`.parquet`、`.ipynb_checkpoints/`

### 7. rules/

**全PJ共通:**
- `rules/chrome-tools.md` — Claude in Chrome（`mcp__claude-in-chrome__*`）はユーザーの明示的指示がない限り使用禁止。ToolSearchでのロードも指示なしでは行わない。
- `rules/quality-gates.md` — 成果物→品質チェックの対応表。詳細は `phase-harness-evolution.md` の生成物4を参照。

**中長期PJのみ:**
- `rules/harness-evolution.md` — 協働進化ルール。詳細は `phase-harness-evolution.md` の生成物3を参照。

**PJ種別固有:**
- **開発PJ:** TypeScript検出 → グローバルの `typescript.md` 参照、Python検出 → `python.md` 参照、React検出 → `react.md` 参照。PJ固有の規約があれば追加。
- **ドキュメントPJ:** 文体ルール（表記揺れ防止、用語集）、レビュー観点
- **データ分析PJ:** 再現性、ドキュメント化、可視化ルール

### 8. Compaction耐性の構築

`phase-compaction.md` を参照し、以下を生成する:

1. **hooks/** — pre-compact.sh（状態保存）+ session-compact.sh（コンテキスト再注入）
2. **skills/anchor/** — コンテキスト固定スキル（PJ種別に応じたテンプレート）
3. **state/roadmap.md** — PJ全体のフェーズ構成と進捗
4. **state/active-task.md** — 現在の作業状態テンプレート
5. **state/decisions.md** — 確定事項ログ（specをまたいで残る決定記録）

### 8.5. ハーネス協働進化の構築（ステップ8完了後）

`phase-harness-evolution.md` を参照し、以下を生成する:

**中長期PJのみ:**
1. **hooks/stop-guardian.sh** — Stop hook（検知・報告）
2. **state/harness-log.jsonl** — 違反ログ（空ファイル）
3. **rules/harness-evolution.md** — 協働進化ルール

**全PJ:**
4. **rules/quality-gates.md** — 品質チェック使用ルール

**短期PJの場合はquality-gates.mdのみ生成し、他はスキップする。**

### 9. PJ固有エージェントの作成判断

詳細は `phase-agent-template.md` を参照して判断・作成する。

### 10. 活動タイプに応じたエージェント・スキル活用ガイド

`phase-agent-table.md` を参照し、ステップ2で判定した活動タイプに該当するテーブルのみをCLAUDE.mdに転記する。

### 11. 完了報告

`phase-completion.md` のフォーマットに従い報告する。
