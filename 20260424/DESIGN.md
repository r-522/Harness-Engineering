# DESIGN.md - アーキテクチャ設計原則

## Design Philosophy

### 1. Context Locality
```
Principle:
  関連する情報は物理的に近い場所に配置
  
Example:
  ✗ Bad: Main logic in A.js, config in B.json, docs in C.md
  ✓ Good: Logic, config, docs are co-located with clear boundaries

Pattern:
  [Feature]/
    ├── feature.js       (logic)
    ├── feature.config   (configuration)
    ├── feature.test     (tests)
    └── README.md        (documentation)
```

### 2. Tool Minimalism
```
Principle:
  Fewer, more powerful tools > Many, specialized tools
  
Example:
  ✗ Bad: 50 micro-tools (tool selection becomes a problem)
  ✓ Good: 10-15 well-scoped tools (clear mental model)

Numbers:
  - Optimal: 8-15 tools
  - Acceptable: 5-30 tools
  - Dangerous: 50+ tools
  - Broken: 100+ tools

Audit:
  Monthly review to remove unused/redundant tools
```

### 3. Explicit > Implicit
```
Principle:
  明示的な宣言・制御 > 魔法・暗黙的な動作

Code Style:
  ✗ Bad: try_operation_if_possible()
  ✓ Good: 
    if preconditions_met():
        perform_operation()

Configuration:
  ✗ Bad: Agent auto-selects tools
  ✓ Good: Explicit tool declarations in CLAUDE.md

Error Handling:
  ✗ Bad: Silent failures with auto-recovery
  ✓ Good: Explicit error reporting
```

### 4. Composition > Inheritance
```
Principle:
  Re-composable components > Deep class hierarchies

Pattern:
  ✗ Bad: 
    class Agent(Tool, Memory, Logger, Auth):
        pass

  ✓ Good:
    agent = Agent(
        tools=toolkit,
        memory=memory_manager,
        logger=logger,
        auth=auth_provider
    )
```

### 5. Measurable > Implicit
```
Principle:
  定量的な指標 > 曖昧な"良さ"

Metrics:
  ✓ Latency (ms)
  ✓ Token usage (%)
  ✓ Success rate (%)
  ✓ Tool accuracy (%)
  ✗ "Performance is good"
  ✗ "Code quality is fine"
```

## Architectural Patterns

### Pattern 1: Tool Routing
```
Use case: Model と Task complexity に基づいて最適なツールを選択

Structure:
    [Request]
       ↓
    [Analyze Complexity]
       ├─ Trivial → Use Haiku + Read/Edit
       ├─ Moderate → Use Sonnet + Bash/Bash
       └─ Complex → Use Opus + WebSearch/Agent

Benefits:
  - Token efficiency
  - Latency optimization
  - Cost reduction
  - Resource allocation

Example:
    if task.tokens < 1K:
        use(haiku, read, edit)
    elif task.tokens < 10K:
        use(sonnet, bash, websearch)
    else:
        use(opus, agent, monitor)
```

### Pattern 2: Context Layers
```
Use case: 多層的な context を効率的に管理

Structure:
    Layer 0 (Core)      - CLAUDE.md, AGENT.md (永続)
    Layer 1 (Task)      - Current task description, requirements
    Layer 2 (History)   - Recent turns (5-10個)
    Layer 3 (Reference) - Code snippets, configs (on-demand)
    Layer 4 (Archive)   - Completed tasks, old outputs (compressed)

Compression Strategy:
    if context_usage > 80%:
        compress(Layer 4)  # Archive
        remove(Layer 3)    # Reference
        summarize(Layer 2) # History to 5 turns

Benefits:
  - Bounded context size
  - Predictable latency
  - Full audit trail (if needed)
```

### Pattern 3: Validation Pipeline
```
Use case: Output quality assurance at multiple stages

Structure:
    [Code Generated]
         ↓
    [Type Check]
         ↓
    [Lint/Format]
         ↓
    [Security Scan]
         ↓
    [Unit Tests]
         ↓
    [Integration Tests]
         ↓
    [User Review] ← Final gate

Benefits:
  - Early error detection
  - Consistent quality
  - Clear failure points
  - Audit trail
```

