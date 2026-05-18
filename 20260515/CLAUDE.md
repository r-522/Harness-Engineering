# CLAUDE.md - Harness Engineering Configuration

**Version**: 2026-05-15  
**Target**: AI Agent Orchestration Platform  
**Lines**: 78 (≤80 limit)

---

## 1. Project Context

**Name**: Harness Engineering  
**Stack**: Python 3.10+, TypeScript, YAML  
**Frameworks**: Claude Code, Claude API (Opus 4.7), Subagent SDK  
**Team Size**: Small team (~5)  
**Persona**: Advanced users requiring Plan mode, Context optimization, Orchestration

---

## 2. Coding Standards

### Naming Conventions
- **Functions/Variables**: `snake_case` (Python), `camelCase` (TS)
- **Classes**: `PascalCase`
- **Constants**: `UPPER_SNAKE_CASE`
- **Files**: `kebab-case.ts`, `snake_case.py`
- **Subagents**: `{role}_{objective}` (e.g., `researcher_literature`)

### Code Quality
- **Python**: PEP 8 + `ruff` + `black`
- **TypeScript**: ESLint + Prettier, `strict` mode required
- **Comments**: WHY only (names explain WHAT)
- **Cyclomatic Complexity**: ≤ 10
- **Function Length**: ≤ 30 lines ideal

---

## 3. Git Workflow

### Commit Format
```
<type>: <subject>
<blank line>
<body>
Refs: #<issue-number>
```
Types: `feat`, `fix`, `refactor`, `test`, `docs`, `perf`

### Branch Strategy
- `main` → production (protected)
- `develop` → integration
- `feature/*` → new features
- `agent/*` → subagent additions
- `fix/*` → bug fixes

### Pull Request Rules
1. Always Plan-mode first (AGENT.md)
2. Minimum 1 sync review
3. All tests + linters must pass

---

## 4. Subagent Rules

### Parallelization
**Use parallel**: Independent research, multiple API calls, context isolation  
**Use serial**: Strong dependencies, single-file edits, debug phases

### Resource Limits
- **Default max parallel**: 3
- **Context per agent**: Total budget ÷ 3
- **Max subagent count**: ≤ 10 (context exhaustion risk)

---

## 5. Forbidden & Restricted

### ❌ Strictly Forbidden
- `git push --force` (requires pre-review)
- Hardcoded API keys in code (use `.env`/`.local.md`)
- Subagent count > 10
- MCP server connections without security review
- New subagents with context < 10% remaining

### ⚠️ Requires Pre-Review
- New MCP integrations
- Agent behavior refactoring
- Package additions
- API specification changes

### ✅ Auto-Allowed
- Local file edits
- Test execution
- Documentation (non-code impact)
- Personal settings in `.local.md`

---

## 6. Testing & Build

```bash
# Unit tests
pytest tests/ -v --cov=src

# Integration (Subagent orchestration)
python -m pytest tests/integration/ -v -k "orchestration"

# E2E (MCP included)
pytest tests/e2e/ --mcp-live
```

**Pass Criteria**: Coverage ≥ 80%, all tests PASS

---

## 7. File References

- **Detailed Design**: DESIGN.md
- **Agent Behavior**: AGENT.md
- **Skills Definition**: SKILLS.md
- **MCP Integration**: MCP-INTEGRATION.md
- **Official Docs**: https://code.claude.com/docs/en/best-practices

---

## 8. Escalation Contacts

- **Technical Lead**: `@team-lead`
- **Security Issues**: Internal security team
- **MCP Support**: `@mcp-owner`
