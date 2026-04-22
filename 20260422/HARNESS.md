# HARNESS.md - ハーネスエンジニアリング設定

このファイルはAIエージェントの「ハーネス」（制約・ガードレール・フィードバックループ）の具体的な実装を定義します。

**ハーネスエンジニアリングの定義**:  
エージェントを本番環境で信頼できるものにするために、その周囲に構築されるシステム、制約、フィードバックループの学問。インフラ設定だけで性能が5%以上向上する可能性がある。

---

## 1. ガードレール設定

### 1.1 権限（Capability）制御

#### エージェント別の権限マトリックス

```yaml
agent_permissions:
  
  orchestrator_agent:
    tools:
      read:
        - "list_all_agents"
        - "check_agent_status"
        - "query_knowledge_base"
      write: []
      dangerous: []
    data_access:
      level: "metadata_only"
      sensitive_data: "blocked"
      pii: "blocked"
    rate_limits:
      api_calls_rpm: 100
      tokens_per_hour: 100000
    resource_limits:
      max_concurrent_tasks: 5
      max_task_duration_seconds: 300
      max_memory_mb: 512

  reasoning_agent:
    tools:
      read:
        - "analyze_code"
        - "search_documentation"
        - "query_logs"
      write:
        - "create_analysis_report"
      dangerous: []
    data_access:
      level: "operational_data"
      sensitive_data: "read_only"
      pii: "blocked"
    rate_limits:
      api_calls_rpm: 80
      tokens_per_hour: 150000
    resource_limits:
      max_concurrent_tasks: 3
      max_task_duration_seconds: 600
      max_memory_mb: 1024

  action_agent:
    tools:
      read:
        - "list_files"
        - "read_files"
      write:
        - "create_files"
        - "modify_files"
        - "execute_commands"
      dangerous:
        - "delete_files"
        - "kill_process"
    data_access:
      level: "full_operational"
      sensitive_data: "restricted"  # 最小権限のみ
      pii: "restricted"
    rate_limits:
      api_calls_rpm: 60
      tokens_per_hour: 120000
    resource_limits:
      max_concurrent_tasks: 3
      max_task_duration_seconds: 900
      max_memory_mb: 2048

  validator_agent:
    tools:
      read:
        - "read_all_data"
        - "audit_logs"
        - "inspect_commands"
      write:
        - "create_audit_entries"
      dangerous: []
    data_access:
      level: "full_read_all"
      sensitive_data: "read_only_for_audit"
      pii: "read_only_for_audit"
    rate_limits:
      api_calls_rpm: 150
      tokens_per_hour: 100000
    resource_limits:
      max_concurrent_tasks: 5
      max_task_duration_seconds: 180
      max_memory_mb: 1024

  retrieval_agent:
    tools:
      read:
        - "query_database"
        - "call_api"
        - "search_cache"
      write: []
      dangerous: []
    data_access:
      level: "database_read_only"
      sensitive_data: "read_only"
      pii: "blocked"
    rate_limits:
      api_calls_rpm: 200
      tokens_per_hour: 80000
    resource_limits:
      max_concurrent_tasks: 8
      max_task_duration_seconds: 120
      max_memory_mb: 512
```

#### Dangerous Operation の事前検認

危険な操作（削除、変更、実行）は必ず Validator が事前承認。

```yaml
dangerous_operations:
  
  operations:
    - operation_id: "delete_file"
      severity: "critical"
      pre_approval_required: true
      approver: "validator_agent"
      sla_minutes: 1
      example: "rm -rf /path"
    
    - operation_id: "execute_command"
      severity: "critical"
      pre_approval_required: true
      approver: "validator_agent"
      sla_minutes: 1
      blocked_commands: ["dd", "shred", "format", "fdisk"]
    
    - operation_id: "modify_database"
      severity: "critical"
      pre_approval_required: true
      approver: "validator_agent"
      sla_minutes: 2
      blocked_sql: ["DROP", "DELETE", "TRUNCATE"]
    
    - operation_id: "push_to_production"
      severity: "critical"
      pre_approval_required: true
      approver: "human_reviewer"
      sla_minutes: 15
    
    - operation_id: "create_credentials"
      severity: "critical"
      pre_approval_required: true
      approver: "validator_agent"
      sla_minutes: 5

approval_workflow:
  step1: "action_agent requests approval"
  step2: "validator_agent reviews"
  step3: "validator returns GO/NO-GO"
  step4: "if GO: action_agent executes"
  step5: "audit_log records execution"
  timeout_escalation: "human_review"
```

