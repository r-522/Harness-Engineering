# DESIGN.md — Design Principles & Architecture Patterns

## 適用する設計原則 (Design Principles)

### 1. SOLID Principles (OOP)

#### S — Single Responsibility Principle
- **Each class/module has one reason to change**
- Example: `SubagentOrchestrator` handles coordination only; `PlanExecutor` handles Plan-Act-Verify
- Violation: "Orchestrator that also runs code" → Split into two

#### O — Open/Closed Principle
- **Open for extension, closed for modification**
- Example: Define `AgentInterface`, implement `SequentialAgent`, `ParallelAgent`, etc.
- Never modify Agent core when adding new execution strategy

#### L — Liskov Substitution Principle
- **Subagent instances are interchangeable within their contract**
- Example: `Explore` agent, `Plan` agent, `Review` agent all satisfy `AgentProtocol`
- Each subagent may have different Model (Sonnet vs. Opus) but same input/output signature

#### I — Interface Segregation Principle
- **Clients depend only on interfaces they use**
- Example: `SubagentCoordinator` interface has `spawn()`, `wait()`, `collect_results()`
- Don't force subagent implementations to support unused methods

#### D — Dependency Inversion Principle
- **Depend on abstractions, not concretions**
- Example: `OrchestratorAgent` depends on `AgentFactory` (abstract), not hardcoded `Sonnet4_6Agent`
- Enables swapping models/providers without code changes

---

### 2. DRY (Don't Repeat Yourself)

**Threshold**: Same logic appears 3+ times
```
Occurrence 1: Accept duplication
Occurrence 2: Accept duplication  
Occurrence 3: REFACTOR into shared function/class
```

**Example: Subagent Spawn**
```python
# ❌ DRY Violation (appears in 3 places)
response = orchestrator.spawn("research", {"prompt": ..., "timeout": 30})
response = orchestrator.spawn("plan", {"prompt": ..., "timeout": 30})
response = orchestrator.spawn("review", {"prompt": ..., "timeout": 30})

# ✓ DRY Compliant
config = {"timeout": 30}
for agent_type in ["research", "plan", "review"]:
    response = orchestrator.spawn(agent_type, {**config, "prompt": ...})
```

---

### 3. YAGNI (You Aren't Gonna Need It)

- **No feature flags for hypothetical future use**
- **No backwards-compatibility shims** (refactor directly)
- **No error handling for impossible scenarios** (trust framework guarantees)
- **No abstractions before 3 occurrences** (premature optimization)

**Example: Config Version Migration**
```
❌ Don't: Add v2_config check, legacy fallback, feature flag
✓ Do: Update config schema directly; migrate in one pass (git commit per schema version)
```

---

### 4. KISS (Keep It Simple, Stupid)

- **Prefer explicit over implicit**
- **Prefer obvious over clever**
- **Prefer boring over creative** (standard patterns > novel solutions)

**Example: Subagent Response Handling**
```python
# ❌ Clever: Magic dict key detection
result = response["data"]["model"]["output"].get("message") or response.get("fallback")

# ✓ Simple: Explicit structure
struct SubagentResponse:
    status: str
    message: str
    metadata: dict
result = response.message
```

---

## アーキテクチャパターン (Architecture Patterns)

### Pattern 1: Orchestrator (Central Coordinator)

**Use Case**: Complex workflows with dynamic routing, conditional branching
**Actors**:
- `Orchestrator`: Reads plan, dispatches subagents, reviews outputs, decides next step
- `Subagents`: Execute focused tasks (Research, Plan, Review, Execute)

**Pros**:
- ✓ Single point of control for complex workflows
- ✓ Easy to add conditional logic (if research complete → plan, else → re-research)
- ✓ Audit trail clear (orchestrator makes all decisions)

**Cons**:
- ✗ Orchestrator can become bottleneck (context accumulation)
- ✗ Sequential by nature (each subagent waits for orchestrator decision)

**Implementation**:
```python
class OrchestratorAgent:
    def coordinate(self, task: Task) -> Result:
        plan = self.spawn("plan", task)
        if plan.feasibility_score < 0.8:
            return self.spawn("research", task)  # Loop back
        execution = self.spawn("execute", plan)
        return self.spawn("review", execution)
```

---

### Pattern 2: Split-and-Merge (Parallel Subtasks)

**Use Case**: Independent subtasks that can run concurrently, then merge results

**Procedure**:
1. Break large task into N independent subtasks
2. Spawn N subagents in parallel (max 3-5 concurrent)
3. Collect results once all complete
4. Merge: Synthesize outputs into unified result

**Output Schema Requirement**:
```python
@dataclass
class SubagentOutput:
    task_id: str
    status: str  # "success" | "failure"
    result: dict
    metadata: dict

# Merge expects consistent schema from all branches
merged = {
    "research_findings": parallel_outputs[0].result,
    "plan_options": parallel_outputs[1].result,
    "review_feedback": parallel_outputs[2].result,
}
```

**Example**: Testing a new feature
```
Main Task: Validate feature readiness
  ├─ [Parallel] Unit tests (Research Agent)
  ├─ [Parallel] Integration tests (Plan Agent)
  ├─ [Parallel] Type checking (Review Agent)
  └─ [Sequential] Merge: All green? → Ship : Fix issues
```

---

### Pattern 3: Agent Teams (Shared State)

**Use Case**: Multiple agents working independently, coordinating via shared task queue/state

**Mechanism**:
- Shared state store (database, file, queue) holds task list
- Each agent picks task, executes, posts result
- No central orchestrator required

