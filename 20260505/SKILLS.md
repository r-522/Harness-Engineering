# SKILLS.md - カスタムスキル定義 & トリガーマッピング

**対象**: Claude Code Skills + MCP統合  
**形式**: Skill YAML定義 + トリガーマッピング  
**最終更新**: 2026-05-05

---

## 1. スキル構造テンプレート

```yaml
skill_id: skill_name
name: "Human Readable Name"
description: "何ができるか（1行）"
category: research | execution | analysis | report
trigger_phrases:
  - "when I need to ..."
  - "analyze ..."
required_tools:
  - WebSearch
  - Read
  - Bash
dependencies:
  - other_skill_name
context_budget: 25000  # tokens
max_parallel_instances: 1  # or 3
steps:
  - name: "Step 1"
    description: "何をするか"
    tool: WebSearch
    command: "example: search for X"
  - name: "Step 2"
    description: "結果をどう使うか"
    tool: Read
    input: "output from Step 1"
success_criteria: "これが出力されたら成功"
fallback: "失敗時の代替案"
```

---

## 2. カテゴリ別スキル定義

### 【Category: Research】リサーチ型スキル

#### Skill #1: `web_research_parallel`

```yaml
skill_id: web_research_parallel
name: "並列Web検索 & ドキュメント分析"
description: "複数キーワードについてWeb検索し、結果を構造化レポート化"
category: research
trigger_phrases:
  - "compare the features of X, Y, Z"
  - "research latest trends in [topic]"
  - "find information about [query]"

required_tools:
  - WebSearch
  - WebFetch
  - Agent (Explore subagent)

context_budget: 25000
max_parallel_instances: 3

steps:
  - name: "キーワード分解"
    description: "複数のリサーチトピックに分割"
    tool: Agent (subagent_type=Explore)
    command: "spawn 3 researcher subagents for [queries]"
  
  - name: "並列Web検索"
    description: "各Subagentが独立してWeb検索"
    tool: WebSearch (× 3並列)
    input: "individual query per subagent"
  
  - name: "結果統合"
    description: "3つの検索結果をMarkdown表に整形"
    tool: (Python script or manual merge)
    output: "structured_comparison.md"

success_criteria: |
  - 3つの検索結果がすべて取得できた
  - 出力ファイルが存在し、表形式で整理されている
  - 各行が source URL を含む

fallback: |
  - 1つ以上の検索失敗時: 手動キーワード入力を要求
  - 全失敗時: 代替APIまたは手動調査を提案
```

**使用例**:
```markdown
/skill web_research_parallel
Topics: React 18.3, Vue 3.5, Svelte 4.2
Output: framework_comparison_2026.md
```

---

#### Skill #2: `code_pattern_discovery`

```yaml
skill_id: code_pattern_discovery
name: "コードパターン検出 & 事例抽出"
description: "大規模コードベースから特定パターンの出現箇所と事例を抽出"
category: research
trigger_phrases:
  - "find all instances of [pattern]"
  - "locate [architecture pattern] in the codebase"
  - "discover [anti-pattern]"

required_tools:
  - Bash (grep, find, awk)
  - Read
  - Agent (Explore subagent)

context_budget: 30000
max_parallel_instances: 1

steps:
  - name: "Explore subagent起動"
    description: "コードベース全体をfind / grep で検索"
    tool: Agent (subagent_type=Explore)
    command: |
      grep -r "[pattern]" src/ --include="*.py" --include="*.ts"
      find . -name "*.py" -type f -exec grep -l "[pattern]" {} \;
  
  - name: "結果フィルタリング"
    description: "偽陽性排除、重複削除"
    tool: Bash (awk, sort -u)
    input: "grep results"
  
  - name: "コンテキスト抽出"
    description: "各出現箇所について周辺コードを読み込み（5～10行）"
    tool: Read (複数ファイル)
    output: "pattern_analysis.md"

success_criteria: |
  - 最低3つの出現箇所が特定できた
  - 各出現箇所の周辺コンテキスト(5行)を含む
  - ファイルパス:行番号 形式で記載

fallback: |
  - パターンなし: "該当パターンは検出されませんでした"を報告
  - 少数のみ: 全ファイル Read (コンテキスト予算許す限り)
```

---

### 【Category: Execution】実行型スキル

#### Skill #3: `parallel_testing_suite`

