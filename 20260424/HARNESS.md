# HARNESS.md - Harness Engineering 実装パターン集

## Harness Engineering とは

> **定義**: "Enforce quality with mechanisms, not prompts"
>
> Agent failures はモデルの問題ではなく、**configuration と context structure の問題**である。

**核心**: プロンプト文言を完璧にするより、context と configuration を厳密に構造化することで、long-running agents は劇的に信頼性と効率が向上する。

---

## 12 Agentic Harness Patterns（2026年版）

### Tier 1: Memory & Context Patterns

#### Pattern 1.1: Layered Context Management
```
Problem:
  Context window が満杯になると、agent は過去の context を喪失
  → Task failures, loop bugs, inconsistency

Solution:
  Context を明示的に層別管理：
    - L0: Core rules (CLAUDE.md, AGENT.md)
    - L1: Current task description
    - L2: Recent history (last 5 turns)
    - L3: Cached references (code, configs)
    - L4: Archive (compressed history)

Implementation:
  on_context_threshold_80_percent():
      compress_archive()      # L4 → summarize
      remove_cache()          # L3 → optional
      summarize_history()     # L2 → condensed
      preserve_task()         # L1 → locked
      preserve_core()         # L0 → locked

Benefit:
  ✓ Bounded token growth
  ✓ Predictable latency
  ✓ No context-loss failures
```

#### Pattern 1.2: Persistent Memory Tokens
```
Problem:
  Agent は session 内でのみ memory を持つ
  → Cross-session consistency なし

Solution:
  Critical learnings を session-persistent storage に保存

Implementation:
  # After each error
  if error_pattern_identified():
      lessons = {
          "pattern": "tool X fails on Y",
          "workaround": "use tool Z instead",
          "since": timestamp,
          "frequency": count
      }
      save_to(persistent_memory, lessons)
      
  # At session start
  on_session_start():
      load_lessons_from(persistent_memory)
      prepend_to_context()

Benefit:
  ✓ Agent learns across sessions
  ✓ Fewer repeated errors
  ✓ Continuous improvement
```

#### Pattern 1.3: Context Compression Hooks
```
Implementation:
  // Pre-defined compression strategy
  if token_usage > 80%:
      // Aggressive
      compress_strategy = "summarize_all_but_recent_5_turns"
  elif latency > 5s:
      // Moderate
      compress_strategy = "remove_old_tool_outputs"
  else:
      // Gentle
      compress_strategy = "archive_completed_tasks"
      
  apply_compression(compress_strategy)

Benefit:
  ✓ Automatic resource management
  ✓ Latency control
  ✓ No manual intervention
```

---

### Tier 2: Workflow & Orchestration Patterns

#### Pattern 2.1: Explicit Task Decomposition
```
Problem:
  Agent に vague task を与える
  → Unfocused efforts, scope creep, token waste

Solution:
  Upfront に task を明示的に分割

Implementation:
  {
    "task": "Build API with auth",
    "subtasks": [
      {
        "id": "auth-schema",
        "title": "Design authentication schema",
        "dependencies": [],
        "success_criteria": ["schema.sql validates"]
      },
      {
        "id": "auth-endpoint",
        "title": "Implement /login endpoint",
        "dependencies": ["auth-schema"],
        "success_criteria": ["POST /login returns token"]
      },
      {
        "id": "auth-middleware",
        "title": "Add middleware for protected routes",
        "dependencies": ["auth-endpoint"],
        "success_criteria": ["GET /protected requires token"]
      }
    ]
  }

Benefit:
  ✓ Clear priorities
  ✓ Dependency detection
  ✓ Progress tracking
  ✓ Failure isolation
```

#### Pattern 2.2: Model Routing at Checkpoints
```
Problem:
  Single model (Opus) を全タスクに使用
  → Token waste on trivial tasks, slow turnaround

Solution:
  Checkpoint ごとに task complexity を再評価 → model を切り替え

Implementation:
  checkpoint("analyze_requirements"):
      current_task = "Read spec and extract requirements"
      complexity = estimate_complexity(current_task)
      
      if complexity == "trivial":
          switch_to(haiku)  # Fast, cheap
      elif complexity == "moderate":
          switch_to(sonnet)  # Balanced
      else:
          switch_to(opus)    # Expensive, powerful

Benefit:
  ✓ Token efficiency (40-60% reduction)
  ✓ Latency optimization
  ✓ Cost reduction
```

#### Pattern 2.3: Cascading Fallbacks
```
Problem:
  Agent は primary approach に固着
  → Unnecessary failures, token waste

Solution:
  複数の戦略を cascade的に試す

Implementation:
  strategies = [
      ("try_toolA_directly", confidence=0.9),
      ("try_toolB_workaround", confidence=0.7),
      ("ask_user", confidence=1.0)  # Last resort
  ]
  
  for strategy, confidence in strategies:
      if attempt(strategy):
          return success()
      else:
          log_failure(strategy)
          if should_continue():
              continue_to_next_strategy()
          else:
              escalate_to_user()

Benefit:
  ✓ Graceful degradation
  ✓ No dead-ends
  ✓ User escalation is explicit
```

---

### Tier 3: Tools & Permissions Patterns

