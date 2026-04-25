# CLAUDE.md - プロジェクト規約と動作制限

このファイルはAIエージェント（Claude）が従うべきプロジェクト固有のルールを定義します。

**最終更新**: 2026-04-25  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

## プロジェクト情報

**プロジェクト名**: Harness Engineering Configuration Hub 2026  
**構成対象**: エンタープライズレベルのマルチエージェント・オーケストレーションシステム  
**チーム規模**: 5-50名（エージェント開発チーム）  
**環境**: 本番環境（セキュリティ・ガバナンス重視）  
**最新アーキテクチャ**: Generator/Evaluator + Guide/Sensor パターン

## 最重要原則

### 1. **Bounded & Deterministic Workflows**
- エージェント数は3-5個に制限（超大規模スワームは禁止）
- 各ワークフローに明確な終了条件を設定
- 段階的フェーズゲーティング：各フェーズ後に人間レビュー挟む

### 2. **Generator / Evaluator の明確な分離**
- **Generator**: 仮説・提案生成に特化（高速・創造的）
- **Evaluator**: 品質検証に特化（厳密・保守的）
- 両者の結果を統合し、精度と創造性のバランスを実現

### 3. **Guide & Sensor 制御**
- **Guide（事前制御）**: 不適切な行動経路を事前に防止
- **Sensor（事後制御）**: 実行後に自己観測し、誤りに気づき自己補正
- 双方向フィードバックループで自律性と安全性を両立

### 4. **コード化ファースト**
- スキル、プロンプト、MCP設定は全てコード化
- Git管理、PR・レビュー必須
- 不備なく再現可能であることが絶対条件

### 5. **Context Discipline（33Kトークン対応）**
- Claude Code 2026-04版バッファ削減に対応（45K→33K）
- `/compact` で定期的にコンテキスト圧縮
- プロンプトキャッシング（1時間TTL）を活用

## コーディング規約

### ファイル構成

```
.
├── agents/
│   ├── generator/          # 仮説生成エージェント
│   ├── evaluator/          # 品質検証エージェント
│   ├── orchestrator/       # オーケストレータ
│   ├── reasoner/           # 推論エージェント
│   ├── retriever/          # 検索エージェント
│   └── actioner/           # アクションエージェント
├── skills/
│   ├── generation.md       # 生成スキル
│   ├── evaluation.md       # 評価スキル
│   └── ...
├── mcp/
│   ├── github.mcp.json
│   ├── database.mcp.json
│   └── ...
├── prompts/
│   ├── generator/
│   ├── evaluator/
│   └── guides/             # Guide制御プロンプト
├── orchestration/
│   ├── phase_gating.yaml
│   ├── adaptive_routing.yaml
│   └── ...
└── monitoring/
    ├── guardrails.yaml
    └── audit_logs.yaml
```

### 命名規則

#### エージェント名
```
{role}_{specialization}_{version}_{variant}

例:
- generator_hypothesis_v1_fast
- evaluator_security_v1_strict
- orchestrator_main_v1_adaptive
- reasoner_complex_v2_extended-thinking
```

#### スキル名
```
{domain}_{action}_{specificity}

例:
- generation_creative_prompt
- evaluation_security_vulnerability_scan
- guide_prevent_permission_escalation
- sensor_detect_anomaly_execution
```

### プロンプト設計のルール

#### 1. 明示性（Explicitness First）
```
✗ 悪い例:
"いつものようにエージェントらしく処理して"

✓ 良い例:
"以下の順序で実行: 
1) 入力を解析
2) [指定ルール]に基づき評価
3) 3つの選肢肢を生成
4) 最も安全な選択肢を推奨"
```

#### 2. コンテキスト配置
- **重要**: クエリ・タスク指示は**コンテキストの最後**に配置
- モデルの注意機構がクエリ直前の情報に集中する性質を利用

#### 3. XML タグの廃止（2026年推奨）
- Claude Opus 4.7 は自然な構造化を理解
- 必要な場合のみマークダウンのセクション記法を使用
```
# Task
## Subtask 1
## Subtask 2
```

#### 4. Generator / Evaluator の明確なプロンプト例

**Generator プロンプト（創造的・高速）**:
```
あなたはハイポセシス生成エージェントです。
以下の課題に対し、**3つの異なるアプローチ**を迅速に提案してください。

課題: [タスク説明]

要件:
- 創造的で実行可能な案を優先
- 各案に簡潔な説明（50語以下）
- 実装難度を★☆☆～★★★で評価

出力形式:
案1: [提案] - 難度: ★★☆
案2: [提案] - 難度: ★☆☆
案3: [提案] - 難度: ★★★
```

**Evaluator プロンプト（厳密・品質重視）**:
```
あなたはシニア品質監査官です。
以下の提案を厳密に評価してください。

提案: [Generator からの案]

評価基準:
1. セキュリティ: OWASP Top 10, CWE対応
2. パフォーマンス: レイテンシ、スループット
3. 保守性: コード可読性、テスト性
4. ガバナンス: 権限、監査可能性
5. リスク: 予期しない副作用

評価結果:
GO / CAUTION(条件: [記載]) / NO-GO(理由: [記載])
```

