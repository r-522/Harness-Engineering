# SKILLS.md - Reusable Capability Definitions

**Version**: 2026-05-15  
**Scope**: Claude Code Skills for Harness Engineering  
**Framework**: Anthropic Skills (cross-platform: Code, API, Web, Mobile)

---

## 1. Core Skills Overview

| Skill | Domain | Trigger Keywords | Model | Tools | Cost |
|---|---|---|---|---|---|
| `research_literature` | Information gathering | "research", "survey", "find", "analyze trends" | Opus 4.7 | WebSearch, WebFetch | Medium |
| `code_review` | Static analysis | "review", "audit code", "check for bugs" | Opus 4.7 | Git-diff, Bash (grep) | Medium |
| `data_validation` | Data integrity | "validate", "check schema", "anomaly" | Sonnet 4.6 | Bash (awk, jq), Read | Low |
| `test_orchestration` | Verification | "run tests", "coverage", "E2E" | Sonnet 4.6 | Bash (pytest), Read | Low-Medium |
| `deployment_automation` | Release management | "deploy", "release", "tag version" | Opus 4.7 | Git, Bash (make) | High |
| `documentation_sync` | Knowledge management | "update docs", "sync README", "changelog" | Sonnet 4.6 | Read, Edit | Low |
| `security_audit` | Compliance & safety | "security review", "threat model", "secrets" | Opus 4.7 | Secret scanning, MCP | High |

---

## 2. Skill Definitions (Detailed)

### Skill: `research_literature`

**Purpose**: Gather, synthesize, and present relevant information from web sources.

**Trigger Conditions**:
- User asks to "research X", "find best practices for Y", "survey Z topic"
- Task requires current information (date-dependent)
- Multiple sources needed for credibility

**Typical Workflow**:
```
1. Parse user query (extract keywords, scope constraints)
2. Formulate search queries (WebSearch: 3-5 variants)
3. Fetch full content (WebFetch: top 3-5 results)
4. Synthesize findings (extract key points, compare perspectives)
5. Present summary (markdown with source citations)
```

**Tools Provided**:
- `WebSearch`: Query engine (rate: 10 req/min)
- `WebFetch`: Retrieve and parse HTML (timeout: 30s)
- `Read`: Access local reference documents

**Output Format**:
```markdown
# Summary: [Topic]
[3-5 bullet points]

## Key Findings
- Finding 1 (Source: URL)
- Finding 2 (Source: URL)
...

Sources:
- [Title](URL)
```

**Context Cost**: ~10-15k tokens (query + 5 fetches + synthesis)  
**Typical Duration**: 1-2 minutes  
**Cost Optimization**: Batch queries (combine 3 related searches into 1)

**Example Invocation** (Subagent frontmatter):
```yaml
---
name: researcher_trends_2026
model: claude-opus-4-7
tools:
  - WebSearch
  - WebFetch
  - Read
---
You are a research specialist. When asked about trends, synthesize info 
from multiple sources and highlight consensus vs. outlier opinions.
```

---

### Skill: `code_review`

**Purpose**: Analyze code changes for correctness, security, and maintainability.

**Trigger Conditions**:
- User asks to "review PR", "check this code", "audit for bugs"
- Pull request exists but needs detailed feedback
- Security concern suspected

**Typical Workflow**:
```
1. Retrieve code diff (git diff, PR API)
2. Analyze against criteria (SOLID, DRY, security, style)
3. Identify violations (functions > 30 lines, hardcoded secrets, N+1 queries)
4. Categorize by severity (CRITICAL, WARNING, MINOR)
5. Suggest fixes (with code examples where possible)
```

**Tools Provided**:
- `Bash` (git commands): `git diff`, `git log`
- `Read`: Inspect full files for context
- `MCP GitHub`: Fetch PR metadata, comments

**Output Format**:
```markdown
# Code Review: [PR Title]

## Critical Issues
1. [Issue]: [Location] → [Fix]
2. ...

## Warnings
1. ...

## Style & Suggestions
1. ...

Approval Status: ✅ APPROVED / ⚠️ REQUEST CHANGES / 📝 COMMENT
```

**Context Cost**: ~8-12k tokens (diff analysis + context lookups)  
**Typical Duration**: 2-5 minutes  
**Cost Optimization**: Set scope limits (e.g., "review only [file] changes")

**Example Invocation**:
```yaml
---
name: reviewer_code_quality
model: claude-opus-4-7
tools:
  - Bash
  - Read
  - mcp__github__pull_request_read
---
You are a senior code reviewer. Focus on SOLID violations, security issues,
and maintainability. Reference CLAUDE.md and DESIGN.md for style guide.
```

---

### Skill: `data_validation`

**Purpose**: Verify data integrity, schema compliance, and anomaly detection.

**Trigger Conditions**:
- CSV/JSON/SQL data needs validation
- Schema migration requires verification
- Data quality gate before processing

**Typical Workflow**:
```
1. Load dataset (CSV, JSON, query results)
2. Define schema (required fields, types, constraints)
3. Check completeness (missing values, nulls)
4. Validate types (int, string, date format)
5. Detect anomalies (outliers, duplicates)
6. Generate report
```

