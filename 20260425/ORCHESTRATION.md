# ORCHESTRATION.md - エージェント間のオーケストレーション戦略

このファイルはマルチエージェント・システムの統合・協調戦略を定義します。

**最終更新**: 2026-04-25  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

---

## オーケストレーション基本概念

### Plan-Do-Evaluate Loop（PDE）

すべてのマルチエージェント・ワークフローの基礎：

```
┌─────────────────────────────────────────────┐
│                  LOOP (繰り返し)             │
├─────────────────────────────────────────────┤
│                                             │
│  PLAN                                       │
│  • タスク分解                               │
│  • エージェント選択                         │
│  • 制約条件設定                             │
│  ↓                                          │
│                                             │
│  DO                                         │
│  • Generator が提案生成                     │
│  • Evaluator が品質検証                     │
│  • Actioner が実行                          │
│  ↓                                          │
│                                             │
│  EVALUATE                                   │
│  • Sensor が結果観測                        │
│  • フィードバック収集                       │
│  • 成功/失敗判定                            │
│  ↓                                          │
│                                             │
│  [成功? → 完了] [失敗? → PLAN に戻る]      │
│                                             │
└─────────────────────────────────────────────┘
```

### Adaptive Orchestration（適応的オーケストレーション）

タスク特性に応じてワークフローを動的に選択：

```
タスク特性の分析
    ↓
    ├─ リスク: LOW, 頻度: HIGH
    │  → Fast-Path (Generator のみ)
    │
    ├─ リスク: MEDIUM, 複雑度: HIGH
    │  → Standard-Path (Generator + Evaluator)
    │
    └─ リスク: HIGH, 重要度: CRITICAL
       → High-Risk-Path (全ステップ + 複重検証)
```

---

## ワークフロー設計テンプレート

### Template 1: Fast-Path（定型タスク用）

**適用条件**: リスク低、頻度高、定型的

```yaml
workflow_fast_path:
  name: "Fast-Path: 定型処理"
  estimated_duration: "90s"
  human_checkpoint: false
  
  stages:
    - stage: "REQUEST"
      agent: "Orchestrator"
      action: "入力チェック（形式、範囲）"
      timeout: "10s"
      output: "validated_request"
    
    - stage: "GENERATE"
      agent: "Generator"
      action: "定型提案生成"
      constraints: ["safety_rules.yaml"]
      timeout: "60s"
      output: "proposal"
    
    - stage: "AUTO_APPROVE"
      agent: "Orchestrator"
      action: "事前登録済みパターンと比較 → 自動承認"
      timeout: "5s"
      output: "approval_decision"
    
    - stage: "EXECUTE"
      agent: "Actioner"
      action: "承認アクション実行"
      timeout: "30s"
      output: "execution_result"
    
    - stage: "SENSOR_CHECK"
      agent: "Sensor"
      action: "軽微な異常チェック"
      timeout: "10s"
      output: "result_with_corrections"
  
  escalation_rules:
    - condition: "Sensor が警告レベル以上の異常検知"
      action: "人間レビューに エスカレーション"
```

**実装例（JSON化）**:

```json
{
  "workflow": "fast_path",
  "user_request": "ログファイル取得",
  
  "orchestrator_plan": {
    "task_analysis": {
      "complexity": "low",
      "risk": "low",
      "estimated_time": "60s",
      "selected_workflow": "fast_path"
    }
  },
  
  "generator_proposal": {
    "id": "proposal_001",
    "description": "grep + awk でログ抽出",
    "difficulty": "★☆☆"
  },
  
  "auto_approval": {
    "pattern_match": "log_retrieval_approved_pattern",
    "approved": true,
    "reason": "事前登録済みの安全なパターン"
  },
  
  "actioner_execution": {
    "command": "grep ... | awk ...",
    "status": "success",
    "result": "logs.txt"
  },
  
  "sensor_check": {
    "anomalies": [],
    "status": "clean",
    "final_result": "SUCCESS"
  }
}
```

