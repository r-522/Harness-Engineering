# ORCHESTRATION.md - Multi-Agent Coordination Patterns

**Generated**: April 21, 2026  
**Framework**: Advanced Agent Orchestration  
**Reference**: Anthropic 3-Agent Harness, Meta Unified Architecture, Haystack/CrewAI Patterns

## 🎯 マルチエージェント構想の必要性

### 単一エージェントの限界

```
Single Agent Problem:
  ├─ 計画と実装が同時実行 → 品質低下
  ├─ 全タスクを同じプロンプトで処理 → コスト増
  ├─ エラー検出が遅い → 手戻り多い
  └─ スケーラビリティが低い → 複雑タスク不可

Success Rate: 40-50% for complex tasks
```

### マルチエージェントの利点

```
Multi-Agent Benefits:
  ✅ 責任分離 → 品質向上
  ✅ 並列実行可能性 → 速度向上（最大3倍）
  ✅ 早期エラー検出 → 手戻り削減
  ✅ スケーラビリティ向上 → 複雑タスク対応

Success Rate: 85-95% for complex tasks
Cost Efficiency: 30-50% improvement through specialization
```

## 🏗️ 3層エージェント構造（推奨標準）

### アーキテクチャ概要

```
┌────────────────────────────────────────────────────────┐
│             USER REQUEST ENTRY POINT                   │
│  "Implement authentication system with tests"         │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│          AGENT 1: PLANNING AGENT (Haiku)              │
│                                                        │
│  Responsibility:                                       │
│    • Task decomposition → 3 subtasks identified       │
│    • Resource estimation → 180K tokens budget         │
│    • Risk assessment → 2 potential blockers           │
│    • Dependency analysis → Auth → Tests → Docs       │
│                                                        │
│  Output:                                               │
│    {                                                   │
│      "subtasks": [                                    │
│        {"id": 1, "task": "Auth API implementation"},  │
│        {"id": 2, "task": "Unit test creation"},       │
│        {"id": 3, "task": "Integration testing"}       │
│      ],                                                │
│      "dependencies": {"2": [1], "3": [1, 2]},        │
│      "estimated_tokens": 150000,                      │
│      "risks": ["DB integration timing", ...]          │
│    }                                                   │
│                                                        │
│  Model: Claude Haiku 4.5                              │
│  Cost: 2-5K tokens                                    │
│  Time: 30-60 seconds                                  │
└──────────────────────────┬─────────────────────────────┘
                           │
                ┌──────────┴──────────┐
                │                     │
                ▼                     ▼
    ┌─────────────────────┐  ┌─────────────────────┐
    │   PARALLEL TRACK    │  │   PARALLEL TRACK    │
    │                     │  │                     │
    │ Subtask 1: Auth API │  │ (wait: Task 1→deps) │
    │ Subtask 2: Tests    │  │ Subtask 3: Integ    │
    │                     │  │                     │
    └──────────┬──────────┘  └──────────┬──────────┘
               │                        │
               ▼                        ▼
┌────────────────────────────────────────────────────────┐
│      AGENT 2: GENERATION AGENT (Sonnet)               │
│                                                        │
│  Parallel Execution:                                   │
│    Agent 2A: Implement Auth API (Sonnet)              │
│      • Code generation                                │
│      • Input validation                               │
│      • Error handling                                 │
│      Output: 200 lines of production code            │
│                                                        │
│    Agent 2B: Create unit tests (Sonnet)               │
│      • Test suite generation                          │
│      • Edge case coverage                             │
│      • Mock setup                                     │
│      Output: 150 lines of test code                  │
│                                                        │
│    Agent 2C: Integration tests (Sonnet)               │
│      • Wait for Task 1, 2 completion                 │
│      • E2E test creation                             │
│      • Performance benchmarks                         │
│      Output: 100 lines of integration code           │
│                                                        │
│  Model: Claude Sonnet 4.6 (x3 parallel)              │
│  Cost: 50-70K tokens per subtask                     │
│  Time: 3-5 minutes (parallelized)                    │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│       AGENT 3: EVALUATION AGENT (Opus)                │
│                                                        │
│  Responsibility:                                       │
│    • Code quality analysis                            │
│      ✅ Type safety check: PASS                       │
│      ✅ Test coverage: 92% (Target: >85%)             │
│      ✅ Performance: 45ms latency (Target: <100ms)    │
│      ✅ Security audit: No vulnerabilities            │
│                                                        │
│    • Production readiness validation                  │
│      ✅ Error handling: Comprehensive                │
│      ✅ Logging: INFO/ERROR levels                   │
│      ✅ Documentation: Complete                      │
│      ✅ Rollback plan: Documented                    │
│                                                        │
│    • Final approval decision                          │
│      ✅ APPROVED FOR PRODUCTION                      │
│      Confidence score: 97%                            │
│      Recommendations: 2 (minor improvements)          │
│                                                        │
│  Model: Claude Opus 4.7                               │
│  Cost: 20-30K tokens                                 │
│  Time: 2-4 minutes                                   │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│              FINAL OUTPUT                              │
│                                                        │
│  Status: ✅ READY FOR PRODUCTION                      │
│  Total Execution Time: 6-10 minutes                   │
│  Total Token Cost: 140K tokens (below 150K budget)    │
│  Quality Score: 97/100                                │
│  All Tests Passing: YES                               │
│                                                        │
│  Deliverables:                                        │
│    ✓ src/auth/api.ts (200 lines)                      │
│    ✓ src/auth/api.test.ts (150 lines)                │
│    ✓ src/auth/integration.test.ts (100 lines)        │
│    ✓ docs/AUTH_API.md (comprehensive)                │
│    ✓ DEPLOYMENT_CHECKLIST.md (completed)             │
└────────────────────────────────────────────────────────┘
```

