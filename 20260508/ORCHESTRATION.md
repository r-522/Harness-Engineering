# ORCHESTRATION.md - Subagent分割統治パターン詳細

**対象**: Subagent Orchestration, Multi-Agent Parallelization  
**フレームワーク**: Claude Code Split-and-Merge, Agent Teams  
**有効期間**: 2026-05-08 〜  
**行数**: 135行（制限: 150行以下）

---

## 1. Orchestration パターン選択

### パターン1: Split-and-Merge（推奨・小規模）

**用途**: 明確に分割可能なタスク（実装A + 実装B + テスト）  
**並列度**: 3 (default)  
**Coordinator**: Orchestrator Agent（コーディネーター）

```
Orchestrator
├─ Worker-1: 機能 A 実装
├─ Worker-2: 機能 B 実装  ← 並列実行
├─ Worker-3: テスト実行
└─ Merge: 結果統合
```

### パターン2: Agent Teams（推奨・中規模）

**用途**: チーム内で自律的に協調（Plan, Impl, Test が共有タスクリスト）  
**並列度**: 5-7  
**通信**: 共有 Task Store（`.claude/tasks.yaml`）

```
Team Lead
├─ Planner Agent    → tasks.yaml に計画書き込み
├─ Impl-1 Agent     → tasks.yaml 読み込み → タスク取得 → 実装
├─ Impl-2 Agent     → tasks.yaml 読み込み → タスク取得 → 実装
└─ Tester Agent     → tasks.yaml 読み込み → テスト検証
```

### パターン3: Hierarchical（大規模・非推奨）

**用途**: 多層エージェント（Meta Orchestrator → Tier1 → Tier2）  
**複雑度**: 高い、Context消費多、デバッグ困難  
**推奨**: Team パターンで代替

---

## 2. Split-and-Merge テンプレート

### ステップ1: タスク分析（Orchestrator）

```markdown
## 🎯 タスク分析

### ゴール
ユーザー認証機能を実装 → テスト → ドキュメント作成

### 制約
- Deadline: 2026-05-10
- API変更なし（既存 interface 維持）
- テストカバレッジ >= 80%

### 入力・出力
- 入力: User model, JWT secret
- 出力: auth_module.py, tests/, docs/

### リスク
- External API遅延（JWT validate）→ Mock使用
- Context枯渇 → 160k tokens予算
```

### ステップ2: タスク分割

```yaml
tasks:
  A: "認証ロジック実装"
    dependencies: []
    assigned_to: worker_1
    estimated_tokens: 40k
    
  B: "API統合実装"
    dependencies: [A]
    assigned_to: worker_1  # 同じAgent続行
    estimated_tokens: 30k
    
  C: "ユニットテスト実装"
    dependencies: [B]
    assigned_to: worker_2  # 並列可能な別作業
    estimated_tokens: 35k
    
  D: "E2E テスト"
    dependencies: [C]
    assigned_to: worker_2
    estimated_tokens: 25k
    
  E: "ドキュメント作成"
    dependencies: [A, B]  # A,B完了待ち
    assigned_to: worker_3
    estimated_tokens: 20k

# 実行順序 (DAG topological sort)
execution_order:
  - parallel: [A, D], [E]  # A→D並列、同時にE
  - sequential: [B] → [C]  # A完了後 B→C
```

### ステップ3: Subagent 定義

```python
# orchestrator.py

workers = [
    SubagentConfig(
        name="worker_1",
        role="implementation_expert",
        context_budget=60_000,  # Token上限
        skills=["python_coding", "api_integration", "git"],
        task_ids=["A", "B"],
        instructions="""
        You are implementation expert.
        Task A & B をこの順番で実装.
        Accept criteria を確認してから next task へ.
        """
    ),
    SubagentConfig(
        name="worker_2",
        role="test_expert",
        context_budget=50_000,
        skills=["pytest", "mocking", "coverage"],
        task_ids=["C", "D"],
        instructions="""
        You are test expert.
        Worker-1 の出力をテスト.
        カバレッジ >= 80% を確保.
        """
    ),
    SubagentConfig(
        name="worker_3",
        role="documentation_writer",
        context_budget=30_000,
        skills=["markdown", "api_docs"],
        task_ids=["E"],
        instructions="""
        You are technical writer.
        API仕様 & 使用例を明確に記述.
        """
    )
]
```

### ステップ4: 並列実行 & 結果統合

```python
async def orchestrate():
    # 実行
    results = await run_parallel_subagents(workers, tasks)
    
    # 検証
    for result in results:
        assert result["status"] == "success", f"{result['task_id']} failed"
        assert result["coverage"] >= 0.8, f"Coverage: {result['coverage']}"
    
    # 統合
    merged_code = merge_outputs(
        implementation=results["worker_1"]["code"],
        tests=results["worker_2"]["tests"],
        docs=results["worker_3"]["docs"]
    )
    
    # コミット
    git.commit(f"feat: authentication module ({len(merged_code)} lines)")
```

---

## 3. 依存関係管理

### 依存グラフの表現（YAML）

