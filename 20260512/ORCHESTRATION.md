# ORCHESTRATION.md - Agent Teams & Subagent 並列実行ガイド

**対象**: Agent Teams構成、Subagent並列管理、依存関係解決  
**作成日**: 2026-05-12  
**基準**: Claude Code Agent Teams（2026年ベストプラクティス）

---

## 1. Agent Teams の概念

### 基本構造

**Agent Teams** = 1 Coordinator + 3-4 Subagents（並列実行）

```
User Request
    ↓
[Coordinator Agent]
  - Plan mode で全体を先読み
  - Subagent への タスク分配
  - 結果統合・コンフリクト解決
    ↓
[Parallel Subagents（max 3）]
  ├─ Researcher: 技術調査（256K context）
  ├─ Implementer: コード実装（256K context）
  └─ Reviewer: コード審査（256K context）
    ↓
[Coordinator]
  - 結果統合
  - ユーザーへ報告
```

### なぜ3-5名か？（ベストプラクティス）

| 並列数 | メリット | デメリット | 推奨度 |
|---|---|---|---|
| 1（単独） | シンプル、Context余裕 | 遅い、視点が限定 | ⭐ |
| 2-3 | バランス型、速い | Context 圧迫少 | ⭐⭐⭐ |
| 4-5 | 高速、視点多彩 | Context 制約厳 | ⭐⭐ |
| 6+ | 並列度MAX | Context 枯渇リスク | ❌ |

**推奨: 3並列** (研究 + 実装 + レビュー)

---

## 2. Agent Teams 初期化（実装例）

### 2.1 Python SDK 例（claude-agent-sdk）

```python
# src/orchestration/team_setup.py
from anthropic import Anthropic
from anthropic.types.agent_sdk import SubagentDefinition

def initialize_agent_team() -> dict:
    """Initialize Harness Engineering Agent Team."""
    
    client = Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
    
    # Coordinator Agent
    coordinator = {
        "model": "claude-opus-4-7",
        "system_prompt": load_file(".claude/agents/orchestrator.md"),
        "tools": ["web_search", "code_analysis", "subagent_runner"],
        "max_tokens": 8000,  # Coordinator は各タスクの説明を書く
    }
    
    # Subagent 定義
    subagents = {
        "researcher": SubagentDefinition(
            name="Researcher",
            description="Investigate technical trends from multiple sources",
            system_prompt=load_file(".claude/agents/researcher.md"),
            tools=["web_search", "http_client"],
            max_tokens=4000,  # Per-subagent limit
            timeout=300,
        ),
        "implementer": SubagentDefinition(
            name="Implementer",
            description="Implement code following Harness standards",
            system_prompt=load_file(".claude/agents/implementer.md"),
            tools=["code_edit", "file_create", "test_runner"],
            max_tokens=4000,
            timeout=600,
        ),
        "reviewer": SubagentDefinition(
            name="Reviewer",
            description="Review code for security, performance, quality",
            system_prompt=load_file(".claude/agents/reviewer.md"),
            tools=["code_analysis", "test_analysis"],
            max_tokens=4000,
            timeout=300,
        ),
    }
    
    return {"coordinator": coordinator, "subagents": subagents}

# 初期化実行
AGENT_TEAM_CONFIG = initialize_agent_team()
```

### 2.2 設定ファイル（YAML）

```yaml
# config/agents.yaml
agent_team:
  name: "Harness Engineering Agent Team"
  coordinator:
    model: "claude-opus-4-7"
    system_prompt_file: ".claude/agents/orchestrator.md"
    max_tokens: 8000
    thinking_budget: 5000  # Plan mode のための思考トークン
  
  subagents:
    researcher:
      description: "Technical investigation and trend analysis"
      system_prompt_file: ".claude/agents/researcher.md"
      context_tokens: 256000
      timeout_seconds: 300
      parallel_factor: 3  # 3つの検索を並列可能
    
    implementer:
      description: "Code implementation with TDD"
      system_prompt_file: ".claude/agents/implementer.md"
      context_tokens: 256000
      timeout_seconds: 600
      parallel_factor: 1  # シングルスレッド実装
    
    reviewer:
      description: "Code review and quality assurance"
      system_prompt_file: ".claude/agents/reviewer.md"
      context_tokens: 256000
      timeout_seconds: 300
      parallel_factor: 1
  
  constraints:
    max_parallel_subagents: 3
    context_buffer_percent: 20  # 200K tokens reserve
    cache_hit_rate_target: 0.60
```

---

## 3. Subagent 並列実行管理

