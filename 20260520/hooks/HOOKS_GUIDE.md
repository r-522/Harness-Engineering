# HOOKS_GUIDE.md — Claude Code Hooks for Workflow Automation

**Purpose**: Configure automated checks/actions at specific lifecycle points. Use for high-value automation (linting, security checks, notifications). | **Updated**: 2026-05-20

---

## Overview

**Hooks** are shell commands executed at lifecycle events in Claude Code:

- **PreToolUse**: Before any tool (Bash, Read, Edit) executes
- **PostToolUse**: After tool completes
- **PostEdit**: After file edit
- **PrePush**: Before git push
- **Notification**: When Claude generates notification
- **Stop**: When agent finishes turn

**Configuration**: `.claude/settings.json` (project-level) or `~/.claude/settings.json` (user-wide)

---

## Hook Configuration (settings.json)

```json
{
  "hooks": {
    "PreToolUse": {
      "linting_check": {
        "script": "scripts/lint-check.sh",
        "blockOnExit": 1,
        "description": "Check formatting before edits"
      }
    },
    "PostEdit": {
      "format_after_edit": {
        "script": "scripts/auto-format.sh",
        "blockOnExit": 0
      }
    },
    "PrePush": {
      "security_audit": {
        "script": "scripts/security-pre-push.sh",
        "blockOnExit": 1
      }
    }
  }
}
```

---

## Hook Examples

### 1. Pre-Tool Linting Check

**Trigger**: Before Claude runs any Bash/Edit command.

**Use Case**: Prevent pushing code that doesn't meet formatting standards.

**Script** (`scripts/lint-check.sh`):
```bash
#!/bin/bash
set -e

# Check Python formatting
if command -v black &> /dev/null; then
  black --check src/ tests/ 2>/dev/null || {
    echo "⚠️ Code not formatted. Run: black src/ tests/"
    exit 1
  }
fi

# Check ruff linting
if command -v ruff &> /dev/null; then
  ruff check src/ tests/ 2>/dev/null || {
    echo "⚠️ Linting errors. Run: ruff check src/ --fix"
    exit 1
  }
fi

echo "✅ Formatting and linting OK"
exit 0
```

**Config**:
```json
{
  "hooks": {
    "PreToolUse": {
      "linting_check": {
        "script": "scripts/lint-check.sh",
        "blockOnExit": 1
      }
    }
  }
}
```

**Effect**: If linting fails, Claude cannot execute Bash/Edit commands. Must fix first.

---

### 2. Auto-Format After Edit

**Trigger**: After Claude edits a Python file.

**Use Case**: Auto-format code to match project style.

**Script** (`scripts/auto-format.sh`):
```bash
#!/bin/bash

# Auto-format Python files
if command -v black &> /dev/null; then
  black src/ tests/ 2>/dev/null
fi

if command -v ruff &> /dev/null; then
  ruff check src/ tests/ --fix 2>/dev/null
fi

echo "✅ Auto-formatted"
exit 0
```

**Config**:
```json
{
  "hooks": {
    "PostEdit": {
      "auto_format": {
        "script": "scripts/auto-format.sh",
        "blockOnExit": 0
      }
    }
  }
}
```

**Effect**: After each edit, code is automatically formatted. Non-blocking (Claude continues).

---

### 3. Security Audit Before Push

**Trigger**: Before git push.

**Use Case**: Prevent secrets from being committed.

**Script** (`scripts/security-pre-push.sh`):
```bash
#!/bin/bash
set -e

echo "🔐 Running pre-push security checks..."

# Check for hardcoded secrets
SECRETS=$(git diff --cached | grep -E '(password|api_key|secret|token)' | wc -l)
if [ "$SECRETS" -gt 0 ]; then
  echo "❌ Found potential secrets in staged changes. Aborting push."
  echo "Review with: git diff --cached | grep -E '(password|api_key)'"
  exit 1
fi

# Run bandit on changed Python files
if command -v bandit &> /dev/null; then
  CHANGED_PY=$(git diff --cached --name-only | grep '.py$' || true)
  if [ -n "$CHANGED_PY" ]; then
    bandit -r $CHANGED_PY -f json 2>/dev/null | \
      jq '.results[] | select(.severity | IN("HIGH", "CRITICAL"))' > /tmp/bandit_findings.json
    CRITICAL=$(wc -l < /tmp/bandit_findings.json)
    if [ "$CRITICAL" -gt 0 ]; then
      echo "❌ Security issues found. Fix before push."
      cat /tmp/bandit_findings.json
      exit 1
    fi
  fi
fi

echo "✅ Security checks passed"
exit 0
```

