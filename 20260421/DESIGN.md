# DESIGN.md - Architecture Guidelines & Design Principles

**Generated**: April 21, 2026  
**Architecture Framework**: Harness Engineering 1.0  
**Baseline**: Meta/Anthropic/Google Cloud 2026 Best Practices

## 🏗️ アーキテクチャ原則

### 原則 1: 複雑性の段階的削減

ハーネスエンジニアリングの核心は「システム設計による品質向上」です。

```
モデル能力依存度: 40%
システムアーキテクチャ: 60%

つまり: どのモデルでもシステムが良ければ出力品質が高い
```

**適用例**:

```
❌ 低品質: スーパーモデル + 平凡なシステム = 不安定
✅ 高品質: 普通モデル + 優れたハーネス = 安定・一貫性

実装: Planning/Generation/Evaluation の分離（3層構造）
```

### 原則 2: 3層エージェント構造（Anthropic Apr 2026）

```
┌─────────────────────────────────────┐
│     Layer 1: PLANNING AGENT         │
│     (Claude Haiku 4.5)              │
│  • Task decomposition               │
│  • Dependency analysis              │
│  • Risk assessment                  │
│  • Resource estimation              │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│    Layer 2: GENERATION AGENT        │
│     (Claude Sonnet 4.6)             │
│  • Implementation                   │
│  • Code generation                  │
│  • Testing                          │
│  • Integration                      │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│    Layer 3: EVALUATION AGENT        │
│     (Claude Opus 4.7)               │
│  • Quality assurance                │
│  • Performance validation           │
│  • Security audit                   │
│  • Production readiness             │
└─────────────────────────────────────┘
```

**メリット**:
- 各層の焦点が明確（責任分離）
- 並列実行可能性あり
- エラーの早期検出
- キャッシング最適化（各層で再利用可能）

### 原則 3: トークン効率と品質のバランス

```
Token Budget vs Quality Trade-off

100% Quality = 150K tokens/task (Opus large context)
80% Quality = 50K tokens/task (Sonnet optimized)
60% Quality = 15K tokens/task (Haiku minimal)

目標: 80% quality at 50K tokens budget
```

**実装方法**:
1. 計画フェーズで十分な分析（浪費防止）
2. キャッシング戦略（同じプロンプト再利用）
3. モデルルーティング（タスクに応じた選択）
4. RAG 統合（全文引用ではなく関連部分のみ）

### 原則 4: 検証可能性（Verifiable by Design）

すべての出力は**独立して検証可能**である必要があります。

```
Code → Tests → Metrics → Approval

各段階で「数字で証明できる」ことが重要
```

**実装例**:

```python
# ❌ Bad: 「品質が良い」だけ
generated_code = generate_code()
print("Code quality is good")

# ✅ Good: 測定可能な指標
test_result = run_tests(generated_code)
coverage = measure_coverage()
perf_score = benchmark_performance()

assert test_result.pass_rate > 0.95
assert coverage >= 0.85
assert perf_score['latency_ms'] < 200
```

## 🔧 システム設計パターン

### Pattern 1: Pipeline Architecture（シーケンシャル）

**用途**: 前のステップの出力が次のステップの入力になる場合

```
Input
  │
  ▼
┌──────────────┐
│  Validate    │ (数値検証)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Transform   │ (形式変換)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Optimize    │ (最適化)
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Output      │
└──────────────┘
```

**実装**:
```python
class Pipeline:
    def __init__(self, *stages):
        self.stages = stages
    
    def execute(self, input_data):
        result = input_data
        for stage in self.stages:
            result = stage(result)
            assert result is not None  # 検証
        return result
```

**使用例**: データ処理、コード生成→テスト→デプロイ

### Pattern 2: Fan-out/Fan-in Architecture（並列）

**用途**: 複数の独立したタスクが並列実行可能な場合

```
Input
  │
  ├─────────────┬─────────────┬─────────────┐
  │             │             │             │
  ▼             ▼             ▼             ▼
┌─────┐     ┌─────┐     ┌─────┐     ┌─────┐
│ Sec │     │ Perf│     │ Test│     │ Doc │
└──┬──┘     └──┬──┘     └──┬──┘     └──┬──┘
  │             │             │             │
  └─────────────┼─────────────┼─────────────┘
                │ (aggregate)
                ▼
            ┌─────────┐
            │ Approve │
            └─────────┘
```

**実装**:
```python
async def parallel_validation(code):
    results = await asyncio.gather(
        security_audit(code),
        performance_benchmark(code),
        unit_tests(code),
        documentation_check(code)
    )
    return aggregate_results(results)
```

**使用例**: 本番対応性確認（Security + Perf + Tests + Docs）

### Pattern 3: Supervisor Agent Pattern（階層的）

**用途**: 複数のスペシャライズドエージェントを制御

```
┌────────────────────────────┐
│     Supervisor Agent       │
│  (Task routing, conflict   │
│   resolution, aggregation) │
└──────────────┬─────────────┘
               │
    ┌──────────┼──────────┬──────────┐
    │          │          │          │
    ▼          ▼          ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Agent1 │ │ Agent2 │ │ Agent3 │ │ Agent4 │
│(Backend)│ │(Frontend)│ │ (DB) │ │(DevOps)│
└────────┘ └────────┘ └────────┘ └────────┘
```

