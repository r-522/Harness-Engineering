# HOOKS.md: Workflow Automation & Event Triggers

## Overview

Hooks are event-driven scripts executed by Claude Code harness in response to git, CI, or manual triggers. They enable transparent automation without manual orchestration.

**Philosophy**: Hooks fail loud, log detailed, escalate intelligently (non-blocking hooks never block workflow).

---

## I. Hook Categories & Execution Model

### Blocking Hooks (Prevent Action)
- **Execution**: Before git commit/push
- **Failure**: Stops action; user must fix and retry
- **Use Case**: Code quality gates (lint, type-check)
- **Cost**: High (must be fast, <10s)

### Non-Blocking Hooks (Notify Only)
- **Execution**: After action completed (merge, deploy, PR)
- **Failure**: Logged + escalated, doesn't block user
- **Use Case**: Notification, logging, cleanup
- **Cost**: Can be slower (background acceptable)

---

## II. Standard Hook Definitions

### Hook 1: `pre-commit`
**Event**: User runs `git commit` with staged files  
**Responsibility**: Prevent committing code with lint/type errors  
**Blocking**: YES (user must fix before commit succeeds)  
**Timeout**: 10 seconds  
**Command**:
```bash
npm run lint --fix && npm run type-check
```

**Actions**:
1. Auto-fix lint issues (`eslint --fix`)
2. Run TypeScript type-check
3. Fail if type errors found (user must fix manually)

**Error Handling**:
- **Type Error**: Print error lines + filename → user fixes → retry commit
- **Lint Error (auto-fixable)**: Auto-fix → re-stage → continue commit
- **Unexpected Error**: Log full stack → abort commit → recommend `npm run lint` manual

---

### Hook 2: `pre-push`
**Event**: User runs `git push` (before network operation)  
**Responsibility**: Verify all tests pass before pushing  
**Blocking**: YES (prevents bad code from reaching remote)  
**Timeout**: 60 seconds (full test suite)  
**Command**:
```bash
npm test -- --coverage && npm run type-check && npm run build
```

**Actions**:
1. Run unit + integration tests
2. Verify TypeScript compilation
3. Build distribution (catch build-time errors)

**Error Handling**:
- **Test Failure**: Print failed test names + error → user debugs → retry push
- **Coverage Below 80%**: Print coverage report → user adds tests → retry push
- **Build Failure**: Print build error → user fixes → retry push

---