### 1.2 出力フィルタリング

エージェント出力に含まれる秘密情報（パスワード、API key等）を自動検出・マスク。

```yaml
output_filtering:
  
  sensitive_patterns:
    - pattern: "password|passwd|pwd"
      regex: "['\"]([a-zA-Z0-9!@#$%^&*]{8,})['\"]"
      action: "mask_and_alert"
      mask_char: "***"
    
    - pattern: "api_key|api_secret|apikey"
      regex: "['\"]([a-zA-Z0-9_-]{32,})['\"]"
      action: "mask_and_alert"
    
    - pattern: "credit_card"
      regex: "[0-9]{4}[\\s-]?[0-9]{4}[\\s-]?[0-9]{4}[\\s-]?[0-9]{4}"
      action: "mask_and_escalate"
    
    - pattern: "email|social_security"
      regex: "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"
      action: "mask_if_context=sensitive"
    
    - pattern: "private_key"
      regex: "-----BEGIN.*PRIVATE KEY-----"
      action: "block_and_escalate"
  
  filtering_rules:
    default_action: "allow"
    detected_sensitive_data: "mask_and_log"
    blocked_patterns: "escalate_to_human"
    audit_log_all_detections: true
```

### 1.3 コマンド実行サンドボックス

```yaml
execution_sandbox:
  
  environment:
    os: "linux"
    container: "docker"
    image: "agent-execution-ubuntu-22.04"
    network_isolation: "enabled"
    
  resource_limits:
    cpu_cores: 4
    memory_gb: 8
    disk_gb: 50
    network_bandwidth_mbps: 100
  
  file_system:
    root_allowed: false
    allowed_directories:
      - "/home/agent"
      - "/tmp/agent_work"
      - "/opt/workspace"
    blocked_directories:
      - "/"
      - "/etc"
      - "/sys"
      - "/proc"
      - "/root"
    file_size_limit_mb: 1000
  
  process_control:
    max_processes: 50
    max_threads: 200
    blocked_syscalls:
      - "reboot"
      - "shutdown"
      - "mount"
  
  timeout:
    default_seconds: 120
    hard_limit_seconds: 300
    kill_on_timeout: true
```

---

## 2. フィードバックループ

### 2.1 エージェント学習フィードバック

エージェントの推奨や実行結果について、人間や他エージェントからのフィードバックを収集。

```yaml
feedback_collection:
  
  feedback_sources:
    - source: "human_user"
      trigger: "after_task_completion"
      question: "エージェントの対応は満足できるものでしたか？"
      options: ["very_satisfied", "satisfied", "neutral", "unsatisfied", "very_unsatisfied"]
      required: false
    
    - source: "validator_agent"
      trigger: "on_approval_decision"
      question: "この推奨は妥当でしたか？"
      tracking: "auto"
      impact: "high"
    
    - source: "outcome_monitoring"
      trigger: "7_days_after_task_completion"
      question: "実行された操作の結果は期待通りでしたか？"
      metric: "success_rate"

  feedback_schema:
    feedback_id: "uuid"
    timestamp: "ISO-8601"
    agent_id: "recommending_agent_id"
    task_id: "related_task_id"
    rating: "1-5_stars"
    comment: "text"
    outcome: "success|partial|failure"
    impact_assessment: "positive|neutral|negative"
```

### 2.2 パフォーマンス監視フィードバック

