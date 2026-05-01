# DESIGN.md - Architectural Principles & Patterns

**Version**: 1.0 | **Last Updated**: 2026-04-30

---

## 1. Core Design Principles

### SOLID Principles Applied to AI Orchestration

#### S - Single Responsibility Principle
**Each file owns ONE concern:**
- `CLAUDE.md` → Project rules & constraints
- `AGENT.md` → Persona & automation logic
- `DESIGN.md` → Architecture & patterns (this file)
- `.claude/settings.json` → Tool access & permissions
- `.claude/agents/*.md` → Individual subagent definitions

**Anti-pattern**: Don't merge CLAUDE.md + AGENT.md. They serve different audiences (developers vs. orchestrators).

#### O - Open/Closed Principle
**Configuration should be:**
- **Open for extension**: New MCP servers added via `settings.json` without touching CLAUDE.md
- **Closed for modification**: Agent personas shouldn't change; if needed, create new subagent

**Example**: Adding a new data source
```json
// Bad: Modify AGENT.md and scripts
// Good: Add new MCP server in settings.json
{
  "mcpServers": {
    "my-datasource": {
      "command": "mcp-server-my-datasource",
      "args": ["--config", "~/.my-datasource.json"]
    }
  }
}
```

#### L - Liskov Substitution Principle
**Subagents are interchangeable implementations of their declared type:**
- If two subagents both declare `type: research`, they should produce equivalent research output
- Different internal strategy (web search vs. document analysis) is fine; contract is the same

#### I - Interface Segregation Principle
**Client-facing tools should have minimal required parameters:**
- WebSearch: Just `query` (optional filters for domain)
- Bash: Just `command` (timeout is optional)
- Agent: Just `description` + `prompt` (everything else configurable)

**Anti-pattern**: Requiring `owner`, `repo`, `pullNumber`, `path`, `body`, `subjectType` for every MCP call.

#### D - Dependency Inversion Principle
**Depend on abstractions, not concrete implementations:**
- Don't hardcode "use Claude Opus"; instead read `model` from settings.json
- Don't hardcode webhook URLs; read from environment
- Don't assume GitHub is available; gracefully degrade if MCP server is missing

---

### DRY (Don't Repeat Yourself)
**Apply across all config files:**

**Bad:**
```markdown
# In CLAUDE.md
Do not edit settings.json directly.

# In AGENT.md  
Do not edit settings.json directly.

# In README.md
Do not edit settings.json directly.
```

**Good:**
```markdown
# In CLAUDE.md
For MCP server changes, update settings.json (see AGENT.md for automation).
```

Then AGENT.md and README.md reference CLAUDE.md, eliminating redundancy.

---

### YAGNI (You Aren't Gonna Need It)
**Avoid premature generalization:**

**Anti-pattern**:
```json
{
  "agents": {
    "generic-research-framework": {...},
    "generic-code-generation-framework": {...},
    "generic-validation-framework": {...}
  }
}
```

**Pattern**: Define only the 3-5 subagents actually needed for the project. When a new need emerges, add one agent.

---

## 2. Architectural Patterns

### Pattern 1: Plan → Act → Verify (PAV) Cycle

**Flow Diagram:**
```
┌─────────────────────────────────────────────┐
│ User Request / Automated Trigger            │
└────────────────────┬────────────────────────┘
                     │
          ┌──────────▼──────────┐
          │  PLAN (Orchestrator) │ ← Analyze, research, design
          │ - Load CLAUDE.md     │
          │ - Web search trends  │
          │ - Sketch output      │
          └──────────┬──────────┘
                     │
      ┌──────────────┼──────────────┐
      │              │              │
  ┌───▼────┐  ┌─────▼───┐  ┌──────▼──┐
  │ ACT 1  │  │  ACT 2  │  │  ACT 3  │ ← Parallel subagents
  │Generate│  │Validate │  │  Audit  │
  │ Config │  │  Syntax │  │Security │
  └───┬────┘  └─────┬───┘  └──────┬──┘
      │              │              │
      └──────────────┼──────────────┘
                     │
          ┌──────────▼────────────┐
          │ VERIFY (Quality Gate)  │ ← Check: parse, content, security
          │ - File integrity       │
          │ - URL realism          │
          │ - No placeholders      │
          └──────────┬─────────────┘
                     │
          ┌──────────▼──────────────┐
          │ REPORT (1-2 sentences)  │ ← What changed, next steps
          └─────────────────────────┘
```

**Implementation in AGENT.md Section 2** — Ensures every major decision follows this flow.

---

### Pattern 2: Layered Configuration

```
┌─────────────────────────────────────┐
│ Session Context (In-Memory)         │
│ - Conversation history              │
│ - Loaded files                      │
│ - Variables during execution        │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ Project Configuration (.claude/*)   │
│ - CLAUDE.md (rules)                 │
│ - settings.json (MCP, permissions)  │
│ - agents/ (subagent definitions)    │
│ - hooks.json (automation)           │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ User/System Settings (~/.claude/)   │
│ - settings.local.json (secrets)     │
│ - keybindings.json                  │
│ - agents/ (global utilities)        │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ Claude Code Defaults                │
│ - Tool definitions                  │
│ - Permission prompts                │
│ - Model selection                   │
└─────────────────────────────────────┘
```

