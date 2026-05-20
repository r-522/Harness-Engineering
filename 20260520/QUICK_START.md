# QUICK START — Production Harness for Harness Engineering

**Last Updated**: 2026-05-20 | **Time to Read**: 5 min | **Files in This Harness**: 10

---

## What Is This?

Production-grade Claude Code configuration for the Harness Engineering team (AI Agent Orchestration Platform). Evidence-based decisions, operationally tested patterns, token-conscious design.

---

## File Guide

| File | Purpose | Read When |
|------|---------|-----------|
| **README.md** | Overview, evidence sources, key decisions | **First** (explains everything) |
| **CLAUDE.md** | Core operational rules (80 lines, loads every session) | **Before first session** |
| **AGENT.md** | Subagent orchestration patterns (120 lines) | Planning complex tasks |
| **DESIGN.md** | Architecture, module structure, implementation patterns | When implementing major features |
| **VERIFY.md** | Verification phase, testing, CI/CD gates | Before pushing code |
| **skills/** | Reusable workflows (test-runner, security-audit, etc.) | When you need `/test-runner`, `/security-audit` |
| **hooks/** | Automated checks at lifecycle points | Configuring pre-push gates |

---

## Copy These Files to Your Project

```bash
# From harness directory
cp -r 20260520/* .claude/

# Then commit
git add .claude/
git commit -m "refactor: apply production harness (20260520)"
git push
```

---

## First 3 Steps

### 1. Read CLAUDE.md (5 min)

**What you'll learn**:
- Tech stack for this project
- Testing commands (copy-paste ready)
- Naming conventions
- Prohibited actions
- Escalation paths

### 2. Review AGENT.md (8 min)

**What you'll learn**:
- When to use parallel subagents (max 3)
- Orchestrator vs. Worker roles
- Error handling & escalation
- Context budget allocation
- Plan mode workflow

### 3. Skim DESIGN.md & VERIFY.md (reference)

**Don't memorize**—just know they exist:
- DESIGN.md: Architecture, module structure, patterns
- VERIFY.md: Testing strategy, CI/CD gates, verification phase

**Reference these when**:
- Adding a new subagent type
- Setting up tests
- Debugging complex tasks

---

## Daily Workflow

### Simple Task (No Subagents)
1. Work in current session
2. Run tests before push: `pytest tests/ -v --cov=src`
3. Push + create PR

### Complex Task (Multiple Independent Pieces)
1. Use `/orchestration-plan` skill (or ask Claude: "let's plan this")
2. Review the plan (decomposition, budget, risks)
3. Approve or modify
4. Execute (Claude spawns subagents)
5. Verify results before merge

### Before Pushing
1. Run full test suite: `pytest tests/ -v --cov=src --cov-fail-under=80`
2. Run security audit: `bandit -r src/ -f json` (or use `/security-audit` skill)
3. Check for secrets: Don't commit `.env`, API keys, tokens
4. Push + create PR (CI runs full checks)

---

## Key Rules (From CLAUDE.md)

**Do**:
- ✅ Use Plan mode for complex tasks
- ✅ Max 3 parallel subagents (default)
- ✅ 30K tokens per subagent budget
- ✅ Test before push (≥80% coverage)
- ✅ Security audit before pushing secrets-sensitive code

**Don't**:
- ❌ Push with `--force` without prior review
- ❌ Commit API keys or `.env` files
- ❌ Spawn >10 subagents (context explosion)
- ❌ New MCP integration without security review
- ❌ Push when context < 10% remaining

---

## Commands You'll Use

**Testing**:
```bash
pytest tests/ -v --cov=src --cov-fail-under=80        # Full suite
pytest tests/unit/ -v                                   # Unit only
pytest tests/ -k "orchestration" -v                     # Pattern match
```

**Formatting/Linting**:
```bash
black src/ tests/                                       # Auto-format
ruff check src/ --fix                                   # Auto-fix lint
mypy src/ --strict                                      # Type check
```

**Security**:
```bash
bandit -r src/ -f json                                  # Security scan
grep -r "password\|api_key" src/                        # Find secrets
safety check --json                                     # Dependency check
```

**Git**:
```bash
git branch -a                                           # List branches
git checkout -b feature/<name>                          # New feature
git push -u origin feature/<name>                       # Push new branch
gh pr create --draft                                    # Create draft PR
```

---

## Skills (Reusable Workflows)

### Available in This Harness

1. **test-runner**: Run pytest, report coverage, flag drops
   - Use: `/test-runner` before push
   - Time: ~45s

2. **security-audit**: Scan for secrets, SQL injection, unsafe patterns
   - Use: `/security-audit` before push
   - Time: ~30s

3. **orchestration-plan**: Decompose complex task, show subagent distribution
   - Use: `/orchestration-plan` for parallel work
   - Time: ~60s

4. **code-review**: Check PR against coding standards
   - Use: `/code-review` during review phase
   - Time: ~120s

---

## Context Budget Reality (2026)

**You have ~170K tokens** to work with in a clean session.

- **5K**: System prompt + tool schemas
- **5K**: Persistent instructions (.claude/*.md)
- **20K**: Typical task conversation
- **140K**: Available for actual work

**Never**: Load massive files unnecessarily. Use `/context` to audit token usage.

---

## Troubleshooting

### Tests Pass Locally But Fail in CI

**Cause**: Python version mismatch, missing dependency.

**Fix**:
```bash
python --version                       # Must be 3.10+
pip install -r requirements.txt        # Update dependencies
pytest tests/ -v                       # Rerun
```

### "Context Window at 85%"

**Cause**: Too many tools loaded or long conversation.

**Fix**:
1. Use `/clear` to start fresh session
2. Use `/context` to see what's consuming space
3. Create subagent to offload work

### Security Audit Finds Secrets

**Cause**: Hardcoded API key, password in code.

**Fix**:
1. Remove from code immediately
2. Add to `.env` file (git-ignored)
3. Load with `os.getenv("VAR_NAME")`
4. Rerun audit to confirm

### Subagent Spawning Fails

**Cause**: Context budget exceeded, task too large.

**Fix**:
1. Split task into smaller pieces
2. Run sequentially instead of parallel
3. Review AGENT.md error handling section

---

## Evidence & Sources

This harness is based on:
- **Official Anthropic guidance** (code.claude.com/docs)
- **Proven operational patterns** (repos with 100+ stars, recent commits)
- **Community best practices** (2026 articles, blogs, conferences)

See **README.md** for full source documentation and confidence levels.

---

## Next Steps

1. **Copy harness to .claude/**
   ```bash
   cp -r 20260520/* .claude/
   ```

2. **Commit**
   ```bash
   git add .claude/ && git commit -m "refactor: apply production harness"
   ```

3. **Run test**
   ```bash
   pytest tests/ -v --cov=src
   ```

4. **Read CLAUDE.md** in context (so Claude learns your rules)

---

## Questions?

- **Orchestration**: See **AGENT.md** section "Error Handling & Escalation"
- **Testing**: See **VERIFY.md** "Testing Integration"
- **Architecture**: See **DESIGN.md** "Module Dependency Graph"
- **Skills**: See **skills/SKILL_INDEX.md**
- **Hooks**: See **hooks/HOOKS_GUIDE.md**

---

## Harness Meta

**Created**: 2026-05-20  
**Based on**: Claude Code research (24h-72h window, May 2026)  
**Team Target**: 5-person advanced team  
**Focus**: Reliability, Verification, Maintainability, Context Efficiency  
**Token Budget**: Persistent files designed for 170K effective context  

**Versioning**: Future updates → new date directory (20260521/, 20260522/, etc.)