```yaml
performance_monitoring:
  
  metrics:
    
    - metric_id: "latency_p95"
      unit: "seconds"
      target: 60
      collection_frequency: "every_task"
      alert_threshold: 120
      trend_analysis: "weekly"
    
    - metric_id: "success_rate"
      unit: "%"
      target: 99.5
      collection_frequency: "hourly_aggregation"
      alert_threshold: 95
      trend_analysis: "daily"
    
    - metric_id: "cost_per_task"
      unit: "USD"
      target: 1.0
      collection_frequency: "every_task"
      alert_threshold: 5.0
      trend_analysis: "daily"
    
    - metric_id: "human_escalation_rate"
      unit: "%"
      target: 5
      collection_frequency: "daily"
      alert_threshold: 15
      trend_analysis: "weekly"
  
  alerting:
    - alert_id: "a001"
      condition: "latency_p95 > 120"
      severity: "warning"
      action: "investigate_and_optimize"
      escalation: "if_persistent_for_3_hours"
    
    - alert_id: "a002"
      condition: "success_rate < 95"
      severity: "critical"
      action: "immediate_investigation"
      escalation: "human_review"
```

### 2.3 継続的改善ループ

```yaml
continuous_improvement:
  
  feedback_to_improvement_process:
    
    step1:
      name: "feedback_aggregation"
      frequency: "daily"
      method: "collect_all_feedback_from_past_24h"
      output: "feedback_report"
    
    step2:
      name: "root_cause_analysis"
      frequency: "weekly"
      method: "pattern_detection_in_negative_feedback"
      output: "problem_categories"
    
    step3:
      name: "improvement_brainstorm"
      frequency: "weekly"
      method: "reasoning_agent analyzes problems"
      output: "improvement_proposals"
    
    step4:
      name: "implementation"
      frequency: "as_decided"
      method: "prompt_refinement OR skill_update OR guardrail_adjustment"
      output: "updated_agent_config"
    
    step5:
      name: "validation"
      frequency: "before_deployment"
      method: "A/B test in staging"
      output: "validation_report"
    
    step6:
      name: "deployment"
      frequency: "if_improvement_confirmed"
      method: "gradual_rollout_to_production"
      output: "deployment_log"

  a_b_testing:
    enabled: true
    staging_environment: "enabled"
    user_split: "5% experimental vs 95% control"
    duration_hours: 48
    success_metric: "improvement > 5% AND no_regression"
```

---

## 3. Observability（監視とロギング）

### 3.1 構造化ログ形式

```json
{
  "log_entry": {
    "timestamp": "2026-04-22T10:25:34.123Z",
    "log_id": "log_uuid",
    "level": "INFO|WARNING|ERROR|CRITICAL",
    
    "agent_context": {
      "agent_id": "reasoning_agent_v1",
      "agent_version": "1.0.2",
      "model": "claude-opus-4-7"
    },
    
    "task_context": {
      "task_id": "task_uuid",
      "execution_id": "exec_uuid",
      "user_id": "user_uuid"
    },
    
    "operation": {
      "operation_type": "task_execution|tool_call|decision_point|escalation",
      "operation_id": "op_uuid",
      "operation_name": "human_readable_name"
    },
    
    "metrics": {
      "latency_ms": 1234,
      "tokens_input": 4200,
      "tokens_output": 1100,
      "cost_usd": 0.21
    },
    
    "result": {
      "status": "success|warning|error",
      "message": "human_readable_message",
      "error_code": "if_error",
      "error_details": "if_error"
    },
    
    "security": {
      "sensitive_data_detected": false,
      "approval_required": false,
      "approval_decision": "if_applicable"
    },
    
    "correlation_id": "for_tracing_entire_request_flow"
  }
}
```

### 3.2 ダッシュボード&アラート定義

