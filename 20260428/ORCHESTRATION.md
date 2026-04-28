# ORCHESTRATION.md — Subagent Configuration & Orchestration Strategies

## Subagent Pool Definition

### Subagent Registry

```yaml
subagents:
  - id: research-v2
    role: Codebase exploration, pattern discovery, architectural research
    model: Sonnet 4.6
    context_window: 200k
    capabilities: [read, search, analyze, report]
    memory_enabled: true
    timeout_seconds: 300
    cost_per_execution: $0.003
    triggers:
      - keywords: ["explore", "find", "where", "search", "codebase"]
      - task_type: ["research", "discovery", "audit"]

  - id: plan-v2
    role: Architectural design, multi-step planning, decision synthesis
    model: Opus 4.7
    context_window: 200k
    capabilities: [analyze, plan, design, decide]
    memory_enabled: true
    timeout_seconds: 600
    cost_per_execution: $0.015
    triggers:
      - keywords: ["how should", "design", "plan", "approach", "architecture"]
      - task_type: ["design", "planning", "strategy"]

  - id: review-v2
    role: Code review, quality assurance, security audit, verification
    model: Sonnet 4.6
    context_window: 200k
    capabilities: [analyze, review, audit, verify]
    memory_enabled: true
    timeout_seconds: 180
    cost_per_execution: $0.003
    triggers:
      - keywords: ["review", "test", "verify", "check", "security"]
      - task_type: ["review", "qa", "audit"]

  - id: orchestrator-v1
    role: Subagent coordination, split-merge execution, team management
    model: Opus 4.7
    context_window: 200k
    capabilities: [coordinate, dispatch, merge, decide]
    memory_enabled: true
    timeout_seconds: 900
    cost_per_execution: $0.020
    triggers:
      - keywords: ["orchestrate", "parallel", "coordinate", "team"]
      - task_type: ["orchestration", "large_workflow"]
```

---

## Orchestration Strategy Patterns

### Strategy 1: Sequential Linear (Default)

**When to Use**: Single-threaded workflow (A → B → C)

**Flow**:
```
Input Task
  ↓
Spawn research-v2 (gather info)
  ↓
Spawn plan-v2 (design solution)
  ↓
Spawn review-v2 (verify quality)
  ↓
Output Result
```

**Implementation**:
```python
class SequentialOrchestrator:
    def execute(self, task: Task) -> Result:
        # Step 1: Research
        research_result = self.spawn_subagent(
            "research-v2",
            prompt=f"Research: {task.description}",
            timeout=300
        )
        
        # Step 2: Plan (uses research output)
        plan_result = self.spawn_subagent(
            "plan-v2",
            prompt=f"Plan based on research:\n{research_result.output}",
            timeout=600
        )
        
        # Step 3: Review
        review_result = self.spawn_subagent(
            "review-v2",
            prompt=f"Review plan:\n{plan_result.output}",
            timeout=180
        )
        
        return Result(
            steps=[research_result, plan_result, review_result],
            final_output=review_result.output
        )
```

**Cost**: 1 × $0.003 + 1 × $0.015 + 1 × $0.003 = **$0.021 per execution**
**Latency**: ~15 minutes (sum of all timeouts)

---

### Strategy 2: Split-and-Merge (Parallel Branches)

**When to Use**: Multiple independent subtasks that can run in parallel

**Flow**:
```
Input Task (decompose into 3 subtasks)
  ├─ [Parallel] research-v2 → Output A
  ├─ [Parallel] plan-v2 → Output B
  └─ [Parallel] review-v2 → Output C
  ↓
[Merge] Synthesis (combines A, B, C)
  ↓
Output Result
```

**Implementation**:
```python
class ParallelOrchestrator:
    async def execute_parallel(self, task: Task) -> Result:
        # Decompose into independent subtasks
        subtasks = self.decompose(task)  # Returns [task_a, task_b, task_c]
        
        # Spawn in parallel
        futures = [
            self.spawn_subagent_async("research-v2", subtasks[0]),
            self.spawn_subagent_async("plan-v2", subtasks[1]),
            self.spawn_subagent_async("review-v2", subtasks[2]),
        ]
        
        # Wait for all to complete
        results = await asyncio.gather(*futures)
        
        # Merge outputs
        merged = self.merge_results(results)
        return merged

    def decompose(self, task: Task) -> List[Task]:
        """Break task into independent subtasks"""
        return [
            Task(desc=f"Research aspect: {task.key_aspect_1}"),
            Task(desc=f"Plan solution for: {task.key_aspect_2}"),
            Task(desc=f"Review approach for: {task.key_aspect_3}"),
        ]

    def merge_results(self, results: List[SubagentOutput]) -> Result:
        """Combine parallel outputs into unified result"""
        return Result(
            research_findings=results[0].output,
            plan_options=results[1].output,
            review_feedback=results[2].output,
            synthesis=self.synthesize(results)
        )
```

