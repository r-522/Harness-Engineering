# CONTEXT_OPTIMIZATION.md - Context Buffer 最適化（33K Token対応）

2026年4月 Claude Code アップデートで、自動コンパクト バッファが 45K → 33K トークンに削減されました。このファイルは、この制限下での効率的なコンテキスト管理戦略を記述します。

**最終更新**: 2026-04-25  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

---

## Context Buffer の仕組み

### バッファ構成（33K Token）

```
┌────────────────────────────────────┐
│ Total Context: 33,000 tokens       │
├────────────────────────────────────┤
│ System Prompt:            2,000    │ (固定)
│ Tool Definitions:         3,000    │ (固定)
│ Project Context/CLAUDE.md: 5,000   │ (固定)
│ Recent Conversation:     15,000    │ (動的)
│ Sub-Agent Results:        5,000    │ (動的)
│ Available for New Query:  3,000    │ (バッファ)
├────────────────────────────────────┤
│ TOTAL:                   33,000    │
└────────────────────────────────────┘
```

### 80% 警告ポイント

```
33,000 * 0.80 = 26,400 tokens

この閾値に達したら、自動的に `/compact` を検討すべき。
```

### 自動コンパクト仕組み

Claude Code は以下の条件で自動的にコンテキスト圧縮を開始：

```yaml
autocompact_trigger:
  buffer_usage: ">= 80%"   # 26,400 tokens
  or:
    conversation_turns: "> 50"
    or:
    old_context_age: "> 60分"
  
  autocompact_action:
    - summarize_old_turns: "最初の10ターンを要約に変換"
    - remove_duplicates: "重複情報削除"
    - compress_code_snippets: "長いコードをファイル参照に"
    - result: "30-40% 削減"
```

---

## Context 最適化テクニック

### Technique 1: Sub-Agent による隔離

**重要**: これが最も効果的な最適化手法。

#### 実装パターン

```yaml
dispatcher_agent:
  role: "親エージェント"
  context_size: 25K tokens
  
  sub_agents:
    - id: "analyzer_1"
      context_size: "独立（30K トークン）"
      task: "コード品質分析"
      output: "500 token サマリー"  ← 重要
      
    - id: "evaluator_1"
      context_size: "独立（30K トークン）"
      task: "セキュリティ検査"
      output: "500 token レポート"  ← 重要
  
  result_synthesis:
    role: "Dispatcher が Sub-Agent の結果のみを参照"
    pattern: "最終結果 = サマリー1 + サマリー2 + ... を統合"
```

#### 利点

```
従来（全て同一エージェント）:
  Dispatcher → サブタスク1（3K tokens）
             → サブタスク2（3K tokens）
             → サブタスク3（3K tokens）
  = 合計 9K tokens 消費

Sub-Agent 分離:
  Dispatcher → Sub-Agent1（独立context）→ サマリー（500 tokens）
             → Sub-Agent2（独立context）→ サマリー（500 tokens）
             → Sub-Agent3（独立context）→ サマリー（500 tokens）
  = 合計 1.5K tokens 消費（83%削減）
```

#### Sub-Agent サマリー テンプレート

```markdown
## [Task] Analysis Summary

**Findings**: [3-5 bullet points]

**Severity**: CRITICAL | HIGH | MEDIUM | LOW

**Recommendation**: [Specific action]

**Details**: [Link to full report if large]
```

### Technique 2: プロンプト キャッシング（1時間 TTL）

2026年4月更新で導入された最新機能。

#### 設定方法

```yaml
enable_prompt_caching_1h:
  setting: "enable"
  ttl_seconds: 3600  # 1 hour
  
  automatically_cached:
    - system_prompt: true
    - project_context: true        # CLAUDE.md等
    - tool_definitions: true
    - safety_guidelines: true
    - frequently_referenced_docs: true
```

#### キャッシュ活用戦略

```
Query 1:
  "What are the project rules?"
  → キャッシュミス（初回）
  → System Prompt + CLAUDE.md をキャッシュに保存

Query 2 (1時間以内):
  "How do I add a new skill?"
  → キャッシュヒット
  → 同じ System Prompt / CLAUDE.md を再利用
  → 入力トークン 60% 削減 ✓

Query 3 (1時間後):
  "What are the project rules?"
  → キャッシュ期限切れ（miss）
  → 再度保存
```

#### Cache Hit を最大化するコツ

```yaml
optimization_tips:
  1_stable_sections_first:
    pattern: |
      # System Context
      [変わらない部分]
      
      # Dynamic Query
      [毎回変わる部分]
    
    reason: "キャッシュエンジンが最初のセクションをロック"
  
  2_batch_similar_queries:
    pattern: |
      Query A:
      Query B:
      Query C:
      (同じコンテキストで3つ実行)
    
    reason: "キャッシュ再利用により各クエリの効率が上昇"
  
  3_reuse_project_context:
    pattern: |
      別のセッション → 同じプロジェクトコンテキストをロード
      → キャッシュ活用
    
    reason: "1時間以内なら同じプロジェクト情報がキャッシュ済み"
```