```yaml
skill_id: parallel_testing_suite
name: "複数テストスイート並列実行"
description: "pytest / npm test を複数ディレクトリで並列実行、JUnit XMLで統合レポート"
category: execution
trigger_phrases:
  - "run all tests in parallel"
  - "execute unit tests for [modules]"
  - "test [features] simultaneously"

required_tools:
  - Bash (pytest, npm test)
  - Read (test results parsing)

context_budget: 25000
max_parallel_instances: 3

steps:
  - name: "テストスイート特定"
    description: "pytest.ini または package.json から実行対象を列挙"
    tool: Bash
    command: |
      pytest --collect-only -q 2>/dev/null | grep '::' | wc -l
      npm test -- --listTests 2>/dev/null | wc -l
  
  - name: "並列分散"
    description: "テストスイートを3グループに分割、Subagent割り当て"
    tool: Agent (spawn 3 executor subagents)
    input: "test suite group A/B/C"
  
  - name: "テスト実行"
    description: "各Subagentで独立テスト実行"
    tool: Bash
    command: "pytest tests/group_X/ -v --junit-xml=result_X.xml"
  
  - name: "結果統合"
    description: "3つのJUnit XML を統合、カバレッジレポート生成"
    tool: (Python script: junit merge + coverage combine)
    output: "test_report_combined.xml"

success_criteria: |
  - 全テストが実行完了
  - JUnit XML出力が存在
  - カバレッジ ≥ 80%

fallback: |
  - 1つのグループ失敗: 直列実行へ自動フォールバック
  - API制限: 待機時間を増やしリトライ
  - コンテキスト枯渇: ログ出力削減
```

---

#### Skill #4: `code_generation_templates`

```yaml
skill_id: code_generation_templates
name: "テンプレート駆動コード生成"
description: "プロジェクト規約に従い、API エンドポイント / コンポーネント / スキーマを生成"
category: execution
trigger_phrases:
  - "create a new API endpoint for [resource]"
  - "generate a React component for [feature]"
  - "scaffold a database model for [entity]"

required_tools:
  - Read (既存コードパターン参照)
  - Write (新規ファイル生成)
  - Bash (ファイル配置、Linting)

context_budget: 20000
max_parallel_instances: 1

steps:
  - name: "パターン抽出"
    description: "既存コード（3～5ファイル）から命名規則・構造を学習"
    tool: Read
    input: "existing_endpoint.py, existing_component.tsx"
  
  - name: "テンプレート適用"
    description: "抽出パターンを新リソースに適用"
    tool: (Template engine: Jinja2 or similar)
    input: "resource_name, field_definitions"
  
  - name: "ファイル生成"
    description: "新規ファイルをディレクトリに配置"
    tool: Write
    output: "generated_file.py / .tsx"
  
  - name: "品質チェック"
    description: "Linting、型チェック、テスト実行"
    tool: Bash (ruff, eslint, pytest)
    command: "ruff check generated_file.py && pytest tests/test_generated.py"

success_criteria: |
  - ファイルが生成されている
  - Linting: 0 errors
  - 型チェック: pass
  - テスト: pass

fallback: |
  - テンプレート不適用: インタラクティブ入力で手動ガイド
  - 既存パターン不足: テンプレートライブラリから標準を使用
```

---

### 【Category: Analysis】分析型スキル

#### Skill #5: `code_quality_audit`

```yaml
skill_id: code_quality_audit
name: "コード品質監査 & メトリクス"
description: "複数セクションのコード品質をスコア化、改善レコメンデーション"
category: analysis
trigger_phrases:
  - "audit code quality for [files/modules]"
  - "generate code quality report"
  - "identify code smells in [area]"

required_tools:
  - Bash (ruff, pylint, eslint, complexity metrics)
  - Read (各ファイル読み込み)
  - Agent (analyzer subagent)

context_budget: 28000
max_parallel_instances: 3

steps:
  - name: "分析対象分割"
    description: "複数モジュールを3グループに分割"
    tool: Bash (find)
    command: "find src/ -name '*.py' -type f | split -n 3"
  
  - name: "メトリクス計測"
    description: "各グループについてLinting・複雑度を並列実行"
    tool: Agent (spawn 3 analyzer subagents)
    input: "module_group A/B/C"
  
  - name: "スコアリング"
    description: "各指標を正規化、統合スコア算出"
    tool: (Python: weighted scoring)
    metrics:
      - cyclomatic_complexity (target: ≤ 10)
      - code_duplication (target: ≤ 10%)
      - test_coverage (target: ≥ 80%)
      - type_hints_ratio (target: ≥ 90%)
  
  - name: "レポート生成"
    description: "スコア、グラフ、改善提案をMarkdownレポート化"
    tool: (Template + matplotlib)
    output: "code_quality_audit.md + visualization.png"

success_criteria: |
  - 全モジュールがスコア化されている
  - スコア範囲: 0～100
  - 改善提案 ≥ 5項目

fallback: |
  - メトリクス計測失敗: スキップして部分結果のみ報告
  - グラフ生成失敗: テキスト表のみで代替
```