### 3.1 タスク分解と依存関係

```python
# src/orchestration/task_scheduler.py
from enum import Enum
from dataclasses import dataclass
from typing import Literal

class TaskType(str, Enum):
    RESEARCH = "research"
    IMPLEMENTATION = "implementation"
    REVIEW = "review"
    DEBUG = "debug"

@dataclass
class Task:
    id: str
    type: TaskType
    description: str
    dependencies: list[str] = field(default_factory=list)  # Task IDs
    subagent_target: str | None = None
    estimated_tokens: int = 0
    timeout_seconds: int = 300
    priority: Literal["high", "normal", "low"] = "normal"

class TaskScheduler:
    def __init__(self):
        self.tasks: dict[str, Task] = {}
        self.execution_order: list[list[str]] = []  # Each level = parallel batch
    
    def add_task(self, task: Task) -> None:
        """Add task to scheduler."""
        self.tasks[task.id] = task
    
    def compute_execution_plan(self) -> list[list[str]]:
        """
        Return execution plan as list of parallel batches.
        
        Example:
        - Level 0: [research_1, research_2, research_3] (parallel)
        - Level 1: [implement_1] (waits for level 0)
        - Level 2: [review_1] (waits for level 1)
        """
        visited = set()
        levels = []
        
        while len(visited) < len(self.tasks):
            current_level = []
            
            for task_id, task in self.tasks.items():
                if task_id in visited:
                    continue
                
                # All dependencies satisfied?
                if all(dep_id in visited for dep_id in task.dependencies):
                    current_level.append(task_id)
            
            if not current_level:
                raise ValueError("Circular dependency detected!")
            
            visited.update(current_level)
            levels.append(current_level)
        
        self.execution_order = levels
        return levels
    
    def get_parallel_batches(self, max_concurrent: int = 3) -> list[list[str]]:
        """
        Split execution plan into batches respecting max_concurrent constraint.
        """
        batches = []
        
        for level in self.execution_order:
            # Split each level into sub-batches if it exceeds max_concurrent
            for i in range(0, len(level), max_concurrent):
                batches.append(level[i:i+max_concurrent])
        
        return batches

# 使用例
scheduler = TaskScheduler()
scheduler.add_task(Task(
    id="research_trends",
    type=TaskType.RESEARCH,
    description="Investigate latest Claude Agent Team patterns",
    subagent_target="researcher"
))
scheduler.add_task(Task(
    id="implement_feature",
    type=TaskType.IMPLEMENTATION,
    description="Implement based on research findings",
    dependencies=["research_trends"],  # 研究後に実装
    subagent_target="implementer"
))

plan = scheduler.compute_execution_plan()
# Output: [[research_trends], [implement_feature]]
```

### 3.2 並列実行エンジン

```python
# src/orchestration/parallel_executor.py
import asyncio
from concurrent.futures import ThreadPoolExecutor
from typing import Callable

class ParallelExecutor:
    def __init__(self, max_concurrent: int = 3):
        self.max_concurrent = max_concurrent
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.results: dict[str, Any] = {}
        self.errors: dict[str, Exception] = {}
    
    async def execute_parallel(
        self, 
        tasks: dict[str, Callable]  # {task_id: async_function}
    ) -> dict[str, Any]:
        """
        Execute up to max_concurrent tasks in parallel.
        
        Args:
            tasks: {task_id: coroutine_function}
        
        Returns:
            {task_id: result}
        """
        async def bounded_execute(task_id: str, coro: Callable):
            async with self.semaphore:
                try:
                    result = await coro()
                    self.results[task_id] = result
                    logger.info(f"Task {task_id} completed")
                except Exception as e:
                    self.errors[task_id] = e
                    logger.error(f"Task {task_id} failed: {e}")
        
        # すべてのタスクを並列実行
        await asyncio.gather(
            *[bounded_execute(tid, func) for tid, func in tasks.items()]
        )
        
        return self.results

# 使用例
async def main():
    executor = ParallelExecutor(max_concurrent=3)
    
    tasks = {
        "researcher_1": async_researcher_search("Claude API trends"),
        "researcher_2": async_researcher_search("Subagent orchestration"),
        "researcher_3": async_researcher_search("Prompt caching patterns"),
    }
    
    results = await executor.execute_parallel(tasks)
    # すべて並列実行、結果を統合
    
    print(f"Successful: {len(results)}")
    print(f"Errors: {len(executor.errors)}")
```

---

## 4. コンフリクト解決パターン

### 4.1 Reviewer vs Implementer コンフリクト

