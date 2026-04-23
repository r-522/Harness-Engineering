# DESIGN.md - 設計原則とアーキテクチャガイドライン

このファイルはマルチエージェントシステムの設計原則とアーキテクチャを定義します。

## 基本設計原則

### 1. マイクロサービス的エージェント設計
```
エージェント = マイクロサービス
├─ 単一責任: 1エージェント = 1ドメイン
├─ 独立性: 他エージェントへの依存を最小化
├─ テスト可能性: 単体テスト可能な単位
└─ スケーラビリティ: エージェント数の増加に対応
```

### 2. 疎結合・高凝集
- **疎結合**: エージェント間は通信プロトコルのみで結合
- **高凝集**: 各エージェント内の機能は密結合（一体性重視）

### 3. 明示的なインターフェース
```yaml
agent_interface:
  inputs:
    - parameter_name: "type"
      required: true|false
      description: "説明"
  outputs:
    - field_name: "type"
      description: "説明"
  errors:
    - error_code: "description"
      recovery_action: "どう対応するか"
```

---

## マルチエージェント・オーケストレーション・パターン

### パターン1: Sequential (シーケンシャル)
**用途**: タスクが段階的で、前のステップ出力が次の入力になる場合

```
User Request
    ↓
[Orchestrator] → Task解析
    ↓
[Reasoning] → 分析・計画立案
    ↓
[Retrieval] → データ取得
    ↓
[Action] → 実行
    ↓
[Validator] → 検証
    ↓
Final Result
```

**実装例**:
```yaml
orchestration_pattern: sequential
steps:
  - step: 1
    agent: reasoning_agent
    input: "user_request"
    output: "task_plan"
    timeout: 30
  
  - step: 2
    agent: retrieval_agent
    input: "task_plan"
    output: "data_context"
    timeout: 60
  
  - step: 3
    agent: action_agent
    input: "data_context + task_plan"
    output: "execution_result"
    timeout: 120
  
  - step: 4
    agent: validator_agent
    input: "execution_result"
    output: "validation_report"
    timeout: 30
```

### パターン2: Hierarchical Supervisor (階層型)
**用途**: 1つのマスターエージェントが複数の専門エージェントを監督

```
              [Orchestrator]
                    ↓
        ┌─────────┬─────────┬─────────┐
        ↓         ↓         ↓         ↓
    [Reasoning] [Retrieval] [Action] [Validator]
        ↓         ↓         ↓         ↓
        └─────────┴─────────┴─────────┘
                    ↓
              Final Result
```

**実装例**:
```yaml
orchestration_pattern: hierarchical_supervisor
supervisor: orchestrator_agent
specialists:
  - id: reasoning_agent
    responsibility: "analysis"
    max_concurrent: 2
  
  - id: retrieval_agent
    responsibility: "data_fetching"
    max_concurrent: 5
  
  - id: action_agent
    responsibility: "execution"
    max_concurrent: 3
  
  - id: validator_agent
    responsibility: "validation"
    max_concurrent: 2

coordination:
  method: "request_response"
  timeout: 300
  error_handling: "escalate_on_failure"
```

### パターン3: Parallel Fan-Out/Gather (並列実行)
**用途**: 複数のエージェントが独立に処理でき、最後に結果を集約する場合

```
User Request
    ↓
[Orchestrator] → Task分割
    ↓
    ├─► [Agent1] ──┐
    ├─► [Agent2] ──┼─► [Synthesizer] → Result
    └─► [Agent3] ──┘
```

**実装例**:
```yaml
orchestration_pattern: parallel_fan_out_gather
tasks:
  - task_id: "security_check"
    agent: validator_agent
    timeout: 60
  
  - task_id: "performance_audit"
    agent: reasoning_agent
    timeout: 90
  
  - task_id: "data_validation"
    agent: retrieval_agent
    timeout: 120

gathering:
  synthesizer: orchestrator_agent
  aggregation_method: "weighted_merge"
  weights:
    security_check: 0.4
    performance_audit: 0.3
    data_validation: 0.3
```

### パターン4: Event-Driven (イベント駆動)
**用途**: リアルタイムイベントの非同期処理

```
Event Stream
    ↓
[Event Broker]
    ↓
    ├─► [Agent1] (alert on "security_issue")
    ├─► [Agent2] (alert on "performance_degradation")
    └─► [Agent3] (alert on "data_change")
```

