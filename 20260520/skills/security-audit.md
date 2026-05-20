---
name: security-audit
description: Scan code for hardcoded secrets, SQL injection, unsafe patterns, credential leakage
author: harness-engineering
tags: [security, compliance, audit, verification]
requires: [Bash, Read]
timeout: 180
version: 1.0.0
---

# Security Audit Skill

## Purpose

Scan source code for security vulnerabilities:
- Hardcoded secrets (API keys, passwords, tokens)
- SQL injection vulnerabilities
- Unsafe deserialization (pickle, yaml.unsafe_load)
- Path traversal risks
- Hardcoded credentials in config
- Dependency vulnerabilities (outdated packages)

**Goal**: Catch security issues before code reaches production.

---

## When to Use

- **Before pushing**: `I've added new API integrations. /security-audit`
- **Pre-release**: `Security check before v1.0.0 release. /security-audit`
- **Code review**: `Review this PR for security issues. /security-audit`
- **Dependency update**: `Updated dependencies. Check for vulnerabilities. /security-audit`

---

## Commands

### Quick Scan (Secrets Only)

```bash
pip install bandit
bandit -r src/ -f json | jq '.results[] | select(.severity=="HIGH" or .severity=="CRITICAL")'
```

**Time**: ~10s | **Scope**: High/critical findings only.

### Full Audit (Secrets + Patterns)

```bash
pip install bandit semgrep
bandit -r src/ -f json -o /tmp/bandit.json
semgrep --config=p/security-audit src/ --json -o /tmp/semgrep.json
echo "Audit complete. See results above."
```

**Time**: ~30s | **Scope**: All patterns.

### Dependency Audit

```bash
pip install safety
safety check --json
```

**Time**: ~5s | **Scope**: Known vulnerabilities in installed packages.

### Check for Hardcoded Credentials

```bash
grep -r "password\|api_key\|secret\|token" src/ \
  --include="*.py" \
  --include="*.ts" \
  --include="*.yaml" \
  | grep -v "TODO\|FIXME\|password =" | head -20
```

**Time**: ~2s | **Output**: Matching lines with context.

---

## Example Output

```
========== bandit ==========
HIGH severity:
  /src/mcp/github_integration.py line 42
    Issue: Hardcoded password
    Code: password="prod_secret_12345"
    Remediation: Load from environment variable (os.getenv)

  /src/core/config.py line 89
    Issue: SQL injection risk
    Code: query = f"SELECT * FROM users WHERE id={user_id}"
    Remediation: Use parameterized query (query = "SELECT * FROM users WHERE id=?", params=[user_id])

========== semgrep ==========
CRITICAL:
  /src/agents/executor.py line 156
    Pattern: unsafe yaml.load()
    Fix: Use yaml.safe_load() instead

=========== safety (pip packages) ==========
MEDIUM:
  Package: django (2.1.0)
    Vulnerability: CVE-2021-12345
    Remediation: Upgrade to 2.2.28+
```

---

## Interpretation Guide

| Severity | Meaning | Action | Block Merge? |
|----------|---------|--------|--------------|
| CRITICAL | Code execution risk (eval, pickle, sql inject) | Fix immediately | YES |
| HIGH | Credential leak, auth bypass | Fix before merge | YES |
| MEDIUM | Bad practice, potential issue | Address in PR | NO (but warn) |
| LOW | Minor code smell | Fix if time allows | NO |

---

## Common Findings & Fixes

### Finding: "Hardcoded Password/API Key"

**Bad**:
```python
api_key = "sk-1234567890abcdef"
db_password = "prod_db_secret"
```

**Good**:
```python
api_key = os.getenv("OPENAI_API_KEY")
db_password = os.getenv("DB_PASSWORD")
```

**In .env** (git-ignored):
```
OPENAI_API_KEY=sk-1234567890abcdef
DB_PASSWORD=prod_db_secret
```

### Finding: "SQL Injection"

**Bad**:
```python
user_id = request.args.get("id")
query = f"SELECT * FROM users WHERE id={user_id}"
db.execute(query)
```

**Good**:
```python
user_id = request.args.get("id")
query = "SELECT * FROM users WHERE id=?"
db.execute(query, [user_id])
```

### Finding: "Unsafe YAML Load"

**Bad**:
```python
import yaml
config = yaml.load(user_input)  # Can execute arbitrary code
```

**Good**:
```python
import yaml
config = yaml.safe_load(user_input)  # Only parses basic types
```

### Finding: "Pickle Deserialization"

**Bad**:
```python
import pickle
user_data = pickle.loads(request.data)  # Can execute arbitrary code
```

**Good**:
```python
import json
user_data = json.loads(request.data)  # Safe for structured data
```

---

## Pre-Commit Hook Integration

Automatically run security audit before each commit:

**In .claude/settings.json**:
```json
{
  "hooks": {
    "PreToolUse": {
      "security_audit_check": {
        "script": "scripts/security-pre-commit.sh",
        "blockOnExit": 1
      }
    }
  }
}
```

**In scripts/security-pre-commit.sh**:
```bash
#!/bin/bash
set -e
pip install -q bandit
FINDINGS=$(bandit -r src/ -f json 2>/dev/null | jq '.results[] | select(.severity | IN("HIGH", "CRITICAL"))' | wc -l)
if [ "$FINDINGS" -gt 0 ]; then
  echo "❌ Security audit failed: $FINDINGS high/critical issues found"
  exit 1
fi
echo "✅ Security audit passed"
exit 0
```

---

## Compliance Checklist

Before pushing to main/staging:

- [ ] No hardcoded passwords/API keys
- [ ] No SQL injection vulnerabilities
- [ ] No unsafe deserialization (pickle, yaml.load)
- [ ] No path traversal risks
- [ ] Dependency audit clean (no critical CVEs)
- [ ] `.env` and secrets files are `.gitignore`'d
- [ ] Secrets loaded from environment, not config files
- [ ] All external inputs validated/sanitized

---

## Escalation: If Critical Issue Found

1. **Block the push** (refuse to merge)
2. **Report finding** with evidence and remediation
3. **Fix immediately** (don't defer to later)
4. **Rerun audit** to confirm fix
5. **Document** why vulnerability existed (process improvement)

---

## See Also

- **VERIFY.md**: Full testing & verification strategy
- **DESIGN.md**: Module structure (where to find code to scan)
- **CLAUDE.md**: Prohibited actions (API keys in code)
- **Bandit Docs**: https://bandit.readthedocs.io/
- **Semgrep Docs**: https://semgrep.dev/docs/
- **OWASP Top 10**: https://owasp.org/www-project-top-ten/
