# DESIGN.md - 設計原則とアーキテクチャガイドライン

このファイルはマルチエージェントシステムの設計原則、アーキテクチャパターン、コンポーネント間の関係を定義します。

**最終更新**: 2026-04-25  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

---

## 基本設計原則

### 1. **Separation of Concerns（関心の分離）**

各エージェントは単一の責任を持つ：

```
Generator:    提案生成に専念（品質判定は Evaluator に任せる）
Evaluator:    品質検証に専念（新規提案生成は Generator に任せる）
Orchestrator: 統合・計画に専念（詳細実装は各エージェントに任せる）
```

**利点**: エージェント設計がシンプル、テストが容易、変更影響が局所化

### 2. **Bounded Workflows（境界を持つワークフロー）**

```
定義済み段階 → 人間レビューポイント → 次段階
```

各段階は：
- **明確な入力**: 何を受け取るか
- **明確な出力**: 何を生産するか
- **明確な終了条件**: いつ完了と見なすか

**禁止事項**:
- 無限ループ / 開放末端ワークフロー
- エージェント数の動的増加（スワーム拡大）
- 人間介入なしの連続自動実行

### 3. **Fail-Safe by Default（デフォルト安全）**

エージェントの行動は、明示的な許可がない限り**実行不可**：

```
Default: ✗ 実行禁止
With approval: ✓ 実行可能
```

### 4. **Context Discipline（文脈規律）**

- Sub-Agent は親エージェントの文脈を継承しない
- 各エージェントは必要最小限の文脈のみ参照
- 33Kトークンバッファを超えない設計

---

## アーキテクチャパターン

### パターン 1: Generator/Evaluator Loop（標準）

```
User Request
    ↓
Orchestrator (タスク分解)
    ↓
Generator (複数提案生成)
    ↓
Evaluator (品質検証)
    ↓
[人間判断] ← GO / CAUTION / NO-GO
    ↓
Actioner (実行)
    ↓
Sensor (事後観測・補正)
    ↓
Result
```

**使用タイミング**: 大多数のビジネスロジックタスク

**工数**: Generator 60秒 + Evaluator 90秒 = 約 3分

**品質**: 高精度（Generator多様性 + Evaluator検証）

### パターン 2: Fast-Path（高速パス）

```
User Request (明確で低リスク)
    ↓
Orchestrator (判定: 低リスクか？)
    ↓ YES
Fast-Path: Generator のみ
    ↓
[自動GO]
    ↓
Actioner
```

**使用タイミング**: 高頻度・低リスク・定型タスク（例：ログ取得）

**工数**: Generator 60秒

**品質**: 中程度（検証スキップ）

### パターン 3: High-Risk Path（高リスク経路）

```
User Request (高リスク・重要決定)
    ↓
Orchestrator + Guide
    ↓
Generator (制約条件下で提案)
    ↓
Evaluator (厳密検証)
    ↓
[強化人間レビュー] ← 複数承認者
    ↓
Validator (再度セキュリティ確認)
    ↓
Sensor (プリ実行チェック)
    ↓
Actioner (段階的実行)
    ↓
Sensor (ポスト実行検証)
    ↓
Result
```

**使用タイミング**: 権限変更、データ削除、システム設定変更

**工数**: 約 30分（人間レビュー含む）

**品質**: 最高精度（複重検証）

---

## 設計パターン：Guide & Sensor

### Guide（事前制御）の設計

```
Generator が実行される前に、潜在的リスクを予測して回避。

実装パターン:

1. Guide テンプレート実行
   ↓
2. リスク行動パターンを予測
   ↓
3. Generator にプロンプト追加
   ↓
4. Generator が制約条件下で提案生成
```

**実装例**:

```yaml
guide_pre_generation:
  task: "新規ユーザー作成スクリプトを生成"
  
  predicted_risks:
    - risk_id: "hardcoded_credentials"
      description: "パスワードをハードコードしてしまう可能性"
      mitigation: "環境変数使用を Generator に明示指示"
    
    - risk_id: "permission_escalation"
      description: "不要な高権限でスクリプト作成"
      mitigation: "最小権限原則を Generator にリマインド"
    
    - risk_id: "infinite_loop"
      description: "エラーハンドルなしの無限再試行"
      mitigation: "max_retries=3 を Generator に強制
  
  generator_prompt_addition: |
    追加制約条件:
    ✓ DO: 環境変数を使用
    ✓ DO: 最小権限で実行
    ✓ DO: max_retries は3に設定
    ❌ DON'T: パスワードをハードコード
    ❌ DON'T: 高権限で実行
    ❌ DON'T: 無限ループを許容
```

