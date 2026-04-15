## ステップ10: 活動タイプに応じたエージェント・スキル活用ガイド

CLAUDE.md 内に、このPJで活用すべきグローバルエージェント・スキルの一覧を記載する。ステップ2で判定した活動タイプに基づき、該当するものだけを載せる。

### つくる（実装）
| 用途 | 使うもの | 呼び出し方 |
|------|---------|-----------|
| コードレビュー | code-reviewer, security-reviewer, performance-reviewer | `/review` |
| UIレビュー | ui-reviewer, ux-reviewer | `/review`（UI変更時自動追加） |
| デバッグ | debugger | Agent tool で起動 |
| E2Eテスト | e2e-tester | Agent tool で起動 |
| アクセシビリティ | a11y-checker | Agent tool で起動 |
| ブラウザテスト | webapp-testing スキル | 自動トリガー |

### つくる（文書）
| 用途 | 使うもの | 呼び出し方 |
|------|---------|-----------|
| 文書下書き | draft スキル | `/draft` |
| 校正・推敲 | proofreader | Agent tool で自動委譲 |
| PDF作成 | pdf スキル | `.pdf` を扱う指示で自動トリガー |
| Word文書 | docx スキル | `.docx` を扱う指示で自動トリガー |
| スライド作成 | slide-creator / pptx スキル | `/pptx` or Agent tool |
| レポート作成 | report-creator | Agent tool で起動 |

### 調べる
| 用途 | 使うもの | 呼び出し方 |
|------|---------|-----------|
| 深掘りリサーチ | researcher | `/research` |
| リサーチ→発表 | research-presenter | Agent tool で起動 |

### 考える（戦略）
| 用途 | 使うもの | 呼び出し方 |
|------|---------|-----------|
| 構造化分析 | framework-thinker | Agent tool で自動委譲 |
| 批判的検証 | devils-advocate | Agent tool で自動委譲 |
| 多角的検証 | stakeholder-simulator | Agent tool で自動委譲 |
| リサーチ | researcher | `/research` |

### 検証する
| 用途 | 使うもの | 呼び出し方 |
|------|---------|-----------|
| コードレビュー | code-reviewer, security-reviewer, performance-reviewer | `/review` |
| 文書校正 | proofreader | Agent tool で自動委譲 |
| UI/UXレビュー | ui-reviewer, ux-reviewer, a11y-checker | `/review` |

### 分析する
| 用途 | 使うもの | 呼び出し方 |
|------|---------|-----------|
| スプレッドシート | xlsx スキル | `.xlsx`/`.csv` を扱う指示で自動トリガー |
| リサーチ | researcher | `/research` |
| レポート作成 | report-creator | Agent tool で起動 |