**Principle**: Lower layers provide defaults; upper layers override (pyramid of specificity).
**Immutability**: Only `settings.local.json` is meant to be dynamic. Others are committed as truth.

---

### Pattern 3: MCP Server Abstraction

**Direct MCP tool reference:**
```
User Request
    ↓
"Use mcp:github:search-code to find X"
    ↓
├─ Check: Is "github" server in settings.json? ✓
├─ Check: Is "search-code" available from that server? ✓
└─ Execute: mcp:github:search-code query="X"
```

**Benefits:**
- Tool invocation is declarative (just name the tool)
- MCP server swapping is transparent (change one config line)
- Offline fallback possible (if server unavailable, skip gracefully)

**Anti-pattern**: Hardcoding "call gh CLI" in automation scripts. Use MCP instead.

---

### Pattern 4: Subagent Delegation Strategy

**When to delegate to subagent:**
```
Is the task:
  ├─ Large (> 50KB of input)?              → Use subagent (separate context)
  ├─ Independent from main flow?            → Use subagent (parallelize)
  ├─ Requires specific tool set?            → Use subagent (tool isolation)
  └─ Blocking main thread?                  → Use subagent (async execution)
```

**Example: Configuration generation**
```
Main Agent:
  1. Decide what config to generate (based on web trends)
  2. Delegate → code-generator subagent (generates files, validates)
  3. Delegate → auditor subagent (scans security)
  4. Collect results, synthesize report
```

**Tool Access in Subagents** (from AGENT.md):
- Subagent 1 (researcher): WebSearch, WebFetch, Read
- Subagent 2 (code-generator): Write, Edit, Bash (validation only)
- Subagent 3 (auditor): Read, Bash, mcp__github__run_secret_scanning

Each subagent has minimal permissions; escalation to main agent if more access needed.

---

## 3. Directory & File Structure

### Canonical Layout

```
harness-engineering/
│
├── README.md                        ← Project overview
├── CLAUDE.md                        ← Project constraints (auto-loaded)
├── AGENT.md                         ← Agent definitions (this repo's agents)
├── DESIGN.md                        ← This file
│
├── .claude/                         ← Project-level config (committed)
│   ├── settings.json                ← MCP servers, permissions, effort defaults
│   ├── hooks.json                   ← Pre/post automation rules
│   ├── agents/                      ← Subagent definitions
│   │   ├── researcher.md
│   │   ├── code-generator.md
│   │   └── auditor.md
│   └── rules/                       ← Custom permission rules (optional)
│
├── 20260430/                        ← Generated config (from this run)
│   ├── README.md                    ← Entry point (overview)
│   ├── CLAUDE.md                    ← Reusable config template
│   ├── AGENT.md                     ← Agent definitions
│   ├── DESIGN.md                    ← Design patterns
│   ├── CONTEXT.md                   ← Context management
│   ├── MCP.md                       ← MCP integration guide
│   ├── CACHING.md                   ← Prompt caching patterns
│   └── HOOKS.md                     ← Automation examples
│
├── .gitignore                       ← Standard excludes
│   entries:
│   - settings.local.json
│   - .claude/state/
│   - node_modules/
│   - *.log
│
└── .github/                         ← (If using GitHub workflows)
    └── workflows/
        └── validate-claude.yml      ← CI for config validation
```

### Naming Conventions

**Files:**
- **Lowercase with hyphens**: `.claude/agents/code-generator.md`
- **Markdown for config**: All agent definitions are `.md` with YAML frontmatter
- **Date-based directories**: `YYYYMMDD/` for generated archives
- **Descriptive names**: `prompt-caching-guide.md` not `caching.md`

**Directories:**
- **Singular for collections**: `.claude/agents/` (multiple agents live here)
- **Plural for multiple files**: `docs/patterns/` (multiple pattern docs)
- **No deep nesting**: Max 3 levels (`.claude/agents/researcher.md` OK; `.claude/teams/ai/agents/research/researcher.md` not OK)

---

## 4. Data Flow & Module Interactions

### Configuration Loading Sequence

```
Session Start
    ↓
1. Load ~/.claude/settings.json (user defaults)
2. Load ~/.claude/settings.local.json (secrets, personal)
3. Load .claude/settings.json (project overrides)
4. Load .claude/agents/*.md (subagent definitions)
5. Load CLAUDE.md (project rules)
6. Ready to execute
```

**Conflict Resolution** (later = higher priority):
- user/system settings < project settings.json < local overrides

---

### Subagent Communication