```yaml
observability_setup:
  
  log_storage:
    backend: "CloudWatch OR Datadog OR ELK"
    retention_days: 90
    encryption: "at_rest_and_in_transit"
  
  dashboards:
    
    - dashboard_name: "Agent_Health_Overview"
      refresh_interval_seconds: 60
      widgets:
        - widget: "Active_Tasks_Count"
          metric: "count(active_tasks)"
          threshold_warning: 100
        
        - widget: "Latency_P95_by_Agent"
          metric: "p95(latency_ms)"
          threshold_warning: 120000
        
        - widget: "Success_Rate"
          metric: "success_count / total_count"
          threshold_critical: 95
        
        - widget: "Cost_Tracking"
          metric: "sum(cost_usd) group by agent"
          trend: "daily_comparison"
    
    - dashboard_name: "Security_and_Compliance"
      refresh_interval_seconds: 300
      widgets:
        - widget: "Sensitive_Data_Detection"
          metric: "count(sensitive_data_detected)"
          threshold_critical: 0
        
        - widget: "Approval_Workflow_SLA"
          metric: "avg(approval_time_seconds)"
          threshold_warning: 60
        
        - widget: "Escalation_Events"
          metric: "count(escalation_type) group by type"
        
        - widget: "Access_Violation_Attempts"
          metric: "count(permission_denied)"
          threshold_critical: 10
    
    - dashboard_name: "Cost_Analytics"
      refresh_interval_seconds: 3600
      widgets:
        - widget: "Daily_Spend"
          metric: "sum(cost_usd) by date"
          budget_limit: 1000
        
        - widget: "Cost_per_Task"
          metric: "avg(cost_usd) by agent"
        
        - widget: "Token_Efficiency"
          metric: "output_tokens / input_tokens"
          benchmark: "historical_average"

  alerting:
    
    - alert_rule: "latency_spike"
      condition: "p95(latency_ms) > 150000 for 10 minutes"
      severity: "warning"
      action: "notify_on_call_engineer"
      escalation: "if_continues_30_min"
    
    - alert_rule: "cost_overage"
      condition: "daily_spend > budget * 1.2"
      severity: "critical"
      action: "notify_finance_AND_engineering"
      action_plan: "review_cost_drivers"
    
    - alert_rule: "high_escalation_rate"
      condition: "escalation_count / total_tasks > 20%"
      severity: "warning"
      action: "trigger_improvement_analysis"
    
    - alert_rule: "security_detection"
      condition: "sensitive_data_detected OR access_denied_count > 50"
      severity: "critical"
      action: "immediate_investigation"
      escalation: "security_team"
```

### 3.3 分布トレーシング

```yaml
distributed_tracing:
  enabled: true
  sampling_rate: 0.1  # 10% of requests
  
  trace_structure:
    root_span: "user_request"
    child_spans:
      - "orchestrator_plan"
      - "task_execution_group_1"
        - "reasoning_task"
        - "retrieval_task"
      - "task_execution_group_2"
        - "action_task"
      - "validator_task"
      - "result_synthesis"
  
  trace_storage:
    backend: "Jaeger OR Datadog APM"
    retention_days: 30
  
  analysis:
    latency_breakdown: "per_span"
    critical_path_analysis: "identify_bottlenecks"
    flame_graph: "visualize_call_hierarchy"
```

---

## 4. セキュリティ設定

### 4.1 認証・認可

```yaml
security_authentication:
  
  agent_authentication:
    method: "mTLS + Service Account"
    certificate_rotation: "every_90_days"
    key_storage: "AWS KMS"
  
  user_authentication:
    method: "OAuth2 + MFA"
    session_timeout_minutes: 60
    ip_whitelist: "enabled"
    device_fingerprint: "enabled"
  
  mcp_server_authentication:
    method: "OAuth2 with scope-based permissions"
    token_ttl_hours: 24
    token_refresh: "automatic"
    revocation_check_interval_minutes: 5

authorization:
  method: "Role-Based Access Control (RBAC)"
  roles:
    - role: "agent_operator"
      permissions: ["execute_tasks", "view_logs"]
    
    - role: "security_auditor"
      permissions: ["audit_all_operations", "view_sensitive_logs"]
    
    - role: "system_admin"
      permissions: ["configure_agents", "manage_mcp_servers", "incident_response"]
  
  policy_enforcement:
    engine: "OPA (Open Policy Agent)"
    policy_language: "Rego"
    audit_policy_changes: "all_changes_logged"
```

### 4.2 暗号化

```yaml
encryption:
  
  data_in_transit:
    protocol: "TLS 1.3"
    certificate_pinning: "enabled"
    perfect_forward_secrecy: "enabled"
  
  data_at_rest:
    algorithm: "AES-256-GCM"
    key_management: "AWS KMS OR HashiCorp Vault"
    rotation_interval_days: 90
    
    encryption_targets:
      - logs: "all_sensitive_logs"
      - cache: "all_cache_entries"
      - database: "all_user_data"
  
  secret_management:
    system: "AWS Secrets Manager OR HashiCorp Vault"
    rotation_interval_days: 30
    audit: "all_accesses_logged"
```