**シナリオ**: Implementer が機能を実装したが、Reviewer がセキュリティ懸念を指摘

```python
# src/orchestration/conflict_resolver.py
@dataclass
class ReviewerFeedback:
    issues: list[str]  # e.g., ["Missing input validation", "SQL injection risk"]
    severity: Literal["critical", "high", "medium", "low"]
    recommendation: str  # e.g., "Add sanitization layer"

@dataclass
class ImplementerResponse:
    status: Literal["accepted", "rejected", "negotiated"]
    reasoning: str
    proposed_fix: str | None

class ConflictResolver:
    def resolve_review_conflict(
        self, 
        feedback: ReviewerFeedback,
        implementer_response: ImplementerResponse
    ) -> dict:
        """
        Resolve disagreement between Reviewer and Implementer.
        
        Returns:
            {
                "decision": "accept_reviewer" | "accept_implementer" | "hybrid",
                "reasoning": "...",
                "action_items": ["...", "..."]
            }
        """
        
        # 優先ルール
        if feedback.severity == "critical":
            # Critical issues は常に Reviewer 優先
            return {
                "decision": "accept_reviewer",
                "reasoning": "Critical security/safety issues take priority",
                "action_items": [
                    f"Implementer: address {issue}" 
                    for issue in feedback.issues
                ]
            }
        
        elif feedback.severity in ["high", "medium"]:
            # Hybrid approach: 修正可能か判定
            if implementer_response.status == "accepted":
                return {
                    "decision": "accepted",
                    "reasoning": "Implementer accepts reviewer feedback",
                    "action_items": [implementer_response.proposed_fix]
                }
            elif implementer_response.status == "negotiated":
                # Coordinator が最終判定
                return {
                    "decision": "coordinator_decision_required",
                    "reasoning": "Implementer proposes alternative solution",
                    "context": {
                        "reviewer_concern": feedback.recommendation,
                        "implementer_proposal": implementer_response.proposed_fix
                    }
                }
            else:
                return {
                    "decision": "accept_reviewer",
                    "reasoning": "Implementer rejected; Reviewer guidance is safer",
                    "action_items": [f"Implement: {feedback.recommendation}"]
                }
        
        else:
            # Low severity: Implementer 優先（時間効率）
            return {
                "decision": "accept_implementer",
                "reasoning": "Low severity; implementation efficiency prioritized",
                "action_items": []
            }
```

### 4.2 複数結果の統合

```python
# src/orchestration/result_aggregator.py
class ResultAggregator:
    def aggregate_parallel_results(
        self, 
        results: dict[str, SubagentResponse]
    ) -> dict:
        """
        Merge results from multiple Subagents.
        
        Example:
            {
                "researcher": { "findings": [...], "confidence": 0.85 },
                "implementer": { "code": "...", "tests": [...] },
                "reviewer": { "issues": [...], "approval": False }
            }
        """
        
        aggregated = {
            "summary": self._generate_summary(results),
            "findings": self._merge_findings(results.get("researcher")),
            "implementation": results.get("implementer"),
            "quality_assessment": self._assess_quality(results.get("reviewer")),
            "recommendations": self._synthesize_recommendations(results),
            "metadata": {
                "total_execution_time_ms": sum(
                    r.execution_time_ms for r in results.values()
                ),
                "total_context_used": sum(
                    r.context_used_tokens for r in results.values()
                ),
                "cache_hit_rate": self._calculate_overall_cache_hit(results),
            }
        }
        
        return aggregated
    
    def _generate_summary(self, results: dict) -> str:
        """Generate executive summary from all subagent results."""
        # Logic to synthesize findings + implementation + review
        pass
    
    def _calculate_overall_cache_hit(self, results: dict) -> float:
        """Average cache hit rate across all subagents."""
        hit_rates = [
            r.metadata.get("cache_hit_rate", 0) 
            for r in results.values()
        ]
        return sum(hit_rates) / len(hit_rates) if hit_rates else 0
```

---

## 5. エラーハンドリング & リカバリー

### 5.1 Subagent Timeout 処理

