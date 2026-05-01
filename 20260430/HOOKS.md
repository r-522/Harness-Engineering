# HOOKS.md - Automation Hooks & Pre/Post-Execution Rules

**Version**: 1.0 | **Last Updated**: 2026-04-30

---

## 1. Hooks Fundamentals

### What are Hooks?

Hooks are shell commands that execute automatically in response to Claude Code events:

```
Event triggers:
├── Pre-commit: Before files are committed
├── Post-commit: After commit completes
├── Pre-push: Before pushing to remote
├── Post-push: After push completes
├── Pre-session: When new session starts
├── Post-session: When session ends
└── Custom: Via `/loop` or `settings.json` triggers
```

**Difference from Agent automation:**
- **Hooks**: Shell scripts, run on the machine (can access filesystem, run any command)
- **Agent automation**: Claude decisions, run in Claude's context (can use tools)

---

## 2. Hook Configuration Structure

### Global Hooks (~/.claude/hooks.json)

```json
{
  "version": "1.0",
  "enabled": true,
  "hooks": {
    "pre-commit": {
      "description": "Validate before committing",
      "command": "sh",
      "args": ["-c", "scripts/validate.sh"],
      "cwd": "${PROJECT_DIR}",
      "timeout": 30,
      "onError": "prompt"
    },
    "post-push": {
      "description": "Notify team after push",
      "command": "curl",
      "args": [
        "-X", "POST",
        "${SLACK_WEBHOOK}",
        "-d", "New commit pushed"
      ],
      "timeout": 10,
      "onError": "ignore"
    }
  }
}
```

### Project-Level Hooks (.claude/hooks.json)

Same structure, but only applies to this project:

```json
{
  "hooks": {
    "pre-commit": {
      "command": "sh",
      "args": ["-c", "make lint"],
      "onError": "fail"
    }
  }
}
```

**Field definitions:**
- `command` (required): Executable to run
- `args` (required): Arguments as array
- `cwd` (optional): Working directory (default: project root)
- `timeout` (optional): Max seconds before killing process (default: 60)
- `onError` (optional): "prompt" (ask), "fail" (stop), "ignore" (continue)

---

## 3. Common Hook Patterns

### Pattern 1: Pre-Commit Validation

**Goal**: Ensure CLAUDE.md and configs are valid before committing

```json
{
  "hooks": {
    "pre-commit": {
      "description": "Validate Markdown and JSON before commit",
      "command": "sh",
      "args": ["-c", "scripts/pre-commit.sh"],
      "onError": "fail"
    }
  }
}
```

**Script**: `.claude/scripts/pre-commit.sh`
```bash
#!/bin/bash
set -e

echo "[PRE-COMMIT] Validating configuration files..."

# 1. Validate JSON
echo "  - Checking JSON syntax..."
find . -name "*.json" -not -path "./node_modules/*" | while read f; do
  jq empty "$f" || { echo "    ✗ $f"; exit 1; }
done
echo "  ✓ JSON valid"

# 2. Check for secrets
echo "  - Scanning for secrets..."
if grep -r "sk_live_\|api_key.*=\|password" . --include="*.json" \
   --exclude-dir=node_modules --exclude-dir=.git; then
  echo "  ✗ Potential secrets found"
  exit 1
fi
echo "  ✓ No secrets detected"

# 3. Validate CLAUDE.md exists and is reasonable
echo "  - Checking CLAUDE.md..."
if [ ! -f "CLAUDE.md" ]; then
  echo "  ✗ CLAUDE.md missing"
  exit 1
fi
lines=$(wc -l < CLAUDE.md)
if [ $lines -gt 80 ]; then
  echo "  ⚠ CLAUDE.md over 80 lines ($lines) - consider archiving"
fi
echo "  ✓ CLAUDE.md valid"

echo "[PRE-COMMIT] ✓ All checks passed"
exit 0
```

### Pattern 2: Post-Push Notifications

**Goal**: Notify Slack when code is pushed

```json
{
  "hooks": {
    "post-push": {
      "description": "Notify Slack of push",
      "command": "sh",
      "args": ["-c", "scripts/notify-slack.sh"],
      "onError": "ignore"
    }
  }
}
```

**Script**: `.claude/scripts/notify-slack.sh`
```bash
#!/bin/bash

BRANCH=$(git rev-parse --abbrev-ref HEAD)
COMMIT_HASH=$(git rev-parse --short HEAD)
COMMIT_MSG=$(git log -1 --pretty=%B)
AUTHOR=$(git log -1 --pretty=%an)

SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

curl -X POST "$SLACK_WEBHOOK" \
  -H 'Content-type: application/json' \
  -d @- <<EOF
{
  "text": "🚀 Code Pushed",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Branch:* $BRANCH\n*Commit:* $COMMIT_HASH\n*Author:* $AUTHOR\n*Message:* $COMMIT_MSG"
      }
    }
  ]
}
EOF
```