### Pattern 4: Graceful Degradation
```
Use case: Tools の失敗時に代替手段を用意

Structure:
    Primary Tool (Fast, feature-rich)
         ↓ (if fails)
    Secondary Tool (Slower, robust)
         ↓ (if fails)
    Manual Fallback (Ask user)

Examples:
    Tool selection:
        ✓ Try WebSearch (fast)
        ↓ fails
        ✓ Try local file search
        ↓ fails
        → Ask user for clarification

    Error handling:
        ✓ Try Bash command
        ↓ fails
        ✓ Retry with different approach
        ↓ fails
        → Report issue and ask user
```

### Pattern 5: Checkpoint & Recovery
```
Use case: Long-running operations の中断・再開

Structure:
    Start
      ↓ Step 1 [checkpoint: "step1_complete"]
      ↓ Step 2 [checkpoint: "step2_complete"]
      ↓ Step 3 [checkpoint: "step3_complete"]
    End

Recovery:
    If interrupted at Step 2:
        Load last checkpoint
        Resume from Step 2
        Preserve prior outputs

Benefits:
  - Token efficiency
  - Fault tolerance
  - Reproducibility
```

## Code Organization

### File Structure Principle
```
Feature-driven organization:
  src/
    feature-a/
      ├── index.js
      ├── feature-a.test.js
      ├── feature-a.config
      └── README.md
    feature-b/
      ├── index.js
      ├── feature-b.test.js
      └── README.md

NOT Module-type organization:
  src/
    ├── models/
    ├── views/
    ├── controllers/
    ├── utils/
    └── (scattered across)
```

### Naming Conventions
```
Variables: snake_case
    ✓ user_data, api_response, max_retries

Classes: PascalCase
    ✓ UserManager, APIClient, ConfigParser

Constants: UPPER_SNAKE_CASE
    ✓ MAX_RETRIES, DEFAULT_TIMEOUT, API_KEY

Functions: verb_noun (snake_case)
    ✓ get_user, validate_input, process_items

Files: kebab-case with extension
    ✓ user-manager.js, api-client.ts, config-parser.py
```

## Quality Gates

### Pre-Commit Gate
```
Must pass:
  ✓ Type check (mypy, TypeScript, etc)
  ✓ Linting (eslint, pylint)
  ✓ Formatting (black, prettier)
  ✓ Security scan (bandit, safety)

Failure → Block commit
```

### Pre-Push Gate
```
Must pass:
  ✓ All pre-commit checks
  ✓ Unit tests (100% modified code)
  ✓ Integration tests
  ✓ Code review (if applicable)

Failure → Block push
```

### Merge Gate
```
Must pass:
  ✓ All pre-push checks
  ✓ CI/CD pipeline
  ✓ Performance tests
  ✓ Security review (for sensitive changes)

Approval: At least 1 reviewer
```

## Performance Targets

### Latency SLO
```
Simple tasks (Haiku):      < 2s
Moderate tasks (Sonnet):   2-8s
Complex tasks (Opus):      8-30s
Timeout:                    60s (hard limit)
```

### Token Efficiency
```
Context usage:  < 70% of available
Tool efficiency: > 95% (tool calls succeed)
Model routing:   90%+ accuracy in selection
```

### Code Quality
```
Type coverage:  > 95%
Test coverage:  > 80% (critical paths)
Lint score:     0 errors
Security issues: 0 critical, < 5 medium
```

## Decision Records

Template for architectural decisions:

```markdown
### ADR: [Decision Title]

**Status**: Proposed / Accepted / Deprecated

**Context**:
[Why are we making this decision?]

**Decision**:
[What are we doing?]

**Consequences**:
[What becomes harder/easier?]

**Alternatives considered**:
[What else could we do?]

**References**:
[Links to related docs/issues]
```

---

**Version**: 1.0  
**Effective**: 2026-04-24  
**Review**: Quarterly