**Constraint**: Max 3-5 parallel subagents (avoid context thrashing)

**Cost**: Same as sequential (~$0.021) but **1/3 the latency** (~5 minutes)

---

### Strategy 3: Agent Teams (Shared State)

**When to Use**: Many independent tasks with dynamic scheduling

**Flow**:
```
Shared Task Queue
  ├─ Task 1: Pending
  ├─ Task 2: In Progress (assigned to research-v2)
  └─ Task 3: Pending

Agent Loop (infinite):
  1. Pick next pending task
  2. Execute
  3. Mark completed
  4. Repeat
```

**State Schema**:
```json
{
  "tasks": [
    {
      "id": "task_001",
      "status": "pending",
      "assigned_to": null,
      "priority": 1,
      "created_at": "2026-04-28T10:00:00Z",
      "deadline": "2026-04-28T12:00:00Z"
    },
    {
      "id": "task_002",
      "status": "in_progress",
      "assigned_to": "research-v2",
      "started_at": "2026-04-28T10:05:00Z"
    },
    {
      "id": "task_003",
      "status": "completed",
      "assigned_to": "plan-v2",
      "result": {...},
      "completed_at": "2026-04-28T10:30:00Z"
    }
  ]
}
```

**Implementation**:
```python
class AgentTeam:
    def __init__(self, agents: List[SubagentConfig], state_store: StateStore):
        self.agents = agents
        self.state = state_store
    
    def agent_loop(self, agent_id: str):
        """Each agent runs this in a loop"""
        while True:
            # 1. Pick next task
            task = self.state.pick_task(agent_id)
            if not task:
                time.sleep(1)
                continue
            
            # 2. Execute
            result = self.execute_task(agent_id, task)
            
            # 3. Mark completed
            self.state.mark_completed(task.id, result)
    
    def execute_task(self, agent_id: str, task: Task):
        agent = next(a for a in self.agents if a.id == agent_id)
        return agent.execute(task)
```

