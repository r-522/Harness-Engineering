# ORCHESTRATION.md — Subagent Orchestration & Cost Optimization

**Updated**: 2026-05-16 | **Audience**: Harness Engineering Team  
**Key Metric**: 5-10× cost reduction via model stratification

---

## 1. Orchestration Fundamentals

### What is Orchestration?

**Orchestration** is the automated coordination of multiple AI agents working in parallel toward a shared goal. Each agent:
- Has a **specific role** (researcher, coder, reviewer, deployer)
- Owns a **distinct concern** (code analysis, test coverage, security audit, release)
- Runs in **isolated context** (own token budget, tool whitelist)
- Returns **structured output** (JSON schema, markdown, metrics)

**Orchestrator** is the conductor agent (expensive, reasoning-heavy) that:
1. Receives the user task
2. Decomposes it into subagent tasks
3. Spawns subagents in parallel
4. Aggregates results + handles failures
5. Synthesizes final output

### Why Orchestrate?

| Benefit | Mechanism |
|---|---|
| **Cost reduction (5-10×)** | Cheap models (Haiku) handle execution; expensive (Opus) handles reasoning only |
| **Context isolation** | Each subagent gets fresh context window; no token exhaustion |
| **Parallelism** | 3 independent tasks run simultaneously vs. sequentially (3× speedup) |
| **Resilience** | If one subagent fails, others continue; orchestrator can retry or skip |
| **Scalability** | Add new subagents without modifying orchestrator core |

---

## 2. Model Stratification Strategy

### Principle: Expensive for Reasoning, Cheap for Execution

The **orchestrator-subagent pattern** splits work by cognitive load:

```
┌─────────────────────────────────────────────────────────┐
│ Task: "Review Python repo, check test coverage, audit   │
│ security, then suggest deployment strategy"             │
└──────────────┬──────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────┐
│ ORCHESTRATOR (Claude Opus 4.7)  ← REASONING HEAVY       │
│ ├─ Parse natural language intent                        │
│ ├─ Decompose into 4 logical subagent tasks              │
│ ├─ Route tasks to cheapest viable model                 │
│ ├─ Monitor progress, handle failures                    │
│ └─ Synthesize final strategy (complex reasoning)        │
│ Cost: ~$0.10 (8k input + 500 output tokens)             │
└────────────────┬─────────────────────────────────────────┘
                 │
      ┌──────────┼──────────┬─────────────┐
      ▼          ▼          ▼             ▼
    ┌──────┐  ┌──────┐  ┌──────┐  ┌──────────┐
    │ SA1  │  │ SA2  │  │ SA3  │  │ SA4      │
    │Code  │  │Test  │  │Sec   │  │Strategy  │
    │Analy │  │Cover │  │Audit │  │Draft     │
    │sis  │  │      │  │      │  │          │
    │(H)  │  │(H)   │  │(H)   │  │(S)       │
    └──┬───┘  └──┬───┘  └──┬───┘  └──┬───────┘
       │         │        │         │
       └─────────┼────────┴─────────┘  ← All exec in parallel
                 ▼
    ┌────────────────────────────────────┐
    │ ORCHESTRATOR (Final synthesis)      │
    │ Cost: ~$0.05 (3k input + 200 output)│
    └────────────────────────────────────┘

Total Cost: $0.10 + 4×$0.015 + $0.05 = ~$0.16
Equivalent 4x Sonnet: ~$0.30
SAVINGS: 47% reduction
```

### Model Choice by Task Type

| Task | Cognitive Load | Recommended Model | Token Budget | Cost/run |
|---|---|---|---|---|
| Code analysis (read, summarize) | Low | Haiku 4.5 | 2k-4k | ~$0.01 |
| Test coverage extraction | Low | Haiku 4.5 | 2k-3k | ~$0.008 |
| Security audit (grep patterns) | Low-Medium | Haiku 4.5 | 2k-4k | ~$0.01 |
| Architecture reasoning | Medium-High | Sonnet 4.6 | 4k-8k | ~$0.03 |
| Strategy synthesis (complex logic) | High | Opus 4.7 | 6k-12k | ~$0.10 |
| Orchestration + decomposition | High | Opus 4.7 | 8k-16k | ~$0.12 |

**Rule**: Default to **Haiku**, escalate to **Sonnet** only if task involves cross-domain reasoning, escalate to **Opus** only for final synthesis or complex optimization.

---

## 3. Subagent Definition Template

### YAML Format (config/agents.yaml)

