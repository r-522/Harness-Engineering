# CLAUDE.md — Harness Engineering Operational Rules

**Project**: AI Agent Orchestration Platform | **Team**: 5-person advanced team | **Updated**: 2026-05-20 | **Lines**: 97

---

## Project Setup

**Tech Stack**: Python 3.10+ (PEP 8 + ruff + black), TypeScript (ESLint, Prettier, strict mode), YAML

**Directories**:
```
src/agents/         # Subagent definitions
src/skills/         # MCP integrations, reusable workflows
src/core/           # Shared libraries
tests/              # Unit, integration, E2E
config/             # settings.yaml, agents.yaml
.claude/            # Operational documentation (CLAUDE.md, AGENT.md, DESIGN.md, VERIFY.md)
```

**Entry Points**: See `DESIGN.md` for module dependency graph.

---

## Code Standards (Naming)

- **Variables/Functions**: `snake_case` (Python), `camelCase` (TypeScript)
- **Classes**: `PascalCase` | **Constants**: `UPPER_SNAKE_CASE`
- **Subagent**: `{role}_{objective}` (e.g., `researcher_competitor_features`)
- **Files**: `kebab-case.ts`, `snake_case.py`

**Code Quality**: Functions ≤ 30 lines, cyclomatic complexity ≤ 10. Comments: WHY only (WHAT is in naming).

---

## Testing & CI

**Run before push**:
```bash
pytest tests/ -v --cov=src                  # Unit + integration (require ≥ 80% coverage)
python -m pytest tests/integration/ -k orchestration  # Subagent orchestration tests
mypy src/                                   # Type checking
ruff check src/ && black --check src/       # Lint + format check
```

**CI/CD**: All tests + linting must pass. Tag `v*.*.* ` on main triggers auto-deploy.

---

## Workflow & Branching

**Plan Mode**: Use for non-trivial tasks. Write plan document, seek approval, execute.

**Branches**: `main` (production), `develop` (integration), `feature/*` (new features), `fix/*` (bugfixes), `agent/*` (subagent work)

**Commits**: `<type>: <subject>` where type ∈ {feat, fix, refactor, test, docs, perf}. Include issue reference: `Refs: #<number>`

---

## Subagent Orchestration (See AGENT.md for details)

**Default**: Max 3 parallel agents, 30K tokens each.

**Orchestrator Role**: Task decomposition, spawning, result integration (Opus 4.7).

**Worker Role**: Single-task execution (Sonnet 4.6).

**Escalation**: Context overrun (>30K per agent) → split task. API errors → retry up to 3× with exponential backoff. Data conflicts → manual verify.

---

## Strict Prohibitions

- ❌ `git push --force` (requires prior review)
- ❌ API keys in code (use `.env`, not version control)
- ❌ Subagent count > 10 (context explosion risk)
- ❌ MCP without security review (new integrations only via DESIGN.md)
- ❌ New Subagent when context < 10% available

---

## Auto-Permitted (No Review)

- ✅ Local file edits
- ✅ Test runs
- ✅ Doc updates (no code impact)
- ✅ `.local.md` personal notes

---

## Context & Tokens

**Effective context**: 170K tokens (clean session). With MCP: 120-130K.

**Rule**: CLAUDE.md loads every session. Keep persistent files lean. See `20260520/README.md` for evidence.

**Checkpoint**: `/context` command shows token allocation.

---

## References

- **Detailed Orchestration**: `.claude/AGENT.md` (orchestrator patterns, error handling)
- **Architecture Patterns**: `.claude/DESIGN.md` (module structure, dependency graph)
- **Verification Workflows**: `.claude/VERIFY.md` (testing, QA gate, CI integration)
- **Official Guidance**: https://code.claude.com/docs/en/best-practices
- **Harness Evidence**: `20260520/README.md` (source documentation, context trade-offs)

---

## Escalation

- **Technical questions**: See `.claude/DESIGN.md` architecture section
- **Orchestration issues**: See `.claude/AGENT.md` error handling
- **Security concerns**: Contact security@company.internal (per CLAUDE.md section 9)
- **MCP integration**: See `MCP-INTEGRATION.md`, then contact MCP owner