**Config**:
```json
{
  "hooks": {
    "PrePush": {
      "security_audit": {
        "script": "scripts/security-pre-push.sh",
        "blockOnExit": 1
      }
    }
  }
}
```

**Effect**: If security issues found, push is blocked.

---

### 4. Test Coverage Check Before Push

**Trigger**: Before git push.

**Use Case**: Ensure coverage thresholds met before merging.

**Script** (`scripts/coverage-pre-push.sh`):
```bash
#!/bin/bash
set -e

echo "📊 Checking test coverage..."

# Run pytest with coverage
pytest tests/ --cov=src --cov-fail-under=80 -q 2>/dev/null || {
  echo "❌ Coverage below 80%. Run: pytest tests/ --cov=src --cov-report=term-missing"
  exit 1
}

echo "✅ Coverage check passed (≥80%)"
exit 0
```

**Config**:
```json
{
  "hooks": {
    "PrePush": {
      "coverage_check": {
        "script": "scripts/coverage-pre-push.sh",
        "blockOnExit": 1
      }
    }
  }
}
```

---

### 5. Notification on Test Failure

**Trigger**: When Claude Code generates a notification (e.g., test failure).

**Use Case**: Log test failures for audit trail.

**Script** (`scripts/log-notification.sh`):
```bash
#!/bin/bash

# Log notification timestamp + message
echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" >> .claude/logs/notifications.log

# Alert if critical
if echo "$*" | grep -q "CRITICAL\|FAILED"; then
  echo "🚨 Critical notification logged to .claude/logs/notifications.log"
fi

exit 0
```

**Config**:
```json
{
  "hooks": {
    "Notification": {
      "log_failures": {
        "script": "scripts/log-notification.sh",
        "blockOnExit": 0
      }
    }
  }
}
```

---

## Hook Lifecycle Reference

| Event | Timing | Use Case | blockOnExit=1 Effect |
|-------|--------|----------|----------------------|
| **PreToolUse** | Before Bash/Read/Edit/etc. | Validation, pre-checks | Prevent tool if script fails |
| **PostToolUse** | After tool completes | Post-processing, logging | Warn if script fails |
| **PostEdit** | After file edit | Auto-format, notifications | Continue regardless |
| **PrePush** | Before git push | Final security gate | Block push if fails |
| **Notification** | When notification fires | Logging, metrics | Doesn't block Claude |
| **Stop** | Agent turn completes | Cleanup, summary | Doesn't block |

---

## Configuration Best Practices

### Do's ✅

- Keep hook scripts **fast** (<5s typical)
- **Exit 0** on success, **exit 1** on failure
- Use `blockOnExit: 1` for **critical gates** (security, tests)
- Use `blockOnExit: 0` for **nice-to-have** (auto-formatting)
- Log output for **debugging** if issues arise
- **Test hooks locally** before committing

### Don'ts ❌

- Don't run expensive operations (full test suite in PreToolUse)
- Don't modify files hook script doesn't control
- Don't use hooks for logic that should be in code
- Don't configure >3 hooks per event (too many checks)
- Don't forget to make scripts executable (`chmod +x`)

---

## Debugging Hooks

**See hook output**:
```bash
# View configured hooks
/hooks

# Run hook manually (for testing)
bash scripts/lint-check.sh
echo $?  # Show exit code (0 = success, 1 = failure)
```

**Disable all hooks temporarily**:
```json
{
  "disableAllHooks": true
}
```

**Disable single hook**:
```json
{
  "hooks": {
    "PreToolUse": {
      "linting_check": {
        "disabled": true
      }
    }
  }
}
```

---

## Hook + Skill Integration

**Skill** = reusable workflow (test-runner, security-audit)  
**Hook** = automated action at lifecycle event

**Example**: Security-Audit Skill + Pre-Push Hook

- **Skill** (`/security-audit`): Manual full scan when you need it
- **Hook** (`PrePush`): Auto-run critical checks before push (lightweight)

**Complementary**: Skill for deep dives, hook for quick gates.

---

## Remote Execution Environment Considerations

When using Claude Code on the web (cloud execution):

- Hooks run in the remote container
- Scripts must be self-contained (no external dependencies assumed)
- File system is ephemeral (logs written to container temp space)
- Network access follows environment policy (Trusted vs. Full)

**Best practice**: Keep hooks independent; don't rely on external services.

---

## See Also

- **Official Hooks Docs**: https://code.claude.com/docs/en/hooks-guide
- **CLAUDE.md**: Operational rules
- **skills/**: Reusable workflows (test-runner, security-audit)
- **VERIFY.md**: Testing & verification strategy
