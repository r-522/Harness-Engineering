# Project Definition: Claude Code Harness Engineering

## Tech Stack & Architecture

**Primary Language**: Markdown + YAML  
**Execution Environments**: Claude Code (Terminal, Web, VS Code, JetBrains, Desktop)  
**Configuration Format**: JSON (`settings.json`, `settings.local.json`)  
**Version Control**: Git (feature branches mandatory)  
**Key Tools**: MCP servers (GitHub, Bash, WebFetch), Custom Skills  

This project applies AI agent harness design patterns to maximize Claude Code capabilities across diverse software engineering tasks.

## Project Structure

```
.
├── .claude/                    # Claude Code configuration
│   ├── CLAUDE.md              # This file (auto-loaded per session)
│   ├── AGENT.md               # Agent behavior definition
│   ├── DESIGN.md              # Design principles
│   └── HARNESS.md             # Harness engineering patterns
├── .git/                       # Version control
├── settings.json              # Global Claude Code settings
└── [project-specific content]
```

### Important Conventions

- **Branch naming**: `feature/*`, `bugfix/*`, `hotfix/*` (never push to `main`/`master` without explicit permission)
- **Commit style**: Imperative, present tense ("Add feature", not "Added"), reference GitHub issues when applicable
- **Feature flags**: No feature flags unless explicitly required; prefer direct code changes
- **Error handling**: Validate only at system boundaries (user input, external APIs); trust internal code guarantees

## Coding Standards

### Naming Conventions

- **Variables/Functions**: `camelCase` (JavaScript) or `snake_case` (Python, Bash)
- **Classes/Types**: `PascalCase`
- **Constants**: `UPPER_SNAKE_CASE` (only for immutable, module-level values)
- **Boolean variables**: Prefix with `is`, `has`, `can`, `should` (e.g., `isActive`, `hasPermission`)

### Code Style Rules

- Default to **no comments** unless the WHY is non-obvious (hidden constraint, workaround, subtle invariant)
- Do NOT document WHAT the code does—use clear naming instead
- Do NOT reference the current task or issue in comments ("used by X flow", "added for Y")
- Prefer editing existing files over creating new ones
- No backwards-compatibility hacks; delete unused code cleanly

### File Organization

- **Markdown files** (`*.md`): Plain UTF-8, trailing newline required
- **YAML configs**: 2-space indentation, no tabs
- **JSON files**: Prettier-compatible formatting where applicable
- **Script files**: Shebang on line 1, executable bit set (`chmod +x`)

## Forbidden Operations

### Destructive Actions (Require Explicit User Approval)

❌ **Do NOT execute** without explicit user confirmation:
- `git reset --hard`
- `git push --force`
- `rm -rf` or similar destructive deletions
- Amending published commits (instead: create new commit)
- Modifying shared infrastructure/permissions

### Tool/Command Restrictions

❌ **Prohibited by default**:
- `git rebase -i` (interactive mode not supported)
- `git stash` with uncommitted sensitive changes (risk of loss)
- Executing shell commands with `eval()` or similar dynamic execution
- Writing to `.env` or files containing secrets without security review

### Code Generation Rules

❌ **Anti-patterns to avoid**:
- Premature abstractions (if fewer than 3 duplicates, keep as-is)
- Comment-heavy docstrings (one-line max)
- Half-finished implementations or TODO placeholders
- Re-exporting types unnecessarily
- Feature flags for single-use toggles

## Session Initialization Protocol

Every Claude Code session automatically loads this `CLAUDE.md` file. Upon session start:

1. **Context Check**: Verify context usage with `/status`; clean at 80% threshold
2. **Scope Confirmation**: If task scope is unclear, use Plan Mode to separate research from execution
3. **Reference Lookup**: Consult `.claude/AGENT.md` for behavior guidelines and skill definitions
4. **Constraint Review**: Re-confirm coding standards, forbidden operations, and file policies

## Testing & Validation

### Required Before Commit

- [ ] **Type checking**: `typescript` projects run `tsc --noEmit`; Python projects use `mypy` if configured
- [ ] **Linting**: Run project-specific linter (ESLint, Pylint, Clippy) and fix issues
- [ ] **Tests**: Execute `npm test`, `pytest`, `cargo test`, or equivalent; achieve baseline coverage
- [ ] **Build verification**: `npm run build`, `cargo build`, or equivalent must succeed
- [ ] **Manual feature test**: Use browser/CLI to verify golden path + edge cases (if UI-based)

### Test Execution Commands

Include project-specific test commands below (examples):

```bash
# Front-end/Node projects
npm test                    # Unit & integration tests
npm run lint               # ESLint/Prettier
npm run build              # Build production artifacts

# Python projects
pytest                     # Run test suite
mypy .                     # Type checking
black --check .            # Code formatting

# Rust projects
cargo test                 # Run all tests
cargo clippy               # Linter
cargo build --release      # Optimized build
```

## File Editing Policy

### Safe to Edit Directly

✅ **Automatic approval** for:
- Implementing new features (single-file changes)
- Bug fixes in existing modules
- Updating documentation, examples, comments
- Configuration changes (within project-specific bounds)

### Requires Confirmation

⚠️ **Must confirm before editing**:
- Cross-file refactoring affecting >5 files
- Changes to core architecture or shared utilities
- Deletions of unused but historically significant modules
- Modifications to CI/CD pipelines or build systems

## Context Management

### Clearing Context Efficiently

When approaching 80% context threshold:

```bash
/clear-all                 # Clear session context (keeps CLAUDE.md)
# OR
claude code --session-id <id> --clear-context
```

### Preferred Tool Usage

- Use **Read** (not `cat`) for file inspection
- Use **Edit** (not `sed`) for targeted modifications
- Use **Bash** for shell-only operations (piping, system calls)
- Use **Agent** (Subagent) for parallel, independent tasks

## MCP Integration

The following MCP servers may be configured in `settings.json`:

- **github** (GitHub API, PR/issue operations)
- **bash** (Local command execution)
- **web-fetch** (URL content retrieval)
- **web-search** (Internet search, requires allowlist)
- **[Custom MCPs]**: Document in HARNESS.md if project-specific

## Communication Style

- **Tone**: Direct, professional, action-oriented
- **Error Reporting**: Include file:line_number references for navigation
- **Updates**: Provide brief status updates at key milestones (found, changed direction, blocked)
- **Brevity**: One sentence per status update; avoid narration of internal deliberation

## Git Workflow Checklist

Before pushing any changes:

```bash
git status                 # Verify correct files staged
git diff --cached         # Review changes to be committed
git log --oneline -5      # Check recent commit history
git push -u origin <branch>  # Push to feature branch only
```

**Never** merge to `main`/`master` directly. Always use pull requests with review.

## References

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Writing a good CLAUDE.md - HumanLayer](https://www.humanlayer.dev/blog/writing-a-good-claude-md)

---

**Last Updated**: 2025-05-03  
**Target Audience**: Medium-level developers using Claude Code for multi-agent orchestration