### Hook 3: `post-merge`
**Event**: Feature branch merged into main (via PR)  
**Responsibility**: Trigger CI pipeline, notify team  
**Blocking**: NO (merge already succeeded; don't interrupt user)  
**Timeout**: 5 seconds (async delegation)  
**Actions**:
```bash
# Trigger GitHub Actions CI pipeline
gh workflow run test.yml --ref main

# Post notification to team (async)
curl -X POST $SLACK_WEBHOOK \
  -d '{"text":"✅ Feature merged: '$PR_TITLE' → main branch. CI running..."}'
```

**Error Handling**:
- **Workflow Trigger Fails**: Log error → manual trigger reminder → continue
- **Slack Webhook Down**: Log error → continue (non-blocking, don't halt merge)

---

### Hook 4: `on-test-failure`
**Event**: CI pipeline reports test failure  
**Responsibility**: Escalate to developer, log for analysis  
**Blocking**: NO (pipeline already failed; log + notify)  
**Timeout**: 5 seconds  
**Trigger Command** (in GitHub Actions workflow):
```bash
# In .github/workflows/test.yml, on failure:
if [ $? -ne 0 ]; then
  sh scripts/hooks/on-test-failure.sh
fi
```

**Actions**:
```bash
# 1. Log test failure to persistent log
echo "$(date '+%Y-%m-%d %H:%M:%S') | Test failed in branch $(git rev-parse --abbrev-ref HEAD)" >> .claude/logs/failures.log

# 2. Notify original committer
AUTHOR=$(git log -1 --pretty=format:'%an')
gh issue comment $PR_NUM --body "@$AUTHOR Tests failed in #$PR_NUM. See logs: $CI_URL"

# 3. Tag PR for manual review
gh label create "needs-investigation" 2>/dev/null || true
gh pr edit $PR_NUM --add-label "needs-investigation"
```

**Error Handling**:
- **GitHub API Down**: Log error + continue (team sees failed CI status)
- **No PR Context**: Log raw failure, skip GitHub notification

---

### Hook 5: `post-deploy`
**Event**: Deployment to staging/production completes  
**Responsibility**: Verify health, log metrics, notify team  
**Blocking**: NO  
**Timeout**: 30 seconds (health checks)  
**Actions**:
```bash
# 1. Run smoke tests against deployed URL
npx playwright test tests/e2e/smoke.spec.ts --base-url=$DEPLOY_URL

# 2. Log deployment metrics
echo "✅ Deployed $(git rev-parse HEAD) to $ENV at $(date)" >> .claude/logs/deployments.log

# 3. Post success notification
curl -X POST $SLACK_WEBHOOK \
  -d "{\"text\": \"🚀 Deployed to $ENV ($(git log -1 --oneline))\"}"
```

**Error Handling**:
- **Smoke Test Fails**: Rollback triggered (external script); log failure + alert; don't notify success
- **Slack Down**: Log failure → continue (deployment succeeded, just notification issue)

---

## III. Hook Configuration (settings.json)

All hooks registered in `.claude/settings.json`:

```json
{
  "hooks": {
    "pre-commit": {
      "command": "npm run lint --fix && npm run type-check",
      "stages": ["staged-files"],
      "timeout_sec": 10,
      "blocking": true,
      "description": "Auto-fix linting, enforce type safety before commit"
    },
    "pre-push": {
      "command": "npm test -- --coverage && npm run type-check && npm run build",
      "timeout_sec": 60,
      "blocking": true,
      "description": "Full test suite + build verification before push"
    },
    "post-merge": {
      "command": "sh scripts/hooks/post-merge.sh",
      "trigger_ref": "main",
      "timeout_sec": 5,
      "blocking": false,
      "description": "Trigger CI pipeline, notify team of merge"
    },
    "on-test-failure": {
      "command": "sh scripts/hooks/on-test-failure.sh",
      "trigger_event": "github-check-failure",
      "timeout_sec": 5,
      "blocking": false,
      "description": "Escalate test failure, log for analysis"
    },
    "post-deploy": {
      "command": "sh scripts/hooks/post-deploy.sh",
      "trigger_event": "deployment-complete",
      "timeout_sec": 30,
      "blocking": false,
      "description": "Smoke test, log metrics, notify team"
    }
  },
  "hook_env_vars": {
    "SLACK_WEBHOOK": "${SLACK_WEBHOOK_URL}",
    "CI_URL": "${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}/actions/runs/${GITHUB_RUN_ID}",
    "DEPLOY_URL": "${DEPLOY_STAGING_URL}",
    "ENV": "${DEPLOYMENT_ENV}"
  }
}
```

---

## IV. Hook Lifecycle & Error Handling

### Execution Flow

```
Event Triggered
    ↓
Load hook config from settings.json
    ↓
Resolve env vars (${VAR_NAME} → value)
    ↓
Execute command (bash, timeout enforced)
    ↓
┌─────────────────┬─────────────────┐
│ EXIT CODE = 0   │ EXIT CODE ≠ 0   │
│ (Success)       │ (Failure)       │
└────────┬────────┴────────┬────────┘
         ↓                  ↓
   Continue Action    ┌─────────────────────┐
   (Pre-hook only)    │ Blocking Hook?      │
                      ├─────────────────────┤
                      │ YES → Abort action  │
                      │       Show error    │
                      │ NO → Log error      │
                      │      Escalate       │
                      └─────────────────────┘
```

### Logging & Debugging

**Hook Output**:
```bash
# All hook execution logged to:
.claude/logs/hooks.log

# Format:
# TIMESTAMP | HOOK_NAME | STATUS | COMMAND | ERROR_MSG (if failed)
```

**View Recent Hook Activity**:
```bash
tail -50 .claude/logs/hooks.log
```

**Disable Hook Temporarily** (for debugging):
```bash
# Edit settings.json: set "enabled": false for hook
git commit --no-verify  # Bypass pre-commit hook if needed (use carefully)
```

---

## V. Adding Custom Hooks

### Template: New Hook Definition

1. **Define in HOOKS.md** (this file):
```markdown
### Hook N: `my-custom-hook`
**Event**: When X happens  
**Responsibility**: Do Y  
**Blocking**: YES/NO  
**Timeout**: N seconds  
**Command**:
```bash
# Command here
```
**Actions**: (numbered list)
**Error Handling**: (per-error type)
```

2. **Implement Script** (`scripts/hooks/my-custom-hook.sh`):
```bash
#!/bin/bash
set -euo pipefail  # Exit on error, undefined var, pipe fail

# Script body
# Log to .claude/logs/hooks.log
echo "$(date '+%Y-%m-%d %H:%M:%S') | my-custom-hook | START" >> .claude/logs/hooks.log

# Perform action
# ... code ...

# Log completion
echo "$(date '+%Y-%m-%d %H:%M:%S') | my-custom-hook | SUCCESS" >> .claude/logs/hooks.log
```

3. **Register in settings.json**:
```json
{
  "hooks": {
    "my-custom-hook": {
      "command": "sh scripts/hooks/my-custom-hook.sh",
      "timeout_sec": 30,
      "blocking": false
    }
  }
}
```

4. **Test**:
```bash
# Manual trigger (if supported by hook framework)
sh scripts/hooks/my-custom-hook.sh
# Or trigger the event manually
```

---

## VI. Hook Monitoring & Troubleshooting

### Common Issues

| Issue | Diagnosis | Fix |
|---|---|---|
| Hook not executing | Check `settings.json` syntax (JSON valid?) | Use JSON linter |
| Hook timeout | Command takes >timeout_sec | Increase timeout or optimize command |
| Pre-commit keeps failing | Lint/type-check rules too strict | Review ruleset, adjust if needed |
| Hook runs but doesn't escalate | Non-blocking hook doesn't notify | Add Slack/email action to hook |
| Env vars not resolved | `${VAR}` appears literal in output | Verify var defined in hook_env_vars |

### Debug Mode

**Enable detailed logging**:
```bash
DEBUG=hooks npm run dev
# Logs every hook invocation + env var resolution
```

---

## VII. Security Considerations

### Secret Handling in Hooks
- **Never commit secrets** to `scripts/hooks/*.sh`
- **Use GitHub Secrets**: `${GITHUB_TOKEN}`, `${SLACK_WEBHOOK_URL}` (injected by CI)
- **Local testing**: Create `.env.local`, add to `.gitignore`

### Privilege Escalation
- **Hooks run with user permissions** (not elevated)
- **Can't modify system files** (by design)
- **Can read/write project files** only

---

## Version & Alignment

- **Last Updated**: 2026-05-02
- **Blocking Strategy**: Pre-commit/pre-push block (quality gates); post-* notify only
- **Aligned With**: CLAUDE.md (automation rules), DESIGN.md (event-driven pattern)

