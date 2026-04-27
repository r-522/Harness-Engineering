# ORCHESTRATION.md - マルチエージェント協調とワークフロー設計

このドキュメントは、複数エージェント間の効果的な協調パターンと、ワークフロー管理のベストプラクティスを定義します。

## 協調パターン選択フロー

### パターン決定ツリー

```
タスク開始
  │
  ├─ 単一エージェント完結可？
  │  └─ YES → Standalone (パターン選択不要)
  │  └─ NO ↓
  │
  ├─ 複数タスクの独立性？
  │  ├─ 高（依存なし） → Orchestrator-Subagent
  │  └─ 低（強い依存） ↓
  │
  ├─ ドメイン知識の蓄積？
  │  ├─ 必須（長期プロジェクト） → Agent Teams
  │  └─ 不要 ↓
  │
  ├─ リアルタイムデータ融合？
  │  ├─ YES（研究、分析） → Shared State
  │  └─ NO ↓
  │
  ├─ イベント駆動型？
  │  ├─ YES（CI/CD、自動応答） → Message Bus
  │  └─ NO ↓
  │
  └─ 品質保証の二重チェック必須？
     ├─ YES（クリティカルコード） → Generator-Verifier
     └─ NO → Orchestrator-Subagent
```

## パターン1: Orchestrator-Subagent（推奨）

### 構造
```
┌─────────────────────────────────┐
│    Orchestrator Agent           │
│    (Task Distribution)          │
└───────┬────────┬────────┬───────┘
        │        │        │
        ▼        ▼        ▼
    ┌─────┐ ┌─────┐ ┌─────┐
    │Sub-1│ │Sub-2│ │Sub-3│
    │(CA) │ │(SR) │ │(PO) │
    └─────┘ └─────┘ └─────┘
```

### 実装例
```yaml
orchestrator:
  name: "Task Router"
  responsibility:
    - タスク分類・最適なSubagent選択
    - 結果統合
    - 最終レポート生成
  
subagents:
  - id: "CodeAnalyst"
    trigger: "code_analysis | architecture | design"
    timeout: 5分
    
  - id: "SecurityReviewer"
    trigger: "security | vulnerability | auth"
    timeout: 3分
    
  - id: "PerformanceOptimizer"
    trigger: "performance | optimize | cost"
    timeout: 5分

execution_mode: "parallel"
fallback: "sequential"
max_retries: 2
```

### メッセージシーケンス
```
User → Orchestrator
         │
         ├─→ Subagent-1 (timeout: 5分)
         ├─→ Subagent-2 (timeout: 3分)
         └─→ Subagent-3 (timeout: 5分)
         │
         └─→ Aggregate & Report
         
User ← Results
```

### 利点と制限
| 項目 | 詳細 |
|-----|------|
| 利点 | - 実装が単純<br>- 低オーバーヘッド<br>- デバッグしやすい<br>- スケーリング容易 |
| 制限 | - Subagent間の直接通信なし<br>- 相互依存タスク不向き<br>- 知識共有が限定的 |
| 推奨用途 | 大多数の実運用タスク |

---

## パターン2: Agent Teams（永続メンバー）

### 構造
```
┌─────────────┐     ┌──────────────┐
│ Agent-A     │────→│ Shared State │
│ (CodeGen)   │     │ Storage      │
└─────────────┘     │              │
                    │ - Knowledge  │
┌─────────────┐     │ - Context    │
│ Agent-B     │────→│ - Results    │
│ (CodeReview)│     │              │
└─────────────┘     └──────────────┘
                    ↑
┌─────────────┐     │
│ Agent-C     │─────┘
│ (Testing)   │
└─────────────┘
```

### 実装例
```yaml
team:
  name: "Development Team"
  members:
    - id: "Coder"
      role: "Generator"
      domain: "software engineering"
      
    - id: "Reviewer"
      role: "Validator"
      domain: "code quality"
      
    - id: "Tester"
      role: "Verifier"
      domain: "testing strategy"
  
  shared_state:
    backend: "Redis"
    tables:
      - project_knowledge
      - task_queue
      - result_cache
      - decision_log
  
  knowledge_persistence:
    - type: "embedding_vector"
      storage: "vector_db"
      update_freq: "on_change"
```

