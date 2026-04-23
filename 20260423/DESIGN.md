# DESIGN.md - アーキテクチャ設計原則

**バージョン**: 2.0 (2026年4月 - ハーネスエンジニアリング最新実装)

## 1. コア設計原則

### 1.1 責任の分離 (Separation of Concerns)

```
┌─────────────────────────────────────────────┐
│         User Interaction Layer              │
├─────────────────────────────────────────────┤
│     Master Orchestrator (Coordination)      │
├──────────────┬──────────────┬───────────────┤
│  Reasoning   │  Retrieval   │  Action Agent │
│    Agent     │    Agent     │               │
├──────────────┴──────────────┴───────────────┤
│      Validation & QA Agent (Gatekeeper)    │
├─────────────────────────────────────────────┤
│    Tool Execution Layer (Harness Core)     │
├──────────┬──────────────┬────────┬──────────┤
│ Security │  Monitoring  │ Retry  │  Audit   │
│ Boundary │  & Logging   │  Mech  │  Trail   │
├──────────┴──────────────┴────────┴──────────┤
│     Data Pipeline & External Services      │
└─────────────────────────────────────────────┘
```

**原則**: 各層は明確に定義された責務を持ち、下層への依存性のみを許可。

### 1.2 防御層設計 (Defense in Depth)

セキュリティとエラー対応が複数層で実装：

```
Layer 1: Input Validation
  ↓
Layer 2: Security Boundary Check
  ↓
Layer 3: Tool Execution Gating
  ↓
Layer 4: Output Validation
  ↓
Layer 5: Audit & Monitoring
```

### 1.3 可観測性優先 (Observability First)

すべての重要操作は観測可能であること：

- **実行時トレーシング**: 各エージェント→ツール呼び出しを追跡
- **パフォーマンスメトリクス**: P50, P95, P99 レイテンシ
- **エラー診断**: 失敗原因の完全ロギング
- **セキュリティ監査**: 全操作の不変記録

## 2. マルチエージェント調整パターン

### 2.1 Task Flow Architecture

```python
class TaskFlowOrchestrator:
    """
    タスク処理フロー:
    1. 分解フェーズ (Reasoning Agent)
    2. 取得フェーズ (Retrieval Agent)
    3. 実行フェーズ (Action Agent)
    4. 検証フェーズ (Validation Agent)
    """
    
    def process_task(self, task):
        # Step 1: Strategic Planning
        plan = self.reasoning_agent.decompose(task)
        
        # Step 2: Information Gathering
        context = self.retrieval_agent.gather(plan)
        
        # Step 3: Execution
        result = self.action_agent.execute(plan, context)
        
        # Step 4: Quality Assurance (必須ゲート)
        validated = self.validation_agent.validate(result)
        
        return validated
```

### 2.2 並列 vs 順序実行

```
並列実行の判定基準:
- タスクに依存性がない場合: 並列実行
- タスクに依存性がある場合: 順序実行
- 最大並列タスク数: 5 (リソース制約)

例:
┌─── Task A ───┐
│              ├─→ Aggregation
└─── Task B ───┘

Task A と Task B は並列実行可能
Aggregation は両者の完了後に実行
```

### 2.3 Conflict Resolution Strategy

```
紛争シナリオと解決方法:

1. Validation Agent vs Action Agent
   → Validation Agent優先（品質ゲート）

2. Reasoning Agent vs Retrieval Agent
   → Orchestrator による調停（タイムアウト30秒）

3. Unresolvable Conflict
   → Human-in-the-loop エスカレーション

4. Timeout Conflict
   → フォールバック戦略実行
   → Human審査スケジュール
```

## 3. データパイプライン設計

### 3.1 Data Flow

```
外部データソース
    ↓
[入力検証] (Schema, Format, Range)
    ↓
[キャッシュ確認] (Freshness < 5分)
    ↓
[品質検証] (Completeness, Accuracy)
    ↓
[変換・正規化]
    ↓
[エージェント利用]
    ↓
[監査ログ]
```

### 3.2 Data Quality Framework

```python
class DataQualityValidator:
    """
    4段階品質検証:
    1. Schema Validation: データ型とフィールド確認
    2. Completeness: NULL値・欠損フィールド確認
    3. Accuracy: サンプル検証とロジックテスト
    4. Freshness: タイムスタンプとキャッシュ確認
    """
    
    QUALITY_THRESHOLD = 0.95  # 95%以上の品質が必須
    
    def validate(self, data):
        schema_score = self.validate_schema(data)
        completeness_score = self.check_completeness(data)
        accuracy_score = self.verify_accuracy(data)
        freshness_score = self.check_freshness(data)
        
        total_score = (schema_score * 0.3 + 
                      completeness_score * 0.2 +
                      accuracy_score * 0.3 +
                      freshness_score * 0.2)
        
        if total_score < self.QUALITY_THRESHOLD:
            raise DataQualityError(f"Quality score: {total_score}")
        
        return data
```

