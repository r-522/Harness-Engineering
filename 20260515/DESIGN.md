# DESIGN.md - Architecture & Design Principles

**Version**: 2026-05-15  
**Scope**: Harness Engineering System Architecture  
**Audience**: Architects, Senior Engineers, Subagent Designers

---

## 1. Core Design Principles

### SOLID Principles (Adapted for Agents)

#### S - Single Responsibility
- **Each subagent** handles one logical domain (research, code review, testing)
- **Each module** has one reason to change
- **Example**: `researcher_literature` searches academic papers; it doesn't validate data
- **Anti-Pattern**: Multi-purpose "do-everything" subagent (causes context bloat, hard to debug)

#### O - Open/Closed
- **Open for extension**: Add new skills via `SKILLS.md`without modifying core CLAUDE.md
- **Closed for modification**: Stable interfaces (Config via YAML frontmatter, fixed tool contracts)
- **Example**: Adding `data_validation` skill doesn't require redeploying orchestrator

#### L - Liskov Substitution
- **Subagents are interchangeable** if they fulfill same contract
- **Example**: `researcher_{arxiv|github|reddit}` should return results in same format
- **Test**: Can one subagent's output be used as another's input without modification?

#### I - Interface Segregation
- **Small, focused tool sets** per subagent (don't grant all tools to all agents)
- **Example**: Testing subagent gets `pytest` tool; review subagent gets `git-diff` tool
- **Benefit**: Security, reduced context burden, clear responsibility

#### D - Dependency Inversion
- **High-level** (orchestrator) depends on abstractions (subagent interface)
- **NOT** on low-level implementations (specific researcher, specific reviewer)
- **Mechanism**: YAML frontmatter contract (model, tools, system prompt structure)

### DRY - Don't Repeat Yourself
- **Reuse Skills**: Define once in `SKILLS.md`, trigger across multiple tasks
- **Reuse Patterns**: Common orchestration sequences (parallel batch, fallback chains)
- **Exception**: Config duplicates acceptable (redundancy for safety)

### KISS - Keep It Simple
- **Trade-off principle**: Prefer working simplicity over premature optimization
- **Corollary**: 3 duplicate lines > premature abstraction
- **Example**: Rather than generic "agent factory", define 3 concrete subagents

---

## 2. Subagent Orchestration Patterns

### Pattern 1: Orchestrator-Worker (Centralized)
```
         ┌─────────────────┐
         │   Orchestrator  │
         │  (Main Agent)   │
         └────┬────┬────┬──┘
              │    │    └─── Worker C
              │    └──────── Worker B
              └───────────── Worker A
```
**Use Case**: Strongly coordinated workflows (sequential dependencies)  
**Example**: Feature → Code Review → Tests → Merge  
**Pros**: Easy debugging, clear causality  
**Cons**: Orchestrator becomes bottleneck, single point of failure  
**Context Cost**: High (orchestrator tracks all worker outputs)

### Pattern 2: Swarm (Decentralized)
```
     ┌─────────┐     ┌─────────┐
     │  Agent A│◄───►│  Agent B│
     └────┬────┘     └────┬────┘
          │               │
          └──────┬────────┘
                 ▼
          (Emergent Results)
```
**Use Case**: Parallel brainstorming, exploration (Reddit/Twitter scans)  
**Example**: 5 researchers independently scan sources, merge findings  
**Pros**: Scalable, no bottleneck, explorative  
**Cons**: Results may conflict, harder to debug  
**Context Cost**: Low per agent (independent contexts)

### Pattern 3: Hierarchical (Tree-Structured)
```
           ┌──────────┐
           │    CEO   │
           └────┬─────┘
        ┌───────┼───────┐
        ▼       ▼       ▼
      [Eng]  [QA]   [Docs]
       ▲ ▲    ▲      ▲
      [B][F] [T]    [W]
```
**Use Case**: Large organizations (domain-specific orchestrators)  
**Example**: Feature team = Dev subagent + QA subagent; both report to team lead  
**Pros**: Scales to 10+ agents, clear reporting structure  
**Cons**: Complexity, context fragmentation, latency  
**Context Cost**: High (multi-level coordination)

### Pattern 4: Pipeline (Sequential Stages)
```
[Input] → [Stage 1] → [Stage 2] → [Stage 3] → [Output]
           (Research)  (Analysis)  (Report)
```
**Use Case**: Linear workflows with clear stages  
**Example**: Data ingestion → Validation → Processing → Visualization  
**Pros**: Simple, easy to test, monitor stage-by-stage  
**Cons**: Bottleneck at slow stage, no parallelization  
**Context Cost**: Medium (keep output from N-1 stage for N stage)

### Pattern 5: Fallback Chain
```
Try [Primary Solver]
  ├─ Success? → Return
  └─ Fail? → Try [Backup Solver A]
              ├─ Success? → Return
              └─ Fail? → Try [Backup Solver B]
                          └─ Return (default) or Escalate
```
**Use Case**: Resilience, graceful degradation  
**Example**: [GPT4 coder] → [GPT3.5 coder] → [Template generator]  
**Pros**: Reliability, cost optimization (use expensive model only if needed)  
**Cons**: Slower (sequential), more complex testing  
**Context Cost**: Medium (cache intermediate attempts)

---

## 3. Directory Structure & Rationale

```
harness-engineering/
├── .claude/                    # Claude Code configuration (always)
│   ├── CLAUDE.md              # This session's rules (80 lines max)
│   ├── AGENT.md               # Subagent persona & orchestration logic
│   ├── DESIGN.md              # This file (architectural decisions)
│   ├── SKILLS.md              # Reusable skill definitions
│   ├── MCP-INTEGRATION.md     # External API/service bindings
│   └── subagents/             # [Optional] Individual subagent definitions
│       ├── researcher.md
│       ├── reviewer.md
│       └── validator.md
│
├── src/                        # Source code
│   ├── agents/                # Subagent orchestration logic
│   │   ├── orchestrator.py    # Main coordinator
│   │   ├── patterns.py        # Reusable pattern templates
│   │   └── config.yaml        # Agent registry
│   ├── skills/                # Skill implementations
│   │   ├── research.py
│   │   ├── validate.py
│   │   └── deploy.py
│   ├── mcp/                   # MCP server integrations
│   │   ├── github_client.py
│   │   └── slack_notifier.py
│   └── core/                  # Shared utilities
│       ├── context_manager.py # Token budget tracking
│       ├── error_handler.py
│       └── logger.py
│
├── tests/                      # Test suite (mirror structure)
│   ├── agents/
│   │   ├── test_orchestration.py
│   │   └── test_patterns.py
│   ├── integration/
│   │   └── test_e2e_workflow.py
│   └── fixtures/              # Test data & mocks
│
├── config/                     # Configuration files
│   ├── settings.yaml          # Global settings (model, rate limits)
│   ├── agents.yaml            # Agent registry (one entry per subagent)
│   └── skills.yaml            # Skill catalog & trigger conditions
│
├── docs/                       # User/maintainer documentation
│   ├── QUICKSTART.md
│   ├── TROUBLESHOOTING.md
│   └── ARCHITECTURE.md
│
├── .gitignore                 # Exclude secrets, local configs
├── Makefile                   # Build targets (build, test, deploy)
└── README.md                  # Project overview
```

**Rationale**:
- **`.claude/` first**: Configuration visible at session start
- **`src/agents/` centralized**: Subagent logic isolated from skills/MCP
- **`tests/` mirrors `src/`**: Easy test discovery, 1:1 module mapping
- **`config/` external**: Non-code configuration stays separate (YAML, JSON)
- **`docs/` for humans**: Keep Markdown prose away from code

---

## 4. Module Coupling & Dependencies

### Dependency Graph (Desired State)
```
Skills ◄─── Subagents (low coupling)
        ◄─── MCP Integrations
        
Subagents ◄─── Orchestrator (fan-in pattern)
          ◄─── Error Handler
```

### Forbidden Patterns
❌ **Circular Dependencies**
- Skill A → Subagent B → Skill A = STOP

❌ **God Objects**
- One module importing 10+ others

❌ **Hard-Coded Secrets**
- API key in source code → Move to `.env` / `.local.md`

---

## 5. Testing Strategy

### Unit Tests (Fast)
- **Target**: Individual skills, utility functions
- **Tools**: pytest
- **Coverage Goal**: ≥ 80%
- **Example**: `test_research_skill.py` → Mock API, test query parsing

### Integration Tests (Medium)
- **Target**: Subagent orchestration, skill chaining
- **Tools**: pytest + fixtures
- **Scenarios**: 
  - Multi-agent parallel execution
  - Output format validation
  - Context budget tracking

### E2E Tests (Slow, valuable)
- **Target**: Full workflow (research → review → test → merge)
- **Tools**: pytest + live MCP servers
- **Trigger**: Pre-deployment only (long timeout)

### Regression Prevention
- Keep failing tests in Git history for 30+ days
- Document why each test was added (link to issue/PR)

---

## 6. Performance & Cost Optimization

### Model Selection per Subagent
| Task | Recommended Model | Reasoning |
|---|---|---|
| Research, synthesis | Opus 4.7 | High-context reasoning |
| Code review, reasoning | Opus 4.7 | Complex analysis |
| Validation, formatting | Sonnet 4.6 | Speed, cost |
| Brainstorming | Sonnet 4.6 | Exploratory, less depth |

### Context Budgeting
```
Total Context = 200k tokens
├─ Main Agent (Orchestrator): 50k
├─ Subagent A (Research): 50k
├─ Subagent B (Code Review): 50k
└─ Reserve: 50k (for compression, final synthesis)
```

### Caching Strategies
- **Git log**: Cache `git log` output (expensive), reuse across subagents
- **API responses**: Keep successful calls in session memory (1hr TTL)
- **Test results**: Memoize test runs if code unchanged

---

## 7. Security Model

### Trust Boundaries
```
[Trusted Code] ──┬─→ [MCP Server A] (HTTPS + auth token)
                 ├─→ [MCP Server B] (GitHub API + SSH)
                 └─→ [External LLM] (Claude API, on-prem)
```

### Secret Management
- **Forbidden**: Hardcoded API keys in `.py`, `.ts`, `.md`
- **Allowed**: Environment variables, `.local.md` (gitignored), SecretManager integrations

### Subagent Isolation
- Each subagent has **distinct tool permissions** (see CLAUDE.md §5)
- Subagent A (Researcher) can't execute `git push`; that's Orchestrator-only

---

## 8. Version Control & Deployment

### Git Workflow
```
main (stable, production)
  ↑
develop (integration, tested)
  ↑
feature/agent-name (development)
```

### Deployment Checklist
- [ ] All tests PASS (coverage ≥ 80%)
- [ ] Linters PASS (ruff, eslint)
- [ ] Security review complete (MCP changes)
- [ ] CLAUDE.md, AGENT.md updated
- [ ] Release notes drafted

---

## 9. Anti-Patterns & What NOT To Do

| Anti-Pattern | Why It Fails | Alternative |
|---|---|---|
| **Monolithic Subagent** | Can't parallelize, Context explosion | Decompose into focused agents |
| **Hardcoded Config** | Can't adapt, secrets leak | Use YAML + env vars |
| **No Testing** | Regressions hide until prod | pytest mandatory pre-commit |
| **Infinite Retries** | Hangs orchestrator | Bounded exponential backoff (§8) |
| **Shared Mutable State** | Race conditions in parallel agents | Immutable data, explicit message passing |

---

## 10. Decision Log

| Date | Decision | Rationale | Impact |
|---|---|---|---|
| 2026-05-15 | Max 3 parallel subagents | Tested threshold; beyond = coherence loss | Context optimization, latency budgets |
| 2026-05-15 | Opus 4.7 default, Sonnet 4.6 opt-in | Balance quality/cost | Speed: +30%, Cost: -20% |
| 2026-05-15 | CLAUDE.md ≤ 80 lines | Community best practice (HumanLayer, Anthropic) | Claude attentiveness ↑ |

---

**References**:
- [Microsoft AI Agent Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [AI Agent Orchestration Guide 2026](https://www.knowlee.ai/blog/ai-agent-orchestration-guide-2026)
- [Redis AI Agent Architecture Blog](https://redis.io/blog/ai-agent-architecture/)

**Last Updated**: 2026-05-15  
**Next Review**: 2026-08-15 (quarterly)