**実装例**:
```yaml
orchestration_pattern: event_driven
event_broker: "Apache Kafka / AWS SNS"
event_subscriptions:
  - event_type: "code_commit"
    agent: reasoning_agent
    handler: "analyze_code_changes"
  
  - event_type: "security_alert"
    agent: validator_agent
    handler: "emergency_review"
  
  - event_type: "data_update"
    agent: retrieval_agent
    handler: "refresh_cache"

sla:
  processing_latency_ms: 500
  delivery_guarantee: "at_least_once"
```

---

## Model Context Protocol (MCP) 統合

MCP は エージェント ← → 外部システム の統合標準です。

### MCP アーキテクチャ

```
┌─────────────────────────────────────────┐
│         Claude Agent (Claude)            │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │  Skill/Tool Layer                │  │
│  └────────────┬─────────────────────┘  │
└───────────────┼──────────────────────────┘
                │
        ┌───────┴────────┐
        │   MCP Client   │
        └───────┬────────┘
                │
        ┌───────┴────────────────┐
        │  MCP Server Protocol   │
        │  (HTTP + WebSocket)    │
        └───────┬────────────────┘
                │
        ┌───────┴────────────────────────┐
        │   External Systems               │
        ├────────────────────────────────┤
        │ - GitHub API                   │
        │ - Slack API                    │
        │ - Database (SQL/NoSQL)         │
        │ - File Systems                 │
        │ - LLM APIs                     │
        │ - Monitoring Tools             │
        └────────────────────────────────┘
```

### MCP 設定ファイル例

```json
{
  "mcp_version": "1.0",
  "servers": [
    {
      "id": "github_mcp",
      "type": "github",
      "config": {
        "api_endpoint": "https://api.github.com",
        "auth_method": "oauth2",
        "token_source": "AWS_Secrets_Manager:github_pat",
        "encryption": "TLS_1_3"
      },
      "resources": [
        {
          "resource_id": "pr_review",
          "method": "POST /repos/{owner}/{repo}/pulls/{pr_number}/reviews",
          "rate_limit": "100 req/hour",
          "requester_agents": ["validator_agent"]
        },
        {
          "resource_id": "code_search",
          "method": "GET /search/code",
          "rate_limit": "30 req/minute",
          "requester_agents": ["reasoning_agent", "retrieval_agent"]
        }
      ]
    },
    {
      "id": "database_mcp",
      "type": "database",
      "config": {
        "engine": "postgresql",
        "endpoint": "${DB_ENDPOINT}",
        "auth_method": "service_account",
        "connection_pool": 20
      },
      "resources": [
        {
          "resource_id": "query_execute",
          "allowed_operations": ["SELECT"],
          "table_whitelist": ["users", "transactions", "logs"],
          "requester_agents": ["retrieval_agent"]
        },
        {
          "resource_id": "data_write",
          "allowed_operations": ["INSERT", "UPDATE"],
          "table_whitelist": ["audit_logs"],
          "requester_agents": ["action_agent"],
          "requires_approval": "validator_agent"
        }
      ]
    },
    {
      "id": "slack_mcp",
      "type": "slack",
      "config": {
        "api_endpoint": "https://slack.com/api",
        "auth_method": "bot_token",
        "token_source": "AWS_Secrets_Manager:slack_bot_token"
      },
      "resources": [
        {
          "resource_id": "send_message",
          "method": "POST /chat.postMessage",
          "channel_filter": "#alerts|#dev-team",
          "requester_agents": ["orchestrator_agent"]
        }
      ]
    }
  ]
}
```

### MCP セキュリティ設定

```yaml
mcp_security:
  transport:
    protocol: "HTTPS"
    tls_version: "1.3"
    certificate_pinning: true
  
  authentication:
    method: "OAuth2 + Service Account"
    token_rotation_hours: 24
    token_storage: "AWS_Secrets_Manager"
  
  rate_limiting:
    per_agent_rpm: 100
    per_mcp_server_rps: 1000
    circuit_breaker: "enabled"
  
  audit_logging:
    log_all_requests: true
    log_retention_days: 90
    sensitive_data_masking: true
```

---

## データフロー設計

### 安全なデータの流れ

```
1. Data Classification
   ├─ Public (誰でも見られる情報)
   ├─ Internal (社内のみ)
   ├─ Sensitive (限定アクセス)
   └─ Restricted (最小権限のみ)

2. Agent-to-Agent Data Transfer
   ├─ TLS 1.3 暗号化
   ├─ メッセージ署名（HMAC-SHA256）
   ├─ レート制限
   └─ 監査ログ

3. Data at Rest
   ├─ Encryption: AES-256
   ├─ Key Management: AWS KMS / HashiCorp Vault
   ├─ Access Control: IAM
   └─ Retention Policy: [指定期間後自動削除]
```