### 協調フロー
```
Iteration 1:
  Coder → generates_code → Shared State
  ↓
  Reviewer reads → validates → Shared State (feedback)
  ↓
  Coder reads_feedback → refines → Shared State
  ↓
  Tester reads → validates_tests → Shared State

Iteration 2: (より少ない往復回数)
  ...
```

### 利点と制限
| 項目 | 詳細 |
|-----|------|
| 利点 | - エージェント間の知識共有<br>- 反復的改善<br>- エラーリカバリ<br>- ドメイン学習 |
| 制限 | - 状態管理が複雑<br>- デバッグ困難<br>- 一貫性保証が必要<br>- レイテンシ増加 |
| 推奨用途 | 長期プロジェクト、進化型タスク |

---

## パターン3: Generator-Verifier（品質保証）

### 構造
```
        ┌─────────────────┐
        │ Task Input      │
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │ Generator Agent │
        │ (produces)      │
        └────────┬────────┘
                 │
        ┌────────▼────────┐
        │ Verifier Agent  │
        │ (validates)     │
        └────────┬────────┘
                 │
        ┌────────▼────────────────────┐
        │ Decision                    │
        │ ├─ OK → Output             │
        │ ├─ Revise → back to Gen    │
        │ └─ Reject → escalate       │
        └─────────────────────────────┘
```

### 実装例
```yaml
generator_verifier:
  generator:
    name: "CodeGenerator"
    prompt: |
      You generate production-ready code.
      Output MUST be:
      - Type-safe
      - Well-documented
      - Following style guide
      - With test cases
  
  verifier:
    name: "CodeAuditor"
    checks:
      - syntax_valid
      - type_safe
      - security_scan
      - performance_ok
      - test_coverage > 80%
    
    criteria:
      acceptable: "all checks PASS"
      revisable: "minor_issues <= 2"
      rejected: "critical_issues present"
  
  feedback_loop:
    max_iterations: 3
    timeout_per_iteration: 2分
```

### フロー例
```
Generate:  "Create authentication system"
    ↓
Output: (1000 lines of code)
    ↓
Verify:
  ✗ Security: Passwords stored in plaintext
  ✓ Syntax: Valid Python 3.11
  ✗ Tests: No test cases
    ↓
Feedback → Generator
    ↓
Regenerate:
  ✓ Security: Using bcrypt
  ✓ Syntax: Valid
  ✓ Tests: 95% coverage
    ↓
Output: Ready to ship
```

### 利点と制限
| 項目 | 詳細 |
|-----|------|
| 利点 | - 品質保証<br>- 明示的な検証基準<br>- 人間的なレビュー代替<br>- 信頼性向上 |
| 制限 | - レイテンシが長い(2-3倍)<br>- コスト2倍<br>- 完全自動化困難 |
| 推奨用途 | クリティカルなコード、本番環境 |

---

## パターン4: Shared State（非同期協調）

### 構造
```
┌────────────────────────────────────────┐
│       Shared State Store               │
│  (Redis / DynamoDB / PostgreSQL)       │
│                                        │
│  ├─ Knowledge Base (Vector DB)        │
│  ├─ Task Queue (Event Stream)         │
│  ├─ Result Cache (K-V Store)          │
│  └─ Decision Log (Time Series)        │
└────────────────────────────────────────┘
    ▲              ▲              ▲
    │              │              │
    │ (pub/sub)    │ (pub/sub)    │ (pub/sub)
    │              │              │
  ┌─┴──┐        ┌──┴─┐        ┌──┴──┐
  │AgA │        │AgB │        │AgC  │
  └────┘        └────┘        └─────┘
  Research    Synthesis    Validation
```

### 実装例（Redis Streams）
```python
# Agent A: Research
research_agent.publish("research:findings", {
    "source": "paper_001",
    "key_insights": ["insight1", "insight2"],
    "confidence": 0.95,
    "timestamp": "2026-04-27T10:30:00Z"
})

# Agent B: Synthesis (listening)
synthesis_agent.subscribe("research:*")
# On message:
synthesis_agent.merge_findings()
synthesis_agent.publish("synthesis:summary", {...})

# Agent C: Validation (listening)
validation_agent.subscribe("synthesis:*")
validation_agent.verify()
```

### 利点と制限
| 項目 | 詳細 |
|-----|------|
| 利点 | - 真の非同期性<br>- スケーラビリティ<br>- リアルタイム性<br>- 疎結合 |
| 制限 | - 状態一貫性の保証難<br>- デバッグ複雑<br>- 順序保証が限定的 |
| 推奨用途 | 大規模研究システム、データパイプライン |