#### コスト削減効果

```
キャッシング未使用:
  入力: 100万トークン
  コスト: $3.00

キャッシング 50% Hit Rate:
  入力キャッシュ無し: 50万トークン × $3.00 = $1.50
  入力キャッシュ有り: 50万トークン × $0.30 = $0.15
  コスト: $1.65
  削減: 45%
```

### Technique 3: `/compact` コマンド

手動でコンテキストを圧縮。

#### 使用タイミング

```yaml
use_compact_when:
  - buffer_usage: ">= 80%"       # 26,400 tokens
  - conversation_turns: "> 50"   # 老いた対話が蓄積
  - agent_switch: "次エージェント切り替え前"
  - context_refresh_needed: true
```

#### 実行例

```bash
# コマンド実行
/compact

# 出力
Compacting context...
- Summarized turns 1-20: 2,000 tokens → 200 tokens saved
- Removed duplicate explanations: 500 tokens saved
- Compressed code snippets: 300 tokens saved
- Result: 3,000 tokens freed (9% total reduction)

New buffer usage: 24,000 / 33,000 (73%)
```

#### `/compact` の詳細な効果

```yaml
compact_operations:
  
  summarize_old_turns:
    operation: "最初の N ターンを1行サマリーに変換"
    example:
      before: |
        User: "プロジェクト規約は？"
        Claude: "CLAUDE.md には以下が書いてあります..."
        (500 tokens)
      
      after: |
        [Summarized] User inquired about project rules. Claude explained CLAUDE.md contents.
        (50 tokens)
    
    token_saved: "450 tokens"
  
  remove_duplicates:
    operation: "同一情報の重複記述を削除"
    example:
      before: |
        "Generator は提案生成します。"
        ...
        "Generator は提案を生成します。"
      
      after: |
        "Generator は提案を生成します。" (重複は削除)
    
    token_saved: "varies"
  
  move_code_to_files:
    operation: "長いコード片をファイル参照に変換"
    example:
      before: |
        ```python
        def very_long_function():
          ... (100 lines)
        ```
        (500 tokens)
      
      after: |
        See implementation in `src/utils/helper.py`
        (30 tokens)
    
    token_saved: "470 tokens"
```

### Technique 4: ファイル参照（Absolute Path）

大容量コンテキストはファイルとして管理。

#### パターン

```yaml
heavy_context:
  - project_specification: "spec.md" (2000 lines)
    → Include: "See `/path/to/spec.md`"
    → Token saving: 95%
  
  - large_codebase_snippet: "src/main.py" (500 lines)
    → Include: "See line 150-200 in `/path/to/src/main.py`"
    → Token saving: 90%
  
  - historical_logs: "logs/april.txt" (10MB)
    → Include: "See archived logs at `/var/logs/april.txt`"
    → Token saving: 99%
```

#### 実装例

```markdown
## Context Reference

For detailed project rules, see:
- `/home/project/CLAUDE.md` - coding standards
- `/home/project/AGENT.md` - agent definitions
- `/home/project/DESIGN.md` - architecture guide

For code review, examine:
- `/src/agent_orchestrator.py` (lines 1-50: Orchestrator class)
- `/src/generator_agent.py` (lines 100-200: Generator logic)

For historical context:
- See `/var/logs/2026-04/summary.txt` for April execution logs
```

---

## Sub-Agent Context 制御戦略

### Parent-Child Context Hierarchy

```
Parent Agent (25K tokens)
├─ System Prompt (cached): 2K
├─ Project Context (cached): 5K
├─ Recent Conversation: 10K
├─ Sub-Agent Results: 5K
└─ Available Buffer: 3K
    
    └─ Sub-Agent A (30K tokens)
       ├─ Task Description: 2K
       ├─ Required Context: 5K
       ├─ Processing: 20K
       └─ Output → 500 tokens サマリー (Parent に返す)
    
    └─ Sub-Agent B (30K tokens)
       └─ [Similar structure]
```

### Sub-Agent 実行テンプレート

```yaml
sub_agent_execution:
  parent_instruction: |
    # Parent から Sub-Agent A への指示
    以下の分析を実行し、500トークン以下のサマリーを返してください。
    
    ## Task
    [詳細なタスク説明]
    
    ## Context
    [必要な背景情報]
    
    ## Expected Output
    ### Finding 1
    - [1文以下]
    - Severity: [HIGH|MEDIUM|LOW]
    
    ### Finding 2
    - [1文以下]
    - Severity: [HIGH|MEDIUM|LOW]
    
    ### Recommendation
    [最重要な推奨アクション 1行]
  
  sub_agent_internal_processing:
    context_limit: "30K tokens"
    processing: "充分な文脈で詳細分析可能"
    output_format: "厳格に500トークン以下のサマリー"
    
  parent_receives:
    input: "サマリー（500 tokens）"
    action: "複数Sub-Agentのサマリーを統合"
    efficiency_gain: "83% context 削減"
```

---

## Context 経済学