#### 5. Guide（事前制御）の実装

```
Guide テンプレート:

あなたはGuideエージェントです。
次のGeneratorエージェントの行動を**予測**し、
不適切な経路を回避させてください。

推奨される行動:
1. [安全な行動A]
2. [安全な行動B]

避けるべき行動:
❌ [危険な行動X] - 理由: ...
❌ [危険な行動Y] - 理由: ...

Generatorへの指示:
"[タスク]を実行する際、上記の'避けるべき行動'
を絶対に取らないでください"
```

#### 6. Sensor（事後制御）の実装

```
Sensor テンプレート:

あなたはSensorエージェントです。
次のアクションが実行された後、
不正な結果を検出し自己補正してください。

実行内容: [実施したアクション]
実行結果: [実際の結果]

チェック項目:
1. 意図した結果が得られたか?
2. 予期しない副作用はないか?
3. セキュリティ侵害の兆候はないか?
4. パフォーマンス劣化はないか?

検出結果:
✓ 問題なし / ⚠️ 警告 / ✗ エラー

補正アクション:
[修正が必要な場合の対応]
```

## ガードレール（Guardrails）

### 1. エージェント権限マトリックス

```yaml
roles:
  generator:
    read: "high"        # 広くリサーチ
    write: "none"       # 読み取り専用
    escalate: "normal"
  
  evaluator:
    read: "high"
    write: "none"       # 検証のみ
    escalate: "normal"
  
  orchestrator:
    read: "high"
    write: "none"       # 読み取り専用
    escalate: "critical"
  
  actioner:
    read: "normal"
    write: "restricted" # 割り当て範囲のみ
    escalate: "immediate"
```

### 2. 出力検証（Sanitization）

```yaml
output_validation:
  - rule: "sensitive_data_leak"
    patterns:
      - "password|passwd|pwd"
      - "api[_-]?key|apikey"
      - "token|secret|credential"
      - "aws_secret|private_key"
    action: "BLOCK_AND_ALERT"
    
  - rule: "unsafe_command"
    patterns:
      - "rm -rf|rm -f \/"
      - "DROP TABLE|DELETE FROM"
      - "kill -9"
      - "> /dev/sda"
    action: "ESCALATE_TO_HUMAN"
    
  - rule: "permission_escalation"
    patterns:
      - "sudo|chmod 777|chown root"
      - "grant.*admin|create user.*admin"
    action: "ESCALATE_TO_HUMAN"
```

### 3. リトライ & タイムアウト戦略

```yaml
resilience:
  max_retries: 3
  retry_policy: "exponential_backoff"
  retry_intervals_seconds: [2, 4, 8]
  timeout_seconds:
    generator: 60
    evaluator: 90
    orchestrator: 300
    actioner: 120
  circuit_breaker: "enabled"
  circuit_breaker_threshold: 5  # 5回連続失敗でブレーカー開く
```

## Generator / Evaluator Workflow

### 標準フロー

```
User Request
    ↓
Orchestrator (タスク解析)
    ↓
Guide (事前制御: 避けるべき行動を提示)
    ↓
Generator (3-5個の仮説・提案を迅速生成)
    ↓
Evaluator (各提案を厳密に品質チェック)
    ↓
Sensor (実行予定前に副作用を検出)
    ↓
Actioner (承認された提案を実行)
    ↓
Sensor (実行後の自己観測・補正)
    ↓
Feedback Loop (次回Generator試行に反映)
```

## 監視とObservability

### 必須ログフォーマット

```json
{
  "timestamp": "ISO-8601",
  "task_id": "unique_uuid",
  "agent_id": "generator_hypothesis_v1_fast",
  "stage": "generation|evaluation|execution|feedback",
  "input_tokens": 1500,
  "output_tokens": 800,
  "cost_usd": 0.042,
  "status": "success|warning|error",
  "metrics": {
    "latency_ms": 2500,
    "cost_per_token": 0.00003,
    "quality_score": 0.92
  },
  "escalation": {
    "required": false,
    "reason": "if_any"
  }
}
```

### 監視指標（KPI）

```yaml
metrics:
  latency:
    p50: "< 5s"
    p95: "< 30s"
    p99: "< 60s"
  
  quality:
    generator_diversity: "> 0.7"      # 提案の多様性スコア
    evaluator_accuracy: "> 0.95"      # 検証の正確性
    overall_success_rate: "> 0.92"    # タスク成功率
  
  cost:
    per_task_target: "< $0.50"
    monthly_budget: "< $10,000"
  
  safety:
    human_escalation_rate: "< 5%"
    security_breach: 0                # ゼロ
    unauthorized_access: 0            # ゼロ
```

## Human-on-the-Loop ポリシー

### エスカレーション条件（自動）

以下の場合は自動的に人間レビューにエスカレーション：

1. **セキュリティ影響** (即座)
   - 権限変更
   - データ削除・修正予定
   - ネットワーク設定変更