---

### 【Category: Report】レポート型スキル

#### Skill #6: `markdown_report_generation`

```yaml
skill_id: markdown_report_generation
name: "マークダウン構造化レポート生成"
description: "複数の分析結果・データを1つの構造化Markdownレポートに統合"
category: report
trigger_phrases:
  - "create a report from [data sources]"
  - "compile findings into [format]"
  - "generate executive summary"

required_tools:
  - Read (各分析結果ファイル読み込み)
  - Write (レポートファイル生成)

context_budget: 15000
max_parallel_instances: 1

steps:
  - name: "入力収集"
    description: "複数のソースファイル（.json, .md, .csv）を読み込み"
    tool: Read (複数ファイル)
    input: "results_A.json, results_B.md, results_C.csv"
  
  - name: "構造化"
    description: |
      要件に従いセクション化:
      - Executive Summary
      - Key Findings
      - Detailed Analysis
      - Metrics & Visualizations
      - Recommendations
      - Appendix
    tool: (Template engine)
  
  - name: "トーン統一"
    description: "フォーマット、専門用語を統一（CLAUDE.md 準拠）"
    tool: (Manual or LLM polish)
  
  - name: "品質チェック"
    description: "リンク検証、表形式確認、画像パス確認"
    tool: Bash (markdown-link-check, pandoc)

success_criteria: |
  - ファイルが生成されている
  - 全セクションが存在
  - 外部リンク: all valid
  - 表組み: valid Markdown

fallback: |
  - リンク検証失敗: URL リストのみ出力
  - セクション不足: テンプレートの最小版使用
```

---

## 3. トリガーマッピング（推奨シーン）

| トリガー文 | 推奨スキル | 並列度 | 理由 |
|---|---|---|---|
| "compare X, Y, Z products" | `web_research_parallel` | 3 | 独立検索可能 |
| "find all [pattern] in code" | `code_pattern_discovery` | 1 | 順序依存 |
| "run all tests" | `parallel_testing_suite` | 3 | テストグループ並列化 |
| "create API for [resource]" | `code_generation_templates` | 1 | 順序依存 |
| "audit code quality" | `code_quality_audit` | 3 | モジュール分割可能 |
| "compile findings" | `markdown_report_generation` | 1 | 統合作業 |

---

## 4. スキル実行ベストプラクティス

### ✅ 推奨

```markdown
# Skill実行例 (推奨)
/skill web_research_parallel
Input: 
  Topics: "Claude Code 2026", "Subagent Orchestration", "MCP Integration"
Output:
  - research_comparison_table.md
  - source_urls.txt
```

### ❌ 非推奨（スキル外）

```markdown
# 直接Tool実行（スキル不要なケース）
- 単一ファイル読み込み → Read tool直接
- 単一コマンド実行 → Bash tool直接
- 簡単な質問応答 → LLM直接
```

---

## 5. カスタムスキル追加チェックリスト

新規スキルを定義する際:

- [ ] **トリガー文**を最低3つ定義（ユーザ発話パターン）
- [ ] **必要Tool**を明記（Read, Bash, WebSearch, Agent等）
- [ ] **Context予算**を見積もり（step毎の トークン考慮）
- [ ] **最大並列数**を設定（1～3推奨）
- [ ] **Success criteria**をテスト可能な形で定義
- [ ] **Fallback**を明記（失敗時の代替案）
- [ ] チーム内でレビュー・同意

---

## 6. スキルライフサイクル

```
定義 (SKILLS.md)
  ↓
実装 (src/skills/ に Python/TS)
  ↓
ローカルテスト (テストシナリオで検証)
  ↓
チームレビュー
  ↓
本番環境へ昇格 (.claude/ に統合)
  ↓
監視 (使用頻度、成功率)
  ↓
改善 (フィードバック反映)
```

---

## 参考

- **Skill Implementation Guide**: https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf
- **MCP Integration**: `MCP-INTEGRATION.md`
- **Agent Orchestration**: `AGENT.md`
