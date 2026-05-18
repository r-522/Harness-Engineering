# CLAUDE.md — Harness Engineering Production Configuration

**Updated**: 2026-05-16 | **Lines**: 71 | **Token Budget**: ~140/200

---

## 1. Project Identity

**Name**: Harness Engineering - AI Agent Orchestration Platform  
**Stack**: Python 3.10+, TypeScript, YAML  
**Framework**: Claude Code, Claude Agent SDK, Subagent Orchestration  
**Team**: Small (〜5), Advanced (Plan mode, context optimization, multi-agent required)

---

## 2. Code Standards

### Naming
- **Variables/Functions**: `snake_case` (Python), `camelCase` (TypeScript)
- **Classes**: `PascalCase`
- **Constants**: `UPPER_SNAKE_CASE`
- **Subagents**: `{role}_{objective}` (e.g., `researcher_code`, `optimizer_perf`)

### Quality Checks
- **Python**: PEP 8, `ruff` + `black` auto-format
- **TypeScript**: ESLint strict mode + Prettier
- **Comments**: WHY only (WHAT is naming)
- **Function length**: ≤30 lines
- **Cyclomatic complexity**: ≤10

---

## 3. Restrictions & Policies

### 🚫 Forbidden
- `git push --force` (pre-review required)
- Hardcoded API keys (use `.env` / `.local.md`)
- Subagent count > 5 (context exhaustion risk)
- MCP connections without security review
- Context remaining < 10% when spawning new subagents

### ⚠️ Pre-Review Required
- New MCP integrations
- Large agent behavior changes
- Package additions / dependencies
- API spec changes

### ✅ Auto-Allowed
- Local file edits
- Test execution
- Documentation (non-code)
- `.local.md` personal settings

---

## 4. Testing & Build

```bash
# Unit tests
pytest tests/ -v --cov=src

# Integration tests (Subagent orchestration)
python -m pytest tests/integration/ -v -k "orchestration"

# Coverage requirement: ≥80%, all PASS
```

---

## 5. Orchestration Rules

### Parallel Execution (Use When)
- Multiple independent research tasks
- Batch external API calls
- Context separation needed

### Sequential Execution (Use When)
- Strong dependencies between steps
- Single file editing
- Debug / verify phase

### Max Parallel Agents
- **Default**: 3 concurrent subagents
- **Budget**: token_budget ÷ 3 per subagent
- **Override**: Only with explicit pre-review

---

## 6. References

- Claude Code Docs: https://code.claude.com/docs/en/best-practices
- Subagent Guide: https://code.claude.com/docs/en/sub-agents
- Agent SDK: https://www.ksred.com/the-claude-agent-sdk-what-it-is-and-why-its-worth-understanding/
- Cost Optimization: https://designbeep.com/2026/05/02/claude-code-agent-teams-how-to-orchestrate-ai-subagents-for-real-development-work/