2. **コスト閾値超過** (即座)
   - 単一タスク > $100
   - 月間累計 > $9,000

3. **エラーの反復** (自動)
   - 同じエラーが3回以上発生
   - Circuit Breaker 発動

4. **品質不足** (自動)
   - Evaluator スコア < 0.7
   - Generator 提案数 < 2

5. **意思決定の重要性** (手動)
   - ビジネス影響度 = HIGH
   - リスク評価 = CRITICAL

### 人間レビューのタイムライン

```yaml
sla:
  critical_escalation: 5 minutes       # またはタイムアウト自動実行
  high_priority: 30 minutes
  normal_priority: 2 hours
  backlog: 24 hours
```

## Context Optimization（33K Token対応）

### Context Buffer 管理

```yaml
claude_code_april_2026:
  buffer_size_tokens: 33000
  auto_compact_threshold: 26400  # 80% utilization
  prompt_caching_ttl: "1hour"
  
  compress_strategy:
    - method: "/compact"
      frequency: "hourly"
      keep_recent_turns: 10
    - method: "summarize_old_context"
      threshold_turns: 50
    - method: "move_to_file_references"
      for: "code snippets > 500 lines"
```

### プロンプトキャッシング（Cache Hits 最大化）

```yaml
caching:
  enable_prompt_caching_1h: true      # 1時間TTL
  enable_force_prompt_caching_5m: false
  
  cacheable_patterns:
    - system_prompt: "always"
    - tool_definitions: "always"
    - project_context: "always"
    - repeated_queries: "on_match"
  
  cache_hit_optimization:
    - group_similar_tasks: true
    - reuse_context_across_agents: true
    - deduplicate_instructions: true
```

## テスト要件

### ユニットテスト

```yaml
test_coverage:
  generator_skills:
    coverage_minimum: 0.85
    test_types:
      - "hypothesis_diversity_test"
      - "speed_benchmark"
      - "safety_baseline"
  
  evaluator_skills:
    coverage_minimum: 0.90
    test_types:
      - "accuracy_test"
      - "edge_case_robustness"
      - "security_detection_rate"
  
  guide_sensor:
    coverage_minimum: 0.80
    test_types:
      - "prevention_effectiveness"
      - "correction_accuracy"
```

### インテグレーションテスト

```yaml
integration_tests:
  - scenario: "generator_evaluator_loop"
    agents: ["generator", "evaluator"]
    expected: "agreement_rate > 0.90"
  
  - scenario: "guide_sensor_protection"
    inject_hazard: true
    expected: "hazard_prevented_or_corrected"
  
  - scenario: "phase_gating_enforcement"
    agents: ["orchestrator", "actioner"]
    expected: "human_review_triggered"
```

## デプロイメント規約

### 本番へのプロモーション

```yaml
pre_deployment:
  - name: "staging_validation"
    duration_hours: 24
    load_profile: "production_equivalent"
  
  - name: "security_review"
    required_approvers: 2
    scan_tools: ["SAST", "dependency-check", "secret-scanner"]
  
  - name: "performance_baseline"
    latency_p95_target: "< 30s"
    success_rate_target: "> 0.95"
```

### 段階的ロールアウト

```yaml
rollout_phases:
  phase_1:
    percentage: 5
    duration_hours: 4
    metric_gate: "error_rate < 2%"
  
  phase_2:
    percentage: 25
    duration_hours: 8
    metric_gate: "error_rate < 1%"
  
  phase_3:
    percentage: 100
    metric_gate: "all_slos_met"

rollback_plan: "enabled"
rollback_trigger: "error_rate > 5% OR latency_p95 > 60s"
```

## チェックリスト：新規エージェント追加時

- [ ] エージェント実装（agents/ ディレクトリ）
- [ ] Generator/Evaluator パターンの採用確認
- [ ] Guide/Sensor メカニズムの組み込み
- [ ] テストカバレッジ（最低80%）
- [ ] プロンプト定義が明示的か確認
- [ ] ガードレール設定（権限、出力検証）
- [ ] ドキュメント更新
- [ ] セキュリティレビュー（秘密情報リーク検査）
- [ ] パフォーマンステスト（レイテンシ、コスト）
- [ ] PR提出 & 最低2名のレビュー待機

## FAQ

**Q: Generator と Evaluator、どちらが先に実行？**  
A: Generator が先。Evaluator が提案を検査。フィードバックは Generator の次回試行に反映。

**Q: Guide と Sensor の違いは？**  
A: Guide は事前制御（実行前）。Sensor は事後制御（実行後）。両者で安全性と自律性を両立。

**Q: 3-5エージェント という制限理由は？**  
A: エージェント間通信・調整コストが指数増加。超大規模スワームは制御不能。

**Q: Context Buffer 33K で十分？**  
A: Sub-Agent による文脈隔離、プロンプトキャッシング、定期的な `/compact` で対応可能。

---

**レビュー対象**: 全ファイル変更時のPRで確認必須

