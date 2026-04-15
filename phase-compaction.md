## ステップ8: Compaction耐性の構築

会話のコンテキスト圧縮（compaction）で Claude の性能が劣化する問題に対処する。
全PJ種別で共通のパターンを適用する。

### 設計思想: 目的は次セッション Claude のパフォーマンス維持

スナップショットの目的を明確化:

- **○** compaction summary で潰される「直前会話の原文」「ツール結果の事実」を保全し、次セッションの Claude が再発見せずに済むようにする（= 性能劣化防止）
- **×** 人間が後から読むための forensic ログ（それは別用途、今回のスコープ外）
- **×** active-task.md / decisions.md の内容を重複して保存する（ファイルは compaction で消えない、再 Read すれば同じものが読める）

この方針から導かれる 4 層設計:

```
[起点] Spec-Driven ワークフロー
  → ロードマップでフェーズ分け → 各フェーズの計画（spec）を先に作成・合意してから実行
  → spec がcompaction後の文脈復元の権威あるソースになる

[主役] ルール（CLAUDE.md / rules/）
  → メインClaudeが全文脈を持った状態で判断・更新
  → コストゼロ、判断精度最高

[補助] PreCompact hook（dialogue + evidence 保全）
  → transcript.jsonl を dialogue-only な md に変換 → .claude/snapshots/ に保存
  → 次セッション Claude が summary で不足を感じた時に LAST を Read できる
  → PostCompact hook は使わない（active-task.md 等は compaction で消えないので注入不要）

[根本] 作業分割
  → /clear → active-task.md読み込み → 続行
  → compactionを起こさない粒度設計
```

---

### 生成物1: CLAUDE.md への追記

CLAUDE.md に以下の2セクションを追加する（既存の内容に追記）:

```markdown
## コンテキスト管理

### active-task.md 更新タイミング（必須）

以下のタイミングで `.claude/active-task.md` を更新すること:

1. **フィードバック・方針変更を受けたとき** — アクションに進む前に更新
2. **技術判断・設計決定をしたとき** — 判断理由も1行で記録
3. **フェーズ・タスク完了時** — チェックを付けて次のTODOを明記
4. **仕様書やドキュメントを作成・更新したとき** — 参照パスを記録

active-task.mdは**30行以内**に保つ。完了済みは詳細省略。

### decisions.md — 確定事項ログ

ユーザーと合意した決定事項を `.claude/decisions.md` に記録すること。specをまたいで残る「これまでに決まったこと」の台帳。

- specは完了時にクローズされる。確定事項はspec完了後も有効なので、specに混ぜない
- 新しい確定事項が会話で生まれたら、その場でdecisions.mdに追記する
- 凍結・立て直し時にも引き継ぐ最優先ファイル

### Compact Instructions

compaction時に以下を保持すること:

- 変更したファイルパスと行番号
- 現在のテスト結果（pass/fail + ファイル名）
- アクティブなタスク計画と残TODOリスト
- アーキテクチャ判断とその理由
- エラーメッセージとスタックトレース
```

### 生成物2: rules/ へのルール追加

`rules/context-persistence.md` として以下を作成:

```markdown
# コンテキスト永続化ルール

## active-task.md — 最重要ファイル

compaction後の文脈復元の起点。以下のタイミングで**必ず**更新する:

| タイミング | 何を記録するか |
|-----------|---------------|
| 新規タスク開始時（中〜大） | spec 作成（docs/specs/） → 参照パスを記録（**着手前に**） |
| フィードバック・方針変更を受けた時 | spec に反映（**アクション前に**） |
| 技術判断・設計決定をした時 | 判断内容と理由（1行） |
| フェーズ・タスク完了時 | チェック付け + 次のTODO |
| 仕様書・ドキュメント作成・更新時 | 参照パスを追記 |

### 書き方の制約

- **30行以内**に保つ
- 完了済みタスクは詳細省略し「完了」とだけ記載
- 未完了・進行中の情報を優先
- 参照ドキュメントのパスを必ず含める（復元時に辿れるように）

## decisions.md — 確定事項ログ

specをまたいで残る「これまでに決まったこと」の台帳。specは完了時にクローズされるが、確定事項はPJ全体にわたって有効。

| タイミング | 何を記録するか |
|-----------|---------------|
| ユーザーと合意した時 | 決定内容と日付（1-2行） |
| specを凍結・立て直す時 | 凍結specの中で「確定として残る事項」を移す |
| 方針転換した時 | 旧方針を「没にした方針」として記録 |

### 書き方の制約

- 確定事項と没にした方針を分けて書く
- 日付を必ず入れる（相対日付は禁止、絶対日付で）
- specに確定事項を混ぜない（完了時に埋もれる）
- active-task.mdから「確定事項: decisions.md 参照」でポインタを張る

## compaction後の復帰

`/clear` やcompaction後に文脈を失った場合:

1. `.claude/active-task.md` を読む
2. `.claude/decisions.md` を読む（確定事項の把握）
3. 記載された参照ドキュメントを読む
4. `git status` と `git log -3 --oneline` で直近の変更を確認
5. **summary だけで不足を感じたら `.claude/snapshots/LAST` を Read する**
   （compaction 直前の dialogue + tool_result が dialogue-only 形式で入っている）
6. 作業を継続する
```