### Sensor（事後制御）の設計

```
Actioner が実行後、予期しない結果や副作用を検出・補正。

実装パターン:

1. 実行ログ収集
   ↓
2. 期待値との比較
   ↓
3. 異常検知（統計的 or ルールベース）
   ↓
4. 自動補正 OR エスカレーション
```

**実装例**:

```yaml
sensor_post_execution:
  executed_action: "新規ユーザー作成スクリプト実行"
  
  monitoring_rules:
    - metric: "api_calls"
      expected: "< 10"
      actual: 50
      status: "ANOMALY_DETECTED"
      action: "self_correct"
      
    - metric: "permission_changes"
      expected: "user_read"
      actual: "system_admin"
      status: "CRITICAL_ANOMALY"
      action: "escalate_to_human"
    
    - metric: "execution_time"
      expected: "< 5s"
      actual: "300s"
      status: "WARNING"
      action: "investigate"
  
  self_correction:
    - anomaly: "過剰な API 呼び出し"
      action: "キャッシュ有効化して再実行"
      success_rate: 0.7
    
    - anomaly: "権限逸脱"
      action: "不可逆的 → エスカレーション"
      success_rate: 0.0  # 自動補正不可
```

---

## Model Context Protocol (MCP) 統合

### MCP の役割

外部システム（GitHub、Database、Slack等）との安全で監視可能な接続。

### MCP 統合ガイドライン

```yaml
mcp_integration:
  principle: "エージェントは MCP 経由でのみ外部システムにアクセス"
  
  github_integration:
    capability: "PR レビュー、Issue 処理"
    authentication: "OAuth2 + Service Account"
    rate_limit: 60 requests/min
    audit: "全操作ログ記録"
  
  database_integration:
    capability: "SELECT / UPDATE / INSERT / DELETE"
    authentication: "Service Account with minimal privilege"
    timeout: "30 seconds"
    query_size_limit: "1MB"
  
  slack_integration:
    capability: "メッセージ送信、ステータス更新"
    rate_limit: "30 messages/min"
    content_filter: "sensitive_data_block"
```

---

## Context Buffer 最適化（33K Token）

### コンテキスト使用パターン

```
Total Buffer: 33,000 tokens

System Prompt:           2,000 tokens
Tool Definitions:        3,000 tokens
Project Context:         5,000 tokens
Recent Conversation:    15,000 tokens
Sub-Agent Results:       5,000 tokens
Available for Query:     3,000 tokens
────────────────────────────────
TOTAL:                  33,000 tokens
```

### コンテキスト節約テクニック

#### 1. Sub-Agent による隔離

```
Dispatcher Agent (メインコンテキスト: 25K tokens)
    ├─ Sub-Agent A (独立コンテキスト)
    │   └─ 完了 → 結果サマリー（500 tokens）を Dispatcher に返す
    │
    └─ Sub-Agent B (独立コンテキスト)
        └─ 完了 → 結果サマリー（500 tokens）を Dispatcher に返す

効果: Sub-Agent の詳細処理が Dispatcher バッファを圧迫しない
```

#### 2. プロンプトキャッシング（1時間 TTL）

```yaml
caching_strategy:
  enable_prompt_caching: true
  ttl: "1 hour"
  
  always_cache:
    - system_prompt: true
    - project_context: true
    - tool_definitions: true
    - safety_guidelines: true
  
  cache_hit_benefits:
    - reduced_input_tokens: "60% fewer"
    - lower_cost: "50% discount on cached tokens"
    - faster_response: "25% faster"
```

#### 3. `/compact` コマンド

```bash
# 冗長なコンテキストを自動圧縮
/compact

# 効果:
# - 古い対話を要約に置き換え
# - 重複情報を削除
# - 前置きを簡潔に

# 使用タイミング:
# - 対話ターン数 > 50
# - バッファ使用率 > 80%
# - Agent切り替え前
```

---

## エージェント間の協調パターン

### パターン A: シーケンシャル（順序実行）

```
Generator
    ↓ (結果を渡す)
Evaluator
    ↓ (結果を渡す)
Actioner
```

**特徴**: シンプル、デバッグ容易、遅い

