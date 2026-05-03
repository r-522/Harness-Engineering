# Harness Engineering: Orchestration Patterns & Configuration

## Harness Engineering Philosophy

**Harness Engineering** is the design and optimization of everything surrounding an LLM except the model itself:

```
Agent = Model + Harness

Where Harness = Configuration + Constraints + Feedback Loops + Tools + Environment
```

This document defines the orchestration patterns, MCP integration strategies, and feedback loop implementations that enable Claude Code to operate at maximum effectiveness.

---

## Core Orchestration Patterns

### 1. Subagent Pattern (Hierarchical Delegation)

**Use Case**: Break a complex task into specialized sub-tasks, delegate to focused workers.

**Structure**:
```
Main Agent (Orchestrator)
  ├─ Subagent 1 (Focused Task A)
  ├─ Subagent 2 (Focused Task B)
  └─ Subagent 3 (Focused Task C)
       ↓
   Synthesize Results
```

**Implementation**:
```typescript
// In your Claude Code session
Agent({
  description: "Task A research",
  prompt: "Find X, report with sources",
  subagent_type: "general-purpose"
});

Agent({
  description: "Task B implementation",
  prompt: "Build Y according to spec",
  subagent_type: "general-purpose"
});

// Main agent synthesizes subagent results
```

**When to Use**:
- ✅ Independent subtasks (research + coding in parallel)
- ✅ Specialized workers (code-reviewer subagent, architecture-planner subagent)
- ✅ Context isolation (keep subagent context fresh, small)

**Limitations**:
- ❌ Requires careful state serialization (results flow back to orchestrator)
- ❌ Token overhead increases with number of subagents
- ❌ Orchestrator's context grows with each round

---

### 2. Split-and-Merge Pattern (Parallel Execution)

**Use Case**: Divide work into parallel chunks, merge results seamlessly.

**Structure**:
```
Main Task
  ├─ [Parallel] Run Tests
  ├─ [Parallel] Build Artifacts
  ├─ [Parallel] Lint Code
       ↓
    Collect Results
       ↓
    Merge & Validate
```

**Implementation**:
```typescript
// Issue all parallel tasks in one message
const parallelTasks = [
  Agent({ description: "Run tests", prompt: "..." }),
  Agent({ description: "Build", prompt: "..." }),
  Agent({ description: "Lint", prompt: "..." })
];
```

**When to Use**:
- ✅ Independent, time-consuming tasks (CI/CD pipeline simulation)
- ✅ Load balancing across available compute
- ✅ Reduced total execution time

**Limitations**:
- ❌ Results must be serializable (JSON, markdown)
- ❌ Cross-task dependencies not allowed
- ❌ Requires merge logic in main agent

---

### 3. Agent Teams Pattern (Collaborative Coordination)

**Use Case**: Multiple agents working as peers, sharing findings, challenging each other.

**Structure**:
```
Team Lead (Orchestrator)
  │
  ├─ Agent A (Frontend specialist)
  ├─ Agent B (Backend specialist)
  └─ Agent C (DevOps specialist)
       ↓
    Shared Task List / Context
    (agents read & write directly)
```

**When to Use**:
- ✅ Complex projects requiring diverse expertise
- ✅ Agents need to challenge/review each other's work
- ✅ Long-running projects with feedback cycles

**Limitations**:
- ❌ More complex to orchestrate than subagents
- ❌ Potential for conflicting edits
- ❌ Requires careful synchronization

---

### 4. Sequential Orchestration (Strict Ordering)

**Use Case**: Task B depends on Task A's output.

**Implementation**:
```typescript
// Step 1: Complete Task A
const resultA = await executeTaskA();

// Step 2: Use A's result in Task B
const resultB = await executeTaskB(resultA);

// Step 3: Finalize
synthesizeResults(resultA, resultB);
```

**When to Use**:
- ✅ Clear dependencies between steps
- ✅ Later steps require earlier outputs
- ✅ Simple, linear workflows

**Avoid If**:
- ❌ Steps are independent (use Split-and-Merge instead)

---

## Feedback Loop Architecture

### The Plan → Work → Review Cycle

