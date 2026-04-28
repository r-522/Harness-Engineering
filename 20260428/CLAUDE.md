# CLAUDE.md — Project Configuration

## WHAT: Project Identity

**Harness-Engineering**: Claude Code AI orchestration research & framework.

- **Tech Stack**: Python 3.11+, Claude API (Opus 4.7/Sonnet 4.6), MCP servers
- **Primary Goal**: Build optimized subagent orchestration patterns for 2026
- **Repository**: r-522/harness-engineering (GitHub private)
- **Branch**: `claude/keen-curie-uR96n` (development branch)

---

## WHY: Design Constraints & Values

1. **Context Efficiency**: Token usage optimization via hierarchical config, <200-line CLAUDE.md, lazy-loaded subagents
2. **Parallel Execution**: Support Split-and-Merge + Agent Teams patterns for complex multi-step workflows
3. **Deterministic Planning**: Plan Mode enforced; research phase separated from implementation phase
4. **Auditability**: Every generated file must trace back to a verified information source (no hallucination)

---

## HOW: Coding Standards & Practices

### File Organization

```
.
├── 20260428/                          # Daily harness snapshot
│   ├── README.md                      # This date's overview & sources
│   ├── CLAUDE.md                      # Project rules (you are here)
│   ├── AGENT.md                       # Agent persona & workflow
│   ├── DESIGN.md                      # Design patterns
│   └── ORCHESTRATION.md               # Subagent config
├── .claude/
│   ├── settings.json                  # Project settings (checked in)
│   ├── settings.local.json            # Personal settings (gitignored)
│   └── skills/                        # Custom skill definitions
└── src/                               # Implementation code (as needed)
```

### Naming Conventions

- **Files**: `lowercase-with-hyphens.py`, `CamelCase.ts`, `UPPERCASE_CONSTANTS.md`
- **Functions**: `snake_case()`
- **Classes**: `PascalCase`
- **Subagent IDs**: `agent-purpose-version` (e.g., `research-agent-v2`)

### Code Standards

- **Python**: Black formatter (line length 88), type hints required
- **Markdown**: GFM (GitHub Flavored), max 100 chars per line (docs), no emoji unless explicitly requested
- **Git**: Atomic commits, descriptive messages, reference tracking via `Sources:` blocks
- **Comments**: None unless WHY is non-obvious (hidden constraint, workaround, subtle invariant)

---

## 禁止事項 (Forbidden Actions)

### Git Operations
- ❌ `git reset --hard` (destructive reset)
- ❌ `git push --force` to main/master (loss of history)
- ❌ `git commit --amend` on published commits
- ❌ Committing to branches other than `claude/keen-curie-uR96n` without explicit approval

### File Operations
- ❌ Deleting dated subdirectories (20260421 等) without version control review
- ❌ Writing to `.claude/settings.local.json` from automated scripts (personal only)
- ❌ Committing secrets, API keys, credentials (use `.gitignore` + MCP secrets management)

### Implementation Practices
- ❌ Feature flags for incomplete implementations (ship working features only)
- ❌ Backwards-compatibility shims (refactor directly)
- ❌ Error handling for impossible scenarios (trust framework guarantees)
- ❌ Premature abstractions (DRY after 3+ identical occurrences, not before)

### Planning & Execution
- ❌ Skipping Plan Mode (use `/plan` command before implementation)
- ❌ Spawning 3+ subagents sequentially (use Split-and-Merge pattern instead)
- ❌ Large documentation blocks in CLAUDE.md (max 200 lines total)

---

## エージェント動作ポリシー (Agent Behavior Policy)

### Plan Mode (Entry Point)

1. **Clarify Intent**: If user request is ambiguous, ask for specifics (one sentence max)
2. **Research Phase**: Use Explore/Plan agents to gather codebase info, design space
3. **Present Plan**: Structured steps with file paths, line numbers, architectural decisions
4. **User Approval**: Wait for explicit consent before implementation begins

### Execution Phase (Act)

- Use dedicated tools (Read, Edit, Write) over Bash when applicable
- Parallelize independent tool calls; batch dependent calls sequentially
- Update user every 2-3 steps with brief (one-sentence) status
- Never commit without explicit user request

### Verification Phase (Verify)

- Run type checkers, linters, tests before reporting completion
- For UI changes: manual testing in browser (golden path + edge cases)
- Check for regressions in related features before merging
- Provide reproducible test commands

### Escalation Rules

- **Ambiguous Requirements**: Ask user to clarify
- **Risky Operations** (force-push, delete branches): Confirm before acting
- **Context Overload** (approaching token limit): Spawn specialized Agent with isolation
- **Unexpected State**: Investigate root cause, don't delete/overwrite

---

## Tool & Permission Matrix

| Tool | Permission | Condition |
|---|---|---|
| **Read** | auto | All files except `.env`, `credentials.json` |
| **Write** | auto | Config files, implementation code, tests |
| **Edit** | auto | Existing files only (after Read) |
| **Bash** | ask | Network ops (git push), system changes |
| **Agent** | auto | Subagent delegation (Research/Plan/Review) |
| **TodoWrite** | auto | Multi-step task tracking (3+ steps) |
| **WebSearch** | ask | Only for sources that must be recent (2026+) |

---

## Subagent Configuration

**Default Subagent Pool**:

| Subagent | Role | Model | Trigger |
|---|---|---|---|
| `research-v2` | Codebase exploration, design research | Sonnet 4.6 | "explore", "find", "where" queries |
| `plan-v2` | Architecture design, implementation planning | Opus 4.7 | Multi-step tasks, architectural decisions |
| `review-v2` | Code review, QA, security audit | Sonnet 4.6 | Pre-commit verification, PR review |
| `orchestrator-v1` | Subagent coordination, split-merge | Opus 4.7 | Large parallel tasks, complex workflows |

---

## Context Management Rules

- **CLAUDE.md Max**: 200 lines (you are at ~180, acceptable margin)
- **Session Window**: Reserve 10k tokens for verification phase
- **Tool Calls**: Max 5 independent calls per response (batch limit)
- **Agent Isolation**: Run heavy research via Agent with `isolation: "worktree"` to avoid context bleed

---

## References

- Anthropic Claude Code Docs: https://code.claude.com/docs
- Latest Best Practices: See `README.md` §参考文献
- Previous Config: `/20260427/CLAUDE.md`, `/20260425/CLAUDE.md` (history)