**実装**:
```python
class SupervisorAgent:
    def __init__(self, specialist_agents: dict):
        self.agents = specialist_agents
    
    def route_task(self, task):
        best_agent = self.select_agent(task)
        result = self.agents[best_agent].execute(task)
        return self.validate_result(result)
    
    def select_agent(self, task):
        # タスク内容から最適なエージェント判定
        if "backend" in task.keywords:
            return "backend_expert"
        elif "ui" in task.keywords:
            return "frontend_expert"
        # ...
```

**使用例**: マルチエージェント開発（Backend/Frontend/DB/DevOps）

## 📊 データフロー・キャッシング戦略

### Cache Hierarchy（キャッシュ階層）

```
┌─────────────────────────────────────────┐
│  L1 Cache (Prompt Caching - 5m/1h TTL) │
│  • System instructions                  │
│  • Project context                      │
│  • Reusable code libraries              │
│  Hit Rate Goal: 75-90%                  │
│  Cost Savings: 90% on cached tokens     │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  L2 Cache (Session Context)              │
│  • Conversation history (recent 10)     │
│  • Generated artifacts                  │
│  • Test results                         │
│  Scope: Current session only            │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│  L3 Cache (File System)                  │
│  • .claude/ folder (settings, hooks)    │
│  • Generated reports                    │
│  • Performance metrics                  │
│  Scope: Project persistent              │
└─────────────────────────────────────────┘
```

### Caching Implementation

```python
# L1: Prompt Caching (Claude API)
system_prompt = {
    "type": "text",
    "text": full_instructions,
    "cache_control": {"type": "ephemeral"}  # or "prefill"
}
# Cache TTL: 1 hour (optimized for 2026)
# Cost: 10% of normal input token price

# L2: Session Context
recent_messages = session_memory[-10:]  # Automatic pruning

# L3: Persistent Storage
@cache_result(ttl="24h", scope="project")
def load_project_context():
    return load_from_file(".claude/context.json")
```

## 🔐 セキュリティ設計

### Threat Model

```
Threat                          Mitigation Strategy
──────────────────────────────  ─────────────────────────
Credential leakage              Secret scanning (pre-commit)
SQL Injection                   Parameterized queries
XSS attacks                     HTML escaping, CSP headers
Unauthorized access             RBAC, API key rotation
Data breach                     Encryption at rest/transit
DoS attacks                     Rate limiting, circuit breakers
Dependency vulnerabilities      npm audit, supply chain scan
Prompt injection                Input validation, sandboxing
```

### Security Checklist (Design Phase)

```yaml
authentication:
  - MFA enabled for all accounts
  - API keys rotated monthly
  - No hardcoded credentials
  
authorization:
  - RBAC implemented
  - Least privilege principle
  - API endpoint protection
  
data_protection:
  - Encryption in transit (TLS 1.3+)
  - Encryption at rest (AES-256)
  - PII handling documented
  
audit_logging:
  - All API calls logged
  - Sensitive operations flagged
  - Log retention > 90 days
```

## 📈 スケーラビリティ設計

### Horizontal Scaling

```
Single Instance (1-10K req/day)
    ▼
Horizontal Scaling (10K-100K req/day)
  • Load balancer (nginx, AWS ALB)
  • Stateless API servers
  • Shared database
  • Cache layer (Redis)
    ▼
Distributed System (>100K req/day)
  • Microservices
  • Message queue (RabbitMQ, Kafka)
  • Event-driven architecture
  • CDN for static content
```

### Database Scaling

```
Single Database (Suitable for <10M records)
    ↓ (Partitioning strategy)
Database Sharding (10M-1B records)
    ↓ (Multi-region replication)
Distributed Database (>1B records)
```

## 🎯 Performance Targets

### API Response Time SLA

```
API Endpoint Type          Target Latency    Priority
──────────────────────────  ────────────────  ───────
GET (read-only)            < 100ms           P0
POST (creation)            < 200ms           P0
Complex query              < 500ms           P1
Batch operation            < 2s              P1
Async operation            < 5s timeout      P2
```

### AI Model Latency (April 2026)

```
Model              First Token (ms)  Total Latency (s)
──────────────────  ────────────────  ──────────────────
Haiku 4.5           80-150           1.5-3s
Sonnet 4.6          150-250          3-8s
Opus 4.7            250-400          8-15s

First-token latency is critical for streaming responses
```

## 🚀 Deployment Strategy

### Stage 1: Development
```
- All 3 agents enabled
- Full logging
- Cost: Highest
- Risk: None (local)
```

### Stage 2: Staging
```
- Reduced context windows
- Selective caching
- Cost: 50% of Dev
- Risk: Test with real data subset
```

### Stage 3: Production
```
- Optimized caching (TTL: 1h)
- Minimal logging (INFO/ERROR only)
- Model routing enabled (Haiku for planning)
- Cost: 20% of Dev
- Risk: Monitored, rollback ready
```

## 📋 Design Review Checklist

新しい機能実装時は必ず確認:

```yaml
architecture:
  - [ ] Design pattern chosen and documented
  - [ ] Data flow diagrams created
  - [ ] Error handling paths identified
  - [ ] Security threats assessed
  - [ ] Scalability plan defined

performance:
  - [ ] Database indexes planned
  - [ ] Caching strategy optimized
  - [ ] API response time SLA defined
  - [ ] Load test plan created
  - [ ] Bottleneck identified and mitigated

operations:
  - [ ] Monitoring/alerting configured
  - [ ] Logs structure defined
  - [ ] Backup/recovery plan documented
  - [ ] Runbook created for incidents
  - [ ] Deployment procedure tested
```

---

**最終更新**: 2026年04月21日  
**バージョン**: 1.0  
**管理**: Harness Engineering Design Committee