### Pattern 3: Pre-Session Setup

**Goal**: Load environment variables and dependencies when Claude starts

```json
{
  "hooks": {
    "pre-session": {
      "description": "Setup environment",
      "command": "sh",
      "args": ["-c", "scripts/setup.sh"],
      "onError": "fail"
    }
  }
}
```

**Script**: `.claude/scripts/setup.sh`
```bash
#!/bin/bash

# Load environment from .env.local
if [ -f ".env.local" ]; then
  export $(grep -v '^#' .env.local | xargs)
fi

# Check Node version
if ! node --version | grep -q "v18\|v20"; then
  echo "⚠ Node version may be incompatible"
fi

# Install dependencies if needed
if [ ! -d "node_modules" ] && [ -f "package.json" ]; then
  echo "Installing dependencies..."
  npm ci --silent
fi

echo "✓ Session environment ready"
```

### Pattern 4: Periodic Health Check

**Goal**: Run tests every 30 minutes (use `/loop` command)

```bash
# In Claude conversation:
/loop 30m npm test

# Or via hook:
{
  "hooks": {
    "periodic-test": {
      "description": "Run tests every 30 minutes",
      "command": "npm",
      "args": ["test"],
      "interval": 1800,
      "timeout": 120,
      "onError": "ignore"
    }
  }
}
```

---

## 4. Environment Variables in Hooks

### Available Variables

```bash
# Git context
${GIT_BRANCH}         # Current branch name
${GIT_COMMIT}         # Latest commit SHA
${GIT_AUTHOR}         # Commit author name

# Project context
${PROJECT_DIR}        # Root directory
${PROJECT_NAME}       # Repo name

# System
${SLACK_WEBHOOK}      # From settings.local.json
${GITHUB_TOKEN}       # From environment
${HOME}               # User home directory

# Claude context
${CLAUDE_SESSION_ID}  # Current session ID
${CLAUDE_MODEL}       # Active model (opus, sonnet, etc.)
```

### Setting Variables

**In ~/.claude/settings.local.json:**
```json
{
  "env": {
    "SLACK_WEBHOOK": "https://hooks.slack.com/services/T00.../B00.../XXXX",
    "GITHUB_TOKEN": "ghp_...",
    "DEPLOY_ENV": "staging"
  }
}
```

**In system environment:**
```bash
export SLACK_WEBHOOK="https://..."
export GITHUB_TOKEN="ghp_..."
```

---

## 5. Hook Execution Order & Conditions

### Execution Sequence

```
User action (commit/push)
    ↓
Check: Is hook enabled? (settings.json: enabled: true)
    ↓
Match event? (pre-commit, post-push, etc.)
    ↓
Check conditions (if specified)
    ↓
Run command with timeout
    ↓
Check: Did command succeed?
    ├─ onError: "fail" → Stop (show error)
    ├─ onError: "prompt" → Ask user (continue/abort)
    └─ onError: "ignore" → Continue silently
    ↓
Next step
```

### Conditional Execution

```json
{
  "hooks": {
    "pre-commit": {
      "command": "npm",
      "args": ["test"],
      "conditions": {
        "branch": ["main", "develop"],
        "filesChanged": ["src/**/*", "test/**/*"],
        "timeOfDay": "09:00-18:00"
      },
      "onError": "prompt"
    }
  }
}
```

**Condition types:**
- `branch`: Only run on specific branches (array)
- `filesChanged`: Only if matching files were modified (glob patterns)
- `timeOfDay`: Only during business hours (HH:MM-HH:MM)
- `environment`: Only if env var is set (object with var:value pairs)

---

## 6. Debugging & Troubleshooting Hooks

### Issue: Hook Not Executing

**Checklist:**
```
1. Is hookExecutionMode enabled?
   $ grep hookExecutionMode ~/.claude/settings.json
   Expected: "enabled"

2. Is the hook defined for the right event?
   $ grep -A5 "pre-commit" .claude/hooks.json
   Check: correct event name

3. Do conditions match?
   $ git rev-parse --abbrev-ref HEAD
   Check: branch matches "branch" condition

4. Can the command execute?
   $ sh -c "your-command"
   Check: no permission errors
```

### Issue: Hook Fails Silently

**Enable debug logging:**
```bash
# In settings.json
{
  "hookDebugMode": true,
  "hookLogPath": "/tmp/claude-hooks.log"
}

# View logs
tail -f /tmp/claude-hooks.log
```

### Issue: Hook Timeout

