# DESIGN.md — Architecture & Design Principles

**Updated**: 2026-05-16 | **Scope**: Harness Engineering Platform

---

## 1. Core Design Principles

### SOLID Principles (Applied to Agentic Systems)

#### S — Single Responsibility
Each Subagent owns **one concern** (research, code generation, validation, deployment).
- ❌ Don't: Agent that researches AND codes AND tests
- ✅ Do: Separate `researcher_code`, `generator_implementation`, `reviewer_quality`

#### O — Open/Closed
Agents are **open for extension** (new skills, new tools) but **closed for modification** (core persona unchanged).
- ✅ Add new Skills in `.claude/skills/`
- ❌ Don't modify `/AGENT.md` thinking process for edge cases

#### L — Liskov Substitution
All Subagents conform to the same **interface contract**: input → processing → output.
- Contract: `(task, context, constraints) → (result, metadata, cost)`
- Result: Orchestrator can swap models (Opus ↔ Sonnet) without breaking flow

#### I — Interface Segregation
Agents expose **only relevant tools**. Each subagent has a whitelist (no tool bloat).
- Example: `researcher_*` has {WebSearch, Read}, no {Write, Bash, Git}
- Example: `developer_*` has {Read, Edit, Write, Bash}, no {WebSearch, MCP}

#### D — Dependency Inversion
Agents depend on **abstractions** (tool interfaces), not implementations.
- Abstraction: "data fetch capability" (implemented by WebSearch or local Read)
- Benefit: Swap implementations without redeploying agents

### DRY (Don't Repeat Yourself)
- **Shared logic**: Move to `.claude/core/` utilities (logging, error handling, retry)
- **Configuration**: Centralize in `config/agents.yaml` (model choices, tool whitelist, timeouts)
- **Skills**: Reusable instruction packages in `.claude/skills/`

---

## 2. Directory Structure & Module Design

```
Harness-Engineering/
├── .claude/                          # Claude-specific config (shared with team)
│   ├── CLAUDE.md                     # Project rules (non-negotiable)
│   ├── AGENT.md                      # Agent persona & process
│   ├── DESIGN.md                     # This file
│   ├── ORCHESTRATION.md              # Subagent orchestration guide
│   ├── skills/                       # Reusable Skills (SKILL.md format)
│   │   ├── orchestrate-subagents/
│   │   ├── review-security/
│   │   └── measure-cost/
│   ├── hooks/                        # GitHub/git integration (optional)
│   │   ├── pre-commit.sh
│   │   └── session-start.yaml
│   └── settings.json                 # Claude Code permissions & env vars
│
├── src/
│   ├── agents/                       # Subagent definitions (YAML)
│   │   ├── researcher_code.yaml      # Code analysis, documentation
│   │   ├── generator_implementation.yaml  # Generate code stubs
│   │   ├── reviewer_quality.yaml     # Code review, test coverage
│   │   └── deployer_release.yaml     # Deploy, tag, notify
│   │
│   ├── core/                         # Shared libraries (utils, logging, retry)
│   │   ├── orchestrator.py           # Subagent spawn + aggregate
│   │   ├── cost_tracker.py           # Token + cost accounting
│   │   ├── error_handler.py          # Retry, escalation, fallback
│   │   └── __init__.py
│   │
│   ├── mcp/                          # MCP integrations (if used)
│   │   ├── github_mcp.py
│   │   ├── web_fetch_mcp.py
│   │   └── credentials.local.md      # GITIGNORED
│   │
│   └── main.py                       # Entry point / orchestration dispatcher
│
├── config/
│   ├── agents.yaml                   # Model choices, tool whitelists, timeouts
│   ├── settings.yaml                 # Environment defaults
│   └── secrets.local.yaml             # GITIGNORED — API keys, tokens
│
├── tests/
│   ├── agents/                       # Test each subagent behavior
│   │   ├── test_researcher_code.py
│   │   ├── test_generator_impl.py
│   │   └── test_reviewer_quality.py
│   │
│   ├── integration/                  # Test subagent orchestration
│   │   ├── test_orchestration_flow.py
│   │   ├── test_cost_tracking.py
│   │   └── test_error_recovery.py
│   │
│   ├── fixtures/                     # Mock data, sample tasks
│   │   ├── sample_tasks.json
│   │   └── mock_api_responses.json
│   │
│   └── conftest.py                   # pytest configuration
│
├── docs/
│   ├── ARCHITECTURE.md               # High-level system design
│   ├── DEPLOYMENT.md                 # Release process
│   └── CONTRIBUTING.md               # Contribution guide
│
├── .gitignore                        # Ignore secrets, .local files
├── pyproject.toml                    # Python dependencies, black/ruff config
├── package.json                      # TypeScript deps (if applicable)
└── Makefile                          # Build targets (test, lint, deploy)
```

---

## 3. Subagent Decomposition Strategy

### How to Decide: When to Create a New Subagent