## 🔄 Orchestration Patterns

### Pattern A: Sequential Workflow (順序実行)

**使用場面**: 各タスクが前のタスク出力に依存

```
Task 1 (Plan)
    ↓
Task 2 (Implement Based on Plan)
    ↓
Task 3 (Test)
    ↓
Task 4 (Deploy)

Advantages:
  ✓ Simple logic
  ✓ Full context available
  ✓ Easy debugging

Disadvantages:
  ✗ Slower (sequential)
  ✗ Error in Task N stops all
  ✗ Poor resource utilization
```

**実装**:

```python
class SequentialOrchestrator:
    def __init__(self, agents: dict):
        self.agents = agents
    
    async def execute(self, user_request: str):
        results = {}
        
        # Step 1: Planning
        plan = await self.agents["planner"].execute(user_request)
        results["plan"] = plan
        
        # Step 2: Implementation (uses plan)
        code = await self.agents["developer"].execute(
            request=user_request,
            context=plan
        )
        results["code"] = code
        
        # Step 3: Testing
        test_results = await self.agents["tester"].execute(
            code=code,
            context=plan
        )
        results["tests"] = test_results
        
        # Step 4: Deployment
        if test_results["all_passed"]:
            deploy = await self.agents["deployer"].execute(
                code=code,
                tests=test_results
            )
            results["deployment"] = deploy
        
        return results
```

### Pattern B: Parallel Workflow (並列実行)

**使用場面**: 複数のタスクが独立している

```
Task 1A ───┐
           ├→ Aggregation
Task 1B ───┤
           │
Task 1C ───┘

Advantages:
  ✓ Faster (3x speedup with 3 parallel tasks)
  ✓ Independent failures isolated
  ✓ Better resource utilization

Disadvantages:
  ✗ Complex orchestration
  ✗ Limited context sharing
  ✗ Debugging harder
```

**実装**:

