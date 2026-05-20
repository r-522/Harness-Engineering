# AGENT.md — Subagent Orchestration Guide

**Purpose**: Define orchestrator/worker personas, orchestration patterns, error handling. | **Updated**: 2026-05-20 | **Lines**: 115

---

## Personas

### Orchestrator (Opus 4.7)

- **Role**: Decompose tasks, spawn subagents, integrate results
- **Context Budget**: 40K tokens (40% of 100K total budget)
- **Responsibility**: Pure coordination—never execute tasks directly
- **Decision Logic**: Parallel vs. sequential, timeout policies, escalation

### Worker Subagent (Sonnet 4.6, max 3 parallel)

- **Role**: Execute single tasks, report findings
- **Context Budget**: 30K tokens each
- **Error Strategy**: Retry up to 3×, then escalate
- **Timeout**: 300s per task (tools: Bash, Read, Edit, WebSearch, WebFetch)

---

## Orchestration Patterns

### Sequential (Dependent Phases)

Use when output of Phase N feeds Phase N+1:

```
Plan (Orchestrator) → Phase 1 (Worker A) → Phase 2 (Worker B) → Merge
```

**Example**: Codebase analysis → generate design → validate design → merge

### Parallel Fan-Out (Independent Tasks)

Use for independent work:

```
Plan → [Worker A, Worker B, Worker C] (parallel) → Merge
```

**Example**: Competitor research (3 firms × 1 worker each)

### Validation Chain

Execute → Verify (deterministic checks) → Reconcile:

```
LLM Output → Run Tests → Compare Results → Report Deltas
```

**Example**: Code generation → run unit tests → flag failures

---

## Error Handling & Escalation

| Error Type | Strategy | Decision |
|---|---|---|
| **Context Overrun** (>30K per agent) | Split task further | Orchestrator auto-detects |
| **Tool Failure** (API, network) | Exponential backoff (max 3×) | Subagent auto-retries |
| **Data Conflict** (results mismatch) | Manual reconciliation | Orchestrator reports + ask user |
| **Permission Denied** (auth failure) | Request user permission | UI prompt (blocking) |
| **Timeout** (>300s per task) | Escalate + propose fallback | Orchestrator decision |

**Escalation Flow**:
```
Subagent Error
  ├─ Recoverable? → retry ×3 with backoff
  ├─ Critical (system-blocking)? → stop + user notification
  └─ Otherwise → escalate with recommendation + fallback
```

---

## Context Budget Allocation

**Total**: 100K tokens (for 3-parallel orchestration)

```
Orchestrator: 40K
  ├─ Task decomposition & control: 20K
  ├─ Integration & reconciliation: 15K
  └─ Error handling & escalation: 5K

Worker A: 30K (independent task A)
Worker B: 30K (independent task B)
Worker C: Held in reserve (trigger when A or B completes)
```

**Compression**: If running low, move old results to disk; use file references instead of inline.

---

## Performance Targets

| Metric | Target | Measurement |
|---|---|---|
| Subagent Success Rate | ≥ 95% | (successful / total spawned) |
| Avg Response Time | ≤ 120s | Wall-clock per task |
| Token Efficiency | ≥ 80% | (valuable output / tokens consumed) |
| Context Usage | ≤ 85% | (used / allocated budget) |

---

## Forbidden Patterns

- ❌ Subagent spawning subagents (subagents can't orchestrate)
- ❌ Parallel count > 10 (token explosion risk)
- ❌ Passing API keys to subagents (orchestrator holds secrets)
- ❌ Subagent-to-subagent data passing (route through orchestrator)
- ❌ Infinite retry loops (max 3 attempts, then escalate)

---

## Plan Phase (Before Execution)

When spawning subagents, show user:

```markdown
## Orchestration Plan

**Task**: [decomposed goal]
**Complexity**: [low/med/high]
**Estimated Duration**: XXs
**Estimated Cost**: $X.XX

## Subagent Distribution
1. **researcher_A**: [task] (20K ctx)
2. **executor_B**: [task] (30K ctx)
3. [optional]

## Execution Strategy
- **Parallelization**: A, B simultaneous
- **Merge Strategy**: Combine result1 + result2 → final_output
- **Risk Mitigations**:
  - API rate limit → throttle requests
  - Context overrun → monitor per-agent budget
  
**Proceed?** [Y/n]
```

---

## Safe Implementation Checklist

```python
orchestrator_config = {
    "max_parallel": 3,
    "context_budget_per_agent": 30_000,
    "timeout_seconds": 300,
    "retry_policy": {
        "max_attempts": 3,
        "backoff_multiplier": 2,
        "max_backoff_seconds": 16
    },
    "escalation_threshold": 0.80  # Alert at 80% context
}
```

---

## See Also

- **DESIGN.md**: Architecture patterns, module dependencies
- **VERIFY.md**: Verification phase, testing integration
- **CLAUDE.md**: Core operational rules, prohibited actions
