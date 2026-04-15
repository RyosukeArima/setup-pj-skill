## ステップ8.5: ハーネス協働進化の構築

PJの運用中にハーネスがユーザーとの協働で進化する仕組みを構築する。
hookは「検知・報告」に徹し、判断はユーザーに委ねる。

### 前提条件

- ステップ0でPJ規模が「中長期」と判定されていること
- 短期PJ（1-2セッション）の場合、このステップはスキップする（quality-gates.mdのみ生成）

### 設計原則

- hookは「ないと困る」最小限のチェックのみ。縛りすぎはミスの元
- 迷ったらrules/（確率的制約）から始め、運用で必要が確認されたらhookに昇格
- Claudeの自発的行動に頼らず、hookのstderrで具体的行動を指示する

---

### 生成物1: hooks/stop-guardian.sh

Stop hookの核心。タスク完了時のチェックとユーザーへの報告を指示する。

PJ種別に応じて `%%PJ_SPECIFIC_CHECKS%%` 部分をカスタマイズする。
**初期状態では全PJ共通のactive-task.mdチェックのみ**。ドメイン固有チェックは運用中にユーザー承認を得て追加する。

```bash
#!/bin/bash
# Stop Guardian: タスク完了時のチェック → ユーザーへの報告指示
set -euo pipefail

# jq存在チェック
if ! command -v jq &>/dev/null; then
  exit 0  # jqがなければ素通り
fi

INPUT=$(cat)
DIR="$CLAUDE_PROJECT_DIR/.claude"
LOG="$DIR/harness-log.jsonl"

# 無限ループ防止
if [ "$(echo "$INPUT" | jq -r '.stop_hook_active // false')" = "true" ]; then
  exit 0
fi

MSG=$(echo "$INPUT" | jq -r '.last_assistant_message // ""')
ISSUES=""

# --- チェック ---

# チェック1: active-task.md の状態整合性（構造のみ・メッセージ解析なし）
# フォーマット: "## 状況: 進行中" のようにインライン記載（rules/active-task-lifecycle.md で規定）
if [ -f "$DIR/active-task.md" ]; then
  if ! grep -q "最初のタスク着手時に更新" "$DIR/active-task.md" 2>/dev/null; then
    STATUS_LINE=$(grep '^## 状況' "$DIR/active-task.md" 2>/dev/null | head -1 || echo "")
    UNCHECKED=$(grep -c '^- \[ \]' "$DIR/active-task.md" 2>/dev/null || true)
    UNCHECKED=${UNCHECKED:-0}
    CHECKED=$(grep -c '^- \[x\]' "$DIR/active-task.md" 2>/dev/null || true)
    CHECKED=${CHECKED:-0}

    # 不整合1: 状況が「完了」なのに未チェック項目がある
    if echo "$STATUS_LINE" | grep -q "完了"; then
      if [ "$UNCHECKED" -gt 0 ]; then
        ISSUES="${ISSUES}- active-task.md: 状況「完了」ですが未完了項目が${UNCHECKED}件あります\n"
      fi
    fi

    # 不整合2: 全項目チェック済みなのに状況が「進行中」
    if [ "$CHECKED" -gt 0 ] && [ "$UNCHECKED" -eq 0 ]; then
      if echo "$STATUS_LINE" | grep -q "進行中"; then
        ISSUES="${ISSUES}- active-task.md: 全項目完了ですが状況が「進行中」のままです\n"
      fi
    fi
  fi
fi

# チェック2: PJ固有（拡張ポイント。運用中にユーザー承認を得て追加する）
# %%PJ_SPECIFIC_CHECKS%%

# --- 問題なし → 素通り ---

if [ -z "$ISSUES" ]; then
  exit 0
fi

# --- 問題あり → ログ記録 + 報告指示 ---

# ログローテーション（100件超なら最新50件に切り詰め）
if [ -f "$LOG" ]; then
  LINE_COUNT=$(wc -l < "$LOG" | tr -d ' ')
  if [ "$LINE_COUNT" -gt 100 ]; then
    TMP=$(mktemp)
    tail -50 "$LOG" > "$TMP" && mv "$TMP" "$LOG"
  fi
fi

# 違反ログに記録
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
ISSUE_FLAT=$(printf '%b' "$ISSUES" | tr '\n' '|' | sed 's/|$//')
echo "{\"ts\":\"$TIMESTAMP\",\"issues\":\"$ISSUE_FLAT\"}" >> "$LOG"

# エスカレーション判定
ESCALATION=""
if [ -f "$LOG" ]; then
  while IFS= read -r issue_line; do
    [ -z "$issue_line" ] && continue
    CLEAN=$(echo "$issue_line" | sed 's/^- //')
    # issueパターンの部分一致でカウント（完全一致だと件数部分が変わるため）
    KEYWORD=$(echo "$CLEAN" | sed 's/ [0-9]*件.*$//' | head -c 30)
    COUNT=$(grep -cF "$KEYWORD" "$LOG" 2>/dev/null || echo "0")
    if [ "$COUNT" -ge 3 ]; then
      ESCALATION="${ESCALATION}  - 「${CLEAN}」が${COUNT}回目: hookチェック項目の追加を検討してください\n"
    elif [ "$COUNT" -ge 2 ]; then
      ESCALATION="${ESCALATION}  - 「${CLEAN}」が${COUNT}回目: rules/へのルール追加を検討してください\n"
    fi
  done <<< "$(echo -e "$ISSUES" | sed '/^$/d')"
fi

# 棚卸し判定
HOUSEKEEPING=""
if [ -f "$LOG" ]; then
  TOTAL=$(wc -l < "$LOG" | tr -d ' ')
  if [ "$TOTAL" -ge 50 ]; then
    HOUSEKEEPING="【棚卸し推奨】違反ログが${TOTAL}件に達しました。ハーネスの棚卸し（不要ルールの除去、hookの見直し）を提案してください。"
  fi
fi

# stderrにフィードバック（Claudeへの指示）
{
  echo "【Stop Guardian】タスク完了前に以下の問題を検知しました。"
  echo ""
  echo -e "$ISSUES"
  echo ""
  echo "対応方法: ユーザーに以下を報告し、どう対応するか確認してください（AskQuestionを使用）:"
  echo "  1. 検知された問題の内容"
  echo "  2. 対応案（修正して完了する / このまま完了する / スキップする）"
  if [ -n "$ESCALATION" ]; then
    echo ""
    echo "【エスカレーション推奨】以下の問題が繰り返されています:"
    echo -e "$ESCALATION"
    echo "ユーザーにハーネス強化の要否も併せて確認してください。"
  fi
  if [ -n "$HOUSEKEEPING" ]; then
    echo ""
    echo "$HOUSEKEEPING"
  fi
} >&2
exit 2
```

