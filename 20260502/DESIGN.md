# DESIGN.md: Design Principles & Architecture Patterns

## I. Core Design Principles (SOLID + Pragmatic)

### 1. Single Responsibility Principle (SRP)
**Definition**: Each agent/skill handles ONE primary concern.

**Application in Harness**:
- `Agent::Dev` → Feature implementation only (no testing, no deployment)
- `Skill::security-review` → Security scanning only (not style review)
- `Hook::pre-commit` → Lint + type-check only (not full test suite)

**Anti-Pattern**: "Jack-of-all-trades" agent doing implementation, review, AND deployment.

**Check**: If skill description contains "and", it likely violates SRP.

---

### 2. Open/Closed Principle (OCP)
**Definition**: System open for extension (new agents/skills) but closed for modification (existing interfaces).

**Application**:
- **New Skill**: Add to `AGENT.md` without modifying core logic
- **New Agent**: Register in `settings.json` without rewriting CLAUDE.md
- **New Hook**: Extend `HOOKS.md` without touching CI/CD config

**Extension Point**: Use `@imports` in CLAUDE.md to reference external docs (not embedding).

---

### 3. Liskov Substitution Principle (LSP)
**Definition**: Any Agent/Skill must be replaceable without breaking system.

**Application**:
- All agents must respect common interface: (task → result | error)
- All skills must return structured output (JSON-like shape)
- Subagent Teams agents must write to shared task state in consistent format

**Example**: If `Agent::Dev` is replaced with a newer version, downstream `Agent::Review` must work unchanged.

---

### 4. Interface Segregation Principle (ISP)
**Definition**: Don't force agents/skills to implement interfaces they don't need.

**Application**:
- Read-only agent (code reviewer) → No Write, Edit, Bash tools needed
- Code-writing agent → Full tool access (Read, Write, Edit, Bash, Bash)
- Deployment agent → GitHub + Bash tools only (no code modification tools)

**Tool Grouping** (see `settings.json`):
```
- minimal: Read, Bash (status checks)
- reviewer: Read, mcp__github__ (PR operations)
- writer: Read, Write, Edit, Bash, Glob (code changes)
- deployer: Bash, mcp__github__ (deployment + notification)
```

---

### 5. Dependency Inversion Principle (DIP)
**Definition**: Depend on abstractions, not concrete implementations.

**Application**:
- Skills define **trigger patterns** (abstraction) instead of "invoke when user types X" (concrete)
- Hooks define **events** (push, commit, PR) instead of hardcoded command sequences
- Subagent Teams communicate via **shared state schema** (abstraction), not peer-to-peer messages

**Example**: Instead of `Agent::Dev` calling `Agent::Deploy` directly, both write to shared task state.

---

### 6. DRY (Don't Repeat Yourself)
**Definition**: Single source of truth for each piece of knowledge.

**Application in Harness**:
- **Shared Logic**: Extract to Skill or reusable hook script
- **Knowledge**: If mentioned in >2 files, create dedicated doc (e.g., `docs/stripe-guide.md`)
- **Configuration**: Use `settings.json` for all tool permissions (not CLAUDE.md)

**Anti-Pattern**: Duplicating error handling across 3 agents (extract to Skill or wrapper script).

---

### 7. Pragmatic Rule: YAGNI (You Aren't Gonna Need It)
**Definition**: Don't over-engineer for hypothetical future requirements.

**Application**:
- **No Premature Abstraction**: 2 similar agents can exist; wait for 3rd before extracting base class
- **No Feature Flags**: If requirements are unclear, ask first; don't build both code paths
- **No Unused Interfaces**: Skill that no agent invokes → delete it (test coverage will catch)

**Threshold**: 3+ code instances/similar patterns → consider extraction. 1-2 instances → leave alone.

---

## II. Architectural Patterns

### Pattern 1: Subagent Teams (Distributed Execution)

**When to Use**:
- Complex multi-step workflows (implementation → review → deployment)
- Long-running operations requiring parallelism
- Specialized expertise across different agents

**Structure**:
```
User Request
    ↓
┌─────────────────────────────┐
│  Coordinator Agent          │
│  (Break down task)          │
└──────────────┬──────────────┘
               │
    ┌──────────┼──────────┐
    ↓          ↓          ↓
Shared State (Task Board)
    ↑          ↑          ↑
    │          │          │
  ┌──┐       ┌──┐       ┌──┐
  │A1│       │A2│       │A3│
  └──┘       └──┘       └──┘
 (Dev)      (Review)   (Deploy)
```