**PJ種別ごとの追記:**
- **開発PJ**: 「仕様書（`docs/specs/`）が実装の権威ある情報源。会話内決定は仕様書に反映してからコードを書く」
- **リサーチPJ**: 「発見事項はリサーチブリーフ（`docs/briefs/`）に随時追記。会話内の発見を放置しない」
- **戦略・企画PJ**: 「判断とその根拠は意思決定ログ（`docs/decisions/`）に記録。口頭合意を残さない」
- **ドキュメントPJ**: 「構成変更は構成メモ（`docs/outlines/`）に反映。口頭での構成議論を文書化する」
- **データ分析PJ**: 「仮説と中間結果は分析計画（`docs/analysis/`）に記録。再現可能な状態を維持」

---

### 生成物3: hooks/pre-compact.sh

目的は「summary で潰される dialogue と tool_result の事実を保全する」こと。
transcript.jsonl からダイアログ相当のメッセージのみ抽出し、タイムスタンプ付き md に変換する。
thinking ブロックは除外（内部推論は次セッション Claude に読ませても性能改善に寄与しない、
むしろノイズ）。

```bash
#!/bin/bash
# PreCompact hook: compaction で失われる会話履歴を dialogue + 事実エビデンス形式で保存する。
# 目的: summary で潰された事実を次セッション Claude が辿れるようにする。
#
# 残す: user 発言 / assistant text / tool_use の呼び出し痕跡 / tool_result の事実データ
# 捨てる: assistant の thinking ブロック、image ブロック
#
# stdin: {session_id, transcript_path, trigger, cwd, hook_event_name, ...}

set -euo pipefail

DIR="$CLAUDE_PROJECT_DIR/.claude"
SNAPSHOT_DIR="$DIR/snapshots"
mkdir -p "$SNAPSHOT_DIR"

INPUT="$(cat || true)"
if [ -z "$INPUT" ]; then
  exit 0
fi

TRANSCRIPT=$(echo "$INPUT" | jq -r '.transcript_path // empty' 2>/dev/null || true)
SID=$(echo "$INPUT" | jq -r '.session_id // "unknown"' 2>/dev/null | cut -c1-8)
TRIGGER=$(echo "$INPUT" | jq -r '.trigger // "unknown"' 2>/dev/null || true)

if [ -z "$TRANSCRIPT" ] || [ ! -f "$TRANSCRIPT" ]; then
  exit 0
fi

TS=$(date +%Y%m%d-%H%M%S)
DEST="$SNAPSHOT_DIR/${TS}-${SID}.md"

{
  echo "# Dialogue + Evidence snapshot"
  echo ""
  echo "- session: \`$SID\`"
  echo "- trigger: \`$TRIGGER\`"
  echo "- transcript: \`$TRANSCRIPT\`"
  echo "- captured: $(date -Iseconds 2>/dev/null || date)"
  echo ""
  echo "---"
  echo ""

  jq -r '
    select(.type == "user" or .type == "assistant") |
    (.timestamp // "") as $ts |
    (.message.content) as $c |
    (
      if ($c | type) == "string" then
        $c
      else
        (
          $c
          | map(
              if .type == "text" then
                .text
              elif .type == "tool_use" then
                "[tool_use: \(.name)]"
              elif .type == "tool_result" then
                (
                  if (.content | type) == "string" then
                    .content
                  elif (.content | type) == "array" then
                    (.content | map(select(.type == "text") | .text) | join("\n"))
                  else
                    ""
                  end
                ) as $body
                | "[tool_result]\n\($body)"
              else
                empty
              end
            )
          | join("\n\n")
        )
      end
    ) as $text |
    if ($text | length) == 0 then
      empty
    else
      (if .type == "user" then "USER" else "ASSISTANT" end) as $role |
      "### [\($ts)] \($role)\n\n\($text)\n"
    end
  ' "$TRANSCRIPT" 2>/dev/null || echo "(jq parse failed — transcript 形式が想定外)"
} > "$DEST"

ln -sf "$(basename "$DEST")" "$SNAPSHOT_DIR/LAST"

exit 0
```