生成後、`chmod +x` を実行すること。

---

### 生成物2: harness-log.jsonl

空ファイルとして作成:

```bash
touch "$CLAUDE_PROJECT_DIR/.claude/harness-log.jsonl"
```

---

### 生成物3: rules/harness-evolution.md

```markdown
# ハーネス協働進化ルール

## Stop hookからフィードバックを受けたとき

Stop hookが問題を検知すると、stderrにフィードバックが返される。
以下の手順で**必ずユーザーに確認**すること:

1. 検知された問題を簡潔に報告する
2. AskQuestionで対応を確認する:
   - 「修正してから完了する」
   - 「このまま完了する（今回はスキップ）」
   - 「ハーネスを強化する（ルール追加 or hookチェック項目追加）」
3. ユーザーの選択に従って対応する

## エスカレーション提案を受けたとき

同じ問題パターンが繰り返されると、stop-guardian.shがエスカレーションを推奨する。
この場合も**ユーザーに確認**してから対応する:

- L2（2回目）→ 「rules/にルールとして明文化しますか？」
- L3（3回目以降）→ 「stop-guardian.shにチェック項目を追加して自動ブロックしますか？」

ユーザーが承認した場合のみ、対応するファイルを更新する。

## 棚卸し提案を受けたとき

違反ログが蓄積すると棚卸しが推奨される。以下をユーザーに確認する:

1. 不要になったrules/のルールがないか
2. 過剰なhookチェック項目がないか（誤検知が多い等）
3. 違反ログのリセット

全てユーザー確認後に実行する。

## 禁止事項

- ユーザー確認なしでrules/やhooks/を変更しない
- 「自動で直しておきました」としない。必ず報告→確認→対応の順序
```