**Shared State Schema** (Example):
```json
{
  "tasks": [
    { "id": "impl-1", "status": "done", "agent": "dev", "output": "src/auth.ts" },
    { "id": "review-1", "status": "in_progress", "agent": "review", "depends_on": ["impl-1"] },
    { "id": "test-1", "status": "pending", "agent": "test", "depends_on": ["impl-1"] }
  ],
  "shared_context": {
    "branch": "feat/auth",
    "pr_url": "https://github.com/.../pull/42"
  }
}
```

**Handoff Rules**:
- Agent reads task state, claims unclaimed task (`status: pending` → `status: in_progress`)
- Agent executes, writes result (`output`, `logs`)
- Next agent reads result, acts on dependency
- Coordinator monitors for failures/deadlocks

---

### Pattern 2: Skills (Lazy-Load Reference Material)

**When to Use**:
- Long reference docs (Stripe API, PostgreSQL schema)
- Complex workflows (Git advanced operations, performance profiling)
- Context-sensitive operations (conditional skill loading)

**Structure**:
```
CLAUDE.md (50-70 lines)
  ├─ @AGENT.md (Skill triggers, summary)
  └─ "For deep Stripe integration, invoke skill `stripe-guide`"
       ↓ (loaded only when skill invoked)
  [Large Stripe guide, 500+ lines]
```

**Invocation Flow**:
1. User mentions "Stripe API refund"
2. Claude detects trigger pattern → checks AGENT.md
3. Matches to `stripe-payment-skill` → loads full body
4. Executes with full context, no CLAUDE.md bloat

---

### Pattern 3: Hooks (Event-Driven Automation)

**When to Use**:
- Automated quality gates (pre-commit linting)
- CI/CD integration (auto-test on push)
- Notification & escalation (failed deploy → Slack)

**Hook Types**:
| Hook | Trigger | Action | Tool Access |
|---|---|---|---|
| `pre-commit` | Git staging | Lint + type-check | Bash, Git |
| `pre-push` | Git push attempt | Full test suite | Bash, npm |
| `post-merge` | Branch merged to main | Trigger CI + notify | Bash, GitHub API |
| `workflow-error` | CI job fails | Log + escalate | Bash, Slack API |

**Error Handling in Hooks**:
- **Blocking hooks** (pre-commit): Fail fast, prevent bad code from staging
- **Non-blocking hooks** (post-merge): Log error, notify team, don't block workflow

---

## III. Directory Structure & Layering

### Recommended Project Layout

```
project-root/
├── .claude/
│   ├── CLAUDE.md              # Session constraints (50-70 lines)
│   ├── AGENT.md               # Agent personas, skill definitions
│   ├── DESIGN.md              # This file
│   ├── HOOKS.md               # Hook definitions & scripts
│   ├── agents/
│   │   ├── dev.md             # Dev agent specialized prompt (optional)
│   │   ├── review.md          # Review agent prompt (optional)
│   │   └── deploy.md          # Deploy agent prompt (optional)
│   ├── skills/
│   │   ├── security-review.md # Skill body (auto-loaded when invoked)
│   │   ├── git-advanced.md
│   │   └── performance-audit.md
│   ├── logs/
│   │   └── hooks.log          # Hook execution logs
│   └── settings.json          # Tool permissions, MCP config
│
├── src/
│   ├── core/                  # Core business logic (SRP: each file ~250 lines)
│   │   ├── auth.ts
│   │   ├── orchestration.ts   # Subagent Teams coordination
│   │   └── skill-registry.ts  # Skill invocation dispatch
│   ├── agents/                # Agent-specific logic
│   │   ├── dev-agent.ts
│   │   ├── review-agent.ts
│   │   └── deploy-agent.ts
│   ├── hooks/                 # Hook implementations
│   │   ├── pre-commit.sh
│   │   └── post-merge.sh
│   └── types/
│       ├── task-state.ts      # Shared task state schema (single source of truth)
│       └── agent.ts           # Agent interface definitions
│
├── tests/
│   ├── unit/                  # SRP: each agent/skill has dedicated test file
│   ├── integration/           # Subagent Teams workflow tests
│   └── e2e/                   # End-to-end hook + deployment tests
│
├── docs/
│   ├── ADR/                   # Architecture Decision Records
│   │   ├── 0001-agent-teams-over-subagent.md
│   │   └── 0002-skill-lazy-load.md
│   ├── guides/
│   │   ├── stripe-integration.md   # Lazy-loaded via skill
│   │   └── git-workflows.md        # Lazy-loaded via skill
│   └── ARCHITECTURE.md        # High-level overview (this file)
│
├── scripts/
│   ├── setup-hooks.sh         # Initialize hooks from HOOKS.md
│   └── monitor-team.ts        # Subagent Teams monitoring CLI
│
├── .github/workflows/
│   ├── test.yml               # PR tests (hook-triggered)
│   ├── deploy.yml             # Production deployment (hook-triggered)
│   └── security-scan.yml      # SAST + dependency scan
│
└── package.json / pyproject.toml

```