**Pros**:
- ✓ Fully parallel (no bottleneck)
- ✓ Dynamic load balancing
- ✓ Fault-tolerant (one agent failure doesn't block others)

**Cons**:
- ✗ Complex state management (race conditions)
- ✗ Harder to debug

---

### Strategy 4: Hybrid (Orchestrator + Parallel Branches)

**When to Use**: Mixed workflows (some sequential, some parallel)

**Flow**:
```
Input Task
  ↓
[Sequential] research-v2 → Findings
  ↓
[Parallel] Based on findings:
  ├─ plan-v2 (Option A)
  ├─ plan-v2 (Option B) [with different constraints]
  └─ plan-v2 (Option C)
  ↓
[Sequential] review-v2 (all options)
  ↓
Output Result
```

**Use Case**: Large architectural decisions with multiple design options

---

## Subagent Configuration in .claude/settings.json

```json
{
  "$schema": "https://code.claude.com/schema/settings.json",
  "model": "claude-opus-4-7",
  "temperature": 0.7,
  "max_tokens": 16000,
  
  "subagents": {
    "research-v2": {
      "description": "Codebase exploration and pattern discovery",
      "model": "claude-sonnet-4-6",
      "system_prompt_file": ".claude/skills/research-system.md",
      "tools": ["read", "bash", "web_search"],
      "memory": {
        "enabled": true,
        "type": "directory",
        "path": ".claude/agent_memory/research"
      }
    },
    "plan-v2": {
      "description": "Architectural design and planning",
      "model": "claude-opus-4-7",
      "system_prompt_file": ".claude/skills/plan-system.md",
      "tools": ["read", "write", "bash"],
      "memory": {
        "enabled": true,
        "type": "directory",
        "path": ".claude/agent_memory/plan"
      }
    },
    "review-v2": {
      "description": "Code review and verification",
      "model": "claude-sonnet-4-6",
      "system_prompt_file": ".claude/skills/review-system.md",
      "tools": ["read", "bash"],
      "memory": {
        "enabled": true,
        "type": "directory",
        "path": ".claude/agent_memory/review"
      }
    }
  },
  
  "orchestration": {
    "default_strategy": "sequential",
    "max_parallel_agents": 5,
    "timeout_global_seconds": 1800,
    "result_merge_strategy": "structured"
  },
  
  "hooks": {
    "before_subagent_spawn": "scripts/validate_task.sh",
    "after_subagent_complete": "scripts/log_result.sh"
  }
}
```

---

## 調整メカニズム (Tuning & Monitoring)

### Performance Tuning

#### Latency Optimization

| Problem | Solution |
|---|---|
| Slow research phase | Increase `research-v2` context window; parallelize subtasks |
| Slow planning | Use Sonnet 4.6 for plan-v2 if complexity < medium |
| Slow review | Pre-define review checklist (reduce decision time) |

#### Cost Optimization

| Strategy | Savings |
|---|---|
| Use Sonnet for research + review (not Opus) | -60% cost |
| Reuse agent memory across sessions | -40% context reuse overhead |
| Batch similar tasks | -30% avg latency (fixed startup cost amortized) |

**Example Cost Comparison**:
```
Sequential (Opus-only):
  research-v2 (Sonnet): $0.003
  plan-v2 (Opus): $0.015
  review-v2 (Sonnet): $0.003
  Total: $0.021 per execution

Optimized (Hybrid):
  Same agents, same cost
  BUT: 3 parallel instead of sequential
  Latency: 15min → 5min (70% improvement, same cost!)
```

---

### Monitoring Checklist

```yaml
Metrics to Track:
  - Latency per phase (research, plan, review)
  - Cost per execution (model × tokens)
  - Success rate (% tasks completed without error)
  - Quality metrics (review feedback scores)
  - Subagent memory growth (GB per session)
  
Alerts:
  - Latency > 5min (sequential) or > 2min (parallel) → Investigate
  - Cost > $0.05 per execution → Optimize model selection
  - Success rate < 95% → Debug failure modes
  - Memory growth > 1GB per week → Clear cache
```

---

### Memory Management

Each subagent can maintain persistent memory in `.claude/agent_memory/{agent_id}/`:

```
.claude/agent_memory/
├── research/
│   ├── codebase_patterns.md       # Patterns discovered over sessions
│   ├── architecture_insights.md   # Design decisions, tradeoffs
│   └── dependency_graph.json      # Cached codebase relationships
├── plan/
│   ├── design_templates.md        # Reusable architecture patterns
│   └── decision_log.md            # Previous decisions & rationale
└── review/
    ├── checklist.md               # Team-specific review criteria
    └── common_issues.md           # Known problems & solutions
```

**Update Frequency**: Weekly sync (consolidate learnings)

---

## Subagent Lifecycle

### Spawn Phase

```python
response = agent.spawn(
    subagent_id="research-v2",
    prompt="...",
    context=[file1, file2, ...],
    timeout_seconds=300,
    priority="high"
)
```

### Execution Phase

- Subagent runs in isolated context window (200k tokens)
- Memory directory available (read/write)
- No communication with other subagents directly
- Timeout enforced (termination if exceeded)

### Completion Phase

```python
Result = {
    "subagent_id": "research-v2",
    "status": "success" | "timeout" | "error",
    "output": "...",
    "metadata": {
        "tokens_used": 12345,
        "latency_ms": 45000,
        "cost": 0.003
    }
}
```

### Escalation

If subagent fails:
1. Log error with full context
2. Notify orchestrator
3. Retry once (if transient error)
4. Escalate to user (if persistent error)

---

## Best Practices

### ✓ DO

- ✓ Use Split-and-Merge for independent subtasks (reduce latency 70%)
- ✓ Cache subagent outputs (reuse results across sessions)
- ✓ Define clear input/output schemas (enables deterministic merging)
- ✓ Monitor cost & latency metrics (tune model selection)
- ✓ Update agent memory weekly (consolidate learnings)

### ❌ DON'T

- ❌ Spawn 10+ subagents in parallel (context thrashing)
- ❌ Use Opus for all subagents (unnecessary cost)
- ❌ Chain subagents sequentially without dependency (use parallel instead)
- ❌ Ignore timeout errors (they indicate overload or bugs)
- ❌ Accumulate unbounded memory (trim after sessions)

---

## References

- Subagent Orchestration Patterns: README.md sources 5, 8, 9
- Split-and-Merge Implementation: https://medium.com/@richardhightower/claude-code-subagents-and-main-agent-coordination-a-complete-guide-to-ai-agent-delegation-patterns-a4f88ae8f46c
- Agent Teams Pattern: https://turion.ai/blog/claude-code-multi-agents-subagents-guide/
- Previous Config: /20260427/ORCHESTRATION.md, /20260425/ORCHESTRATION.md (if exists)