Every harness should implement a feedback loop that allows Claude to verify its own work:

```
1. PLAN
   ├─ Understand task scope
   ├─ Propose approach
   └─ Get user confirmation

2. WORK
   ├─ Execute implementation
   ├─ Run tests/linting (inline feedback)
   └─ Validate output

3. REVIEW
   ├─ Compare result against plan
   ├─ Check for edge cases
   ├─ Verify no regressions
   └─ Report to user

4. [REPEAT if needed]
```

### Feedback Loop Implementation

#### **Build Feedback Loop**
```bash
npm run build && npm run build:verify
# OR
cargo build --release && cargo test --release
```

**Expected Output**: Build succeeds or specific error messages that guide next action.

#### **Test Feedback Loop**
```bash
npm test -- --coverage
# OR
pytest --cov --tb=short
```

**Expected Output**: Test results + coverage report; failures list specific assertions.

#### **Type Checking Loop**
```bash
tsc --noEmit && mypy .
```

**Expected Output**: Type errors with file:line references.

#### **Linting Loop**
```bash
eslint src/ --format=json
# OR
pylint src/
```

**Expected Output**: Linting violations grouped by severity + error codes.

### Integrating Feedback Loops in CLAUDE.md

```markdown
## Test & Build Commands

Before every commit, Claude should execute:

1. **Type Checking** (TypeScript projects)
   ```bash
   npm run type-check
   ```

2. **Linting**
   ```bash
   npm run lint -- --fix  # Auto-fix where possible
   ```

3. **Tests**
   ```bash
   npm test -- --watch=false
   ```

4. **Build**
   ```bash
   npm run build
   ```

These commands provide real-time feedback; Claude can self-correct errors.
```

---

## MCP (Model Context Protocol) Integration

### MCP Servers as Tools for Claude

An MCP server exposes tools that Claude can invoke. Common MCP servers:

| MCP Server | Purpose | Tools Exposed |
|---|---|---|
| **github** | GitHub API access | create_pr, comment_issue, search_repos, list_issues |
| **bash** | Shell command execution | run_command (local machine) |
| **web-fetch** | URL content retrieval | fetch_content (HTML → markdown) |
| **web-search** | Internet search | search_query (returns ranked results) |
| **postgres** | Database access | query, execute_sql (read/write) |
| **stripe** | Payment processing | create_charge, list_customers, etc. |

### Configuring MCP in settings.json

```json
{
  "mcpServers": {
    "github": {
      "command": "node",
      "args": ["mcp-github-server.js"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    },
    "bash": {
      "command": "/bin/bash"
    },
    "web-search": {
      "command": "python",
      "args": ["web-search-mcp.py"],
      "alwaysAllow": ["search_web"]
    }
  }
}
```

### Invoking MCP Tools from Claude Code

```typescript
// In your Claude Code prompt/session
// GitHub MCP example:
mcp__github__list_pull_requests({
  owner: "your-org",
  repo: "your-repo",
  state: "open"
});

// Bash MCP example:
bash({
  command: "npm test -- --coverage"
});

// Web Search MCP example:
web_search({
  query: "Claude Code best practices 2025"
});
```

### Security Considerations for MCPs

- **API Keys**: Store in `.env` or environment variables, never in source code
- **Permissions**: Grant minimal required permissions to each MCP
- **Allowlist**: Use `alwaysAllow` for safe, read-only operations
- **Audit**: Log MCP invocations for sensitive operations (database writes, API calls)

---

## Monitoring & Observability

### Context Health Monitoring

Track context usage to avoid hitting limits:

```typescript
// Query context status
/status

// Expected output:
// ✓ Session context: 42% used (21,000 of 50,000 tokens)
// ✓ Model: claude-opus-4-7
// ✓ Message history: 8 messages (last 6 hours)
```

**Action Thresholds**:
- 🟢 **0-60%**: Normal, continue work
- 🟡 **60-80%**: Monitor closely, consider clearing non-critical history
- 🔴 **80%+**: Clear context immediately `/clear-all` before proceeding

### Logging Best Practices

