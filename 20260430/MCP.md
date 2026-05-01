# MCP.md - Model Context Protocol Integration Guide

**Version**: 1.0 | **Last Updated**: 2026-04-30

---

## 1. MCP Fundamentals

### What is MCP (Model Context Protocol)?

MCP is a standardized interface for Claude Code to access external tools and services:
- **Decouples** Claude from specific tool implementations
- **Standardizes** tool invocation syntax (namespace:tool-name format)
- **Enables** third-party integrations (GitHub, databases, APIs)
- **Supports** both inline (temporary) and persistent server connections

**Key distinction:**
```
Claude Tools (built-in):        MCP Servers (external integrations):
├── Read (file system)          ├── github (PR/issue management)
├── Write                       ├── database (SQL queries)
├── Edit                        ├── datadog (monitoring)
├── Bash                        ├── slack (messaging)
└── Agent                       └── custom-internal-api
```

---

## 2. Configuration Structure

### Global MCP Setup (~/.claude/settings.json)

```json
{
  "mcpServers": {
    "github": {
      "command": "mcp-server-github",
      "args": [],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    },
    "my-database": {
      "command": "python3",
      "args": ["/path/to/mcp-database-server.py"],
      "env": {
        "DB_CONNECTION_STRING": "${env:DB_CONNECTION_STRING}"
      }
    }
  }
}
```

**Field definitions:**
- `command` (required): Executable or script to launch the MCP server
- `args` (optional): Command-line arguments to pass
- `env` (optional): Environment variables for the server
  - Syntax: `"${env:VAR_NAME}"` references system environment
  - Use for secrets (tokens, API keys)

### Project-Level MCP Setup (.claude/settings.json)

Same structure as global, but only accessible in this project:

```json
{
  "mcpServers": {
    "project-internal-api": {
      "command": "node",
      "args": ["./mcp-servers/internal-api.js"],
      "env": {
        "API_KEY": "${env:INTERNAL_API_KEY}"
      }
    }
  }
}
```

**Merging rule**: Project settings override global settings (same server name).

---

## 3. Built-in vs. Custom MCP Servers

### Official Anthropic-Maintained Servers

| Server | Package | Tools | Use Case |
|--------|---------|-------|----------|
| **github** | `mcp-server-github` | search-code, list-issues, create-pr, get-file-contents | GitHub integration |
| **filesystem** | Built-in | read, write, edit, bash | File operations |
| **web** | Built-in | fetch, search | Web access |

### Popular Third-Party Servers

