# VERIFY.md — Verification Workflows & Testing Gates

**Purpose**: Verification phase design, testing integration, CI/CD gates, QA procedures. Read when setting up test runs or validating outputs. | **Updated**: 2026-05-20

---

## Verification Phase (Post-LLM Deterministic Checks)

**Pattern**: Execute → Verify → Reconcile

After Claude/subagent generates output (code, config, etc.), run deterministic checks before merge:

### Step 1: Syntax & Schema Validation

```bash
# Code generation
python -m py_compile <generated_file>          # Syntax check
mypy <generated_file> --strict                 # Type checking

# Config generation
yamllint <generated_file>                      # YAML syntax
jsonschema validate --instance <file> --schema <schema.json>

# Security
bandit <generated_file> -f json -o report.json # Find hardcoded secrets
```

### Step 2: Deterministic Testing

```bash
# Unit tests
pytest tests/unit -v --tb=short -x            # Stop on first failure

# Integration tests (isolated from external services)
pytest tests/integration -v --markers='not mcp_live'

# Schema/output validation
pytest tests/validators/ -v                    # Custom validators
```

### Step 3: Reconciliation Report

```markdown
## Verification Report

### ✅ Passed Checks
- Syntax: OK
- Type checking: 0 errors
- Schema validation: OK
- Security scan: 0 high-severity findings

### ⚠️ Warnings (non-blocking)
- 2 unused imports (mypy)
- Test coverage: 75% (target: 80%)

### ❌ Blocking Issues
(if any)

### Decision
[Proceed to merge / Require human review]
```

---

## Testing Integration

### Pre-Push Checks (Local)

**Must pass before git push**:
```bash
# Format check
black --check src/ tests/
ruff check src/ tests/

# Lint
pylint src/ --disable=missing-docstring

# Type check
mypy src/ --strict

# Unit tests
pytest tests/unit/ -v --cov=src --cov-fail-under=80

# Security
bandit -r src/ -f json
```

### CI/CD Gates (GitHub Actions)

**On Pull Request**:
1. Linting (ruff, black, pylint)
2. Type checking (mypy)
3. Unit tests (pytest, ≥80% coverage)
4. Integration tests (subagent orchestration)
5. Security scan (bandit, semgrep)

**Required Status**: All checks pass before merge.

**On Merge to main**:
1. Full test suite (unit + integration + E2E)
2. Coverage report (must maintain ≥80%)
3. Build artifact generation
4. Auto-tag release (if version bumped)
5. Deploy to staging

---

## Verification Checklist

### For Code Generation Tasks

- [ ] Generated code has zero syntax errors
- [ ] Type checking passes (`mypy --strict`)
- [ ] Follows project naming conventions (snake_case, PascalCase)
- [ ] Function length ≤ 30 lines
- [ ] Cyclomatic complexity ≤ 10
- [ ] No hardcoded secrets, API keys, or credentials
- [ ] Docstrings for public functions (WHY only, not WHAT)
- [ ] Unit tests written (coverage ≥80%)
- [ ] Import statements valid (all dependencies installed)

### For Configuration/Manifest Generation

- [ ] Valid YAML/JSON syntax
- [ ] Schema validation passes
- [ ] No undefined environment variables
- [ ] All referenced files/modules exist
- [ ] Semantic correctness (e.g., port not in use)

### For Data Processing Outputs

- [ ] Output schema matches specification
- [ ] No null/empty required fields
- [ ] Numeric fields in expected ranges
- [ ] Timestamps in ISO 8601 format
- [ ] No PII leakage (names, emails, IPs)

---

## Test Execution Examples

### Run All Tests Locally

```bash
# Fast: unit tests only
pytest tests/unit/ -v

# Full: unit + integration (no external APIs)
pytest tests/ -v --markers='not mcp_live' --cov=src --cov-fail-under=80

# E2E: includes live MCP (slow, requires network)
pytest tests/e2e/ -v --markers='mcp_live'
```

### Test Single Feature

```bash
# Test orchestrator pattern
pytest tests/integration/test_orchestrator.py::test_parallel_spawn -v

# Test subagent error handling
pytest tests/integration/test_subagent.py -k error_handling -v
```

### Measure Coverage

```bash
# Report by file
pytest tests/ --cov=src --cov-report=term-missing

# HTML report
pytest tests/ --cov=src --cov-report=html
open htmlcov/index.html
```

---

## CI/CD Pipeline (GitHub Actions)

### .github/workflows/ci.yml (Pseudocode)

```yaml
name: CI

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with: { python-version: '3.10' }
      - run: pip install ruff black mypy pylint
      - run: ruff check src/ tests/
      - run: black --check src/ tests/
      - run: mypy src/ --strict
      - run: pylint src/ --disable=missing-docstring

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: pip install -r requirements.txt pytest pytest-cov
      - run: pytest tests/ -v --cov=src --cov-fail-under=80
      - run: coverage report

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: pip install bandit semgrep
      - run: bandit -r src/ -f json -o bandit-report.json
      - run: semgrep --config=p/security-audit src/

  build:
    needs: [lint, test, security]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: python -m build
      - run: twine upload dist/ (if version bump)
```

---

## Regression Testing

**On Each PR**: Run full test suite. If >10% coverage drop → block merge.

**Weekly**: Run E2E tests against staging environment (includes live MCP).

**Pre-Release**: Manual QA checklist (feature acceptance tests).

---

## Common Verification Failures & Recovery

| Failure | Root Cause | Recovery |
|---------|-----------|----------|
| Type check fails (mypy) | Type annotation missing | Add `-> ReturnType:` or use `@overload` |
| Coverage drops below 80% | New code lacks tests | Write tests for uncovered branches |
| Import error in CI (works locally) | Missing dependency in requirements.txt | Add to requirements.txt, reinstall |
| Flaky test (passes locally, fails in CI) | Race condition or temp files | Mock timers, use tmpdir fixture |
| Bandit flags hardcoded secret | API key in code | Move to `.env`, load via `os.getenv()` |
| Semgrep detects SQL injection | User input in query | Use parameterized queries |

---

## See Also

- **DESIGN.md**: Module structure, testing strategy
- **AGENT.md**: Subagent error handling, escalation
- **CLAUDE.md**: Testing commands, coverage requirements
