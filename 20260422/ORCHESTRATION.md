# ORCHESTRATION.md - マルチエージェント・オーケストレーション戦略

このファイルはエージェント間のオーケストレーション実装を詳細に定義します。

## 中核概念: Plan-Do-Evaluate ループ

2026年のベストプラクティスでは、全マルチエージェントシステムが **Plan-Do-Evaluate (PDE)** サイクルに基づいて動作します。

```
┌──────────────────────────────────────────────────────────┐
│                  User Request                            │
└────────────────────────┬─────────────────────────────────┘
                         │
                    ┌────▼────┐
                    │  PLAN   │ (Orchestrator, Reasoning)
                    └────┬────┘
                         │
           ┌─────────────┼─────────────┐
           │             │             │
      ┌────▼──┐      ┌────▼───┐   ┌────▼──┐
      │ DO    │      │ DO     │   │ DO    │ (並列実行)
      │Action │      │Retrieval   │        │
      │Agent  │      │Agent   │   │  ...  │
      └────┬──┘      └────┬───┘   └────┬──┘
           │             │             │
           └─────────────┼─────────────┘
                         │
                    ┌────▼─────────┐
                    │  EVALUATE    │ (Validator)
                    │ & Synthesize │
                    └────┬─────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
          ┌───▼──┐            ┌────▼──────┐
          │ OK   │            │ NOT OK    │
          │Final │            │Re-PLAN or │
          │Result│            │Escalate   │
          └──────┘            └───────────┘
```

---

## ステップ1: PLAN フェーズ

### 1.1 タスク解析 (Orchestrator)

**入力**: ユーザーリクエスト（自然言語）  
**出力**: 構造化タスク計画（JSON）  
**タイムアウト**: 30秒  

```json
{
  "plan_id": "plan_uuid_20260422_001",
  "user_request": "新しいコード機能をレビューして、セキュリティチェックして、テストして。",
  "parsed_tasks": [
    {
      "task_id": "t1",
      "description": "コード品質・複雑度分析",
      "assigned_agent": "reasoning_agent",
      "dependencies": [],
      "priority": "high",
      "estimated_duration_seconds": 45
    },
    {
      "task_id": "t2",
      "description": "セキュリティ脆弱性スキャン",
      "assigned_agent": "validator_agent",
      "dependencies": ["t1"],
      "priority": "critical",
      "estimated_duration_seconds": 60
    },
    {
      "task_id": "t3",
      "description": "テスト実行 & カバレッジ確認",
      "assigned_agent": "action_agent",
      "dependencies": ["t1"],
      "priority": "high",
      "estimated_duration_seconds": 120
    }
  ],
  "orchestration_pattern": "sequential_with_parallel_branches",
  "total_estimated_duration_seconds": 180,
  "escalation_trigger": {
    "condition": "any_critical_severity",
    "action": "immediate_human_review"
  },
  "success_criteria": [
    "code_quality_score >= 80",
    "security_vulnerabilities == 0",
    "test_coverage >= 85%"
  ]
}
```

### 1.2 エージェント選択と権限確認 (Orchestrator)

```yaml
agent_selection:
  process:
    - step: 1
      action: "match_task_to_agents"
      logic: "タスク内容 → エージェント特性マッピング"
    
    - step: 2
      action: "verify_availability"
      check: "各エージェントのキャパシティ確認"
    
    - step: 3
      action: "check_permissions"
      check: "エージェントが必要なリソースにアクセス可能か"
    
    - step: 4
      action: "estimate_cost"
      output: "予想API呼び出し回数, トークン数, 金額"

  capability_matrix:
    reasoning_agent:
      skills: ["code_analysis", "logic_chain", "risk_assessment"]
      max_concurrent_tasks: 2
      current_queue: 1
      availability: "available"
    
    validator_agent:
      skills: ["security_scan", "compliance_check"]
      max_concurrent_tasks: 2
      current_queue: 0
      availability: "available"
    
    action_agent:
      skills: ["code_execute", "test_runner"]
      max_concurrent_tasks: 3
      current_queue: 2
      availability: "available"
```

### 1.3 計画の検証 (Orchestrator)

計画実行前に、矛盾や不可能な依存関係がないか確認。

