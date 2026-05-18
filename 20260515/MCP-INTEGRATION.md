# MCP-INTEGRATION.md - External Service & API Integration Guide

**Version**: 2026-05-15  
**Scope**: Model Context Protocol (MCP) Server Integration for Harness Engineering  
**Audience**: DevOps, Security, Advanced Engineers (requires CLAUDE.md §5 pre-review)

---

## 1. What Is MCP?

**MCP (Model Context Protocol)** is a standardized interface that allows Claude Code to safely interact with external systems (APIs, services, databases) without embedding secrets or credentials directly in code.

**Key Properties**:
- **Sandboxed**: Subagent can only invoke explicitly-allowed tools
- **Auditable**: All API calls logged and can be reviewed
- **Extensible**: New integrations added without code changes
- **Secure**: Credentials stored separately from application code

---

## 2. MCP Integration Architecture

```
Claude Code Session
  ├─ Main Agent (Orchestrator)
  ├─ Subagent A
  │  └─ MCP Server 1 (GitHub API)
  ├─ Subagent B
  │  └─ MCP Server 2 (Slack)
  └─ Subagent C
     └─ MCP Server 3 (SQL Database)

Legend:
  → Secure channel (OAuth token / API key in env var)
  ← Results (JSON, markdown, structured data)
```

---

## 3. Approved MCP Servers (Out-of-Box)

| Server | Purpose | Auth | Tools | Status |
|---|---|---|---|---|
| **GitHub** | Issue/PR management, code access | OAuth / GitHub token | `pull_request_read`, `issue_write` | ✅ Approved |
| **Slack** | Notifications, message posting | Slack token (env var) | `post_message`, `create_channel` | ✅ Approved |
| **AWS** | EC2, S3, CloudWatch | AWS_KEY + SECRET (env var) | `describe_instances`, `get_logs` | ⚠️ Restricted (pre-approval) |
| **SQL Database** | Query execution, schema inspection | DB_PASSWORD (env var) | `execute_query`, `describe_schema` | ⚠️ Restricted (read-only) |

---

## 4. Adding a New MCP Server: Security Checklist

### ✅ Required Pre-Integration Review

**Before** activating ANY new MCP server, complete this checklist:

#### 4.1 Threat Model
```
[ ] API Surface Analysis
    ├─ What operations can this MCP expose? (CRUD, billing, delete, etc.)
    └─ Which operations do we NEED vs. WANT?

[ ] Trust Boundary
    ├─ Is this service within our org or external?
    ├─ Does it handle sensitive data (PII, financial)?
    └─ What's the blast radius if credentials leak?

[ ] Credential Risk
    ├─ API key/token expiry? (rotate every 90 days)
    ├─ Revocation process? (can we kill keys quickly?)
    └─ Audit trail? (does the service log API calls?)
```

#### 4.2 Implementation Checklist
```
[ ] Credential Storage
    ├─ Stored in .env or environment variable (NOT hardcoded)
    ├─ Added to .gitignore
    └─ Accessible to subagent only (tool permission scoping)

[ ] Tool Scoping
    ├─ List EXACT tools subagent can invoke
    ├─ Deny dangerous operations (delete, create user, billing)
    └─ Example: GitHub can read PRs but NOT delete repos

[ ] Logging & Monitoring
    ├─ All API calls logged (timestamp, user, result)
    ├─ Alerts on suspicious activity (bulk delete, rate limit)
    └─ Retention: 30+ days for audit

[ ] Testing
    ├─ Unit test: Mock MCP server, verify tool format
    ├─ Integration test: Real API call (staging environment only)
    └─ Security test: Attempt unauthorized operation (should fail)

[ ] Documentation
    ├─ MCP server config in YAML (src/config/mcp_servers.yaml)
    ├─ Tool list + parameters documented
    └─ Error handling documented (quota exceeded, auth failure)
```

#### 4.3 Approval Process
```
1. Create GitHub issue: "MCP Integration Request: [ServiceName]"
2. Attach threat model + implementation checklist
3. Request review from: @security-team + @team-lead
4. Approval required: 2+ reviewers, no blocking comments
5. Merge to main, activate in subagent
```

---

## 5. Integration Examples

### Example 1: GitHub API Integration (Already Approved)

**File**: `src/mcp/github_client.py`
```python
import os
import subprocess

# MCP server started by Claude Code (auto-detected)
# GitHub token loaded from environment
GITHUB_TOKEN = os.environ.get("GITHUB_TOKEN")

class GitHubMCP:
    def __init__(self):
        # Claude Code MCP framework handles authentication
        pass
    
    def get_pull_request(self, owner, repo, pr_number):
        """Read-only: Fetch PR metadata and diff"""
        # Tool: mcp__github__pull_request_read
        # Subagent calls this via MCP; credentials transparent
        pass
    
    def post_review_comment(self, owner, repo, pr_number, body):
        """Write: Post review comment (tool-scoped)"""
        # Tool: mcp__github__add_comment_to_pending_review
        # Requires explicit tool permission in subagent def
        pass
```