**Increase timeout and check for slow operation:**
```json
{
  "hooks": {
    "pre-commit": {
      "timeout": 120,  # Increase from 30 to 120 seconds
      "command": "sh",
      "args": ["-c", "time scripts/validate.sh"]
    }
  }
}
```

---

## 7. Hook Cooperation with Agent Automation

### Pattern: Hook Prepares, Agent Executes

```
Hook (pre-commit):
  └─ Validates config files exist
  
Claude Agent (automation):
  ├─ Reads validation results from hook log
  ├─ Runs additional checks (no hardcoded values)
  ├─ Commits if all pass
  └─ Post-commit hook notifies team
```

### Pattern: Agent Triggers Hook

```
Claude Agent:
  ├─ Makes code changes
  ├─ Runs git commit (triggers pre-commit hook)
  └─ Hook validates changes
```

---

## 8. Best Practices

**DO:**
- ✅ Keep hooks fast (< 30 seconds)
- ✅ Log detailed output (for debugging)
- ✅ Use environment variables for secrets (never hardcode)
- ✅ Make hooks idempotent (safe to run multiple times)
- ✅ Test hook scripts locally before deploying
- ✅ Version hooks in git (.claude/scripts/)
- ✅ Use `onError: "ignore"` for non-critical notifications

**DON'T:**
- ❌ Run long-running tasks in pre-commit hooks (blocks commit)
- ❌ Hardcode sensitive data in hook definitions
- ❌ Use interactive commands (curl -i) that wait for input
- ❌ Assume specific shell (use `sh -c` for portability)
- ❌ Modify files in pre-commit hooks (causes commit loops)
- ❌ Use `timeout: 0` (no limit — can hang indefinitely)
- ❌ Rely on hooks for security validation (also do it in code)

---

## 9. Real-World Hook Examples

### Example 1: Multi-Step Validation

```bash
#!/bin/bash
# .claude/scripts/pre-commit.sh

set -e

echo "🔍 Pre-commit validation..."

# Step 1: Format check
echo "  • Checking formatting..."
npm run format --check || {
  echo "  ✗ Format check failed. Run: npm run format"
  exit 1
}

# Step 2: Lint
echo "  • Running linter..."
npm run lint || {
  echo "  ✗ Lint errors found"
  exit 1
}

# Step 3: Test
echo "  • Running tests..."
npm test -- --bail || {
  echo "  ✗ Tests failed"
  exit 1
}

# Step 4: Security
echo "  • Scanning for secrets..."
if grep -r "password\|api_key" src/ --include="*.ts" --include="*.js"; then
  echo "  ✗ Hardcoded secrets detected"
  exit 1
fi

echo "✓ All checks passed!"
```

### Example 2: Smart Notifications

```bash
#!/bin/bash
# .claude/scripts/post-push.sh

# Only notify if pushing to main
BRANCH=$(git rev-parse --abbrev-ref HEAD)
if [ "$BRANCH" != "main" ]; then
  exit 0
fi

COMMIT=$(git rev-parse --short HEAD)
STATS=$(git log --oneline -1)

# Send to Slack
curl -s "${SLACK_WEBHOOK}" \
  -d "text=📍 Main branch updated: $COMMIT - $STATS" \
  > /dev/null

# Send to Discord (optional)
if [ ! -z "$DISCORD_WEBHOOK" ]; then
  curl -s -X POST "$DISCORD_WEBHOOK" \
    -H "Content-Type: application/json" \
    -d "{\"content\":\"Main updated: $STATS\"}" \
    > /dev/null
fi
```

---

## 10. Troubleshooting Reference

| Symptom | Cause | Fix |
|---------|-------|-----|
| Hook runs every time, even after passing | `onError: "prompt"` keeps asking | Change to `"fail"` or `"ignore"` |
| Command not found in hook | Wrong PATH in hook environment | Use absolute paths or source .profile |
| Secrets leaked in hook output | Echo sensitive env vars | Use `${VARIABLE}` (quoted) in scripts |
| Hook timeout on first run | Download/installation happens first time | Increase timeout or use pre-session hook |
| Post-push hook fails, breaks workflow | Hook error not gracefully handled | Set `onError: "ignore"` for notifications |

---

## References

- [Claude Code Documentation](https://code.claude.com/docs) — Official docs (no specific hooks page, but in "Automation" section)
- [Git Hooks Guide](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) — Standard git hooks patterns
- [Bash Scripting Best Practices](https://mywiki.wooledge.org/BashGuide) — Shell script quality
- [Advanced Claude Code Techniques (2026)](https://smartscope.blog/en/generative-ai/claude/claude-code-best-practices-advanced-2026/) — Hooks + subagents integration