**実行権限**: `chmod +x .claude/hooks/pre-compact.sh`

**依存**: `jq` コマンド（mac: `brew install jq`）

### 生成物4: settings.json のhooks部分

PreCompact の matcher は `auto` と `manual` 両方を明示する。
公式仕様上 matcher 省略の扱いは不定のため省略しない。PostCompact は不要。

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

### 生成物5: .gitignore への追記

スナップショットはローカル資産。commit しない。

```gitignore
# Claude snapshot (regenerated automatically on PreCompact)
.claude/snapshots/
```

### 生成物6: skills/anchor/SKILL.md

PJ種別に応じてテンプレートとディレクトリ名を調整する。以下のテンプレートを使用:

```markdown
---
name: anchor
description: Spec-Driven で作業を進める。ロードマップでフェーズを分け、各フェーズの計画（spec）を先に書いてユーザー合意してから着手。フィードバック時は spec を先に更新。compaction耐性も確保する。
user-invocable: true
---

## Spec-Driven ワークフロー（anchor）

**原則: 計画を先に書き、ユーザーと合意してからアクションする。**
会話中の合意・フィードバック・発見も、必ずドキュメントに反映してからアクションする。

### ロードマップとspecの2段構造

1. **ロードマップ**: PJ全体のフェーズ分け。`roadmap.md` で管理する
2. **spec**: 各フェーズの詳細計画。`/anchor create <name>` で作成

ロードマップはPJ開始時にユーザーと合意し、`roadmap.md` に書く。
各フェーズに着手する時にspecを書き、**1つのspecを終えるごとにセッションを切る。**

### コマンド

- `/anchor` — このワークフロー説明を表示
- `/anchor create <name>` — 新規 spec を作成
- `/anchor update <name>` — 既存 spec を更新
- `/anchor list` — spec 一覧を表示
- `/anchor roadmap` — ロードマップを表示・更新

### spec の置き場

`{DOCS_PATH}/specs/<name>.md`

### spec の必須セクション

1. **目的** — この spec が達成したいこと（1-3行）
2. **背景** — なぜ今これをやるのか、どういう文脈か
3. **スコープ** — やること
4. **非スコープ** — やらないこと（ここを明示するのが重要）
5. **成功条件** — 完了判定基準（チェック可能な形で）
6. **進め方** — ステップ分解
7. **検証方法** — 完成物をどう確かめるか
8. **リスク / 未解決事項**

軽量 spec（中規模タスク）は 1/3/4/5/6 のみでも可。大規模タスクは全項目必須。

### /anchor roadmap の仕様

ロードマップの表示・更新を行う。

- 引数なし → `roadmap.md` を表示
- `/anchor roadmap init` → ユーザーとフェーズ分けを合意し、`roadmap.md` を初期化
- `/anchor roadmap done <phase#>` → 指定フェーズを done に更新し、完了日を記録
- `/anchor roadmap activate <phase#>` → 指定フェーズを active に更新。同時に active-task.md の「現在のフェーズ」を更新

フェーズの状態遷移: `pending` → `active` → `done`
active は同時に1つだけ（複数を active にしない）。

### PJ開始フロー

1. **ロードマップ作成**: PJ全体のフェーズ分けをユーザーと合意し、`/anchor roadmap init` で `roadmap.md` に書く
2. **フェーズごとにspec作成**: 各フェーズに着手する時に `/anchor create <name>` でspecを書き、`/anchor roadmap activate <#>` でフェーズをactiveにする
3. **1つのspecを終えるごとにセッションを切る**: `/anchor roadmap done <#>` → `/clear` → 次のフェーズへ

### フェーズ内の作業フロー

1. タスクの規模を判定（小/中/大）
   - **小**（バグ修正、typo、1ファイル修正）→ spec なし。active-task.md 更新のみで着手
   - **中**（機能追加、調査、複数ファイル変更）→ `/anchor create <name>` で軽量 spec 作成 → 着手
   - **大**（新機能設計、アーキテクチャ変更、戦略立案）→ `/anchor create <name>` でフル spec 作成 → **ユーザーレビュー・合意** → 着手
