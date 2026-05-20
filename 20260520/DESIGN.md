# DESIGN.md — Architecture & Implementation Reference

**Purpose**: Detailed design patterns, module structure, decision rationale. Read when implementing major features. | **Updated**: 2026-05-20

---

## Module Dependency Graph

```
src/
├── core/
│   ├── orchestrator.py      (Orchestrator class, task decomposition)
│   ├── subagent.py          (Subagent spawn/monitor)
│   ├── config.py            (YAML parsing, settings)
│   └── errors.py            (Custom exceptions)
├── agents/
│   ├── researcher.py        (WebSearch, WebFetch specialist)
│   ├── executor.py          (Bash, test execution)
│   ├── analyzer.py          (Code analysis, static checks)
│   └── validator.py         (Verification phase)
├── skills/
│   ├── test_runner.py       (Test orchestration skill)
│   ├── code_reviewer.py     (Code review skill)
│   └── deployment.py        (CD pipeline skill)
└── mcp/
    ├── github_integration.py
    └── database_integration.py
```

**Dependency Rules**:
- Agents depend on core, not each other
- MCP integrations isolated in `src/mcp/`
- Skills are self-contained; reference via import

---

## Orchestrator (Core Pattern)

```python
class Orchestrator:
    """Pure coordinator—no task execution."""
    
    def decompose(self, task: str) -> Plan:
        """Break task into independent subagent assignments."""
    
    def spawn_parallel(self, assignments: List[Task], max_workers: int = 3):
        """Launch up to 3 workers simultaneously."""
    
    def monitor_context(self):
        """Alert if any worker exceeds 30K tokens."""
    
    def integrate_results(self, results: List[Result]) -> FinalOutput:
        """Merge subagent findings, reconcile conflicts."""
    
    def escalate(self, error: Exception, subagent_id: str):
        """Route unrecoverable errors to user with recommendation."""
```

---

## Subagent Roles

### researcher_*

**Tools**: WebSearch, WebFetch, Read  
**Timeout**: 300s  
**Use Case**: Parallel data gathering, competitive analysis, documentation review

**Example**:
```python
@subagent
def researcher_competitor_features(company: str):
    """Analyze feature set of competitor."""
    # WebSearch → parse → structure findings
    return structured_comparison
```

### executor_*

**Tools**: Bash, Read, Edit  
**Timeout**: 600s  
**Use Case**: Testing, linting, build verification

**Example**:
```python
@subagent
def executor_unit_tests(test_pattern: str):
    """Run pytest suite, return coverage report."""
    # pytest → parse output → return metrics
    return coverage_report
```

### analyzer_*

**Tools**: Read, Bash (grep/find)  
**Timeout**: 180s  
**Use Case**: Code review, security scan, dependency analysis

**Example**:
```python
@subagent
def analyzer_security_scan(src_dir: str):
    """Check for secrets, hardcoded keys, unsafe patterns."""
    # bandit, semgrep → report findings
    return vulnerabilities
```

### validator_*

**Tools**: Bash (verification only, no side effects)  
**Timeout**: 120s  
**Use Case**: Deterministic validation phase

**Example**:
```python
@subagent
def validator_output_schema(output_file: str, schema: str):
    """Verify JSON/YAML conforms to schema."""
    # load file, validate, report schema errors
    return validation_result
```

---

## Verification Phase (Post-LLM)

Pattern: **Execute → Verify → Reconcile**

```
1. LLM generates code/config
2. Validator runs deterministic checks:
   - Type checking (mypy)
   - Schema validation
   - Syntax checks
   - Security scans (no hardcoded secrets)
3. If mismatches → report deltas to user before merge
4. If all pass → proceed to CI/merge
```

**Example**:
```python
# After code generation
generated_code = llm_output
validation_errors = validator_syntax_check(generated_code)
if validation_errors:
    report_mismatches(generated_code, validation_errors)
else:
    merge_to_branch()
```

---

## Sequential vs. Parallel Decision Tree

**Use SEQUENTIAL when**:
- Phase B depends on output of Phase A
- Task complexity is high and benefits from full context
- No independent parallelizable units exist

**Use PARALLEL when**:
- Tasks are mutually independent
- Each task is self-contained (≤ 30K tokens)
- Total time savings justify coordination overhead

**Decision Heuristic**:
```
Can Phase B start before Phase A finishes?
  ├─ NO → Sequential (feed output forward)
  └─ YES & independent? → Parallel (spawn 2-3 workers)

If >3 independent tasks → batch in rounds:
  Round 1: A, B, C (parallel)
  Round 2: D, E, F (parallel)
  Merge: combine all results
```

---

## Configuration Management

**settings.yaml**: Team defaults (build commands, test thresholds, deployment targets)

**agents.yaml**: Subagent specifications (timeout, max_retries, tool allocations)

```yaml
# agents.yaml
researcher:
  model: sonnet-4.6
  max_parallel: 3
  timeout_seconds: 300
  tools:
    - WebSearch
    - WebFetch

executor:
  max_parallel: 2
  timeout_seconds: 600
  tools:
    - Bash
    - Read
    - Edit
```

---

## Error Scenarios & Mitigations

| Scenario | Root Cause | Mitigation |
|----------|-----------|-----------|
| Subagent context explodes to 50K | Task too large for single agent | Orchestrator splits into 2 tasks |
| API rate limits (WebSearch hits quota) | Parallel calls exceed limits | Backoff + throttle queue |
| Code generation fails syntax check | LLM hallucination | Route to human review, not auto-merge |
| Test suite hangs (>600s) | Infinite loop in generated code | Timeout trap, escalate with diff |
| Memory usage spikes | MCP server leak | Isolate → investigate → update MCP |

---

## Testing Strategy

**Unit Tests** (`tests/unit/`): Test core classes (Orchestrator, config parsing) in isolation.

**Integration Tests** (`tests/integration/`): Spawn 2-3 test subagents, validate merge logic.

**E2E Tests** (`tests/e2e/`): Full orchestration pipeline, including MCP, verification phase.

**Coverage Threshold**: ≥ 80% (enforced in CI).

```bash
# Run all tiers
pytest tests/unit tests/integration tests/e2e -v --cov=src --cov-fail-under=80
```

---

## Deployment Model

1. **Commit to feature/* branch**
2. **Run local tests** (CI gates auto-run on PR)
3. **Code review** (1 team member minimum)
4. **Merge to develop** (integration branch)
5. **Tag v*.*.*** on develop (semver)
6. **Auto-deploy** to staging + production

---

## See Also

- **AGENT.md**: Orchestration personas, error handling
- **VERIFY.md**: Verification workflows, testing gates
- **CLAUDE.md**: Operational rules, naming conventions