### Template 2: Standard-Path（標準タスク用）

**適用条件**: リスク中、複雑度中程度

```yaml
workflow_standard_path:
  name: "Standard-Path: 標準ワークフロー"
  estimated_duration: "5-10分"
  human_checkpoint: true
  
  stages:
    - stage: "GUIDE"
      agent: "Guide"
      action: "リスク行動予測 → Generator に制約追加"
      output: "safety_constraints"
    
    - stage: "GENERATE"
      agent: "Generator"
      action: "3-5個の提案生成（制約条件下）"
      timeout: "60s"
      output: "proposals[]"
    
    - stage: "EVALUATE"
      agent: "Evaluator"
      action: "各提案をセキュリティ・パフォーマンス観点で評価"
      timeout: "90s"
      output: "evaluation_results[]"
    
    - stage: "HUMAN_REVIEW"
      agent: "Human"
      action: "Evaluator 評価を確認 → GO/CAUTION/NO-GO 判定"
      timeout: "30分"
      output: "human_decision"
    
    - stage: "PRE_EXECUTION"
      agent: "Sensor"
      action: "実行予定内容のプリチェック（ドライラン）"
      timeout: "60s"
      output: "pre_check_result"
    
    - stage: "EXECUTE"
      agent: "Actioner"
      action: "承認アクション実行"
      timeout: "varies"
      output: "execution_log"
    
    - stage: "POST_EXECUTION"
      agent: "Sensor"
      action: "実行結果観測・自動補正"
      timeout: "60s"
      output: "final_result"
```

**実装例（YAML）**:

```yaml
execution_log:
  task_id: "task_20260425_001"
  workflow: "standard_path"
  
  guide_output:
    risks_predicted:
      - "DB権限逆脱"
      - "無限クエリループ"
    constraints_added: "max_queries=100, timeout=30s"
  
  generator_output:
    proposals:
      - id: 1
        description: "新規ユーザー作成（メール検証付き）"
        difficulty: "★★☆"
      - id: 2
        description: "バッチ作成（CSV Import）"
        difficulty: "★★★"
      - id: 3
        description: "API ゲートウェイ経由作成"
        difficulty: "★★☆"
  
  evaluator_output:
    evaluation_result:
      proposal_1:
        security: "PASS"
        performance: "PASS"
        maintainability: "PASS"
        overall: "GO"
      proposal_2:
        security: "PASS"
        performance: "CAUTION (bulk operation risk)"
        maintainability: "PASS"
        overall: "CAUTION"
      proposal_3:
        security: "PASS"
        performance: "PASS"
        maintainability: "PASS"
        overall: "GO"
    recommendation: "Proposal 1 が最も安全で推奨"
  
  human_review:
    reviewed_by: "user@example.com"
    reviewed_at: "2026-04-25T10:30:00Z"
    decision: "GO"
    feedback: "提案1で実行を許可"
  
  sensor_pre_execution:
    status: "ready_to_execute"
    warnings: []
  
  actioner_execution:
    status: "executing"
    progress: "75%"
    estimated_completion: "2026-04-25T10:35:00Z"
  
  sensor_post_execution:
    status: "success"
    metrics:
      users_created: 5
      execution_time: "12s"
      api_calls: 8
    anomalies: []
```

### Template 3: High-Risk-Path（高リスク・重要決定用）

**適用条件**: リスク高、規制・ガバナンス重視