## 4. セキュリティとガバナンス

### 4.1 人間-機械協調モデル

```
自律性スペクトラム:

LOW          MEDIUM       HIGH
AUTONOMY     AUTONOMY     AUTONOMY
    ↓          ↓            ↓
┌────────┬──────────┬───────────┐
│ Human  │ Human    │ Human     │
│ IN the │ ON the   │ OUT of    │
│ Loop   │ Loop     │ the Loop  │
└────────┴──────────┴───────────┘
│        │          │
┌──────────────────────────────┐
│ All Tool Calls Logged &     │
│ Auditable                    │
└──────────────────────────────┘

決定基準:
- Exploratory tasks → Human-out-of-loop
- Standard operations → Human-on-the-loop
- High-risk actions → Human-in-the-loop
```

### 4.2 高リスク操作の定義

```python
HIGH_RISK_OPERATIONS = {
    "financial_transaction": {
        "threshold": "$100",
        "approval_required": "human",
        "escalation_time": "immediate"
    },
    "data_deletion": {
        "threshold": "any",
        "approval_required": "human",
        "escalation_time": "immediate"
    },
    "permission_modification": {
        "threshold": "any",
        "approval_required": "human",
        "escalation_time": "immediate"
    },
    "external_api_call": {
        "threshold": "first_time",
        "approval_required": "human",
        "escalation_time": "interactive"
    }
}
```

## 5. エラーハンドリングとリカバリー

### 5.1 Error Classification

```
┌─────────────────────────────────────┐
│     Error Classification            │
├─────────┬───────────────┬───────────┤
│ Retry   │ Log & Alert   │ Escalate  │
│ Able    │               │           │
├─────────┼───────────────┼───────────┤
│Network  │ Parsing       │ Security  │
│Timeout  │ Schema        │ Auth      │
│Busy     │ Validation    │ Permission│
├─────────┴───────────────┴───────────┤
│         Exponential Backoff          │
│    (1s, 2s, 4s, 8s, 16s max)       │
└─────────────────────────────────────┘
```

### 5.2 Circuit Breaker Pattern

```python
class CircuitBreaker:
    """
    状態遷移:
    CLOSED (normal)
        ↓
    [Error threshold exceeded]
        ↓
    OPEN (blocking calls)
        ↓
    [Timeout: 60 seconds]
        ↓
    HALF_OPEN (test mode)
        ↓
    [Success] → CLOSED
    [Failure] → OPEN
    """
    
    ERROR_THRESHOLD = 5  # 5エラーでOPEN
    TIMEOUT_SECONDS = 60
```

## 6. パフォーマンス最適化

### 6.1 Prompt Caching Strategy

```
キャッシング対象:
- システムプロンプト (常時キャッシュ)
- 大型知識ベース (4KB+自動キャッシュ)
- API応答 (< 5分は再利用)
- 計算結果 (session内で再利用)

目標: 70%以上のキャッシュヒット率
```

### 6.2 Context Window Management

```
┌────────────────────────────────┐
│  Total Context (200K tokens)   │
├────────────────────────────────┤
│ System Prompt (cached):   5K   │
│ User History (managed): 50K    │
│ Knowledge Base (cached): 80K   │
│ Available for response:  65K   │
└────────────────────────────────┘

ポリシー:
- 80%超過: 自動圧縮開始
- 95%超過: 古い履歴削除
- 100%超過: エラーで処理中止
```

## 7. スケーラビリティ原則

### 7.1 Horizontal Scaling

```
マルチエージェント環境の拡張:

1 Orchestrator → n Reasoning Agents
            → n Retrieval Agents
            → n Action Agents
            → 1 Validation Agent (常に1つ)
```

### 7.2 Load Balancing

```python
class AgentLoadBalancer:
    """
    ラウンドロビン + 状態ベースのルーティング
    """
    
    def route_task(self, task):
        # 最も応答時間が短いエージェントを選択
        agent = min(
            self.available_agents,
            key=lambda a: a.avg_response_time
        )
        return agent.execute(task)
```

## 8. 監査とコンプライアンス

### 8.1 Immutable Audit Trail

```json
{
  "audit_record": {
    "timestamp": "ISO-8601",
    "transaction_id": "UUID",
    "agent_id": "string",
    "operation": "string",
    "input": "encrypted",
    "output": "encrypted",
    "status": "success|error",
    "user_id": "string",
    "approvals": ["id1", "id2"],
    "hash": "SHA-256"
  }
}
```

**特性**:
- 追加専用 (append-only)
- 暗号署名付き
- 改ざん検出可能
- 12ヶ月以上保持

---

**設計哲学**: "Trust But Verify"
すべてのエージェントは信頼できるが、すべての操作は監視・監査可能である必要があります。

**最終更新**: 2026年4月23日
