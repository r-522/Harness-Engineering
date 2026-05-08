# HARNESS.md - Agent Harness Engineering詳細設定

**対象**: Harness Engineering, System Prompt Design, Tool Architecture  
**フレームワーク**: Claude Code + Anthropic SDK (Opus 4.7)  
**有効期間**: 2026-05-08 〜  
**行数**: 115行（制限: 120行以下）

---

## 1. Harness構成図

```
┌─────────────────────────────────────────┐
│         Harness (CI: Agent Engine)      │
├─────────────────────────────────────────┤
│                                         │
│  ┌──────────────┐  ┌────────────────┐  │
│  │ System Prompt│  │  Tool Registry │  │
│  │  (60行以下)  │  │  (定義と権限)  │  │
│  └──────────────┘  └────────────────┘  │
│                                         │
│  ┌──────────────┐  ┌────────────────┐  │
│  │  Context     │  │  Execution     │  │
│  │  Management  │  │  Engine        │  │
│  └──────────────┘  └────────────────┘  │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │    Feedback Loops & Sensors      │  │
│  │  (テスト結果, エラー, 指標)       │  │
│  └──────────────────────────────────┘  │
│                                         │
└─────────────────────────────────────────┘
         ↓ (Model - typically Claude)
    Claude API (Opus 4.7)
```

**重要**: Agent = Model + Harness  
Harness は Model に仕事をさせるための全インフラ（制約、ツール、反省能力）

---

## 2. System Prompt 設計（60行以下）

### テンプレート（推奨）

```
# System Prompt - [Project Name]

You are an expert [role]: [specific domain expertise].

## Your Mission
[3-4文のゴール記述]

## Critical Constraints (Non-negotiable)
1. [Code quality] Follow CLAUDE.md strictly. Circular complexity <= 10.
2. [Testing] Ensure test coverage >= 80% before submitting.
3. [Context] Track token usage. If > 120k, report immediately.
4. [Safety] Never execute: git push --force, rm -rf, disable security checks.

## Execution Steps (Plan → Act → Verify)
1. Plan: Analyze, design, split into subtasks
2. Act: Implement, test locally, lint
3. Verify: Run full test suite, coverage check, final review

## Your Tools
[5-8個のTool名と簡潔な説明]

## Feedback Loops
- Test failures: Self-correct and retry (max 3 times)
- Context overrun: Report to parent immediately
- Ambiguity: Escalate to staff reviewer (never guess)

## End-of-turn Checklist
- [ ] Goal achieved?
- [ ] Tests passing (coverage >= 80%)?
- [ ] No CLAUDE.md violations?
- [ ] Context usage at X%?
- [ ] Next step clear?
```

### 文字数管理（50-60行が最適）

```
❌ Too verbose (90+ lines):
"This is a comprehensive system prompt that explains..."
"In the context of modern software development..."
→ Model が重要部分を見落とし、トークン浪費

✅ Concise (55 lines):
"You are a planning expert. Break tasks into subtasks."
"Critical: Follow CLAUDE.md, test coverage >= 80%"
"Tools: [list]"
→ Model が明確に理解、高効率実行
```

---

## 3. Tool Design & Registry

### Tool 定義スキーマ

```python
@dataclass
class ToolDefinition:
    name: str                           # "run_tests"
    description: str                    # 1-2文
    input_schema: dict                  # JSONSchema
    required_permissions: List[str]     # ["execute_code"]
    timeout_seconds: int                # 300
    max_retries: int                    # 3
    output_schema: dict                 # 期待される出力形式
    
# 例
run_tests = ToolDefinition(
    name="run_tests",
    description="Execute pytest and report coverage",
    input_schema={
        "type": "object",
        "properties": {
            "test_path": {"type": "string"},
            "coverage_target": {"type": "number"}
        }
    },
    required_permissions=["execute_code", "read_files"],
    timeout_seconds=300,
    output_schema={
        "type": "object",
        "properties": {
            "passed": {"type": "integer"},
            "failed": {"type": "integer"},
            "coverage": {"type": "number"}
        }
    }
)
```

### Tool Registry（権限管理）

```yaml
tools:
  read_file:
    category: "safe"
    permission: "read_files"
    description: "Read file contents"
    
  write_file:
    category: "cautious"
    permission: "write_files"
    description: "Create/modify files"
    
  run_tests:
    category: "safe"
    permission: "execute_code"
    description: "Execute pytest"
    
  git_push:
    category: "restricted"
    permission: "git_write"
    description: "git push (never --force)"
    blocked_flags: ["--force", "-f"]
    
  git_reset_hard:
    category: "dangerous"
    permission: "none"  # ユーザー確認必須
    description: "Destructive operation"
    requires_user_approval: true
```

---

## 4. Context Engineering & Token Management

### フェーズ別Context配置

```
Session開始時
├─ System Prompt: 3k tokens
├─ CLAUDE.md: 4k tokens
├─ AGENT.md: 3k tokens
└─ Task brief: 2k tokens
  = 12k tokens initial

実行中
├─ Code editing: 40k tokens
├─ Test results: 10k tokens
├─ Error logs: 5k tokens
└─ Feedback: 3k tokens
  = 58k tokens active

Archive / Delete（Context cut）
├─ Completed tasks: Remove detailed output
├─ Old logs: Keep only summary
└─ Verbose comments: Delete, keep code only

Result: Max 80k tokens (40% of 200k window)
```