```yaml
workflow_high_risk_path:
  name: "High-Risk-Path: 強化検証ワークフロー"
  estimated_duration: "30-60分"
  human_checkpoint: true
  multi_approval: true
  
  stages:
    - stage: "RISK_ASSESSMENT"
      agent: "Reasoner"
      action: "詳細なリスク分析、規制要件確認"
      output: "risk_report"
    
    - stage: "GUIDE"
      agent: "Guide"
      action: "極めて詳細な制約条件設定"
      constraints:
        - "最小権限のみ"
        - "監査ログ記録必須"
        - "ロールバック計画必須"
    
    - stage: "GENERATE_CONSTRAINED"
      agent: "Generator"
      action: "厳格な制約下で提案生成"
      timeout: "90s"
      output: "conservative_proposals[]"
    
    - stage: "EVALUATE_STRICT"
      agent: "Evaluator"
      action: "最も厳格な基準で評価"
      timeout: "120s"
      output: "strict_evaluation[]"
    
    - stage: "VALIDATOR_REVIEW"
      agent: "Validator"
      action: "セキュリティ・コンプライアンス監査"
      timeout: "30分"
      output: "compliance_report"
    
    - stage: "MULTI_HUMAN_REVIEW"
      agents: ["approver_1", "approver_2"]
      action: "複数承認者によるレビュー・承認"
      timeout: "60分"
      output: "multi_approval_decision"
    
    - stage: "EXECUTE_STAGED"
      agent: "Actioner"
      action: "段階的実行（進捗確認 + ロールバック態勢）"
      steps:
        - step_1: "10% 実行"
        - review_checkpoint: "影響確認"
        - step_2: "50% 実行"
        - review_checkpoint: "再度影響確認"
        - step_3: "100% 実行"
    
    - stage: "AUDIT"
      agent: "Sensor"
      action: "実行結果の完全監査・記録"
      timeout: "60s"
      output: "audit_report"
```

---

## エージェント選択ロジック

### Decision Tree（意思決定木）

```
タスク受信
    ↓
リスク分析
    ├─ リスク = LOW
    │  ├─ 頻度 = HIGH
    │  │  → Fast-Path (Generator のみ)
    │  └─ 頻度 = LOW
    │     → Standard-Path
    │
    ├─ リスク = MEDIUM
    │  └─ 複雑度 = ?
    │     → Standard-Path + Guide/Sensor
    │
    └─ リスク = HIGH
       └─ → High-Risk-Path (全ステップ)
```

### リスク評価マトリックス

```
                低      中      高
複雑度
高            Standard  High    High
中            Standard  Standard High
低            Fast      Standard Standard
```

---

## 段階的フェーズゲーティング

### Phase Gates（段階制御ポイント）

```
Phase 0: REQUEST
├─ 入力形式チェック
├─ 権限確認
├─ リソース確認
└─ [GATE] 合否判定 → NO: エラー返却

Phase 1: ANALYSIS
├─ リスク評価
├─ タスク分解
├─ エージェント選択
└─ [GATE] 実行方針確認 → ユーザー確認（必要に応じて）

Phase 2: GENERATION
├─ Generator が提案生成
├─ 品質チェック
└─ [GATE] Evaluator による初期評価 → NO: 再生成

Phase 3: EVALUATION
├─ Evaluator が厳格検証
├─ セキュリティスキャン
└─ [GATE] GO/CAUTION/NO-GO → 人間判断

Phase 4: PRE_EXECUTION
├─ Sensor がドライラン
├─ 副作用予測
└─ [GATE] 実行可否確認 → 最終人間承認

Phase 5: EXECUTION
├─ Actioner が実行
├─ 進捗ログ記録
└─ [GATE] 段階的チェックポイント（高リスクの場合）

Phase 6: POST_EXECUTION
├─ Sensor が結果観測
├─ 自動補正試行
└─ [GATE] 最終結果確認
```

### Gate 判定ルール

```yaml
gate_rules:
  phase_0_request:
    pass_conditions:
      - input_format_valid: true
      - user_authorized: true
      - resources_available: true
    fail_action: "return_error_to_user"
  
  phase_2_generation:
    pass_conditions:
      - generator_completed: true
      - proposal_count: "> 1"
      - quality_score: "> 0.5"
    fail_action: "retry_generation"
  
  phase_3_evaluation:
    pass_conditions:
      - evaluator_completed: true
      - recommendation_provided: true
      - risk_assessment_done: true
    fail_action: "human_escalation"
  
  phase_5_execution:
    pass_conditions:
      - human_approval: true      # HIGH_RISK の場合
      - pre_check_passed: true
      - rollback_plan_ready: true
    fail_action: "abort_and_escalate"
```