#### Pattern 3.1: Principled Tool Inventory
```
Principle:
  8-15 tools: optimal sweet spot
  50+ tools: functional failure territory

Audit Process:
  quarterly_tool_audit():
      used_tools = analyze_session_logs()
      unused_tools = all_tools - used_tools
      
      for tool in unused_tools:
          if age_in_months > 3:
              mark_for_removal(tool)
          else:
              investigate_why_unused()
      
      if len(all_tools) > 15:
          consolidate_overlapping_tools()

Tool Priority Ranking:
    1. Read (必須)
    2. Edit (必須)
    3. Write (必須)
    4. Bash (必須)
    5. Agent (delegation)
    6. Monitor (long-running)
    7. WebSearch (info)
    8-15. Domain-specific tools

Benefit:
  ✓ Agent focus maintained
  ✓ Tool selection becomes trivial
  ✓ No attention fragmentation
```

#### Pattern 3.2: Permission Staging
```
Problem:
  All-or-nothing permissions
  → Either too permissive (security risk) or too restrictive (velocity loss)

Solution:
  Staged permission levels based on context

Implementation:
  permission_level = evaluate_context():
      if is_code_review():
          return READ_ONLY  # No modifications
      elif is_bugfix():
          return EDIT_EXISTING  # Modify, no delete
      elif is_feature():
          return WRITE_AND_EDIT  # Full access
      elif is_experiment():
          return SANDBOXED  # Isolated environment
  
  apply_permissions(permission_level)

Example Gates:
    - Read + Edit: Code review, bugfix
    - Read + Write: New features
    - Bash (git only): Version control
    - Bash (npm/build): Build automation
    - Bash + Monitor: Long-running operations

Benefit:
  ✓ Security by context
  ✓ Minimal permission prompts
  ✓ Audit trail of who did what when
```

#### Pattern 3.3: Tool Instrumentation
```
Implementation:
  class ToolWrapper:
      def __call__(self, *args, **kwargs):
          start = time.time()
          try:
              result = self.tool(*args, **kwargs)
              duration = time.time() - start
              
              log_metric({
                  "tool": self.name,
                  "status": "success",
                  "duration_ms": duration,
                  "input_size": len(str(args)),
                  "output_size": len(str(result))
              })
              return result
          except Exception as e:
              duration = time.time() - start
              log_metric({
                  "tool": self.name,
                  "status": "failure",
                  "error": str(e),
                  "duration_ms": duration
              })
              raise

Benefit:
  ✓ Precise tool performance tracking
  ✓ Failure pattern detection
  ✓ Data-driven optimization
```

---

### Tier 4: Automation & Feedback Patterns

#### Pattern 4.1: Hook-Based Quality Enforcement
```
Implementation:
  # Pre-commit
  hook("pre-commit"):
      run(mypy)        # Type check
      run(pylint)      # Lint
      run(black)       # Format
      run(bandit)      # Security
      
      if any_failed():
          abort_commit()
          report_failures()
      else:
          allow_commit()

  # Post-tool-use
  hook("post-tool-use"):
      if tool == "bash":
          verify_command_safety()
      elif tool == "edit":
          auto_format(edited_file)
          run_type_check(edited_file)
      else:
          noop()

  # Pre-push
  hook("pre-push"):
      run(test_suite)
      verify_build()
      check_security()
      
      if all_pass():
          allow_push()

Benefit:
  ✓ Automate quality gates
  ✓ Catch issues early
  ✓ Zero-friction enforcement
```

#### Pattern 4.2: Failure Analytics Loop
```
Implementation:
  on_agent_failure():
      failure = {
          "task": current_task,
          "step": current_step,
          "error": exception_message,
          "tool_used": last_tool,
          "context_usage": token_percentage,
          "timestamp": now()
      }
      
      save_to(failure_log, failure)
      categorize_failure()
      aggregate_patterns()
      
  monthly_review():
      patterns = analyze_failure_log()
      
      for pattern in patterns:
          if pattern.frequency > threshold:
              create_mitigation()
              update_CLAUDE_md()
              communicate_to_team()

Example Patterns:
  - "Tool X fails on Y with Z frequency"
      → Add fallback in Pattern 2.3
  
  - "Context fills at step N"
      → Adjust compression trigger
  
  - "Model routing picks wrong model"
      → Refine complexity estimation

Benefit:
  ✓ Data-driven configuration updates
  ✓ Continuous improvement
  ✓ Failure prevention
```

#### Pattern 4.3: Success Metrics Tracking
```
Implementation:
  # Per-session metrics
  metrics = {
      "total_turns": count_turns(),
      "tool_success_rate": successes / attempts,
      "token_efficiency": used_tokens / available_tokens,
      "latency_p95": percentile_95(response_times),
      "cost": sum_of_api_costs(),
      "task_completion": success_indicator()
  }
  
  # Aggregate
  aggregate_to_monthly_report()
  
  # Trends
  if token_efficiency < 70%:
      alert("Context compression needed")
  if tool_success_rate < 95%:
      alert("Tool reliability issue")
  if latency_p95 > 10s:
      alert("Performance degradation")

SLO Targets:
    - Tool success rate: > 95%
    - Token efficiency: < 70% usage
    - Latency p95: < 10s
    - Task success: > 90% first attempt

Benefit:
  ✓ Objective quality measurement
  ✓ Data-driven decisions
  ✓ Accountability
```

---

## Implementation Checklist

- [ ] Context layering を CLAUDE.md に実装
- [ ] Model routing logic を構築
- [ ] Tool inventory を監査（8-15個に調整）
- [ ] Pre-commit hooks を設定
- [ ] Failure logging を有効化
- [ ] Metrics dashboard を構築
- [ ] Monthly review プロセスを確立
- [ ] Team training を実施

---

**Version**: 1.0  
**Source**: 2026年4月最新ベストプラクティス  
**Status**: Production Ready