---

## パターン5: Message Bus（イベント駆動）

### 構造
```
┌─────────────────────────────────────────┐
│      Event Bus (Kafka / RabbitMQ)       │
│                                         │
│  Topics:                                │
│  ├─ code.submitted                     │
│  ├─ code.reviewed                      │
│  ├─ test.passed                        │
│  ├─ deploy.ready                       │
│  └─ alert.critical                     │
└─────────────────────────────────────────┘
    ▲          ▲          ▲          ▲
    │Publisher │          │          │
    │          │Subscriber│          │
    │          │          │          │
  ┌─┴──┐    ┌──┴─┐    ┌───┴──┐   ┌──┴──┐
  │Gen │    │Rev │    │Test  │   │Ops  │
  └────┘    └────┘    └──────┘   └─────┘
```

### 実装例
```yaml
event_bus:
  type: "Kafka"
  topics:
    - name: "code.events"
      partitions: 3
      retention: "7 days"
    
    - name: "test.events"
      partitions: 2
      retention: "3 days"

consumers:
  - name: "CodeReviewer"
    topic: "code.events"
    filter: "event_type == 'code.submitted'"
    handler: "review_code"
    
  - name: "TestRunner"
    topic: "code.events"
    filter: "event_type == 'code.reviewed' AND status == 'approved'"
    handler: "run_tests"
    
  - name: "Deployer"
    topic: "test.events"
    filter: "event_type == 'test.passed'"
    handler: "deploy"
```

### 利点と制限
| 項目 | 詳細 |
|-----|------|
| 利点 | - 完全な疎結合<br>- スケーラビリティ<br>- 障害分離<br>- リトライ容易 |
| 制限 | - セットアップ複雑<br>- 順序保証困難<br>- デバッグ難 |
| 推奨用途 | CI/CD、自動応答システム |

---

## ハイブリッド協調パターン

実運用では複数パターンを組み合わせます。

### 推奨組み合わせ

#### シナリオ1: 標準的なソフトウェア開発
```
Orchestrator-Subagent (メイン)
  ├─ CodeGenerator (simple task)
  └─ CodeReviewer (simple task)

+ Generator-Verifier (クリティカルパス)
  ├─ SecurityCritical code
  └─ AuthenticationLogic
```

#### シナリオ2: 長期研究プロジェクト
```
Agent Teams (メイン)
  ├─ ResearcherA (persistent)
  ├─ ResearcherB (persistent)
  └─ Shared Knowledge Base

+ Shared State (結果統合)
  └─ Vector DB (論文埋め込み)

+ Message Bus (イベント)
  └─ New paper available → trigger synthesis
```

#### シナリオ3: 自動化エンジニアリング
```
Message Bus (メイン)
  ├─ code.submitted → 
  ├─ code.reviewed →
  ├─ test.passed →
  └─ deploy.ready

+ Orchestrator-Subagent (各ステップ)
  └─ 個別タスクの実行
```

---

## 協調パターンの比較表

| パターン | 実装複度 | レイテンシ | スケーラビリティ | 知識共有 | 推奨 |
|---------|--------|----------|-------------|--------|-----|
| Orchestrator | ⭐ | 低 | 高 | なし | ✓ |
| Agent Teams | ⭐⭐⭐ | 中 | 中 | 高 | △ |
| Generator-Verifier | ⭐⭐ | 高 | 低 | なし | △ |
| Shared State | ⭐⭐⭐⭐ | 中 | 非常に高 | 高 | △ |
| Message Bus | ⭐⭐⭐⭐ | 低 | 非常に高 | 限定的 | △ |

---

## ワークフロー管理

### Workflow状態機械

```
┌─────────┐
│ Pending │
└────┬────┘
     │
┌────▼────┐      ┌────────────┐
│ Running  │─────→│ Failed     │
└────┬────┘      └────────────┘
     │
┌────▼───────┐
│ Completed  │
└────────────┘
```

### 重要概念

**Idempotency（べき等性）**
- 同じ入力に対して常に同じ出力
- リトライ時の安全性確保

**Monotonicity（単調性）**
- 状態遷移は前進のみ
- ロールバック不可

**Observability（可観測性）**
- 各ステップの実行ログ
- デバッグ可能性

---

**バージョン**: 1.0  
**最終更新**: 2026-04-27