```yaml
version: "1.0"
tasks:
  task_setup:
    name: "環境準備"
    type: "setup"
    dependencies: []
    
  task_plan:
    name: "設計ドキュメント"
    type: "planning"
    dependencies: [task_setup]
    
  task_impl_a:
    name: "機能A実装"
    type: "implementation"
    dependencies: [task_plan]
    parallel_group: "group_1"
    
  task_impl_b:
    name: "機能B実装"
    type: "implementation"
    dependencies: [task_plan]
    parallel_group: "group_1"  # task_impl_a と同時実行可
    
  task_test:
    name: "統合テスト"
    type: "testing"
    dependencies: [task_impl_a, task_impl_b]  # 両方完了待ち
    
  task_deploy:
    name: "デプロイ"
    type: "deployment"
    dependencies: [task_test]

# 実行順序（topological sort）
execution_order:
  1. [task_setup]
  2. [task_plan]
  3. [task_impl_a, task_impl_b]  # 並列
  4. [task_test]
  5. [task_deploy]
```

### サイクル検出（重要）

```python
def validate_dag(tasks):
    # サイクル検出
    visited, rec_stack = {}, {}
    for task in tasks:
        if has_cycle(task, visited, rec_stack):
            raise CycleDependencyError(f"Cycle in: {task}")
    
    # Topological sort
    return topological_sort(tasks)

# エラー例
invalid_dag = {
    "A": depends_on=["B"],
    "B": depends_on=["C"],
    "C": depends_on=["A"]  # ← サイクル！
}
# → CycleDependencyError 発生
```

---

## 4. Context 予算管理

### 予算配分（200k tokens 総予算）

```
Orchestrator:
  system_prompt: 3k
  planning: 10k
  coordination: 7k
  → 20k 合計

Worker-1 (Impl):
  context: 60k
  code_editing: 40k
  testing: 20k
  → 60k 予算

Worker-2 (Test):
  context: 50k
  test_execution: 35k
  coverage_analysis: 15k
  → 50k 予算

Worker-3 (Docs):
  context: 30k
  markdown_writing: 25k
  review: 5k
  → 30k 予算

Reserve: 40k  # バッファ（予期外の追加作業用）
```

### 予算超過時の対応

```python
def monitor_context(agent_name, tokens_used, budget):
    usage_pct = (tokens_used / budget) * 100
    
    if usage_pct > 100:
        ESCALATE: f"{agent_name} exceeded budget by {usage_pct-100}%"
        ACTION: "Migrate to new session or delegate remaining tasks"
    
    elif usage_pct > 80:
        WARN: f"{agent_name} at {usage_pct}% capacity"
        ACTION: "Wrap up current task, prepare handoff"
    
    elif usage_pct > 60:
        LOG: f"{agent_name} at {usage_pct}% (safe zone)"
```

---

## 5. エラーハンドリング & リトライ戦略

### Subagent失敗時の対応

```
Status: FAILED (Worker-2 テスト失敗)

Step 1: 原因分析
  - Test output: AssertionError at test_auth.py:42
  - Coverage: 75% (Goal: 80%)
  
Step 2: 判断
  IF 新しいバグ: Reassign Worker-2 (同じWorkerが修正)
  IF Context枯渇: Abort + 再計画
  IF Tool失敗: Retry 3回
  
Step 3: 実行
  → Worker-2 に修正指示 + 再テスト
  
Step 4: フォローアップ
  Coverage 80% 達成 ✓
  → Next task へ進む
```

### Timeout & リソース制限

```yaml
subagent_config:
  timeout_minutes: 30
  max_retries: 3
  max_context_tokens: 60_000
  
  on_timeout: "escalate"
  on_retry_exhausted: "fail_fast"
  on_context_exceeded: "abort_and_replan"
```

---

## 6. Agent Teams パターン（中規模以上）

### 共有 Task Store (`tasks.yaml`)

```yaml
tasks:
  - id: "plan_001"
    status: "in_progress"
    assigned_to: "planner_agent"
    created_at: "2026-05-08T10:00:00Z"
    description: "高レベル設計書作成"
    
  - id: "impl_001"
    status: "pending"  # Planner完了待ち
    assigned_to: null  # 未割り当て
    blocking_tasks: ["plan_001"]
    estimated_complexity: "high"
    
  - id: "impl_002"
    status: "pending"
    assigned_to: null
    blocking_tasks: ["plan_001"]
    estimated_complexity: "medium"
    
  - id: "test_001"
    status: "pending"
    assigned_to: null
    blocking_tasks: ["impl_001", "impl_002"]
    estimated_complexity: "medium"
```

### Agent間コミュニケーション

```python
# Planner Agent
def plan_and_publish():
    plan = create_plan()
    store.update_task("plan_001", status="completed", output=plan)
    store.unblock_tasks(["impl_001", "impl_002"])

# Impl-1 Agent
def execute_when_ready():
    while not is_blocked("impl_001"):
        task = store.get_task("impl_001")
        implement(task)
        store.update_task("impl_001", status="completed")

# Tester Agent
def test_when_ready():
    while not is_blocked("test_001"):
        task = store.get_task("test_001")
        test(task)
        store.update_task("test_001", status="completed")
```

---

## 7. ベストプラクティス

### 🟢 Good（推奨）
- ✅ 依存関係を明示的に指定（DAG明記）
- ✅ 並列度を上限3に制限（Context効率）
- ✅ タスク分割は「実装A, 実装B, テスト」単位
- ✅ Subagent完了の明確な Acceptance Criteria

### 🔴 Bad（避けるべき）
- ❌ 自動並列化に期待（明示的指定必須）
- ❌ 10個以上の Subagent 起動（Context枯渇）
- ❌ 細粒度タスク（1Subagent=1関数）
- ❌ 非同期・非決定的な結果

---

**次**: `.claude/HARNESS.md` で System Prompt & Tool Design を確認してください。