```yaml
plan_validation:
  checks:
    - name: "dependency_cycle_detection"
      rule: "循環依存がないか"
      example_violation: "t1 depends on t2, t2 depends on t1"
    
    - name: "agent_capability_match"
      rule: "全タスクが割り当てられたエージェントで実行可能か"
      example_violation: "action_agent に 'security_scan' を割り当てた（できない）"
    
    - name: "resource_availability"
      rule: "必要なリソース（API, DB等）が利用可能か"
    
    - name: "sla_feasibility"
      rule: "計画実行時間が SLA を満たすか"
      target_sla_seconds: 300
    
    - name: "cost_feasibility"
      rule: "予想コストが予算内か"
      monthly_budget_usd: 10000
      current_spend_usd: 4523
      remaining_budget_usd: 5477
      estimated_cost_usd: 1.23
      verdict: "APPROVE"

validation_result:
  status: "APPROVED"
  warnings: []
  execution_go_time: "2026-04-22T10:15:32Z"
```

---

## ステップ2: DO フェーズ

### 2.1 タスク実行（並列実行）

PLAN で決定された依存関係に基づき、**並列化できるタスクは並列に実行**。

```yaml
execution_flow:
  executor: "task_dispatcher"
  
  parallel_groups:
    
    - group_id: "phase1"
      start_condition: "plan_approved"
      tasks:
        - task_id: "t1"  # reasoning_agent: コード分析
          agent: "reasoning_agent"
          concurrency: 1
          timeout_seconds: 45
    
    - group_id: "phase2"
      start_condition: "phase1_completed"
      tasks:
        - task_id: "t2"  # validator_agent: セキュリティスキャン（並列実行）
          agent: "validator_agent"
          concurrency: 1
          timeout_seconds: 60
        
        - task_id: "t3"  # action_agent: テスト実行（並列実行）
          agent: "action_agent"
          concurrency: 1
          timeout_seconds: 120

execution_monitoring:
  heartbeat_interval_seconds: 10
  
  triggers:
    - event: "task_timeout"
      action: "cancel_task"
      retry: false
      escalation: "validator_agent (failure review)"
    
    - event: "agent_unavailable"
      action: "queue_task"
      retry: true
      backoff_seconds: [5, 10, 20]
    
    - event: "unexpected_error"
      action: "log_and_escalate"
      escalation: "human_review"
```

### 2.2 実行ログとトレーシング

```json
{
  "execution_id": "exec_uuid_20260422_001",
  "plan_id": "plan_uuid_20260422_001",
  "start_time": "2026-04-22T10:15:35Z",
  "tasks_executed": [
    {
      "task_id": "t1",
      "agent": "reasoning_agent",
      "status": "completed",
      "start_time": "2026-04-22T10:15:35Z",
      "end_time": "2026-04-22T10:16:10Z",
      "duration_seconds": 35,
      "result": {
        "code_quality_score": 82,
        "complexity_assessment": "moderate",
        "findings": ["could_refactor_module_x", "long_function_y"]
      },
      "token_usage": {
        "input_tokens": 4200,
        "output_tokens": 1100,
        "total": 5300,
        "cost_usd": 0.21
      }
    },
    {
      "task_id": "t2",
      "agent": "validator_agent",
      "status": "completed",
      "start_time": "2026-04-22T10:16:10Z",
      "end_time": "2026-04-22T10:17:15Z",
      "duration_seconds": 65,
      "result": {
        "vulnerabilities_found": 0,
        "compliance_status": "passed",
        "risk_level": "low"
      },
      "token_usage": {
        "input_tokens": 6100,
        "output_tokens": 850,
        "total": 6950,
        "cost_usd": 0.28
      }
    },
    {
      "task_id": "t3",
      "agent": "action_agent",
      "status": "completed",
      "start_time": "2026-04-22T10:16:10Z",
      "end_time": "2026-04-22T10:18:45Z",
      "duration_seconds": 155,
      "result": {
        "tests_passed": 145,
        "tests_failed": 2,
        "coverage_percent": 87,
        "failed_test_details": [
          {
            "test_name": "edge_case_handler",
            "error": "timeout_500ms"
          }
        ]
      },
      "token_usage": {
        "input_tokens": 8500,
        "output_tokens": 2200,
        "total": 10700,
        "cost_usd": 0.43
      }
    }
  ],
  "execution_end_time": "2026-04-22T10:18:45Z",
  "total_duration_seconds": 190,
  "total_cost_usd": 0.92,
  "execution_status": "completed_with_warnings"
}
```

---

## ステップ3: EVALUATE フェーズ

### 3.1 結果評価 (Validator Agent)

実行結果を成功基準に照らして評価。

```yaml
evaluation:
  evaluator: "validator_agent"
  
  success_criteria_check:
    - criterion: "code_quality_score >= 80"
      actual: 82
      status: "PASS"
    
    - criterion: "security_vulnerabilities == 0"
      actual: 0
      status: "PASS"
    
    - criterion: "test_coverage >= 85%"
      actual: 87
      status: "PASS"
  
  warnings:
    - severity: "medium"
      issue: "2 tests failed (edge_case_handler timeout)"
      recommendation: "investigate_timeout_cause"
    
    - severity: "low"
      issue: "Code refactoring opportunities in module_x"
      recommendation: "follow_up_task"

  overall_evaluation:
    status: "APPROVED_WITH_RECOMMENDATIONS"
    approval_decision: "GO_TO_PRODUCTION"
    reasoning: "All critical criteria met. Warnings are non-blocking."
```