```
Main Orchestrator (this session)
    ↓
1. Define task: "Generate config for YYYYMMDD"
2. Create subagent bundle:
   - Model: claude-opus-4-7
   - Tools: [Write, Bash, Read]
   - System prompt: (from .claude/agents/code-generator.md)
   - Context: [Required files to read]
3. Execute (in parallel with other subagents)
4. Collect results:
   - Output files created ✓
   - Parse validation ✓
   - Error log (if any)
5. Next step: Audit subagent (parallel)
    ↓
   Auditor scans files for secrets
    ↓
6. Merge results: [generated] + [validated] + [audited]
7. Return to main flow
```

---

## 5. Extension Points

### How to Add a New Subagent

1. **Define persona**: Create `.claude/agents/new-agent.md`
   ```yaml
   ---
   name: new-agent
   type: [research | implementation | verification]
   description: One-line purpose
   tools: [List of tools needed]
   model: claude-opus-4-7
   ---
   # System prompt
   ```

2. **Register in settings**: Add to `.claude/settings.json`
   ```json
   {
     "agents": {
       "new-agent": { "enabled": true }
     }
   }
   ```

3. **Reference in orchestration**: Update AGENT.md Section 4

4. **Test invocation**: `Agent(description="...", subagent_type="new-agent")`

---

### How to Add a New MCP Server

1. **Install server**: Install package or download binary
2. **Update settings.json**:
   ```json
   {
     "mcpServers": {
       "new-server": {
         "command": "mcp-server-new-server",
         "args": ["--config", "path/to/config.json"]
       }
     }
   }
   ```
3. **Test connectivity**: `mcp:new-server:tool-name`
4. **Document in MCP.md**: Add server to tool matrix

---

## 6. Testing & Validation Strategy

### Unit Tests (per file)
```bash
# CLAUDE.md
- [ ] Parse as markdown ✓
- [ ] No broken references ✓
- [ ] Under 60 lines ✓

# .claude/settings.json
- [ ] Valid JSON ✓
- [ ] All MCP servers have 'command' ✓
- [ ] No secrets (API keys, tokens) ✓

# .claude/agents/*.md
- [ ] YAML frontmatter parses ✓
- [ ] All tools listed exist ✓
- [ ] Model name is valid (Opus, Sonnet, Haiku) ✓
```

### Integration Tests (across files)
```bash
# Configuration consistency
- [ ] Agent names in AGENT.md match .claude/agents/ files ✓
- [ ] MCP server names in settings match tool references ✓
- [ ] Links in README point to actual files ✓

# End-to-end
- [ ] Run /init in project directory ✓
- [ ] Load all subagents without YAML errors ✓
- [ ] Execute sample task (e.g., "research latest Claude practices") ✓
```

### Continuous Validation

**Pre-commit** (local):
```bash
find . -name "*.json" -exec jq empty {} \;
find . -name "*.md" -path "./.claude/agents/*" | xargs head -20 | grep "^---$"
```

**CI/CD** (GitHub Actions, if enabled):
```yaml
- name: Validate CLAUDE Configs
  run: |
    find . -name "*.json" -exec jq empty {} \;
    python3 scripts/validate-agents.py
    grep -r "TODO\|TBD" . && exit 1 || echo "No placeholders"
```

---

## 7. Security & Compliance

### Secret Management
- **Never commit**: API keys, tokens, passwords
- **Use**: `settings.local.json` (in .gitignore)
- **Scan before push**: `mcp__github__run_secret_scanning`

### Permission Model
**Principle**: Least privilege per subagent

- **researcher**: Can read web; cannot write/delete files
- **code-generator**: Can write to `/20260430/`; cannot access GitHub API
- **auditor**: Can read files; cannot execute arbitrary commands

### Compliance Checklist

- [ ] No hardcoded credentials in CLAUDE.md, AGENT.md, or `.claude/settings.json`
- [ ] MCP server definitions don't expose sensitive API endpoints
- [ ] Audit log captures who ran what (via git commit messages)
- [ ] Subagent definitions don't have over-broad tool access

---

## 8. Scaling Considerations

### For 1 person
- Single CLAUDE.md (project-level rules)
- settings.local.json for personal preferences
- No team sync overhead

### For 1-5 person team
- Commit CLAUDE.md to repo (everyone sees the rules)
- Keep settings.local.json in .gitignore (personal overrides)
- Use hooks for pre-commit validation
- Document decisions in `decisions.md`

### For 5+ person organization
- Central `.claude/` directory (committed)
- Org-level `~/.claude/` (shared machine or NFS)
- CI/CD validation required for all changes
- Code review for CLAUDE.md + AGENT.md changes
- Separate approval process for MCP server additions

---

## References

- [SOLID Principles](https://en.wikipedia.org/wiki/SOLID) — Wikipedia
- [DRY Principle](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)
- [Claude Code Docs - Configuration](https://code.claude.com/docs/en/how-claude-code-works)
- [The Twelve-Factor App](https://12factor.net/) — Configuration management approach (adapted)