### トークンコスト計算

```
Model: Claude Opus 4.7

Input Token Cost:  $0.003 / 1,000 tokens
Output Token Cost: $0.015 / 1,000 tokens

Example Task:
  Generator: 2,000 input + 1,000 output
    Cost: (2K * $0.003) + (1K * $0.015) = $0.006 + $0.015 = $0.021
  
  Sub-Agent Optimization:
    Dispatcher: 15K input (cached to 9K) + 500 output
    Sub-Agent1: 20K input + 500 output (서로 독립)
    Sub-Agent2: 20K input + 500 output (독립)
    
    Cost with Caching:
    - Dispatcher: (9K * $0.0009) + (500 * $0.015) = $0.0081 + $0.0075 = $0.0156
    - Sub-Agent1: (20K * $0.003) + (500 * $0.015) = $0.06 + $0.0075 = $0.0675
    - Sub-Agent2: (20K * $0.003) + (500 * $0.015) = $0.06 + $0.0075 = $0.0675
    Total: $0.1506
    
    Savings vs. Non-Cached: 25%
```

### Monthly Budget Planning

```yaml
monthly_budget:
  target: "$10,000"
  
  cost_breakdown:
    generator_tasks: 500 tasks × $0.06 = $30
    evaluator_tasks: 500 tasks × $0.15 = $75
    sub_agent_tasks: 1000 tasks × $0.05 = $50
    orchestration: 500 tasks × $0.02 = $10
    
    total: "$165 / month"  (予算内)
  
  optimization_targets:
    cache_hit_rate: "> 50%"        → 20% コスト削減
    sub_agent_adoption: "> 70%"    → 25% コスト削減
    batch_processing: "> 60%"      → 15% コスト削減
    
    projected_savings: "60% reduction"
    optimized_cost: "$66 / month"
```

---

## トラブルシューティング

### Issue: Context Buffer 枯渇（>90%）

**症状**: Claude が "context limit exceeded" と返答

**原因**: Sub-Agent 出力が大きすぎる

**対策**:
```yaml
fix:
  1_reduce_sub_agent_output:
    action: "Sub-Agent に出力サイズ制限を追加"
    example: "Return top 5 findings only, not all 20"
  
  2_use_file_references:
    action: "大容量結果をファイルに保存"
    reference: "Full report in `/tmp/analysis_report.txt`"
  
  3_enable_auto_compact:
    action: "/compact コマンド実行"
```

### Issue: Cache Hit が発生しない

**症状**: プロンプトキャッシング有効でも コスト削減なし

**原因**: キャッシュキー（System Prompt等）が毎回変わっている

**対策**:
```yaml
fix:
  1_stable_system_prompt:
    problem: "各リクエストで System Prompt が変わる"
    solution: "System Prompt を固定セクションに移動"
    
  2_group_queries:
    problem: "異なるプロジェクト コンテキストを混在"
    solution: "同じプロジェクトのクエリをバッチ処理"
    
  3_verify_caching:
    command: "/config | grep enable_prompt_caching"
    action: "設定確認 & 有効化"
```

### Issue: Sub-Agent が親の文脈に依存している

**症状**: Sub-Agent が親の詳細コンテキストを要求

**原因**: Sub-Agent 設計が不適切

**対策**:
```yaml
fix:
  1_self_contained_instruction:
    bad: "プロジェクト規約に基づいて..."
    good: "以下の規約に基づいて: [規約全文を埋め込み]"
  
  2_explicit_output_format:
    action: "Sub-Agent の出力形式を厳格に定義"
    example: "Return EXACTLY 5 bullet points, 100 tokens max"
```

---

## ベストプラクティス チェックリスト

```yaml
context_optimization_checklist:
  - [ ] Sub-Agent による隔離を採用 (context 83% 削減)
  - [ ] プロンプトキャッシング有効化 (コスト 45-50% 削減)
  - [ ] `/compact` スケジュール設定 (毎50ターンごと)
  - [ ] ファイル参照戦略を実装 (大容量コンテキスト)
  - [ ] Sub-Agent サマリー形式を統一（500 tokens 以下）
  - [ ] 月次コスト追跡ダッシュボード構築
  - [ ] キャッシュ Hit Rate 監視（目標: > 50%）
  - [ ] エージェント別トークン使用量ログ開始
```

---

## 参考資料

- [Claude Code Context Buffer: The 33K-45K Token Problem](https://claudefa.st/blog/guide/mechanics/context-buffer-management)
- [Master Claude Code in 2026: Using Limits & Context Walls](https://medium.com/@rameshkannanyt0078/master-claude-code-in-2026)
- [Claude Code Context Discipline: CLAUDE.md, Memory, MCPs, Subagents](https://techtaek.com/claude-code-context-discipline-memory-mcp-subagents-2026/)
- [Using Claude Code Quota More Efficiently: Models, Context, Caching, /compact](https://www.knightli.com/en/2026/04/19/claude-code-usage-context-compact-notes/)

---

**最終更新**: 2026-04-25  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