### 3.2 決定ツリー

```yaml
decision_tree:
  
  if: "all_success_criteria_met AND no_critical_issues"
    then: "APPROVED"
    action: "finalize_and_deliver_result"
  
  elif: "some_success_criteria_met AND no_critical_issues"
    then: "APPROVED_WITH_WARNINGS"
    action: "deliver_with_warnings"
    escalation: "inform_user_of_issues"
  
  elif: "critical_issue_found"
    then: "REJECTED"
    action: "escalate_to_human"
    reason: "manual_review_required"
  
  elif: "inconsistent_results_from_agents"
    then: "CONFLICT_DETECTED"
    action: "trigger_re_analysis"
    reassign: "reasoning_agent"
    max_retries: 2
```

### 3.3 結果の統合と報告 (Synthesizer)

複数エージェントの出力を 1つの最終レポートに統合。

```json
{
  "final_report": {
    "report_id": "report_uuid_20260422_001",
    "timestamp": "2026-04-22T10:20:00Z",
    "user_request": "新しいコード機能をレビューして、セキュリティチェックして、テストして。",
    
    "executive_summary": {
      "overall_status": "APPROVED_FOR_PRODUCTION",
      "risk_level": "LOW",
      "confidence_score": 0.95
    },
    
    "detailed_findings": {
      "code_review": {
        "source": "reasoning_agent",
        "quality_score": 82,
        "key_findings": [
          "Logic is sound and well-structured",
          "Could benefit from refactoring in module_x",
          "Performance is acceptable"
        ]
      },
      
      "security_review": {
        "source": "validator_agent",
        "vulnerabilities": 0,
        "compliance_status": "PASS",
        "risk_assessment": "No security risks identified"
      },
      
      "testing_report": {
        "source": "action_agent",
        "tests_passed": 145,
        "tests_failed": 2,
        "coverage": "87%",
        "action_items": [
          {
            "priority": "medium",
            "task": "Investigate edge_case_handler timeout",
            "estimated_effort_hours": 2
          }
        ]
      }
    },
    
    "recommendations": [
      {
        "priority": "high",
        "action": "Deploy to production",
        "reason": "All critical criteria met"
      },
      {
        "priority": "medium",
        "action": "Schedule follow-up optimization for module_x refactoring",
        "effort_hours": 4
      },
      {
        "priority": "medium",
        "action": "Fix edge_case_handler timeout before next release",
        "effort_hours": 2
      }
    ],
    
    "cost_summary": {
      "total_cost_usd": 0.92,
      "breakdown": {
        "reasoning_agent": 0.21,
        "validator_agent": 0.28,
        "action_agent": 0.43
      }
    },
    
    "next_steps": "Deploy to production with follow-up optimization tasks",
    "approval_signature": "validator_agent",
    "approval_timestamp": "2026-04-22T10:20:00Z"
  }
}
```

---

## Adaptive Orchestration（動的なオーケストレーション）

2026年の革新：固定パターンではなく、**タスク特性に応じて動的にオーケストレーション戦略を選択**。

### 4.1 タスク特性分析

```yaml
task_characteristic_analysis:
  
  dimensions:
    - name: "parallelizability"
      values: [low, medium, high]
      calculation: "何タスクが並列実行可能か"
    
    - name: "decision_complexity"
      values: [simple, moderate, complex]
      calculation: "論理判断の複雑さ"
    
    - name: "urgency"
      values: [normal, high, critical]
      calculation: "実行時間の制約"
    
    - name: "resource_dependency"
      values: [minimal, moderate, heavy]
      calculation: "外部リソース依存度"

example_analysis:
  task: "Deploy code to production"
  characteristics:
    parallelizability: "medium"    # テスト並列実行可, デプロイは直列
    decision_complexity: "moderate"  # デプロイ前確認が必要
    urgency: "high"                  # 営業時間内での対応必須
    resource_dependency: "heavy"     # GitHub, 本番環境等
```

### 4.2 オーケストレーション戦略の自動選択