### 圧縮戦略（Context rot対策）

```python
def compress_history():
    """
    200k tokens消費後は history を圧縮するか
    新規 Session へ移行するか判定
    """
    
    if token_usage > 300_000:
        # Context rot が発生
        ACTION: "Create new session"
        OUTPUT: "Summary of progress + next tasks"
        return
    
    if token_usage > 150_000:
        # 圧縮可能なら compress
        compress_by_removing:
            - Intermediate test output (keep only: PASS/FAIL)
            - Verbose debug logs (keep only: key errors)
            - Completed task details (keep only: 1-line summary)
        return

# 例: 1000行のテスト出力
before: "FAILED test_auth.py::test_login (AssertionError at line 42...)"  # 2k tokens
after: "FAILED: test_login (coverage 75%)"  # 10 tokens
ratio: 200倍削減
```

---

## 5. Feedback Loops & Sensors（自己修正能力）

### フィードバック構造（Feedforward + Feedback）

```
Feedforward Control (予防的)
  ↓
┌──────────────────────────────────┐
│ System Prompt → "Check CLAUDE.md" │  ← 行動前に予防
│ Tool Validation → "Lint before"   │
└──────────────────────────────────┘
  ↓
Agent Action (実装・テスト)
  ↓
Feedback Control (修正的)
┌──────────────────────────────────┐
│ Test Results → PASS/FAIL          │  ← 行動後に検証
│ Coverage Check → 80% 達成？        │
│ Code Review → SOLID準拠？          │
└──────────────────────────────────┘
  ↓
Agent Self-Correction (再実装)
```

### Sensor 実装例

```python
class Agent:
    def execute_with_feedback(self):
        while not goal_achieved:
            # 1. Feedforward: 実装前チェック
            self.validate_against_claude_md()
            
            # 2. Act: 実装
            result = self.implement()
            
            # 3. Feedback: 結果検証
            feedback = self.run_test_suite()
            
            if feedback["passed"]:
                return result
            else:
                self.log_error(feedback)
                self.adjust_and_retry()
```

---

## 6. Tool Execution & Error Handling

### Tool実行フロー

```
┌─────────────────────┐
│ Tool Request        │
│ (例: run_tests)     │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Permission Check    │  ← Security Gate
│ (実行権限 有無？)    │
└──────────┬──────────┘
           │ Yes
           ↓
┌─────────────────────┐
│ Execute Tool        │  ← Sandboxed
│ (timeout: 300s)     │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Validate Output     │  ← Sanity Check
│ (schema match?)     │
└──────────┬──────────┘
           │ Match
           ↓
┌─────────────────────┐
│ Return Result       │  ← Feedback to Agent
│ (JSON)              │
└─────────────────────┘
```

### エラーリカバリ

```python
def execute_tool_with_retry(tool, args, max_retries=3):
    for attempt in range(max_retries):
        try:
            result = tool.execute(args)
            validate_output(result)
            return result
        except ToolError as e:
            if attempt < max_retries - 1:
                log.warn(f"Retry {attempt+1}: {e}")
                time.sleep(2 ** attempt)
            else:
                escalate_to_human(
                    issue=e,
                    agent_context=self.context_snapshot,
                    recommendation="Manual review needed"
                )
                raise
```

---

## 7. Orchestrator 設計

### Orchestrator Job

```
Input: タスク定義 (複数 Subagent への分割)

Phase 1: Plan
  → 依存グラフ生成
  → リソース配分（Token予算）
  → リスク評価

Phase 2: Dispatch
  → Subagent にタスク割り当て
  → 並列実行開始

Phase 3: Monitor
  → Context使用率監視
  → Failure検出
  → エスカレーション判定

Phase 4: Merge
  → 全 Subagent 完了待ち
  → 出力統合
  → 最終検証 (tests, coverage, docs)
```

### Orchestrator Tool Set

```yaml
tools:
  - analyze_task
    input: task_description
    output: split_plan + dependency_graph
    
  - split_and_assign
    input: split_plan
    output: subagent_configs + task_assignments
    
  - monitor_execution
    input: subagent_status_list
    output: progress_report + escalation_alerts
    
  - merge_results
    input: all_subagent_outputs
    output: integrated_deliverable + quality_report
```

---

## 8. Production Deployment チェックリスト

before deploying to main:

```markdown
## Pre-Deployment Harness Verification

- [ ] System Prompt: 60行以下、明確な指示
- [ ] Tools: 全て registry に登録、権限明確
- [ ] Test Suite: coverage >= 80%, all PASS
- [ ] CLAUDE.md: チーム合意済み
- [ ] Context Budget: 各 Subagent に予算割り当て
- [ ] Orchestrator DAG: サイクル検出 OK
- [ ] Error Handling: Level 1-3 対応ロジック実装
- [ ] Security Review: OWASP top10 対応
- [ ] Documentation: README + API docs完備

Status: [PASS/FAIL] → Deploy / Abort
```

---

**最後に**: このHARNESS.md と `ORCHESTRATION.md` を組み合わせて、初回 Plan セッションを実施してください。
