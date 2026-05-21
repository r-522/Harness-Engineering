---
name: test-runner
description: Execute pytest suite, report coverage, flag coverage drops, suggest untested paths
author: harness-engineering
tags: [testing, verification, ci, quality]
requires: [Bash, Read]
timeout: 300
version: 1.0.0
---

# Test Runner Skill

## Purpose

Run the full pytest test suite with coverage analysis. Report:
- Test results (pass/fail counts, duration)
- Coverage percentage by file
- Uncovered lines that should be tested
- Coverage trend (vs. previous run)
- Recommendations for new test cases

## When to Use

- **Verify changes before push**: `I've refactored the orchestrator. /test-runner`
- **Check coverage compliance**: `Does this PR meet the 80% coverage requirement? /test-runner`
- **Diagnose test failures**: `Why are integration tests failing? /test-runner --integration-only`
- **Pre-release validation**: `Full test suite before v1.0.0 release. /test-runner --full`

---

## Commands

### Quick Test (Unit Tests Only)

```bash
pytest tests/unit/ -v --tb=short
```

**Time**: ~15s | **Coverage**: No | **Use When**: Fast feedback on changes.

### Full Test (Unit + Integration)

```bash
pytest tests/ -v --cov=src --cov-report=term-missing --cov-fail-under=80
```

**Time**: ~45s | **Coverage**: Yes (≥80% required) | **Use When**: Ready to push.

### E2E Test (Includes Live MCP)

```bash
pytest tests/e2e/ -v --markers='mcp_live'
```

**Time**: ~120s | **Coverage**: No | **Use When**: Validating external integrations.

### Coverage Report (HTML)

```bash
pytest tests/ --cov=src --cov-report=html
echo "Coverage report: htmlcov/index.html"
```

**Time**: ~50s | **Output**: Interactive coverage map.

---

## Example Output

```
================== test session starts ====================
platform linux -- Python 3.10.0, pytest-8.x.x
collected 87 tests

tests/unit/test_orchestrator.py ........................ [ 30%]
tests/unit/test_subagent.py ........................... [ 60%]
tests/integration/test_orchestration.py ................ [ 90%]
tests/integration/test_error_handling.py ............... [100%]

====================== 87 passed in 42.3s =====================

Name                              Stmts   Miss Cover   Missing
─────────────────────────────────────────────────────────────
src/core/orchestrator.py            145     4    97%   234-236, 501
src/core/subagent.py                98      2    98%   421, 543
src/agents/researcher.py            56      1    98%   189
src/agents/executor.py              67      3    95%   234, 456, 789
─────────────────────────────────────────────────────────────
TOTAL                              489     12    98%

Coverage: 98% (exceeds 80% target) ✅
Trend: +2% since last run ✅
```

---

## Interpretation Guide

| Metric | Healthy | Concerning | Action |
|--------|---------|------------|--------|
| Pass Rate | 100% | <95% | Debug failing tests |
| Coverage | ≥85% | <80% | Add tests for uncovered lines |
| Trend | +1% or stable | -5% | Review what lost coverage |
| Duration | <60s | >180s | Optimize slow tests |

---

## Common Issues & Recovery

### Issue: Test Fails Locally But Not in CI

**Cause**: Environment difference (Python version, missing dependency).

**Fix**:
```bash
python --version           # Check Python version (need 3.10+)
pip list | grep pytest     # Verify pytest installed
pip install -r requirements.txt --upgrade  # Reinstall deps
```

### Issue: Coverage Drops Below 80%

**Cause**: New code without test coverage.

**Fix**:
1. Identify uncovered lines: `pytest tests/ --cov=src --cov-report=term-missing`
2. Write tests for those lines
3. Re-run to verify coverage restored

### Issue: Test Hangs (Timeout >300s)

**Cause**: Infinite loop, deadlock, or slow external API.

**Fix**:
1. Run with `-v` and watch which test hangs
2. Review that test for blocking I/O or infinite loops
3. Add pytest timeout fixture:
   ```python
   @pytest.mark.timeout(10)  # 10s max per test
   def test_my_feature():
       ...
   ```

### Issue: "ModuleNotFoundError" in Tests

**Cause**: Test can't find src/ module.

**Fix**: Ensure PYTHONPATH includes src/:
```bash
export PYTHONPATH="${PYTHONPATH}:$(pwd)/src"
pytest tests/
```

---

## Integration with CI/CD

This skill output feeds GitHub Actions CI:

1. **Local Run** (before push): `/test-runner` → coverage report
2. **PR Check** (GitHub Actions): Automatic run on push
3. **Merge Gate**: CI must pass (all tests + coverage ≥80%)
4. **Release**: `/test-runner --full` before tagging release

---

## Advanced Options

### Run Single Test File

```bash
pytest tests/unit/test_orchestrator.py -v
```

### Run Tests Matching Pattern

```bash
pytest tests/ -k "error_handling" -v
```

### Run with Verbose Output

```bash
pytest tests/ -vv --tb=long
```

### Generate JUnit XML (for CI Integration)

```bash
pytest tests/ --junitxml=test-results.xml
```

---

## See Also

- **VERIFY.md**: Full testing strategy, CI/CD pipeline
- **CLAUDE.md**: Testing commands reference
- **DESIGN.md**: Testing tier breakdown (unit/integration/E2E)
- **pytest Docs**: https://docs.pytest.org/
