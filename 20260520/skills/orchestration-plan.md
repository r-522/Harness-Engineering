---
name: orchestration-plan
description: Decompose complex tasks into parallel subagent work, show context budget, risks, and fallbacks
author: harness-engineering
tags: [orchestration, planning, subagent, verification]
requires: [Agent, Read]
timeout: 120
version: 1.0.0
---

# Orchestration Plan Skill

## Purpose

Before executing complex tasks with subagents, create and review a plan:

- **Task Analysis**: Complexity, dependencies, estimated time/cost
- **Decomposition**: Break into independent subagent assignments
- **Parallelization**: Sequential vs. parallel decision
- **Resource Planning**: Context budget, token estimates
- **Risk Identification**: API limits, context overrun, failure modes
- **Fallback Strategy**: What to do if a subagent fails

**Goal**: Prevent wasted tokens and failed orchestrations by planning first.

---

## When to Use

- **Complex Data Processing**: `I need to analyze 5 competitors' pricing pages. Let me plan this first. /orchestration-plan`
- **Parallel Code Review**: `Review this large PR across 3 specialist agents. /orchestration-plan`
- **Batch External API Calls**: `I need to pull data from 10 API endpoints. /orchestration-plan`
- **Multi-Stage Pipeline**: `Design a 3-phase data pipeline. /orchestration-plan`

---

## Plan Template

```markdown
# Execution Plan: [Task Title]

## Task Analysis

**Objective**: [One-sentence goal]

**Complexity**: [Low / Medium / High]
- [Reason 1]
- [Reason 2]

**Estimated Duration**: XX seconds
**Estimated Cost**: $X.XX (vs. Opus-only: $Y.YY)

---

## Proposed Subagent Distribution

| # | Agent | Task | Context | Tools | Timeout |
|---|-------|------|---------|-------|---------|
| 1 | researcher_A | [Task A] | 25K | WebSearch, Read | 300s |
| 2 | executor_B | [Task B] | 30K | Bash, Edit | 600s |
| 3 | [optional] | [Task C] | 25K | [tools] | Xs |

**Total Assigned**: 75K / 100K budget (leaving 25K orchestrator + 10% margin)

---

## Execution Strategy

### Parallelization
- **Parallel Phase**: agents 1, 2, 3 run simultaneously
- **Merge Phase**: Orchestrator integrates results → final output
- **Sequential Fallback**: If any agent fails, fall back to single-agent approach

### Estimated Timeline
1. Spawn subagents: 2s
2. Parallel execution: ~45s
3. Integrate & report: 5s
4. **Total**: ~52s (vs. sequential: ~180s)

---

## Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|-----------|
| **API Rate Limit** (WebSearch × 3) | High | Medium | Throttle requests; use fallback search |
| **Context Overrun** (Worker >30K) | Medium | High | Monitor per-agent budget; split if needed |
| **Network Failure** | Low | Medium | Retry up to 3× with exponential backoff |
| **Merge Conflict** (conflicting results) | Low | Low | Manual reconciliation by orchestrator |

---

## Fallback Strategy

**If researcher_A fails**:
- Retry with different search terms
- Fall back to manual documentation review
- Continue with researchers B + C; incorporate A's findings later

**If executor_B fails**:
- Escalate with error logs
- Offer manual verification alternative
- Block merge until resolved

**If context overrun**:
- Orchestrator splits remaining work into Round 2
- Re-prioritize high-value subagents first

---

## Success Criteria

- [ ] All subagents complete within timeout
- [ ] Context usage ≤ 85% per agent
- [ ] No API rate limit errors
- [ ] Merged output matches specification
- [ ] Verification phase passes (if applicable)

---

## Decision

**Proceed with this plan?** [Yes / No / Modify]

If **No** or **Modify**: Provide feedback and request revised plan.
```

---

## Example Plans

### Example 1: Competitive Analysis (3 Parallel Researchers)

```markdown
# Plan: Analyze 3 Competitors' Pricing

## Task Analysis
- Objective: Compare pricing pages of competitors A, B, C
- Complexity: Medium (3 independent searches + synthesis)
- Est. Duration: 60s | Est. Cost: $0.12

## Subagent Distribution
| # | Agent | Task |
|---|-------|------|
| 1 | researcher_A | Scrape competitor_a.com pricing page |
| 2 | researcher_B | Scrape competitor_b.com pricing page |
| 3 | researcher_C | Scrape competitor_c.com pricing page |

## Execution
- Parallel: A, B, C fetch pages simultaneously
- Merge: Orchestrator creates comparison table
- Cost Savings: 3× Sonnet << 1× Opus (parallel scales)

## Fallback
- If A fails: Manual lookup competitor_a.com
- Re-run once; escalate if 2nd attempt fails
```

### Example 2: Code Review Across 3 Reviewers

```markdown
# Plan: Security + Performance + Maintainability Review

## Decomposition
| # | Agent | Specialism | Focus |
|---|-------|-----------|-------|
| 1 | analyzer_security | Security | SQL injection, secrets, auth bypass |
| 2 | analyzer_perf | Performance | N+1 queries, cache misses, optimization |
| 3 | analyzer_maintainability | Code Quality | Complexity, naming, documentation |

## Execution
- Parallel: Each analyzer reviews full PR with their lens
- Merge: Orchestrator groups findings by file + severity
- Report: Consolidated review with severity levels

## Timeline
- Single-agent review: ~5 min
- Parallel 3-agent review: ~2 min (3× faster)
```

### Example 3: Failed Orchestration Recovery

```markdown
# Plan (Attempt 2): Retry with Sequential Fallback

## Original Plan
- Parallel: A, B, C
- Result: A succeeded, B timed out, C succeeded

## Revised Plan
- Phase 1: A, C (already done) ✅
- Phase 2: B (retry with extended timeout 600s)
- Merge: B results + A + C

## Why Sequential?
B appears to have stalled on external API.
Sequential allows manual API check between phase 1 & 2.
```

---

## Using Plan Mode in Claude Code

**Command**: Use the `/plan` command or ask explicitly:

```
I need to orchestrate 5 data collection tasks. Let's plan this before execution.
```

**Claude Response**: Shows comprehensive plan from template above.

**Your Action**:
1. Review plan (complexity, cost, fallbacks reasonable?)
2. Suggest modifications (e.g., "reduce to 2 parallel agents")
3. Approve or reject

**Execution**: Once approved, Claude spawns subagents per plan.

---

## Metrics to Track

After each orchestration execution:

```markdown
## Orchestration Metrics

- **Subagent Success Rate**: 3/3 (100%)
- **Parallel Duration**: 47s (vs plan: 60s) ✅
- **Context Usage**: Agent A: 22K (budget: 25K), B: 28K (30K), C: 19K (25K) ✅
- **Cost**: $0.18 (vs plan: $0.12) ⚠️ (overestimate OK)
- **Quality**: Merged output passes verification ✅
```

**Compare to plan**: Did actual match estimates? Learn for next plan.

---

## Anti-Patterns (What NOT to Do)

❌ **Don't spawn 10 subagents** (context explosion)
❌ **Don't parallelize sequential work** (will fail or merge incorrectly)
❌ **Don't skip planning for "simple" tasks** (definition of simple changes)
❌ **Don't ignore risks** (APIs will rate-limit, networks will fail)
❌ **Don't merge without verification** (trust but verify)

---

## See Also

- **AGENT.md**: Orchestrator persona, error handling, context budgets
- **DESIGN.md**: Orchestration patterns, module dependencies
- **VERIFY.md**: Verification phase after orchestration
- **CLAUDE.md**: Operational rules, parallelization limits