```python
class ParallelOrchestrator:
    async def execute(self, user_request: str):
        # Run three agents in parallel
        results = await asyncio.gather(
            self.agents["security_auditor"].execute(user_request),
            self.agents["performance_tester"].execute(user_request),
            self.agents["documentation_writer"].execute(user_request),
            return_exceptions=True
        )
        
        # Aggregate results
        return self.aggregate_results(results)
    
    def aggregate_results(self, results: list):
        return {
            "security": results[0],
            "performance": results[1],
            "documentation": results[2],
            "overall_status": self.determine_status(results)
        }
    
    def determine_status(self, results: list) -> str:
        if any(isinstance(r, Exception) for r in results):
            return "FAILED"
        if all(r.get("approved") for r in results):
            return "APPROVED"
        return "NEEDS_REVIEW"
```

### Pattern C: Adaptive Workflow (動的実行)

**使用場面**: タスク複雑度に応じて戦略を変更

```
Complexity Analysis
    ↓
Low Complexity          Medium Complexity      High Complexity
(Haiku only)           (Haiku + Sonnet)      (All 3 agents)
    ↓                      ↓                      ↓
1 step                 3 steps                5 steps
2 min                  5 min                  10 min
$2 cost                $10 cost               $30 cost
```

**実装**:

```python
class AdaptiveOrchestrator:
    def __init__(self, agents: dict):
        self.agents = agents
    
    async def execute(self, user_request: str):
        # Analyze complexity (lightweight)
        complexity = self.analyze_complexity(user_request)
        
        # Select strategy based on complexity
        if complexity == "low":
            return await self._simple_execution(user_request)
        elif complexity == "medium":
            return await self._standard_execution(user_request)
        else:
            return await self._comprehensive_execution(user_request)
    
    def analyze_complexity(self, request: str) -> str:
        factors = {
            "num_files": request.count(".py") + request.count(".js"),
            "has_security": "security" in request.lower(),
            "has_performance": "performance" in request.lower(),
            "request_length": len(request.split()),
            "num_languages": len(set(
                re.findall(r'(\.\w+)', request)
            ))
        }
        
        score = sum([
            factors["num_files"] * 0.1,
            factors["has_security"] * 0.3,
            factors["has_performance"] * 0.2,
            factors["request_length"] // 100 * 0.1,
            (factors["num_languages"] - 1) * 0.3
        ])
        
        if score > 2.0:
            return "high"
        elif score > 1.0:
            return "medium"
        else:
            return "low"
    
    async def _simple_execution(self, request: str):
        """Only Haiku for simple tasks"""
        return await self.agents["haiku"].execute(request)
    
    async def _standard_execution(self, request: str):
        """Plan (Haiku) + Implement (Sonnet)"""
        plan = await self.agents["planner"].execute(request)
        implementation = await self.agents["developer"].execute(request, plan)
        return {"plan": plan, "implementation": implementation}
    
    async def _comprehensive_execution(self, request: str):
        """All 3 agents: Plan, Develop, Review"""
        plan = await self.agents["planner"].execute(request)
        code = await self.agents["developer"].execute(request, plan)
        review = await self.agents["reviewer"].execute(code, plan)
        return {"plan": plan, "code": code, "review": review}
```

## 💬 Agent Communication Protocol

### Message Format

```json
{
  "sender": "agent_id",
  "recipient": "agent_id_list",
  "message_type": "task|update|error|request_context",
  "timestamp": "2026-04-21T10:30:00Z",
  "payload": {
    "task_id": "unique_task_id",
    "content": "message body",
    "metadata": {
      "priority": "P0|P1|P2",
      "estimated_time": "5m",
      "token_budget": 20000,
      "cache_keys": ["system_prompt", "project_context"]
    }
  },
  "context_request": {
    "files": ["auth.ts", "auth.test.ts"],
    "recent_decisions": true,
    "code_snippets": 5
  }
}
```

### Context Sharing Strategy