---

### 生成物4: rules/active-task-lifecycle.md

stop-guardian.sh がパースする `## 状況` フィールドの管理ルール。

```markdown
# active-task.md ライフサイクルルール

## 状況フィールドの管理

active-task.md の見出しに `## 状況: {値}` をインラインで記載する（stop guardian がパースする）。

値は以下のいずれか:
- `進行中` — 作業中（デフォルト）
- `ユーザー確認待ち` — こちらの作業は完了、ユーザーのアクション待ち
- `完了` — 全タスク完了

例: `## 状況: ユーザー確認待ち`

## 完了報告の手順

1. 全 `- [ ]` チェックボックスを `- [x]` に更新する
2. `## 状況` を `完了` に更新する
3. その上で完了報告する

`- [ ]` が残っている状態で `## 状況` を `完了` にしてはならない。
逆に、全チェックが付いているのに `## 状況` が `進行中` のままにしてもいけない。

## サブタスク完了は通常報告

「Slides 5-8 完了」のようなサブタスク完了報告は自由。
`## 状況` を変える必要はない。対応する `- [ ]` だけチェックする。
```

---

### 生成物5: rules/quality-gates.md

**これは短期PJを含む全PJで生成する。**

```markdown
# 品質ゲートルール

成果物を作成・変更したら、完了報告前に対応する品質チェックを実行する。
チェック完了前の「完了しました」は禁止。

## 成果物 → 品質チェックの対応表

| 成果物 | 必須チェック | 呼び出し方 |
|--------|------------|-----------|
| コード変更 | コードレビュー + セキュリティ | /review |
| UI変更 | + UI/UXレビュー | /review（自動追加） |
| 文書・資料 | 校正 | /review または proofreader |
| スライド | 構成・内容レビュー | /review |
| リサーチ結果 | 出典・論理検証 | /review |
| 戦略・企画 | 多角的検証 | /review（devils-advocate等） |

## 例外

- 小規模タスク（typo修正、1行変更等）はチェック不要
- ユーザーが「レビュー不要」と明示した場合はスキップ
- 判断に迷ったらユーザーに確認（AskQuestion）
```

PJ種別に応じて、該当する行のみを含めること:
- 開発PJ → コード変更、UI変更
- ドキュメントPJ → 文書・資料
- データ分析PJ → リサーチ結果（分析結果として読み替え）
- 戦略・企画PJ → 戦略・企画
- リサーチPJ → リサーチ結果
- 複数種別 → 該当する全行

---

### 生成物5: settings.json へのStop hook追加

既存のsettings.jsonのhooksセクションに以下をマージする（既存hookを上書きしない）:

```json
{
  "Stop": [
    {
      "hooks": [{
        "type": "command",
        "command": "bash \"$CLAUDE_PROJECT_DIR/.claude/hooks/stop-guardian.sh\"",
        "timeout": 10
      }]
    }
  ]
}
```

---

### 生成物6: CLAUDE.md への追記

既存のCLAUDE.mdに以下の**2行のみ**を追記する:

```markdown
### ハーネス協働進化
Stop hookがタスク完了時にチェックを行い、問題があればユーザーに報告する。詳細は rules/harness-evolution.md 参照。
```

CLAUDE.mdの肥大化を防ぐため、これ以上の詳細は書かない。

---

### チェックリスト

- [ ] PJ規模が「中長期」であることを確認（短期ならquality-gates.mdのみ生成してスキップ）
- [ ] hooks/stop-guardian.sh を生成し chmod +x
- [ ] harness-log.jsonl を touch で作成
- [ ] rules/harness-evolution.md を生成
- [ ] rules/active-task-lifecycle.md を生成
- [ ] rules/quality-gates.md を生成（全PJ）
- [ ] settings.json にStop hookをマージ
- [ ] CLAUDE.md に2行のポインタを追記