**使用**: 標準的なワークフロー

### パターン B: 階層型（Top-Down）

```
        Orchestrator
       /      |      \
   Gen    Eval    Plan
    |      |       |
   Result Results Results
       \      |      /
       Result Synthesis
```

**特徴**: スケーラブル、並列度高い

**使用**: 複雑なタスク分解

### パターン C: イベント駆動

```
Generator 完了
    ↓
Event: "generation_complete"
    ↓
Evaluator リッスン → 自動起動
    ↓
Event: "evaluation_complete"
    ↓
Actioner リッスン → 自動起動
```

**特徴**: 非同期、疎結合

**使用**: リアルタイム要件が高い場合

### パターン D: Peer-to-Peer（相互参照）

```
Generator ← → Evaluator
    ↑         ↓
    ← → Orchestrator
```

**特徴**: 柔軟、複雑、高オーバーヘッド

**使用**: 相互レビューが必要な場合

---

## スケーリング戦略

### Vertical Scaling（エージェント強化）

```
Generator v1 → Generator v2
- モデル: Sonnet 4.6 → Opus 4.7
- 提案数: 3 → 5
- タイムアウト: 60s → 90s
```

### Horizontal Scaling（エージェント追加）

```
推奨: 3-5 エージェント（Orchestrator + Generator + Evaluator + 1-2 Sub）

警告: > 5 エージェントはコーディネーション複雑度が指数増加
```

### 避けるべきスケーリング

```
✗ スワーム型（50+ エージェント）
✗ 動的エージェント生成
✗ 無制限の再試行ループ
```

---

## 監視とオブザーバビリティ

### 監視ポイント

```
Orchestrator:
  - タスク分解時間
  - エージェント選択精度
  
Generator:
  - 提案多様性スコア
  - 実行時間（目標: < 60s）
  
Evaluator:
  - 検証精度（偽陽性/偽陰性率）
  - 実行時間（目標: < 90s）
  
Actioner:
  - 成功率
  - ロールバック頻度
  
Sensor:
  - 異常検知精度
  - 自動補正成功率
```

### ダッシュボード例

```
Real-time Metrics:
┌─────────────────────────────────────┐
│ Agent Health                         │
│ ┌─────┬────────┬──────┬────────┐   │
│ │ Gen │ Eval   │ Act  │ Sensor │   │
│ │ 99% │  98%   │ 97%  │  96%   │   │
│ └─────┴────────┴──────┴────────┘   │
├─────────────────────────────────────┤
│ Cost & Performance                  │
│ Avg Latency: 180s | Cost: $0.34    │
├─────────────────────────────────────┤
│ Human Escalation Rate: 4.2%         │
└─────────────────────────────────────┘
```

---

## トラブルシューティングガイド

### Issue: Generator が不適切な案を生成

**原因**: プロンプトが曖昧

**対策**:
1. CLAUDE.md の「明示性」ルールを確認
2. Guide を強化：避けるべき行動を明示
3. Generator のプロンプトを具体化

### Issue: Evaluator が過度に保守的

**原因**: リスク閾値が低すぎる

**対策**:
1. Evaluator プロンプト見直し
2. リスク基準を明文化（ローカル/本番別）
3. 段階的に閾値を上げ

### Issue: Context Buffer が枯渇（33Kを超過）

**原因**: Sub-Agent結果が大きすぎる

**対策**:
1. Sub-Agent の出力をサマライズ
2. `/compact` でコンテキスト圧縮
3. ファイル参照に変更

---

## デプロイメント検査リスト

本番環境へのプロモーション前に以下を確認：

```yaml
pre_deployment_checklist:
  - name: "Architecture Review"
    items:
      - [ ] Generator/Evaluator 分離が明確か
      - [ ] Guide/Sensor メカニズムが統合か
      - [ ] Phase-Gating が実装されているか
      - [ ] Sub-Agent による文脈隔離が有効か
  
  - name: "Context Management"
    items:
      - [ ] コンテキスト使用量 < 33K tokens
      - [ ] プロンプトキャッシング有効か
      - [ ] `/compact` 戦略が定義されているか
  
  - name: "Monitoring"
    items:
      - [ ] ダッシュボード整備完了
      - [ ] アラート設定完了（エラー率 > 5%）
      - [ ] 監査ログ記録確認
```

---

**最終更新**: 2026-04-25  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