### 例：機密データ処理フロー

```yaml
data_flow:
  input: "user_credentials"
  classification: "restricted"
  
  flow_rules:
    - agent: "orchestrator"
      action: "pass_through"
      transformation: "none"
    
    - agent: "validator"
      action: "audit_only"
      transformation: "mask_sensitive_fields"
      log_destination: "audit_log_encrypted"
    
    - agent: "action"
      action: "use_for_execution"
      transformation: "encrypt_in_transit"
      ephemeral_storage: "memory_only"
      cleanup_after_minutes: 5

retention:
  audit_log: "12_months"
  operational_log: "30_days"
  sensitive_data: "0_days"  # 即刻削除
```

---

## エラーハンドリング戦略

### エラー分類と対応

```yaml
error_handling:
  categories:
    
    - name: "transient_error"
      examples: ["network_timeout", "rate_limit_exceeded"]
      strategy: "auto_retry"
      retry_policy: "exponential_backoff"
      max_attempts: 3
      backoff_seconds: [2, 4, 8]
    
    - name: "recoverable_error"
      examples: ["data_validation_failed", "missing_dependency"]
      strategy: "fallback_or_escalate"
      fallback_agent: "validation_agent"
      escalation_threshold: 2  # 2回失敗したらエスカレーション
    
    - name: "unrecoverable_error"
      examples: ["security_violation", "authorization_denied", "resource_not_found"]
      strategy: "immediate_escalation"
      escalation_target: "human_review"
      notification: "critical_alert"
    
    - name: "business_logic_error"
      examples: ["constraint_violation", "invalid_state"]
      strategy: "structured_analysis"
      analysis_agent: "reasoning_agent"
      decision_point: "human_approval"
```

### Graceful Degradation

```yaml
degradation_mode:
  trigger: "critical_resource_unavailable"
  
  example_scenario:
    unavailable_service: "external_api"
    fallback_plan:
      - step: 1
        action: "try_cache"
        timeout_seconds: 5
      
      - step: 2
        action: "try_readonly_mode"
        functionality: "limited"
        data_freshness: "may be stale"
      
      - step: 3
        action: "escalate_to_human"
        message: "API unavailable, manual intervention needed"
```

---

## スケーラビリティ設計

### 水平スケーリング

```yaml
scalability:
  agent_replication:
    strategy: "load_balanced_pool"
    
    pools:
      - pool_id: "reasoning_agents"
        min_replicas: 1
        max_replicas: 10
        scale_trigger: "queue_length > 50"
      
      - pool_id: "action_agents"
        min_replicas: 1
        max_replicas: 20
        scale_trigger: "latency_p95 > 60s"
  
  message_queue:
    broker: "Apache_Kafka"
    partitions: "num_agents * 2"
    replication_factor: 3
    retention_hours: 24

  database:
    read_replicas: "auto_scale"
    caching: "Redis_cluster"
    sharding_key: "agent_id"
```

---

## 監視とObservability

### メトリクス定義

```yaml
metrics:
  
  agent_performance:
    - name: "latency_p50"
      unit: "seconds"
      target: "< 10"
    
    - name: "latency_p95"
      unit: "seconds"
      target: "< 60"
    
    - name: "success_rate"
      unit: "%"
      target: "> 99.5"
    
    - name: "error_rate"
      unit: "%"
      target: "< 0.5"
  
  resource_usage:
    - name: "token_usage_per_task"
      unit: "tokens"
      alert_threshold: 50000
    
    - name: "mcp_call_count"
      unit: "calls"
      alert_threshold: "rate_limit_80%"
  
  business_metrics:
    - name: "cost_per_task"
      unit: "USD"
      budget_monthly: 10000
    
    - name: "human_escalation_rate"
      unit: "%"
      target: "< 5"
```

### ダッシュボード構成

```yaml
dashboards:
  
  - name: "Agent_Health"
    widgets:
      - latency_percentiles
      - error_rate
      - active_tasks
      - queue_depth
  
  - name: "Cost_Tracking"
    widgets:
      - daily_spend
      - cost_per_agent
      - cost_trend
      - budget_vs_actual
  
  - name: "Security_Audit"
    widgets:
      - access_patterns
      - data_movement
      - privilege_escalation_attempts
      - audit_log_health
```

---

**最終更新**: 2026-04-22  
**対応システム**: Enterprise-scale Multi-Agent Orchestration