**Tools Provided**:
- `Bash`: awk, jq, sort, uniq
- `Read`: Inspect sample data
- `WebFetch`: Retrieve remote datasets

**Output Format**:
```
Data Validation Report: [dataset.csv]
✅ 9,950 / 10,000 records valid (99.5%)

Issues Found:
- Missing values in column 'email': 45 records
- Type mismatch in 'age' (string vs int): 12 records
- Duplicate IDs: 3 records

Anomalies:
- Age outlier: 999 (row 4521)
- Salary > 10x median: 8 records

Recommendations:
1. Backfill missing 'email' or exclude rows
2. Cast 'age' to integer, validate 0-120 range
```

**Context Cost**: ~3-5k tokens (analysis + lightweight processing)  
**Typical Duration**: 1-3 minutes  
**Cost Optimization**: Use Sonnet 4.6; process streaming for large datasets

---

### Skill: `test_orchestration`

**Purpose**: Execute test suites, track coverage, verify CI/CD gates.

**Trigger Conditions**:
- "run tests", "check coverage", "E2E validation"
- Pre-commit / pre-merge gate
- Regression detection needed

**Typical Workflow**:
```
1. Identify test scope (unit, integration, E2E)
2. Run pytest with coverage flag
3. Parse results (pass/fail count, coverage %)
4. Flag regressions (coverage drop, new failures)
5. Generate report + recommendations
```

**Tools Provided**:
- `Bash`: pytest, coverage.py
- `Read`: Review test code if needed

**Output Format**:
```
Test Results
===========
Passed: 127 / 130 tests (97.7%)
Skipped: 2 (known flaky tests)
Failed: 1 (test_concurrent_write - REGRESSION)

Coverage: 84% → 82% (↓ 2%) [ALERT]

Recommendations:
1. Investigate test_concurrent_write (new failure)
2. Add tests for new codebase sections (lines 340-380 uncovered)
```

**Context Cost**: ~5-8k tokens (execution + result parsing)  
**Typical Duration**: 2-10 minutes (depends on test suite size)  
**Cost Optimization**: Skip slow tests with `-k` flag, use Sonnet 4.6

---

### Skill: `deployment_automation`

**Purpose**: Orchestrate releases, tag versions, manage rollbacks.

**Trigger Conditions**:
- User requests "deploy to staging/production"
- Release checklist needed
- Version bump + tag requested

**Typical Workflow**:
```
1. Verify readiness (tests pass, changelog updated)
2. Bump version (semantic versioning)
3. Create tag + push
4. Trigger CI/CD pipeline
5. Monitor deployment (health checks, logs)
6. Verify rollback plan
```

**Tools Provided**:
- `Bash`: git, make, kubectl/docker
- `MCP GitHub`: Create releases, tag management
- `Monitor`: Watch deployment logs

**Output Format**:
```
Deployment: v2.1.0 → [Staging]
Checklist:
✅ Tests passing (127/127)
✅ Changelog updated
✅ Security review completed
✅ Docs synced

Timeline:
- 14:23 UTC: Tag created (v2.1.0)
- 14:25 UTC: CI pipeline triggered
- 14:35 UTC: Staging deployment complete
- 14:36 UTC: Health checks PASS

Rollback Plan:
git revert v2.1.0, retag as v2.0.1-hotfix
```

**Context Cost**: ~8-12k tokens (automation + monitoring)  
**Typical Duration**: 5-15 minutes  
**Cost Optimization**: Use Opus 4.7 only; cost is inherent (safety-critical)

---

### Skill: `documentation_sync`

**Purpose**: Auto-update docs when code changes, keep README current.

**Trigger Conditions**:
- Code changes detected (especially API changes)
- User requests "update docs"
- Release process needs changelog

**Typical Workflow**:
```
1. Identify code changes (git diff)
2. Extract changes (new functions, API updates, config)
3. Update docs (README.md, API reference, CHANGELOG)
4. Validate links
5. Commit + push (or suggest manual review)
```

**Tools Provided**:
- `Read`: Scan code & docs
- `Edit`: Update .md files
- `Bash`: git commands

**Output Format**:
```
Documentation Sync Report
Files updated: 3
- README.md (+ 5 lines, new installation section)
- CHANGELOG.md (+ 2 entries for v2.1.0)
- docs/API.md (updated endpoint list)

Suggested changes (review before commit):
[Diff preview]
```

**Context Cost**: ~4-6k tokens (analysis + editing)  
**Typical Duration**: 1-2 minutes  
**Cost Optimization**: Use Sonnet 4.6; safe to automate with review gate

---

### Skill: `security_audit`

**Purpose**: Threat modeling, vulnerability scanning, compliance checks.

**Trigger Conditions**:
- New MCP server integration
- Hardcoded secrets detected
- Security review requested
- Pre-deployment audit needed

**Typical Workflow**:
```
1. Identify assets (code, configs, API endpoints)
2. Scan for secrets (API keys, tokens, credentials)
3. Threat model (attack surfaces, unauthorized access)
4. Check OWASP top 10 (injection, XSS, CSRF, etc.)
5. Rate severity + recommend mitigations
```