| Server | Docs | Example Use |
|--------|------|-------------|
| **datadog** | [Datadog MCP](https://github.com/datadog/mcp-server-datadog) | Query metrics, logs, incidents |
| **slack** | [Slack MCP](https://github.com/slackapi/slack-sdk-python) | Send messages, list channels |
| **database** | [SQL MCP](https://github.com/model-context-protocol/servers) | Execute queries safely |
| **stripe** | [Stripe MCP](https://github.com/stripe/stripe-python) | Query transactions, disputes |

### Custom MCP Servers (Your Own)

Create in `.claude/mcp-servers/`:

```bash
# Example: my-api-server.js
#!/usr/bin/env node
const mcp = require("@anthropic-ai/mcp-sdk");

const server = new mcp.Server();

// Define a tool
server.defineTool({
  name: "query-reports",
  description: "Query internal reports database",
  inputSchema: {
    type: "object",
    properties: {
      report_id: { type: "string" },
      date_range: { type: "string" }
    }
  }
});

server.onToolCall(async (toolName, params) => {
  if (toolName === "query-reports") {
    return { result: fetchReport(params.report_id) };
  }
});

server.start();
```

Register in `.claude/settings.json`:
```json
{
  "mcpServers": {
    "my-api": {
      "command": "node",
      "args": [".claude/mcp-servers/my-api-server.js"]
    }
  }
}
```

---

## 4. Tool Invocation Syntax

### Standard Tool Reference

```
mcp:<server-name>:<tool-name>
```

### Examples

**GitHub integration:**
```
mcp:github:search-code           ← Search repo code
mcp:github:list-issues           ← Get issues list
mcp:github:create-pull-request   ← Create new PR
mcp:github:get-file-contents     ← Read file from repo
```

**Custom server:**
```
mcp:my-api:query-reports         ← Call query-reports tool
mcp:my-api:submit-analysis       ← Call submit-analysis tool
```

**Inline server (temporary):**
```
mcp:inline-python:execute-script ← One-off Python execution
```

### Tool Invocation in Practice

In agent definitions or subagent configs:

```yaml
---
name: research-agent
tools:
  - Read                         # Built-in tool
  - WebSearch                    # Built-in tool
  - mcp:github:search-code       # External via MCP
  - mcp:my-api:query-reports     # Custom server
---
```

When Claude uses the tool:
```
Agent: "I'll search for caching implementations"
→ Calls: mcp:github:search-code with query="prompt caching"
→ Returns: [file1.py, file2.py, ...]
```

---

## 5. Subagent + MCP Integration

### Pattern: Subagent with MCP Access

**Scenario:** Code research subagent needs both GitHub + internal API access

```yaml
---
name: research-analyst
type: research
description: Research code patterns across GitHub and internal repos
tools:
  - Read
  - mcp:github:search-code
  - mcp:my-internal-api:query-codebase
model: claude-opus-4-7
---
You are a code research specialist. Your task:
1. Search GitHub for implementations of [topic]
2. Query our internal API for company-specific usage
3. Synthesize findings into a comparison matrix
Return both sources and analysis.
```

### Pattern: Inline MCP Servers (Temporary)

For one-off tasks, define MCP inline without global config:

```json
{
  "subagents": {
    "data-analyst": {
      "mcpServers": {
        "csv-processor": {
          "command": "python3",
          "args": ["-m", "mcp_csv_server"],
          "env": { "DATA_PATH": "/tmp/data/" }
        }
      },
      "tools": ["mcp:csv-processor:query-csv"]
    }
  }
}
```

**Lifetime**: Server starts when subagent spawns, stops when subagent finishes.

---

## 6. Security & Permissions

### Principle: Least Privilege

**Restrict MCP access by subagent:**

```json
{
  "subagents": {
    "auditor": {
      "tools": ["Read", "mcp:github:search-code"],
      "deny": ["Write", "mcp:my-api:execute-script"]
    },
    "implementer": {
      "tools": ["Read", "Write", "mcp:github:create-pull-request"],
      "deny": ["mcp:my-api:query-reports", "Bash"]
    }
  }
}
```

### Secret Management

**Never hardcode API keys in settings.json. Always use environment variables:**

```json
// ❌ BAD
{
  "mcpServers": {
    "stripe": {
      "env": { "STRIPE_API_KEY": "sk_live_....." }
    }
  }
}

// ✅ GOOD
{
  "mcpServers": {
    "stripe": {
      "env": { "STRIPE_API_KEY": "${env:STRIPE_API_KEY}" }
    }
  }
}
```

Then set in shell:
```bash
export STRIPE_API_KEY="sk_live_....."
# or in settings.local.json:
{ "env": { "STRIPE_API_KEY": "..." } }
```

---

## 7. Debugging MCP Issues

### Issue: MCP Server Not Responding

**Diagnostic steps:**

1. **Check server definition:**
   ```bash
   jq '.mcpServers.github' ~/.claude/settings.json
   # Verify: command exists, args are valid
   ```

2. **Test server manually:**
   ```bash
   # For Node.js server:
   node /path/to/server.js --help
   
   # For Python server:
   python3 -m mcp_server_github --help
   ```

3. **Enable debug logging:**
   ```bash
   # In settings.json:
   { "mcpServers": { "github": { "debug": true } } }
   # Or via environment:
   export MCP_DEBUG=1
   ```

4. **Check permissions:**
   ```bash
   ls -la /path/to/server.js
   # Must be executable: chmod +x server.js
   ```

### Issue: Tool Not Available

**Symptom:** "Tool mcp:github:search-code not found"

**Causes & fixes:**
```
1. Server not in settings.json
   → Add to mcpServers

2. Server name typo
   → mcp:github vs. mcp:githubl (note extra 'l')
   → Check exact name in settings.json

3. Tool doesn't exist on that server
   → Verify tool name: mcp:github:search-code (correct)
   → NOT mcp:github:searchCode (wrong case)

4. Server failed to start
   → Check error logs: ~/Library/Application Support/Claude/logs/
   → Enable --enable-mcp-debug flag
```

### Issue: Permission Denied

**Symptom:** "Permission denied: cannot access API token"

**Fix:**
1. Verify env var is set:
   ```bash
   echo $GITHUB_TOKEN
   ```

2. Check settings.json references correct env var:
   ```json
   { "env": { "GITHUB_TOKEN": "${env:GITHUB_TOKEN}" } }
   ```

3. If using settings.local.json, ensure it's loaded:
   ```bash
   # settings.local.json should be in ~/.claude/ or .claude/
   ls -la ~/.claude/settings.local.json
   ```

---

## 8. Performance Optimization

### Caching MCP Results

**Problem**: MCP calls can be slow (network latency, server processing)

**Solution**: Cache results in intermediate variables

```yaml
---
name: analysis-agent
---
# Bad: Calls search-code 3 times
Files mentioning "cache": [search results 1]
Files mentioning "ttl": [search results 2]
Files mentioning "redis": [search results 3]

# Good: Single search, filter results
All cache-related files: [single search result]
→ Files mentioning "cache": [filtered]
→ Files mentioning "ttl": [filtered]
→ Files mentioning "redis": [filtered]
```

### Batch Operations

**For databases via MCP:**

```
❌ Inefficient:
FOR EACH user_id:
  CALL mcp:database:query "SELECT * FROM users WHERE id = ?"

✅ Efficient:
CALL mcp:database:query "SELECT * FROM users WHERE id IN (...)"
```

---

## 9. Integration Patterns

### Pattern 1: GitHub-Driven Workflow

```yaml
name: github-analyst
tools:
  - mcp:github:search-code
  - mcp:github:list-issues
  - mcp:github:get-file-contents
---
Your workflow:
1. Search for issues tagged "bug" and "urgent"
2. For each, search code for related files
3. Fetch files and analyze root causes
4. Output: ranked bug list with fixes
```

### Pattern 2: Multi-Source Research

```yaml
name: research-synthesizer
tools:
  - WebSearch
  - mcp:github:search-code        # Search open-source repos
  - mcp:my-internal-api:query     # Search company code
  - Read                          # Read local files
---
Synthesize findings from:
- Public GitHub (trends)
- Internal codebase (current usage)
- Web (external resources)
- Local documentation
```

### Pattern 3: Approval & Execution

```yaml
name: executor
tools:
  - Read
  - mcp:github:create-pull-request
  - mcp:my-api:deploy
---
1. Read implementation plan
2. Create PR with changes
3. Wait for human approval (manual step)
4. On approval, trigger deployment via MCP
```

---

## 10. Best Practices

**DO:**
- ✅ Document server capabilities in README.md
- ✅ Use environment variables for secrets
- ✅ Test MCP server startup separately from Claude
- ✅ Log all MCP tool invocations (in agent description)
- ✅ Start simple: one MCP server, then expand

**DON'T:**
- ❌ Commit API keys in settings.json (use .local.json)
- ❌ Give all subagents access to all MCP servers
- ❌ Assume MCP server is always available (handle failures)
- ❌ Create too many custom MCP servers (use built-in tools first)
- ❌ Ignore server logs when debugging issues

---

## References

- [Model Context Protocol Spec](https://modelcontextprotocol.io/) — Official MCP specification
- [Claude Code MCP Docs](https://code.claude.com/docs/en/sub-agents) — Claude integration guide
- [MCP Server Repository](https://github.com/model-context-protocol/servers) — Official server implementations
- [Anthropic Engineering Blog - Agents](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) — Agent + MCP patterns