```python
# src/orchestration/error_handler.py
class SubagentTimeoutHandler:
    async def handle_timeout(self, task_id: str, timeout_sec: int) -> dict:
        """
        Handle subagent timeout.
        
        Strategy:
        1. Log the timeout
        2. Check if partial results are available
        3. If critical: escalate to Debugger
        4. If non-critical: proceed with available results
        """
        
        logger.warning(f"Task {task_id} timed out after {timeout_sec}s")
        
        # Partial results を回収
        partial_result = self.get_partial_result(task_id)
        
        if partial_result:
            logger.info(f"Partial results available for {task_id}")
            return {
                "status": "partial_success",
                "result": partial_result,
                "warning": f"Timeout after {timeout_sec}s; partial results returned"
            }
        else:
            # Debugger に転送
            logger.error(f"No partial results for {task_id}; invoking Debugger")
            debugger_result = await self.invoke_debugger(
                issue=f"Task {task_id} timed out",
                context={
                    "timeout_sec": timeout_sec,
                    "estimated_remaining_work": "unknown"
                }
            )
            return {
                "status": "debugged",
                "debugger_insights": debugger_result,
                "action": "user intervention required"
            }
```

### 5.2 Context Overflow 対応

```python
class ContextOverflowHandler:
    def handle_context_overflow(
        self, 
        subagent: str,
        used_tokens: int,
        budget_tokens: int
    ) -> dict:
        """Handle subagent context budget overflow."""
        
        overflow_percent = (used_tokens - budget_tokens) / budget_tokens * 100
        
        if overflow_percent > 20:
            # 20% 超過は危険
            return {
                "action": "terminate_and_split_task",
                "reason": f"Context overflow {overflow_percent:.1f}%",
                "recommendation": "Split into 2 smaller tasks"
            }
        else:
            # 警告のみ
            logger.warning(
                f"{subagent} near budget: {used_tokens}/{budget_tokens}"
            )
            return {
                "action": "continue_with_warning",
                "message": f"High memory usage {overflow_percent:.1f}%"
            }
```

---

## 6. 実行パターン（よくある構成）

### パターン A: 研究 → 実装 → レビュー（順次）

```yaml
# config/patterns/sequential_development.yaml
pattern: "Sequential Research → Implement → Review"
description: "Classic waterfall for well-defined requirements"

execution_flow:
  - level_1:  # 並列度 = 1
      - research_trends
  - level_2:  # 依存: level_1
      - implement_feature
  - level_3:  # 依存: level_2
      - review_code

max_duration: 30 min
use_cases:
  - "Bug fix with clear root cause"
  - "Small feature addition"
  - "Refactoring task"
```

### パターン B: 複数研究 + 1実装 + 1レビュー（並列研究）

```yaml
pattern: "Parallel Research + Sequential Implement + Review"
description: "Fast research phase for multiple angles"

execution_flow:
  - level_1:  # 並列度 = 3
      - research_api_design
      - research_performance_patterns
      - research_security_best_practices
  - level_2:  # 依存: level_1 すべて
      - implement_design
  - level_3:  # 依存: level_2
      - review_implementation

max_duration: 45 min
use_cases:
  - "New framework integration"
  - "Major feature design"
  - "Architecture decision"
```

### パターン C: 並列実装 + 集約レビュー

```yaml
pattern: "Parallel Implement + Aggregate Review"
description: "Fast prototyping with concurrent implementation"

execution_flow:
  - level_1:  # 並列度 = 2
      - implement_api_handler
      - implement_database_schema
  - level_2:  # 依存: level_1 すべて
      - review_integration

max_duration: 60 min
use_cases:
  - "Complex features requiring multiple components"
  - "Microservice coordination"
```

---

## 7. チェックリスト（Agent Teams 導入）

### 初期化フェーズ

- [ ] Coordinator agent を `.claude/agents/orchestrator.md` で定義
- [ ] 各 Subagent を `.claude/agents/{name}.md` で定義
- [ ] `config/agents.yaml` に Agent Team 構成を記載
- [ ] 各 Subagent のタイムアウト・コンテキスト予算を設定
- [ ] Task Scheduler を実装（`src/orchestration/task_scheduler.py`）

### 実行フェーズ

- [ ] 並列度が MAX 3 に制限されているか
- [ ] Subagent 間メモリ隔離が機能しているか（コンテキスト独立）
- [ ] Task 依存関係が正しくキャプチャされているか
- [ ] Timeout ハンドラが Debugger を自動で呼び出しているか

### 監視フェーズ

- [ ] 各 Subagent の応答時間をログ記録
- [ ] Context 使用率を監視（警告閾値: 90%）
- [ ] 並列実行による時間短縮を測定
- [ ] キャッシュヒット率が 60% 以上か確認

---

**参考リソース**:
- [Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)
- [Claude Code Agent Teams: Setup & Usage Guide 2026](https://claudefa.st/blog/guide/agents/agent-teams)
- [GitHub: wshobson/agents - Multi-agent orchestration](https://github.com/wshobson/agents)