```yaml
adaptive_routing:
  
  rules:
    
    - condition: "parallelizability=high AND urgency=critical"
      selected_pattern: "parallel_fan_out_gather"
      rationale: "複数タスク並列実行で最短時間実現"
    
    - condition: "decision_complexity=complex AND resource_dependency=heavy"
      selected_pattern: "hierarchical_supervisor"
      rationale: "オーケストレータが中央統制で安全性確保"
    
    - condition: "urgency=normal AND decision_complexity=simple"
      selected_pattern: "sequential"
      rationale: "シンプルで監視・ロールバックが容易"
    
    - condition: "resource_dependency=heavy AND availability=real_time"
      selected_pattern: "event_driven"
      rationale: "イベント駆動で即時反応"

strategy_confidence_scoring:
  scoring_factors:
    - factor: "past_success_rate"
      weight: 0.4
    
    - factor: "estimated_latency_fit"
      weight: 0.3
    
    - factor: "cost_optimization"
      weight: 0.2
    
    - factor: "risk_profile_match"
      weight: 0.1
```

---

## エスカレーションと Human-on-the-Loop

### 5.1 自動エスカレーション条件

```yaml
escalation_triggers:
  
  - trigger_id: "t001"
    condition: "critical_severity_issue"
    examples: ["security_vulnerability", "data_loss_risk"]
    escalation_path: "immediate_human_review"
    sla_minutes: 15
  
  - trigger_id: "t002"
    condition: "agent_disagreement"
    examples: ["validator rejects reasoning output"]
    escalation_path: "human_decision_required"
    sla_minutes: 30
  
  - trigger_id: "t003"
    condition: "cost_threshold_exceeded"
    cost_limit_usd: 100
    escalation_path: "budget_approval"
    sla_minutes: 60
  
  - trigger_id: "t004"
    condition: "repeated_failures"
    failure_threshold: 3
    escalation_path: "technical_investigation"
    sla_minutes: 120
  
  - trigger_id: "t005"
    condition: "sla_breach_imminent"
    threshold_percent: 80
    escalation_path: "priority_review"
    sla_minutes: 10
```

### 5.2 Human Review Interface

```json
{
  "escalation_request": {
    "escalation_id": "esc_uuid_20260422_001",
    "timestamp": "2026-04-22T10:25:00Z",
    "trigger": "critical_severity_issue",
    "severity": "critical",
    "sla_deadline": "2026-04-22T10:40:00Z",
    
    "context": {
      "original_task": "Deploy new authentication module to production",
      "current_status": "validator_agent detected potential security vulnerability",
      "detailed_issue": "SQL injection risk in user_input_validator function"
    },
    
    "agent_recommendations": [
      {
        "agent": "validator_agent",
        "recommendation": "DO_NOT_DEPLOY",
        "reasoning": "Security vulnerability must be fixed first",
        "confidence": 0.98
      },
      {
        "agent": "reasoning_agent",
        "recommendation": "CONDITIONAL_DEPLOY",
        "reasoning": "Risk can be mitigated with input sanitization",
        "confidence": 0.75
      }
    ],
    
    "human_action_required": [
      {
        "action_id": "1",
        "type": "decision",
        "question": "Deploy or roll back?",
        "options": ["APPROVE_DEPLOY", "REJECT_DEPLOY", "CONDITIONAL_WITH_MITIGATION"]
      },
      {
        "action_id": "2",
        "type": "instruction",
        "question": "If CONDITIONAL_WITH_MITIGATION, specify required fixes:",
        "input_type": "text"
      }
    ]
  }
}
```

---

## パフォーマンス最適化

### 6.1 キャッシングとメモ化

```yaml
optimization:
  
  caching:
    query_result_cache:
      ttl_hours: 24
      max_size_mb: 1000
      key: "hash(agent_id + query)"
    
    mcp_response_cache:
      ttl_hours: 1
      external_api: "github_api"
      rate_limit_avoidance: true
  
  memoization:
    frequent_analyses:
      - analysis: "code_quality_check"
        cache_key: "repo_sha"
        ttl_hours: 24
      
      - analysis: "dependency_scan"
        cache_key: "package_json_hash"
        ttl_hours: 12

  parallelization:
    max_concurrent_agents: 10
    task_batching: true
    batch_size: 5
```

### 6.2 コスト最適化

```yaml
cost_optimization:
  
  model_selection:
    - task_type: "simple_analysis"
      model: "claude-haiku-4-5"
      tokens_saved_percent: 60
    
    - task_type: "complex_reasoning"
      model: "claude-opus-4-7"
      tokens_saved_percent: 0
  
  token_efficiency:
    technique: "prompt_compression"
    compression_ratio: 0.7
    monthly_savings_usd: 1200
  
  batch_processing:
    enabled: true
    batch_size: 50
    discount_percent: 10
```

---

**最終更新**: 2026-04-22  
**参考**: DESIGN.md のパターン定義と組み合わせて使用