**Tools Provided**:
- `mcp__github__run_secret_scanning`: Detect secrets
- `Read`: Code inspection
- `Bash`: grep for patterns

**Output Format**:
```
Security Audit Report
Date: 2026-05-15

Findings:
🔴 CRITICAL
  - Hardcoded AWS_KEY in src/config.py:42
    → Move to .env, use boto3.Session() instead

🟡 WARNING
  - SQL query concatenation in db/query.py:156
    → Use parameterized queries (sqlalchemy.text())

🟢 INFO
  - HTTPS enforced in CI/CD (✓)
  - No CORS wildcards detected (✓)

Compliance Status: 85% (8/10 controls)
```

**Context Cost**: ~10-15k tokens (deep analysis)  
**Typical Duration**: 5-10 minutes  
**Cost Optimization**: Mandatory Opus 4.7 (security-critical); no cost optimization

---

## 3. Skill Composition & Chaining

### Example: Full Feature Workflow
```yaml
Orchestrator launches:
  ├─ Parallel:
  │  ├── Subagent A (researcher_trends): Find best practices
  │  ├── Subagent B (code_review): Analyze existing codebase
  │  └── Subagent C (data_validation): Validate test data
  ├─ Wait for results (↑3 agents)
  └─ Serial:
     ├── Implement feature (main agent + Subagent D)
     ├── Run test_orchestration
     └── Run security_audit
     └── deployment_automation (if prod)
```

**Context Allocation**:
- Total: 200k tokens
- Researchers (3): 50k
- Dev + QA: 75k
- Reserve: 25k

---

## 4. Skill Activation & Deactivation

### Enable a Skill
```bash
# Add to .claude/SKILLS.md (this file)
# Update agents.yaml entry
# Restart Claude Code session
# /agents command (create new subagent with skill)
```

### Disable a Skill
```bash
# Remove from SKILLS.md
# Comment out in agents.yaml
# Delete corresponding .md in subagents/
# Session restart not required (no active references)
```

---

## 5. Cost Optimization Strategies

| Strategy | Savings | Trade-off |
|---|---|---|
| Use Sonnet 4.6 for validation-heavy tasks | ~50% | Slightly lower accuracy |
| Batch WebSearch queries | ~30% | Slower iteration |
| Cache API responses (1hr TTL) | ~20% | Stale data risk |
| Parallelize independent subagents | ~40% wall-clock time | Higher context peak |
| Skip slow tests with `-k` filter | ~60% for unit tests | Coverage blind spots |

**Default Optimization Profile** (conservative):
- Use Opus 4.7 unless explicitly low-risk
- Batch searches when possible
- Enable caching by default
- Parallelize only for independent tasks

---

## 6. Monitoring & Observability

### Metrics Per Skill

| Metric | Tool | Alert Threshold |
|---|---|---|
| **Execution Time** | Monitor tool | > 10 min (timeout risk) |
| **Token Consumption** | Claude API logs | > context allocation |
| **Error Rate** | Session logs | > 10% failed attempts |
| **Cost** | Billing dashboard | > 2x historical avg |

### Example: Monitor Research Skill
```bash
Monitor command:
  every 30s: log execution time, check for timeouts
  alert if: > 15 min elapsed
  stop when: researcher subagent completes
```

---

## 7. Troubleshooting & Common Issues

| Issue | Cause | Solution |
|---|---|---|
| **WebFetch timeout** | Slow server or network | Increase timeout, try alt source |
| **Git diff too large** | PR > 100 files | Scope to specific files (`-p <file>`) |
| **Test suite hangs** | Concurrent test bug | Run serially (`-n0`) or skip flaky tests |
| **Secret scanning false positive** | Regex mismatch | Tune patterns in MCP config |
| **Coverage drop after refactor** | Dead code removed or new complexity | Regenerate baseline, add tests |

---

## 8. Template: Define a New Skill

To add a new skill `my_skill`:

```yaml
---
name: [skill_name]
domain: [domain_name]
triggers: ["keyword1", "keyword2"]
model: [claude-opus-4-7 | claude-sonnet-4-6]
tools:
  - [Tool1]
  - [Tool2]
cost: [Low | Medium | High]
---

# My Skill Documentation

## Purpose
[One sentence: what does this skill do?]

## Trigger Conditions
- [Condition 1]
- [Condition 2]

## Workflow
1. [Step 1]
2. [Step 2]
...

## Tools Provided
[List tools]

## Output Format
[Example markdown output]

## Context Cost
~[X]k tokens

## Example Invocation
[Subagent definition with YAML frontmatter]
```

---

**Last Updated**: 2026-05-15  
**Next Review**: 2026-08-15  
**Maintainer**: Harness Engineering Team

**References**:
- [Anthropic Skills Documentation](https://anthropic.skilljar.com/introduction-to-agent-skills)
- [GitHub - ColinMcNamara: Understanding Skills, Agents, Subagents](https://colinmcnamara.com/blog/understanding-skills-agents-and-subagents-and-mcp-in-claude-code)