```yaml
subagents:

  # Low-cost executor: code analysis
  researcher_code:
    model: claude-haiku-4-5-20251001
    description: >
      Analyze Python codebase. Extract architecture patterns,
      list all functions/classes, identify code smells.
    tools:
      - read          # Read files from project
      - bash          # grep, find, tree, wc
    tools_deny:
      - write         # Read-only!
      - edit
      - git
      - mcp_github
    system_prompt: |
      You are a code analyst. Your job:
      1. Read the target codebase structure
      2. List all modules, classes, functions with brief descriptions
      3. Identify code smells (long functions >50 lines, high CC, dead code)
      4. Return JSON with keys: {modules, functions, smells, total_lines}
      
      Be precise. Return ONLY valid JSON. No markdown, no commentary.
    timeout_seconds: 120
    max_retries: 2
    cost_estimate_tokens:
      input: 3000
      output: 500

  # Low-cost executor: test coverage
  reviewer_tests:
    model: claude-haiku-4-5-20251001
    description: >
      Measure test coverage, identify untested code paths.
    tools:
      - read
      - bash
    system_prompt: |
      You are a QA specialist. Your job:
      1. List all test files and their target modules
      2. Count test cases per module
      3. Estimate coverage % (lines with at least one test)
      4. Return JSON: {modules, test_count, coverage_percent, gaps}
    timeout_seconds: 90
    max_retries: 2
    cost_estimate_tokens:
      input: 2500
      output: 400

  # Medium-cost: security review
  reviewer_security:
    model: claude-haiku-4-5-20251001  # Patterns are simple; Haiku suffices
    description: >
      Audit code for security vulnerabilities (injection, auth, crypto).
    tools:
      - read
      - bash
    system_prompt: |
      You are a security auditor. Check for:
      1. Hardcoded secrets (keys, passwords, tokens)
      2. Injection vectors (SQL, shell, XSS)
      3. Weak crypto (MD5, DES, weak RNG)
      4. Missing auth/authz checks
      5. Unsafe deserialization (pickle, eval)
      
      Return JSON: {findings, severity_high_count, severity_med_count, severity_low_count}
    timeout_seconds: 120
    max_retries: 2
    cost_estimate_tokens:
      input: 3000
      output: 400

  # Higher-cost: strategy synthesis
  strategist_deployment:
    model: claude-sonnet-4-6  # Reasoning-heavy; use Sonnet
    description: >
      Synthesize deployment strategy based on code analysis,
      test coverage, security audit. Output actionable plan.
    tools:
      - read         # May need to read config files
    system_prompt: |
      You are a DevOps architect. You receive inputs from 3 analysts:
      {code_analysis, test_coverage, security_audit}
      
      Synthesize a deployment strategy that:
      1. Addresses all security findings before release
      2. Ensures >80% test coverage critical paths
      3. Recommends deployment pipeline (staging → prod)
      4. Identifies blockers and risks
      
      Return Markdown with sections: Readiness, Risks, Action Items, Timeline.
    timeout_seconds: 120
    max_retries: 2
    cost_estimate_tokens:
      input: 6000
      output: 800

  # Top-level orchestrator
  orchestrator_main:
    model: claude-opus-4-7          # Complex decomposition; expensive
    description: >
      Main orchestrator. Parse user task, spawn subagents,
      aggregate results, synthesize final output.
    tools:
      - read
      - bash
      - agent (for Task spawning)
    system_prompt: |
      You are the orchestrator. Your workflow:
      1. Parse the user task
      2. Break into ≤4 subagent tasks
      3. Spawn subagents in parallel (use Task tool)
      4. Wait for all results
      5. Aggregate into coherent output
      
      Subagents available:
      - researcher_code: Extract codebase structure, smells
      - reviewer_tests: Measure test coverage, gaps
      - reviewer_security: Audit security vulnerabilities
      - strategist_deployment: Synthesize deployment strategy
      
      Return final output as Markdown report.
    timeout_seconds: 300  # Longer; waits for subagents
    max_retries: 1
    cost_estimate_tokens:
      input: 10000
      output: 1000
```

---

## 4. Orchestration Workflow (Step-by-Step)

### Phase 1: Task Reception & Decomposition

**Input**: User request (natural language)

```
User: "Review my Python repo before deploying to production.
       I'm worried about test coverage and security."
```

**Orchestrator action**:
1. Parse intent: "Full pre-deployment review"
2. Decompose into subagent tasks:
   - `researcher_code`: Analyze codebase structure + patterns
   - `reviewer_tests`: Measure test coverage + gaps
   - `reviewer_security`: Audit for vulnerabilities
   - `strategist_deployment`: Synthesize deployment readiness

### Phase 2: Parallel Spawning

**Orchestrator spawns 3 subagents simultaneously**:

```python
results = await asyncio.gather(
    spawn_subagent("researcher_code", 
                   task="Analyze /home/user/myproject"),
    spawn_subagent("reviewer_tests",
                   task="Check test coverage in /home/user/myproject/tests"),
    spawn_subagent("reviewer_security",
                   task="Audit security in /home/user/myproject/src"),
)
```

**Timeline**:
- Without orchestration: 120s + 90s + 120s = 330s (sequential)
- With orchestration: max(120s, 90s, 120s) = 120s (parallel) → **2.75× faster**

### Phase 3: Result Aggregation

**Orchestrator waits for all subagents, then synthesizes**:

```
Code Analysis (from SA1):
├─ 47 files, 12 modules
├─ 234 functions, 18 classes
├─ Code smells: 5 long functions (>80 lines)
└─ Total: 8,234 lines

Test Coverage (from SA2):
├─ Test files: 23
├─ Test cases: 156
├─ Coverage: 68% (below 80% target)
└─ Gaps: Auth module untested, error handlers

Security Audit (from SA3):
├─ Findings: 3 HIGH (hardcoded secrets), 2 MED (weak hashing)
├─ No injection vectors detected
└─ Recommendation: Fix HIGH before deploy

[Strategist synthesizes into final report...]
```

### Phase 4: Final Synthesis

**Orchestrator (with strategist input) produces**:

```markdown
# Pre-Deployment Review Report

## Status: ⚠️ NOT READY

### Blockers (Fix Before Deploy)
1. **HIGH**: Remove 3 hardcoded API keys from `config/secrets.py`
2. **MED**: Replace MD5 hashing with SHA256 in `auth.py`
3. **HIGH**: Test coverage below 80% (currently 68%)

### Recommended Actions
- Fix security findings (1 day)
- Add tests for auth + error paths (2 days)
- Re-run full review (1 day)
- Deploy to staging (1 day)

### Timeline
- Ready for production: May 23, 2026 (assuming fixes start today)

[Full details...]
```

---

## 5. Cost Accounting & Optimization

### Token Cost Tracking

Track cost per run:

```python
def track_cost(subagent_name, model, input_tokens, output_tokens):
    """Log token usage and cost"""
    pricing = {
        "claude-haiku-4-5": {"input": 0.80/M, "output": 4.00/M},
        "claude-sonnet-4-6": {"input": 3.00/M, "output": 15.00/M},
        "claude-opus-4-7": {"input": 15.00/M, "output": 75.00/M},
    }
    cost = (input_tokens * pricing[model]["input"] + 
            output_tokens * pricing[model]["output"]) / 1e6
    log(f"{subagent_name}: {input_tokens}in + {output_tokens}out = ${cost:.4f}")
    return cost
```

### Monthly Cost Baseline (Example)

Assuming 20 code reviews per month:

| Scenario | Models | Avg Cost/Review | Monthly |
|---|---|---|---|
| **Single Opus** | 1× Opus | $0.24 | $4.80 |
| **Orchestrated** | 1× Opus + 3× Haiku | $0.07 | $1.40 |
| **Savings** | — | **71%** | **71%** |

---

## 6. Failure Handling in Orchestrated Workflows

### Common Failures & Recovery

| Failure | Impact | Recovery |
|---|---|---|
| **Subagent timeout** | 1 result missing | Retry subagent OR skip + use default |
| **Subagent error** (invalid JSON | Undecodable output | Re-prompt with stricter schema |
| **All subagents timeout** | Complete workflow failure | Escalate to user; offer degraded mode |
| **Network latency** (slow API | One subagent slow | Others continue; aggregate partial results |

### Retry Strategy (Per Subagent)

```yaml
subagent_retry_policy:
  max_attempts: 3
  backoff: exponential  # 1s, 2s, 4s
  
  recoverable_errors:
    - timeout
    - rate_limit
    - json_parse_error
  
  non_recoverable:
    - permission_denied
    - file_not_found
    - invalid_task
```

---

## 7. Monitoring & Observability

### Metrics to Track

```yaml
metrics:
  - orchestration_latency: Time from user request to final output (target: <2min)
  - subagent_success_rate: % of subagents succeeding (target: >95%)
  - cost_per_review: Total cost aggregated (target: <$0.10)
  - model_mix: % Opus vs Sonnet vs Haiku (target: >80% Haiku)
  - context_utilization: Avg tokens used vs budget (target: <70%)
```

### Logging Template

```json
{
  "timestamp": "2026-05-16T14:23:00Z",
  "workflow_id": "review_pr_42",
  "orchestrator": "claude-opus-4-7",
  "subagents": [
    {"name": "researcher_code", "status": "success", "tokens_in": 3000, "tokens_out": 500, "duration_s": 45},
    {"name": "reviewer_tests", "status": "success", "tokens_in": 2500, "tokens_out": 400, "duration_s": 38},
    {"name": "reviewer_security", "status": "success", "tokens_in": 3000, "tokens_out": 400, "duration_s": 52}
  ],
  "total_cost": "$0.068",
  "total_duration_s": 52,
  "final_output_quality": "high"
}
```

---

## 8. When NOT to Orchestrate

**Don't use subagent orchestration if:**
- ❌ Task < 5 minutes runtime (overhead > benefit)
- ❌ High interdependency (result of task A feeds into task B → sequential)
- ❌ Single expert needed (one Opus better than many Haikus)
- ❌ Requires full context (orchestrator overhead + communication cost)

**Use simple orchestration if:**
- ✅ Task easily parallelizable (independent analyses)
- ✅ Cost matters (Haiku << Opus)
- ✅ Latency matters (parallel > sequential)

---

**Orchestration review required for: new subagent additions, model changes, token budget adjustments.**