**Subagent Configuration**:
```yaml
---
name: reviewer_code_quality
model: claude-opus-4-7
tools:
  - mcp__github__pull_request_read  # Approved
  - mcp__github__add_comment_to_pending_review  # Scoped write
  - Bash  # For git commands (local, no MCP)
---
You are a code reviewer...
```

**Security Notes**:
- ✅ GITHUB_TOKEN in env var (not hardcoded)
- ✅ Tool scoping: read + comment only (no force-push, delete)
- ✅ All calls logged by Claude Code
- ✅ No access to billing, org settings, or user management

---

### Example 2: Slack Notification Integration (Custom MCP)

**Threat Model**:
```
[ ] API Surface: post_message, create_channel
[ ] Trust: Internal service, handles team messages (not sensitive)
[ ] Credentials: SLACK_TOKEN, rotated monthly
[ ] Blast Radius: Can post to public channels; contained
[ ] Approval Status: ✅ APPROVED (2026-04-01)
```

**File**: `src/mcp/slack_notifier.py`
```python
import os

SLACK_TOKEN = os.environ.get("SLACK_TOKEN")

class SlackMCP:
    def post_notification(self, channel_id, message):
        """Send notification (post-deployment or test results)"""
        # MCP tool invocation (Claude handles OAuth flow)
        pass
```

**Subagent Config**:
```yaml
---
name: notifier_deployment
model: claude-sonnet-4-6
tools:
  - mcp__slack__post_message  # Send only, read-only on channels
environment:
  SLACK_TOKEN: ${SLACK_TOKEN}  # Loaded from runtime env
---
You notify the team of deployment progress...
```

**Usage**:
```python
# Subagent invokes (transparent to main agent)
notifier.post_notification(
    channel_id="#deployments",
    message="✅ v2.1.0 deployed to staging"
)
```

---

### Example 3: AWS Integration (Restricted, Pre-Approval)

**Threat Model**:
```
[ ] API Surface: EC2 describe, CloudWatch read-only
[ ] Trust: External cloud provider
[ ] Credentials: AWS_KEY + SECRET (highest risk)
[ ] Blast Radius: Could spin up expensive resources, delete data
[ ] Approval Status: ⚠️ RESTRICTED (read-only queries only)
```

**Approval Conditions**:
```
1. Only EC2 DESCRIBE operations (no launch, no terminate)
2. CloudWatch READONLY (metrics only, no config changes)
3. Credentials rotated EVERY 30 DAYS
4. Rate limit: 100 API calls/min (burst: 200)
5. Audit: Daily log review for suspicious activity
```

**Subagent Config** (minimal):
```yaml
---
name: ops_monitor_infrastructure
model: claude-opus-4-7
tools:
  - mcp__aws__describe_instances  # Read-only
  - mcp__aws__get_cloudwatch_logs  # Read-only
environment:
  AWS_KEY: ${AWS_KEY}
  AWS_SECRET: ${AWS_SECRET}
max_api_calls_per_min: 100
---
You monitor infrastructure status...
```

**Forbidden Operations** (will be rejected):
```python
# ❌ FORBIDDEN: Would be blocked by tool scoping
mcp.run_instance(...)      # No launch tool
mcp.terminate_instance()   # No delete tool
mcp.modify_security_group()  # No config tool
```

---

## 6. Managing MCP Credentials

### Storage Strategy

```
.env (root, gitignored)
├─ GITHUB_TOKEN=ghp_xxxxxxxxxxxx
├─ SLACK_TOKEN=xoxb-xxxxxxxxxxxx
└─ AWS_KEY=AKIAIOSFODNN7EXAMPLE

.claude/.local.md (personal, gitignored)
├─ Team lead notes, personal API keys
├─ Debugging hints
└─ Local overrides

.github/secrets/ (GitHub Actions, encrypted)
├─ PROD_GITHUB_TOKEN
├─ PROD_AWS_KEY
└─ PROD_SLACK_TOKEN
```

### Rotation Checklist

| Token | Rotation Interval | Process | Last Rotated |
|---|---|---|---|
| `GITHUB_TOKEN` | 90 days | `gh auth refresh` | 2026-05-01 |
| `SLACK_TOKEN` | 30 days | Regenerate in Slack workspace settings | 2026-05-15 |
| `AWS_KEY` | 30 days | AWS Console → Security Credentials | 2026-05-01 |