**Create a new subagent if:**
1. **Distinct concern** — Task is logically separate (e.g., code review != code generation)
2. **Different tool access** — Needs different tools (researcher needs WebSearch; developer needs Write)
3. **Scalability** — Task can run in parallel with others
4. **Cost optimization** — Can use cheaper model (Haiku instead of Opus)

**Don't create a subagent if:**
- ❌ Task is small (< 5 min run time)
- ❌ High interdependency with previous step (sequential only)
- ❌ Requires full project context (orchestrator overhead > benefit)

### Template: Subagent YAML

```yaml
name: researcher_code
description: >
  Analyze codebase, identify patterns, extract documentation.
  Returns markdown summary of architecture + test coverage.

model: claude-haiku-4-5  # Cheap; no complex reasoning needed
tools:
  - read           # Read files
  - bash           # grep, find, tree commands
  - # NO write, edit, git — read-only

system_prompt: |
  You are a code analyst specialist. Your job:
  1. Read the target codebase
  2. Identify architectural patterns (MVC, microservices, etc.)
  3. Extract test coverage metrics
  4. Summarize in Markdown (max 500 words)
  
  Be precise. Prefer data over opinion.

timeout_seconds: 120
max_retries: 2
```

---

## 4. Data Flow & Orchestration Pattern

### Orchestrator-Subagent Flow

```
┌─────────────────────────┐
│  User Task Request      │
│  (e.g., "review PR")    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────────────────┐
│  ORCHESTRATOR (Opus, expensive)     │
│  ├─ Parse task intent               │
│  ├─ Decompose into subagent tasks   │
│  ├─ Estimate token cost             │
│  └─ Spawn subagents (parallel)      │
└───────────┬─────────────────────────┘
            │
      ┌─────┼─────┐
      ▼     ▼     ▼
    ┌─────┐ ┌─────┐ ┌─────────────┐
    │SA1  │ │SA2  │ │SA3          │
    │Code │ │Test │ │Security     │
    │Anal │ │Cov  │ │Audit        │
    │Haiku│ │Haiku│ │Haiku        │
    └──┬──┘ └──┬──┘ └──────┬──────┘
       │       │           │
       └───┬───┴─────┬─────┘
           ▼         ▼
    ┌────────────────────────────┐
    │ ORCHESTRATOR (Aggregate)   │
    │ ├─ Combine results         │
    │ ├─ Fill gaps / retry       │
    │ ├─ Track cost              │
    │ └─ Synthesize output       │
    └────────────┬───────────────┘
                 ▼
            ┌──────────┐
            │User      │
            │Output    │
            └──────────┘
```

**Cost Benefit**:
- Orchestrator (Opus): ~2 min run time, ~8000 input + ~500 output tokens → ~$0.10
- 3 × Subagent (Haiku): ~1 min each, ~2000 input + ~300 output tokens each → ~$0.01 total
- **Total**: ~$0.11 vs. 4× Sonnet (~$0.30) = **63% cost reduction**

---

## 5. Error Handling & Recovery

### Retry Strategy (Exponential Backoff)

```python
def retry_with_backoff(fn, max_attempts=4, base_delay=2):
    """Exponential backoff: 2s, 4s, 8s, 16s"""
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError as e:
            if attempt == max_attempts - 1:
                raise
            delay = base_delay ** attempt
            sleep(delay)
    raise FatalError("Max retries exceeded")
```

**Applies to**: Git operations (push, pull), API calls, Subagent spawning

### Graceful Degradation

| Failure | Response | Fallback |
|---|---|---|
| Subagent timeout | Retry with extended timeout | Skip subagent, use orchestrator |
| API rate limit | Exponential backoff | Cache previous result |
| File write permission | Report error, don't retry | Escalate to user |
| Code generation error | Re-prompt with constraints | Use template skeleton |

---

## 6. Testing Strategy

### Unit Tests (Per Subagent)
- Input: Task definition + context
- Expected: Output matches schema, metadata complete, cost tracked
- **Coverage**: ≥80% per subagent

### Integration Tests (Orchestration)
- Spawn 3 parallel subagents
- Aggregate results
- Verify no race conditions, correct cost accounting
- **Pass criteria**: All subagents succeed or fail gracefully

### E2E Tests (End-to-End)
- Full workflow: user request → orchestrator → subagents → output
- Simulate real constraints (limited context, network latency)
- **Monthly**: Cost regression testing (ensure 5-10x savings maintained)

---

## 7. Versioning & Deprecation

### Semantic Versioning
- **MAJOR**: Breaking change (agent interface, tool removal)
- **MINOR**: New skill, new subagent, new cost optimization
- **PATCH**: Bug fix, documentation update

### Deprecation Policy
- Announce in CHANGELOG.md (3 versions ahead)
- Update docs / migration guide
- Remove only after 3 minor versions

---

**Design review required for: new MCP integrations, subagent count > 5, major orchestration pattern changes.**