### 4.3 監査ログ（Immutable）

```yaml
audit_logging:
  
  immutability:
    storage: "Write-Once-Read-Many (WORM) storage"
    blockchain_backup: "optional_for_ultra_critical"
  
  log_content:
    - who: "agent_id, user_id"
    - what: "operation_description"
    - when: "timestamp_with_timezone"
    - where: "source_ip, resource_id"
    - why: "authorization_reason"
    - result: "success|failure"
  
  retention_policy:
    standard_retention_years: 1
    critical_operations_years: 3
    regulatory_requirement_years: 7
  
  review_frequency:
    automated_review: "daily"
    manual_review: "weekly_for_critical"
    quarterly_audit: "by_external_auditor"
```

---

## 5. 本番環境チェックリスト

エージェント設定を本番環境に昇格させる前に、必ず確認。

```yaml
production_readiness_checklist:
  
  functionality:
    - [ ] 全スキルが期待通り動作（ステージング環境で48時間テスト）
    - [ ] エラーハンドリング動作確認
    - [ ] エッジケース処理確認
  
  performance:
    - [ ] P95 レイテンシが SLA 内（目標: < 60秒）
    - [ ] スループット要件を満たす（目標: >= 10 tasks/分）
    - [ ] コスト予測が予算内
  
  security:
    - [ ] セキュリティレビュー合格
    - [ ] 秘密情報リーク検査合格
    - [ ] 権限スコープが最小権限原則準拠
    - [ ] MCP サーバー認証が確保される
  
  reliability:
    - [ ] フェイルオーバーテスト合格
    - [ ] リカバリー手順がドキュメント化
    - [ ] ロールバック計画がある
  
  monitoring:
    - [ ] ダッシュボード構築完了
    - [ ] アラート設定完了
    - [ ] ログ記録が正常に動作
  
  documentation:
    - [ ] 運用マニュアル作成完了
    - [ ] トラブルシューティングガイド作成
    - [ ] エスカレーション手順が明記
  
  compliance:
    - [ ] 監査ログ機能検証
    - [ ] データ保護規制準拠確認（GDPR等）
    - [ ] インシデント対応計画がある
  
  approval:
    - [ ] セキュリティチーム承認
    - [ ] インフラストラクチャチーム承認
    - [ ] ビジネスオーナー承認
```

---

## 6. インシデント対応

### 6.1 Incident Detection & Response

```yaml
incident_response:
  
  severity_levels:
    
    - level: "critical"
      examples: ["security_breach", "data_loss", "service_unavailable"]
      sla_minutes: 15
      escalation: "immediate_ceo_notification"
      response_lead: "ciso"
    
    - level: "high"
      examples: ["significant_performance_degradation", "escalation_rate_spike"]
      sla_minutes: 60
      escalation: "engineering_management"
      response_lead: "engineering_lead"
    
    - level: "medium"
      examples: ["increased_latency", "cost_overage"]
      sla_minutes: 240
      escalation: "team_lead"
      response_lead: "on_call_engineer"
  
  response_workflow:
    
    - phase: "detection"
      action: "automated_alert OR human_report"
      documentation: "create_incident_ticket"
    
    - phase: "triage"
      action: "assess_severity AND impact_scope"
      documentation: "update_ticket_with_context"
    
    - phase: "mitigation"
      action: "stop_bleeding (disable_agent, revert_change, etc)"
      documentation: "action_log"
    
    - phase: "investigation"
      action: "root_cause_analysis"
      documentation: "detailed_findings"
    
    - phase: "remediation"
      action: "implement_fix"
      documentation: "fix_verification"
    
    - phase: "deployment"
      action: "deploy_to_production"
      documentation: "deployment_log"
    
    - phase: "postmortem"
      action: "post_incident_review"
      documentation: "postmortem_report + action_items"
```

---

**最終更新**: 2026-04-22  
**参考**: CLAUDE.md, DESIGN.md と組み合わせて使用することを推奨