### Layering Rules

**Layer 1: Core Abstraction** (`src/types/`)
- Task state schema
- Agent interface (input/output contract)
- Skill registry (lookup table)

**Layer 2: Business Logic** (`src/core/`)
- Orchestration engine (task scheduling, dependency resolution)
- Skill dispatcher (match trigger pattern → load body → execute)
- Error aggregation (collect failures across agents)

**Layer 3: Agents & Hooks** (`src/agents/`, `src/hooks/`)
- Agent implementations (leverage Layer 2 orchestration)
- Hook scripts (call Layer 2, log results)

**Layer 4: Integration** (`tests/integration/`, `.github/workflows/`)
- Multi-agent workflows (Layer 3 agents + shared state)
- CI/CD integration (hooks + external systems)

**Rule**: Lower layers don't depend on higher layers. Agents can call orchestration; orchestration doesn't call agents.

---

## IV. Code Organization by Concern

| Concern | Responsible File(s) | Principle Applied |
|---|---|---|
| Agent behavior | `AGENT.md`, `src/agents/*.ts` | SRP (each agent = one role) |
| Skill definitions | `AGENT.md` (summary), `src/skills/*.md` (body) | OCP (extensible) |
| Shared state | `src/types/task-state.ts` | DIP (abstract schema) |
| Tool permissions | `.claude/settings.json` | ISP (tool groups by capability) |
| Hook automation | `HOOKS.md`, `src/hooks/*.sh` | SRP (each hook = one trigger) |
| Error handling | `src/core/error-handler.ts` | DRY (centralized) |

---

## V. Testing Strategy (Aligned with SRP)

### Unit Tests (Per-Agent)
```
tests/unit/agent-dev.test.ts       # Dev agent behavior
tests/unit/agent-review.test.ts    # Review agent behavior
tests/unit/skill-security.test.ts  # Security review skill
```

**Focus**: Individual agent/skill logic, input validation, error responses.

### Integration Tests (Workflow)
```
tests/integration/teams-workflow.test.ts
```

**Focus**: Multi-agent handoff, task state transitions, dependency resolution.

### E2E Tests (Hook + Deployment)
```
tests/e2e/pre-commit-hook.test.ts  # Verify linting hook blocks bad code
tests/e2e/deploy-workflow.test.ts  # Verify deployment agent chains correctly
```

**Focus**: Hook trigger → action → result, external system integration.

---

## VI. Extensibility & Customization

### Adding a New Agent
1. Define in `AGENT.md` (summary)
2. Create `src/agents/new-agent.ts` (implementation)
3. Register in `settings.json` (tool permissions)
4. Add unit tests in `tests/unit/agent-new.test.ts`
5. Add workflow test if part of Teams pattern

### Adding a New Skill
1. Define trigger pattern in `AGENT.md`
2. Create `src/skills/new-skill.md` (body) if >200 lines
3. Reference in CLAUDE.md with `See src/skills/new-skill.md when ...`
4. Test via `/skills` command in Claude Code
5. Update DESIGN.md if architectural significance

### Adding a New Hook
1. Define in `HOOKS.md` (trigger + action)
2. Create `src/hooks/hook-name.sh` (script)
3. Register in `settings.json` (`hooks.hook-name`)
4. Add E2E test in `tests/e2e/`
5. Document in docs/ADR if modifies workflow

---

## Version & Alignment

- **Last Updated**: 2026-05-02
- **Principles**: SOLID + Pragmatic YAGNI
- **Patterns**: Agent Teams, Skills lazy-load, Hooks automation
- **Aligned With**: CLAUDE.md (constraints), AGENT.md (personas)