#### **Implement structured logging**
```javascript
// Good: Structured, context-rich logs
console.log(JSON.stringify({
  timestamp: new Date().toISOString(),
  level: "INFO",
  service: "user-service",
  userId: "user_123",
  action: "login",
  duration_ms: 145,
  status: "success"
}));

// Avoid: Unstructured logs
console.log("User logged in successfully");
```

#### **Log levels**
- **ERROR**: Failures that require immediate attention
- **WARN**: Unexpected conditions (retries, fallbacks)
- **INFO**: Major business events (user login, payment received)
- **DEBUG**: Detailed execution flow (only in dev)

---

## Performance Optimization for Harness Engineering

### Token Efficiency

| Strategy | Benefit | Trade-off |
|---|---|---|
| **Subagent isolation** | Smaller context per worker | Merge overhead |
| **Caching results** | Avoid re-computation | Stale data risk |
| **Prompt compression** | Fewer tokens per request | Reduced context |
| **Async execution** | Parallel tasks complete faster | Coordination complexity |

### Context Optimization

```markdown
## Context Cleanup Protocol

1. Run `/status` before starting new major task
2. If >70% context used:
   - Summarize findings and save to file
   - Run `/clear-all` (preserves CLAUDE.md)
   - Start fresh with focused subtask

3. Keep CLAUDE.md lean (<300 lines)
   - Move detailed docs to separate .md files
   - Reference files, don't inline examples
```

---

## Practical Example: Multi-Agent Code Review

### Scenario
Review a complex Pull Request in parallel (code quality + security).

### Orchestration

```typescript
Agent({
  description: "Code quality review",
  prompt: `Review PR for readability, maintainability, and SOLID principles.
    Focus: naming, duplication, error handling, test coverage.
    Report: structured findings with file:line references.`,
  subagent_type: "general-purpose"
});

Agent({
  description: "Security review",
  prompt: `Review PR for security vulnerabilities.
    Focus: input validation, authentication, secrets, SQL injection, XSS.
    Report: severity-ranked findings with remediation steps.`,
  subagent_type: "general-purpose"
});

// Main agent synthesizes both reviews
// → Creates consolidated review comment
```

### Feedback Loop

1. **Plan**: Identify PR scope, review checkpoints
2. **Work**: Spin up parallel subagents
3. **Review**: Merge findings, check for conflicts
4. **Verify**: Confirm findings with diff context

---

## Configuration Template for settings.json

```json
{
  "model": "claude-opus-4-7",
  "contextWindowPercentage": 80,
  "clearContextAt": 0.80,
  "mcpServers": {
    "github": { "command": "node", "args": ["gh-mcp.js"] },
    "bash": { "command": "/bin/bash" },
    "web-search": { "alwaysAllow": ["search"] }
  },
  "hooks": {
    "before_commit": "npm run lint && npm test",
    "before_push": "npm run build",
    "on_error": "log_error_context"
  },
  "permissions": {
    "bash": ["npm", "pytest", "git"],
    "github": ["create_pr", "list_issues"],
    "write": ["src/**", "tests/**"]
  }
}
```

---

## Troubleshooting Harness Issues

| Issue | Root Cause | Solution |
|---|---|---|
| **Subagents not running in parallel** | Single-message requirement not met | Wrap multiple `Agent()` calls in same response block |
| **Context bloat** | Subagent results accumulating | Clear non-critical history with `/clear` between cycles |
| **MCP tool not found** | MCP server not running | Verify MCP in `settings.json`, restart Claude Code |
| **Slow feedback loops** | External tool latency | Cache results, use async where possible |
| **Token limit exceeded** | Context window overrun | Implement more aggressive clearing, use Split-and-Merge |

---

## References

- [Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [Claude Code Subagents: Parallel Multi-Agent Workflows](https://turion.ai/blog/claude-code-multi-agents-subagents-guide/)
- [Harness Engineering for Coding Agents - HumanLayer](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)
- [Building 11 Skills in Claude Code - Zenn](https://zenn.dev/ryuka_lucas/articles/agent-teams-harness-engineering)

---

**Last Updated**: 2025-05-03  
**Version**: 1.0 (Initial Release)  
**Audience**: Senior engineers, SREs, AI platform architects