---

## Feedback Loop & Learning

### タスク完了後のフィードバック統合

```
Execution Complete
    ↓
Feedback Collection
├─ 実行時間（予測との比較）
├─ コスト（予測との比較）
├─ 品質スコア（ユーザー評価）
├─ エラー/警告（Sensor 検出）
└─ 人間フィードバック（アンケート）
    ↓
Analytics
├─ Generator の精度向上
├─ Evaluator の閾値調整
├─ 制約条件の最適化
└─ ワークフロー選択ロジック改善
    ↓
Apply Updates
├─ Generator プロンプト調整
├─ Evaluator ガイドライン修正
└─ ワークフロー定義アップデート
```

### メトリクス追跡

```yaml
feedback_metrics:
  generator_metrics:
    - proposal_diversity_score
    - time_accuracy (predicted vs actual)
    - user_satisfaction
  
  evaluator_metrics:
    - false_positive_rate (不要な NO-GO)
    - false_negative_rate (見逃した問題)
    - evaluation_time_accuracy
  
  workflow_metrics:
    - fast_path_adoption_rate
    - phase_gate_failure_rate
    - overall_success_rate
```

---

## コスト管理

### Cost Awareness（コスト認識）

```yaml
cost_control:
  per_task_limit: "$1.00"
  monthly_budget: "$10,000"
  
  cost_breakdown:
    generator_60s: "$0.15"
    evaluator_90s: "$0.25"
    sensor_monitor: "$0.05"
    orchestrator: "$0.05"
    
  cost_optimization:
    - cache_hit_target: "50%"  # Prompt Caching
    - batch_processing: "group similar tasks"
    - model_selection: "use Haiku for fast-path"
```

### Cost Gate

```
タスク開始時: 予想コスト * 1.5 をチェック
タスク実行中: リアルタイムコスト監視
超過時: 自動ストップ / 人間警告
```

---

## トラブルシューティング・フローチャート

```
[エージェントが失敗]
    ↓
リトライポリシー確認
├─ 最大3回まで exponential backoff (2s, 4s, 8s)
├─ 3回以上失敗 → [フェーズバック or エスカレーション]
└─ Circuit breaker 発動 → [停止 + 人間通知]
    ↓
失敗の分類
├─ 一時的エラー（network timeout 等）
│  → 自動リトライ
├─ 入力エラー（不正フォーマット）
│  → ユーザーに再入力要求
├─ 権限エラー（パーミッション不足）
│  → 権限昇格リクエスト
└─ 論理エラー（計画そのもの不可能）
   → ワークフロー見直し / 人間判断
```

---

##監視・アラート

### Key Metrics to Monitor

```yaml
kpis:
  latency:
    p50: "< 5分"
    p95: "< 15分"
    p99: "< 30分"
    alert_threshold: "p95 > 20分"
  
  success_rate:
    target: "> 95%"
    alert_threshold: "< 90%"
  
  human_escalation_rate:
    target: "< 5%"
    alert_threshold: "> 10%"
  
  cost_per_task:
    target: "< $0.50"
    alert_threshold: "> $1.00"
```

### Alert Conditions

```yaml
alerts:
  - condition: "agent_error_rate > 5%"
    severity: "critical"
    action: "page_on_call_engineer"
  
  - condition: "phase_gate_failure_rate > 20%"
    severity: "high"
    action: "notify_team"
  
  - condition: "cost_per_task > $1.00"
    severity: "medium"
    action: "notify_manager"
  
  - condition: "latency_p95 > 20分"
    severity: "medium"
    action: "investigate_bottleneck"
```

---

**最終更新**: 2026-04-25  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