**Pros**:
- ✓ Fully parallel (no bottleneck)
- ✓ Scales to many agents
- ✓ Fault-tolerant (one agent failure doesn't block others)

**Cons**:
- ✗ Complex state management (race conditions, consistency)
- ✗ Harder to debug (no single control flow)

**State Schema**:
```json
{
  "tasks": [
    {"id": "t1", "status": "pending", "assigned_to": null},
    {"id": "t2", "status": "in_progress", "assigned_to": "review-v2"},
    {"id": "t3", "status": "completed", "result": {...}}
  ]
}
```

---

### Pattern 4: Headless (No Orchestrator)

**Use Case**: Single-purpose agents, no coordination needed

**Example**: "Generate documentation from code"
```
User → SingleAgent(model=Sonnet, task="document") → Output
```

**Simplest pattern; use when task is atomic and doesn't require branching**

---

## ディレクトリ構成ガイド (Project Structure)

### Small Project (< 5 files)

```
.
├── CLAUDE.md
├── main.py          # Single entry point
├── agents/
│   ├── orchestrator.py
│   └── subagents.py
└── tests/
    └── test_agents.py
```

---

### Medium Project (5-50 files)

```
.
├── CLAUDE.md
├── DESIGN.md
├── src/
│   ├── __init__.py
│   ├── orchestrator.py      # Main orchestration logic
│   ├── agents/
│   │   ├── base.py          # AgentProtocol, BaseAgent
│   │   ├── research.py      # ResearchAgent implementation
│   │   ├── plan.py          # PlanAgent implementation
│   │   └── review.py        # ReviewAgent implementation
│   ├── models/
│   │   ├── subagent.py      # SubagentResponse dataclass
│   │   └── config.py        # Configuration schema
│   └── utils/
│       ├── logging.py       # Structured logging
│       └── cache.py         # Result caching (optional)
├── tests/
│   ├── test_orchestrator.py
│   ├── test_agents.py
│   └── fixtures.py          # Shared test data
├── examples/
│   ├── sequential_workflow.py
│   ├── parallel_workflow.py
│   └── hybrid_workflow.py
└── docs/
    ├── ARCHITECTURE.md
    └── API.md
```

---

### Large Project (50+ files)

Add per-domain directories:
```
src/
├── orchestration/           # Orchestrator + coordination logic
├── agents/                  # Subagent implementations
├── workflows/               # Workflow definitions (sequential, parallel, etc.)
├── storage/                 # Persistent state, caching
├── telemetry/               # Logging, tracing, monitoring
└── cli/                      # CLI interface (if applicable)
```

---

## 選択パターンのマトリックス (Pattern Selection Guide)

| Workflow Type | Pattern | Model | Notes |
|---|---|---|---|
| Single independent task | Headless | Sonnet 4.6 | Fastest, cheapest |
| Sequential tasks (A→B→C) | Orchestrator | Opus 4.7 | Main agent orchestrates |
| Parallel independent tasks | Split-and-Merge | Opus (main) + Sonnet (branches) | Max 5 parallel; merge results |
| Long-running, many tasks | Agent Teams | Opus 4.7 per agent | Use shared state store |
| Hybrid (some parallel, some sequential) | Orchestrator + Split-and-Merge | Mixed | Main orchestrator spawns parallel branches |

---

## 禁止パターン (Anti-Patterns)

### ❌ Anti-Pattern 1: Feature Flag Hell

```python
# Don't do this
if config.USE_PARALLEL_AGENTS:
    result = orchestrator.spawn_parallel(tasks)
else:
    result = orchestrator.spawn_sequential(tasks)
```

**Why**: Feature flags prevent code cleanup; code accumulates "legacy support"

**Instead**: Refactor directly, commit as separate feature branch

---

### ❌ Anti-Pattern 2: God Agent

```python
# Don't do this
class SuperAgent:
    def research(self): ...
    def plan(self): ...
    def execute(self): ...
    def review(self): ...
    def log(self): ...
    def cache(self): ...
    # 1000+ lines
```

**Why**: Violates Single Responsibility; untestable

**Instead**: Separate concerns into `ResearchAgent`, `PlanAgent`, etc.

---

### ❌ Anti-Pattern 3: Premature Parallelization

```python
# Don't do this (unless tasks are truly independent)
parallel_results = spawn_parallel([
    task_read_file,
    task_write_file,
    task_compile,
])
```

**Why**: Sequential is faster for dependent tasks; parallelization overhead > speed gain

**Instead**: Analyze dependencies first; parallelize only independent subtasks

---

### ❌ Anti-Pattern 4: Silent Failures

```python
# Don't do this
try:
    result = subagent.execute(task)
except Exception:
    return None  # Silent failure
```

**Why**: No audit trail; hard to debug

**Instead**: Log error with context, escalate to orchestrator

```python
try:
    result = subagent.execute(task)
except Exception as e:
    logger.error(f"Subagent failed: {subagent.id}, task: {task.id}, error: {e}")
    raise  # Let orchestrator handle
```

---

## Testing Strategy

### Unit Tests
- Test each agent independently (mock collaborators)
- Test orchestrator logic separately (mock subagents)

### Integration Tests
- Full workflow: Orchestrator + 2-3 subagents
- Test result merging (Split-and-Merge pattern)

### E2E Tests
- Real agents, real models (Sonnet/Opus)
- Single workflow end-to-end
- Measure: latency, cost, output quality

---

## References

- SOLID Principles: https://en.wikipedia.org/wiki/SOLID
- Design Patterns (Gang of Four): https://en.wikipedia.org/wiki/Design_Patterns
- Claude Subagent Patterns: See README.md §参考文献 (sources 4, 8, 9)
- Previous DESIGN docs: /20260427/DESIGN.md, /20260425/DESIGN.md