```python
class SharedContextManager:
    def __init__(self):
        self.shared_context = {
            "system_instructions": "",
            "project_overview": "",
            "architecture": "",
            "recent_decisions": [],
            "code_snippets": {}
        }
        self.context_cache = {}
    
    def register_agent(self, agent_id: str):
        """Register agent for context sharing"""
        self.context_cache[agent_id] = {
            "cached_at": None,
            "hits": 0,
            "misses": 0
        }
    
    def get_context(self, agent_id: str, request_keys: list) -> dict:
        """Serve context with caching metrics"""
        result = {}
        for key in request_keys:
            if key in self.shared_context:
                result[key] = self.shared_context[key]
                self.context_cache[agent_id]["hits"] += 1
            else:
                self.context_cache[agent_id]["misses"] += 1
        
        return result
    
    def update_context(self, agent_id: str, key: str, value):
        """Update shared context from any agent"""
        self.shared_context[key] = value
        self.invalidate_caches()  # Notify other agents
    
    def report_efficiency(self) -> dict:
        """Report context sharing efficiency"""
        report = {}
        for agent_id, stats in self.context_cache.items():
            total = stats["hits"] + stats["misses"]
            hit_rate = stats["hits"] / total if total > 0 else 0
            report[agent_id] = {
                "hit_rate": f"{hit_rate*100:.1f}%",
                "efficiency": "high" if hit_rate > 0.8 else "low"
            }
        return report
```

## ⚖️ 負荷分散・スケーリング

### Agent Pool Management

```python
class AgentPool:
    def __init__(self, num_agents: int = 5):
        self.agents = {
            "haiku": [clone_agent("haiku") for _ in range(num_agents)],
            "sonnet": [clone_agent("sonnet") for _ in range(num_agents)],
            "opus": [clone_agent("opus") for _ in range(num_agents)]
        }
        self.task_queue = asyncio.Queue()
    
    async def assign_task(self, task, target_model="sonnet"):
        """Assign task to least-busy agent"""
        available_agent = self._find_least_busy(target_model)
        if available_agent is None:
            await self.task_queue.put((task, target_model))
            return
        
        await available_agent.execute(task)
    
    def _find_least_busy(self, model_type: str):
        """Find agent with minimum queue length"""
        available = [
            agent for agent in self.agents[model_type]
            if agent.current_load < agent.max_load
        ]
        
        if not available:
            return None
        
        return min(available, key=lambda a: a.current_load)
```

## 📊 Monitoring & Observability

### Multi-Agent Metrics

```python
@dataclass
class AgentMetrics:
    agent_id: str
    execution_time: float  # seconds
    token_usage: int
    success_rate: float  # 0-1
    cache_hit_rate: float  # 0-1
    cost: float  # USD
    timestamp: datetime

class OrchestrationMonitor:
    def __init__(self):
        self.metrics: list[AgentMetrics] = []
    
    def report_agent_metrics(self, metrics: AgentMetrics):
        self.metrics.append(metrics)
    
    def generate_report(self) -> dict:
        return {
            "total_execution_time": self._sum_times(),
            "total_tokens": self._sum_tokens(),
            "total_cost": self._sum_costs(),
            "average_success_rate": self._avg_success(),
            "cache_efficiency": self._cache_efficiency(),
            "bottleneck_agent": self._identify_bottleneck()
        }
    
    def _identify_bottleneck(self) -> str:
        """Find slowest agent in the pipeline"""
        return max(self.metrics, key=lambda m: m.execution_time).agent_id
```

## ✅ Orchestration Checklist

```yaml
design_phase:
  - [ ] エージェント責務の明確化
  - [ ] 通信プロトコル定義
  - [ ] コンテキスト共有戦略
  - [ ] エラーハンドリング計画

implementation_phase:
  - [ ] オーケストレーション実装
  - [ ] 負荷分散機構
  - [ ] キャッシング最適化
  - [ ] 監視・ロギング

testing_phase:
  - [ ] 単体テスト（各エージェント）
  - [ ] 統合テスト（全体流れ）
  - [ ] 負荷テスト（並列実行）
  - [ ] 障害シナリオテスト

production_phase:
  - [ ] カナリアリリース（10% トラフィック）
  - [ ] メトリクス監視
  - [ ] ロールバック計画確認
  - [ ] SLA 達成確認
```

---

**最終更新**: 2026年04月21日  
**推奨アーキテクチャ**: 3層エージェント構造 (Planning/Generation/Evaluation)  
**期待効果**: 成功率 85-95%, コスト 30-50% 削減
