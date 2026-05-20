# SKILL_INDEX.md — Skills Directory & Reusable Workflows

**Purpose**: Catalog of custom skills (reusable Claude Code workflows). Skills reduce repetition; use for patterns you invoke frequently. | **Updated**: 2026-05-20

---

## Overview

**Skills** are reusable Claude Code workflows stored in `.claude/skills/`.

Location: `~/.claude/skills/` (user) or `.claude/skills/` (project)

Each skill is a directory with a `SKILL.md` file:

```
.claude/skills/
├── test-runner/
│   ├── SKILL.md
│   ├── scripts/
│   │   └── run_tests.sh
│   └── resources/
│       └── pytest.ini
├── code-review/
│   ├── SKILL.md
│   └── scripts/
│       └── review_checklist.md
└── security-audit/
    ├── SKILL.md
    └── scripts/
        └── bandit_template.sh
```

---

## Creating a Skill

**Step 1**: Create directory:
```bash
mkdir -p .claude/skills/my-skill/scripts
```

**Step 2**: Create `SKILL.md` with YAML frontmatter:

```markdown
---
name: test-runner
description: Run pytest suite, report coverage, suggest improvements
author: team
tags: [testing, verification, ci]
---

# Test Runner Skill

## Trigger
Run this skill to:
- Execute full pytest suite with coverage
- Report results in markdown
- Flag coverage drops
- Suggest test cases for uncovered lines

## Commands
\`\`\`bash
pytest tests/ -v --cov=src --cov-report=term-missing
\`\`\`

## Output Example
[Generated report with coverage analysis]

## See Also
- `.claude/VERIFY.md` for full testing strategy
- `tests/` directory for test structure
```

---

## Built-In Skills (Template)

### 1. test-runner

**Purpose**: Execute tests, report coverage, flag regressions.

**Trigger**: When you say "run tests" or `/test-runner`.

**Output**: Coverage report, failing tests, suggestions.

### 2. code-review

**Purpose**: Review PR changes against coding standards.

**Trigger**: When you say "review this PR" or `/code-review`.

**Output**: Issues found, suggestions, checklist.

### 3. security-audit

**Purpose**: Scan code for secrets, injection vulnerabilities, unsafe patterns.

**Trigger**: When you say "security audit" or `/security-audit`.

**Output**: Findings, severity, remediation steps.

### 4. orchestration-plan

**Purpose**: Decompose task, generate subagent orchestration plan.

**Trigger**: When you say "plan this task" or `/plan`.

**Output**: Parallel/sequential strategy, context budget, risks.

---

## Skill YAML Frontmatter Reference

```yaml
---
name: skill-name                  # kebab-case identifier
description: One-line summary    # What this skill does
author: team | author-name       # Creator
tags: [tag1, tag2]               # Searchable categories
version: 1.0.0                   # Optional: semver
requires: [tool1, tool2]         # Optional: required Claude Code tools
timeout: 300                     # Optional: max execution seconds
---
```

---

## Invoking Skills

**In Claude Code**:
```
I need to review this PR for security issues. /security-audit

Run the full test suite and report coverage. /test-runner

Plan how to parallelize this data processing task. /orchestration-plan
```

**Result**: Claude loads skill markdown, executes embedded commands, integrates output.

---

## Skill Best Practices

1. **Keep skills focused**: One skill = one workflow. Don't combine testing + security into one.

2. **Embed commands**: Include bash/python commands directly in SKILL.md so Claude can copy-paste.

3. **Document output**: Show example output so Claude knows what to expect.

4. **Reference project docs**: Link to CLAUDE.md, DESIGN.md sections so skill context is clear.

5. **Tag for discoverability**: Use consistent tags so users can find skills by category.

6. **Include fallbacks**: If external service (API) is required, explain offline alternative.

7. **Test skills**: Manually invoke each skill on sample data before committing.

---

## Roadmap: Skills to Implement

| Skill | Purpose | Complexity | Benefit |
|-------|---------|-----------|---------|
| test-runner | Pytest orchestration, coverage | Low | High (frequent) |
| code-review | Standards compliance check | Medium | High (PR reviews) |
| security-audit | Secrets, injection, unsafe patterns | Medium | High (compliance) |
| orchestration-plan | Task decomposition, subagent layout | Medium | High (complex tasks) |
| mcp-health-check | Verify MCP integrations working | Low | Medium (troubleshooting) |
| deployment-checklist | Pre-release validation | Medium | Medium (release cycle) |
| cost-audit | Token budget analysis, optimization | High | Medium (optimization) |

---

## Disabling Skills

To temporarily disable all skills:
```bash
# In .claude/settings.json
{
  "disableAllSkills": true
}
```

To disable a single skill:
```bash
# Remove or rename .claude/skills/skill-name/
mv .claude/skills/test-runner .claude/skills/.test-runner.disabled
```

---

## See Also

- **Official Skills Docs**: https://code.claude.com/docs/en/skills
- **Anthropic Skills Examples**: https://github.com/anthropics/skills
- **Skills Directory**: https://www.skillsdirectory.com/
- **CLAUDE.md**: Core operational rules
- **VERIFY.md**: Testing strategy (skill references)
