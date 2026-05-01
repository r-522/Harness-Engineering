# CLAUDE.md - Harness Engineering Project Configuration

**Project**: Harness Engineering AI Orchestration  
**Stack**: Claude Code + MCP Servers + Subagents  
**Team Size**: 1-5 members  
**Updated**: 2026-04-30

---

## 1. Project Context

This project implements a Claude Code "harness" – a configuration system for orchestrating AI agent workflows, managing MCP servers, and optimizing token/cache usage. The primary artifact is automated generation of configuration documents based on real-time web trend analysis.

**Core Goals:**
- Auto-generate best-practice Claude Code settings based on latest trends
- Implement prompt caching strategies for cost reduction (~70% savings possible)
- Enable MCP server integration with typed tool access
- Define subagent orchestration patterns for parallel task execution

---

## 2. Code Style & Naming Conventions

### Markdown Documents
- **Line length**: 100 characters max (exception: code blocks, URLs)
- **Headings**: Use `#` level 1 for top-level only; h2-h6 for sections
- **Lists**: Use `-` for unordered, `1.` for ordered; indent nested items 2 spaces
- **Code blocks**: Wrap in triple backticks with language specifier (e.g. ```json)
- **Links**: Use relative paths within repo; external URLs must include protocol

### JSON Configuration Files
```json
{
  "comment": "Keys in UPPER_SNAKE_CASE for constants, camelCase for objects",
  "description": "Each object has 'type' and 'required' fields in comments",
  "indentation": 2
}
```

### YAML Frontmatter (Agent Definitions)
```yaml
---
name: agent-name
type: agentic-reasoner
model: claude-opus-4-7
description: Single-line description
tools:
  - Read
  - Write
  - mcp:server-name:tool-name
---
# Markdown body becomes system prompt
```

### Variable Naming
- **Config keys**: `SCREAMING_SNAKE_CASE` (e.g., `MAX_CONTEXT_TOKENS`)
- **File paths**: lowercase with hyphens (e.g., `.claude/agents/analyze-code.md`)
- **MCP tool refs**: `namespace:tool-name` (e.g., `mcp:github:search-code`)

---

## 3. Prohibited Actions & Constraints

### Git Operations
❌ **NEVER:**
- Force-push to `main` or `master` without explicit approval from repository owner
- Commit to branches other than `claude/keen-curie-xtGUj` without prior agreement
- Merge branches without a passing CI check
- Rebase public commits (use merge if already pushed)

✅ **DO:**
- Create new commits rather than amending published ones
- Use `git pull origin <branch>` before pushing changes
- Reference issue numbers in commit messages (e.g., `Fixes #123`)

### File Editing
❌ **NEVER:**
- Edit `.gitignore` without documenting why in commit message
- Modify `settings.local.json` and commit it (it's in .gitignore for a reason)
- Delete test files or configuration files without replacing them
- Remove or re-export types marked with `// @internal`

✅ **DO:**
- Use relative imports; avoid absolute paths
- Keep CLAUDE.md under 60 lines by archiving old instructions to separate docs
- Document workarounds with comments only if the reason is non-obvious

### Security
❌ **NEVER:**
- Commit API keys, tokens, or credentials (use environment variables)
- Run `sudo` commands without explicit user approval
- Disable security checks (e.g., `--no-verify`, `--no-gpg-sign`)
- Fetch from untrusted external MCP servers without inspection

✅ **DO:**
- Use `settings.local.json` for personal API keys
- Scan sensitive files with `mcp__github__run_secret_scanning` before commit
- Validate MCP server definitions before enabling in settings.json

---

## 4. Testing & Verification

### Before Committing
1. **Markdown Linting**: Validate all .md files have proper frontmatter
   ```bash
   grep -l "^---$" *.md | while read f; do echo "Checking $f"; done
   ```
2. **JSON Validation**: Parse all .json files
   ```bash
   find . -name "*.json" -exec jq empty {} \;
   ```
3. **YAML Validation**: Check agent definitions
   ```bash
   find .claude/agents -name "*.md" -exec sh -c 'head -20 "$1" | grep "^---$"' _ {} \;
   ```

### Build & Documentation Generation
```bash
# No explicit build step; verification is via file structure
# To verify configuration consistency:
ls -la .claude/
ls -la CLAUDE.md AGENT.md DESIGN.md
```

### Manual Testing Checklist
- [ ] Run `/init` in project and compare output with CLAUDE.md (should be compatible)
- [ ] Test MCP server connections with: `settings.json` lists correct servers
- [ ] Verify subagent definitions load without YAML errors
- [ ] Confirm all URLs in README.md are valid (no trailing slashes on domains)

---

## 5. Development Workflow

### Local Setup
```bash
git clone https://github.com/r-522/harness-engineering.git
cd harness-engineering
git checkout claude/keen-curie-xtGUj
```

### Making Changes
1. Edit files in the current branch
2. Run validation (Markdown, JSON, YAML)
3. Commit with descriptive message
4. Push to `claude/keen-curie-xtGUj`

### Code Review Process
- All changes should be peer-reviewed before merge (when team size > 1)
- Use `/review` to request Copilot feedback on changes
- Address review comments in a new commit (don't amend public commits)

---

## 6. Context Management

### Token Budget per Session
- **Available**: 200,000 tokens
- **Pre-consumed** (system prompt + tools + CLAUDE.md): ~20,000 tokens
- **Safe working window**: 120,000 tokens (60% rule)
- **Warning threshold**: 150,000 tokens (75% capacity)

### When Context Fills Up
1. Create a new session (context auto-compresses older messages)
2. Ask Claude to summarize findings into a separate document
3. Reference that document in the new session instead of scrolling history
4. Use `/clear` to start fresh if needed

### Large File Handling
- Avoid reading entire files > 500 lines at once
- Use `offset` parameter in Read tool
- Delegate large code reviews to `Explore` subagent
- Store findings in intermediate summary files

---

## 7. Hooks & Automation

### Enabled Hooks
```json
{
  "hookExecutionMode": "enabled",
  "hooks": {
    "pre-commit": "scripts/validate.sh",
    "post-push": "scripts/notify.sh"
  }
}
```

### Pre-Commit Checks (if enabled)
- Markdown syntax validation
- JSON/YAML parsing
- No secrets detected (via `mcp__github__run_secret_scanning`)

---

## 8. Dependencies & MCP Servers

### Required MCP Servers (in settings.json)
```json
{
  "mcpServers": {
    "github": {
      "command": "mcp-server-github",
      "args": []
    }
  }
}
```

### Optional Integrations
- Filesystem tools (built-in): `Read`, `Write`, `Edit`, `Bash`
- GitHub MCP: For issue/PR management
- Custom servers: Define in `.claude/agents/` with inline `mcpServers` field

---

## 9. Effort & Model Settings

### Recommended Settings
```
/config
- Model: Claude Opus 4.7 (main), Claude Sonnet 4.6 (fast mode)
- Effort: Auto (let Claude choose) or set to "medium" for balanced speed/quality
- Thinking Mode: true (show reasoning for complex tasks)
- Output Style: Explanatory (detailed step-by-step with ★ Insight boxes)
```

### When to Use /ultrathink
- Research & trend analysis (like generating this config)
- Complex architecture decisions
- Debugging subtle bugs

---

## 10. Troubleshooting & Escalation

### If Claude encounters an error:
1. **Markdown parse error**: Check frontmatter syntax (`---` on separate lines)
2. **JSON/YAML error**: Validate with `jq` / `yamllint` before retry
3. **MCP server timeout**: Check `settings.json` server definition and network
4. **Permission denied**: Check `settings.json` and `settings.local.json` for allowlist

### Escalation Protocol
- **Minor issues** (formatting, typos): Fix directly
- **Architecture changes** (new MCP, new subagent type): Ask in conversation
- **Destructive operations** (force-push, delete branches): Always confirm first

---

## References

- [Claude Code Docs](https://code.claude.com/docs) — Official documentation
- [CLAUDE.md Best Practices](https://www.gradually.ai/en/claude-md/) — Template guide
- [Subagent Documentation](https://code.claude.com/docs/en/sub-agents) — Subagent specification