**Automation**:
```bash
# Add to CI/CD pipeline (monthly)
- name: Rotate API tokens
  run: |
    aws iam create-access-key --user-name ci-user
    aws iam delete-access-key --user-name ci-user --access-key-id [OLD]
    gh auth refresh
```

---

## 7. Monitoring & Observability

### MCP Call Logging

**Location**: `logs/mcp_calls.log` (rotated daily)

```json
{
  "timestamp": "2026-05-15T14:23:45Z",
  "subagent": "reviewer_code_quality",
  "mcp_server": "github",
  "tool": "pull_request_read",
  "parameters": {"owner": "r-522", "repo": "harness-engineering", "pr_number": 42},
  "result": "success",
  "duration_ms": 234,
  "cost_tokens": 150
}
```

### Alert Rules

```yaml
alerts:
  - name: "Unusual API Activity"
    condition: "mcp_calls[5m].count > 1000"
    action: "Slack @security-team"
  
  - name: "Auth Failure"
    condition: "mcp_calls[1h] | select(.result == 'auth_failed') | length > 5"
    action: "Rotate credentials immediately"
  
  - name: "Dangerous Tool Invoked"
    condition: "tool in ['delete', 'terminate', 'destroy']"
    action: "Block + alert immediately"
```

---

## 8. Troubleshooting

| Issue | Cause | Solution |
|---|---|---|
| **Auth failure (401)** | Expired token | Rotate credential (see §6), restart session |
| **Rate limit (429)** | Too many API calls | Implement backoff, batch requests, lower parallelism |
| **Timeout (>30s)** | Slow external API | Increase timeout in MCP config, add caching |
| **Tool not found** | Typo in tool name | Check MCP server documentation, list available tools |
| **Permission denied (403)** | Token lacks scope | Regenerate token with broader scopes (if applicable) |

### Debug Commands

```bash
# List active MCP servers
/agents

# Check MCP credentials (safe)
echo $GITHUB_TOKEN | head -c 10  # Show first 10 chars only

# Test API call manually
gh api repos/r-522/harness-engineering

# View MCP logs
tail -f logs/mcp_calls.log | jq '.[] | select(.result == "error")'
```

---

## 9. Approved MCP Servers (Current State)

| Server | Approval Date | Tools | Restrictions | Reviewer |
|---|---|---|---|---|
| GitHub | 2025-02-01 | read PRs, comment, merge | No force-push, no delete | @team-lead |
| Slack | 2025-03-15 | post_message | Public channels only, no delete | @security-team |
| AWS | 2025-04-01 | describe_instances, get_logs | Read-only, 100 req/min | @security-team |

---

## 10. Requesting New MCP Integration

### Step-by-Step Process

1. **Create GitHub Issue** (template):
   ```markdown
   # MCP Integration Request: [ServiceName]
   
   ## Purpose
   Why do we need this? (e.g., "Automate deployment notifications")
   
   ## Threat Model
   [Attach completed threat model from §4.1]
   
   ## Implementation Plan
   [Link to PR with MCP config + subagent definition]
   
   ## Approval Checklist
   - [ ] Threat model reviewed
   - [ ] Credentials secured (.env, not hardcode)
   - [ ] Tests pass (unit + integration)
   - [ ] Tool scoping verified (deny dangerous ops)
   ```

2. **Attach Evidence**:
   - Threat model (§4.1 checklist)
   - Example subagent config (YAML)
   - Test results (pytest output)

3. **Reviewer Approval** (2+ required):
   - @security-team: Threat model, credentials
   - @team-lead: Design, integration points

4. **Activate**:
   - Merge to main
   - Add to `src/config/mcp_servers.yaml`
   - Restart Claude Code session
   - Verify in `/agents` command

---

## 11. Best Practices

✅ **Do**:
- [ ] Store credentials in `.env` (never in code)
- [ ] Scope tools (deny delete, terminate, admin ops)
- [ ] Log all API calls for audit
- [ ] Rotate credentials monthly
- [ ] Test in staging before prod
- [ ] Document tool parameters & error codes
- [ ] Use read-only scopes when possible

❌ **Don't**:
- [ ] Hardcode API keys
- [ ] Grant all tools to all subagents
- [ ] Skip security review
- [ ] Log sensitive data (PII, tokens)
- [ ] Use prod credentials in dev/test
- [ ] Make irreversible API calls (delete, terminate)
- [ ] Ignore rate limits

---

## 12. References

- [Anthropic MCP Documentation](https://modelcontextprotocol.io/)
- [Claude Code Docs: Agent Teams](https://code.claude.com/docs/en/agent-teams)
- [OWASP Top 10: API Security](https://owasp.org/www-project-api-security/)
- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)

---

**Last Updated**: 2026-05-15  
**Next Security Review**: 2026-08-15  
**Emergency Contact**: @security-team