2. `.claude/active-task.md` を更新（参照 spec、チェックリスト）
3. 作業に着手
4. 項目完了ごとに active-task.md を更新

**spec なしでの着手は禁止。** 「小さそうだから」で始めて中規模化した場合は、その場で spec を書き起こす。

### 確定事項の記録

ユーザーと合意した決定事項は `.claude/decisions.md` に記録する。

- specは完了時にクローズされるが、確定事項はspec完了後も有効
- 新しい確定事項が会話で生まれたら、その場でdecisions.mdに追記する
- specを凍結・立て直す時は、確定として残る事項をdecisions.mdに移す
- active-task.mdの「確定事項」欄は「decisions.md 参照」のポインタのみ

### フィードバック受領時のフロー

1. フィードバックを項目化する
2. `/anchor update <name>` で該当 spec を**アクション前に**更新
3. 確定事項が含まれていれば `.claude/decisions.md` に追記
4. `.claude/active-task.md` を更新（チェックリスト形式）
5. アクションに着手
6. 項目完了ごとに active-task.md を更新

### spec 作成手順

1. `reference/doc-template.md` を読み、PJ種別に応じたテンプレートで作成
2. 上記「必須セクション」を満たす
3. 大規模タスクの場合、ユーザーと内容を確認・合意してから着手
4. `.claude/active-task.md` を更新

### active-task.md の管理

作業開始時・中断時・完了時に `.claude/active-task.md` を更新する。
compaction後の文脈復元に使われる最重要ファイル。

\```markdown
# Active Task
## 作業中のテーマ
[テーマ名]
## 参照ドキュメント
{DOCS_PATH}/[name].md
## 現在の進捗
- [x] 完了した項目
- [ ] 次にやること
## ブロッカー/決定事項
[あれば記載]
\```
```

### 生成物6.5: roadmap.md（初期テンプレート）

```markdown
# ロードマップ

PJ全体のフェーズ構成と進捗。各フェーズの詳細は spec を参照。

## フェーズ一覧

| # | フェーズ | 状態 | spec | 完了日 |
|---|---------|------|------|--------|
| 1 | （フェーズ名） | pending | - | - |
| 2 | （フェーズ名） | pending | - | - |
| 3 | （フェーズ名） | pending | - | - |

状態: `pending` → `active` → `done`（active は同時に1つだけ）

## 依存関係

（フェーズ間の依存がある場合に記載。なければ「上から順に実行」）
```

### 生成物7: active-task.md（初期テンプレート）

```markdown
# Active Task

## 状況: 進行中

## 作業中のテーマ
（セットアップ完了。`/anchor roadmap init` でロードマップ作成後に更新）

## 現在のフェーズ
（`/anchor roadmap activate <#>` でフェーズ着手時に更新。参照specのパスを記載）
roadmap.md 参照

## 残TODO
（フェーズ内の残作業をチェックリストで管理）

## 確定事項
decisions.md 参照
```

### 生成物8: decisions.md（初期テンプレート）

```markdown
# 確定事項ログ

specをまたいで残る「これまでに決まったこと」の台帳。
specは完了時にクローズされるが、ここに記録した事項はPJ全体にわたって有効。

## 確定済みの事項

（まだなし。ユーザーと合意した決定事項をここに追記する）

<!-- 記入例:
- 2026-04-10: 認証方式はOAuth2.0（API Key方式は不採用）。理由: マルチテナント対応が必要
- 2026-04-12: APIレスポンスはsnake_case統一。フロントでcamelCaseに変換
-->

## 没にした方針

（まだなし。試して却下した方向性を記録し、同じ失敗を繰り返さない）

<!-- 記入例:
- ❌ REST API で全件取得（理由: ページネーション未対応で大量データ時にタイムアウト → GraphQL に変更）
-->
```

---

### チェックリスト

- [ ] CLAUDE.md に「コンテキスト管理」セクション追記（Compact Instructions + 更新タイミング）
- [ ] rules/context-persistence.md を作成（PJ種別に応じた内容）
- [ ] hooks/pre-compact.sh を生成（chmod +x、jq 依存を README に記載）
- [ ] settings.json に PreCompact hooks（matcher: auto/manual 両方）を追加。**PostCompact は不要**
- [ ] .gitignore に `.claude/snapshots/` を追加
- [ ] skills/anchor/SKILL.md をPJ種別に応じて生成
- [ ] skills/anchor/reference/doc-template.md をPJ種別に応じて生成
- [ ] active-task.md を初期テンプレートで作成
- [ ] decisions.md を初期テンプレートで作成
