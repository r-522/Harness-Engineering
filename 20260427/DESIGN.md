# DESIGN.md - ハーネスアーキテクチャ設計原則

このドキュメントは、AIエージェント実行環境（ハーネス）の設計原則と、2026年のアーキテクチャベストプラクティスを規定します。

## 基本設計原則

### 1. 決定性とフェイルセーフ性（CAAF - Convergent AI Agent Framework）
```
生成（Generation）
  ↓
原子的分解（Atomic Decomposition）
  ↓
機械可読レジストリ（Machine-Readable Registry）
  ↓
検証・フィードバック（Verification Loop）
  ↓
決定的アウトプット（Deterministic Output）
```

**原則**:
- エージェント出力は**常に検証可能**な形式
- ツールチェーンは**トレーサブル**であること
- フェイルセーフ分岐は明示的に定義
- 不確実性は数値化・可視化

### 2. コンテキストこそが最高の最適化変数
```
モデル性能  < ハーネス設計
5%差       5-10%差
```

**推奨構成**:
- システムプロンプト容量: 全コンテキストの15-20%
- 入力タスク: 5-10%
- ワーキングメモリ: 30-40%
- 予約領域（キャッシュ, レジストリ）: 20-30%

### 3. 環境工学（Environment Engineering）
AIが理解しやすいコードベース設計が新しい競争優位性。

**適用例**:
```python
# ❌ AI可読性低い
def process_data(d, f):
    return [x*2 for x in d if f(x)]

# ✓ AI可読性高い
def filter_and_double_values(
    values: list[int],
    predicate: Callable[[int], bool]
) -> list[int]:
    """
    Filters values by predicate, then doubles each.
    """
    filtered = [v for v in values if predicate(v)]
    return [v * 2 for v in filtered]
```

**設計チェックリスト**:
- [ ] 関数名は実行内容を完全に表現
- [ ] 型ヒント / ドックストリング完備
- [ ] 副作用は明示的に記述
- [ ] エラーケースは明示化
- [ ] APIドキュメントは機械解析可能

## ハーネスレイヤー設計

### Layer 1: Core LLM Interface
```
┌──────────────────────────┐
│  Claude Model (API)      │
│  (Opus 4.7 / Sonnet 4.6) │
└────────────┬─────────────┘
             │
        (Token stream)
```

**責務**: 推論エンジン  
**制約**: なし（ステートレス）

### Layer 2: Tool Orchestration
```
┌────────────────────────────────┐
│ Tool Coordinator               │
│ - Tool Selection Logic         │
│ - Parallel Execution           │
│ - Error Recovery               │
└────────┬───────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
 [Tools]  [Tool Registry]
```

**責務**: ツール選択、実行、結果統合  
**制約**: ツール間の依存関係を解決

### Layer 3: State Management
```
┌─────────────────────────────┐
│ State Manager               │
│ - Conversation History      │
│ - Session State             │
│ - Knowledge Base            │
│ - Cache Control             │
└─────────────────────────────┘
```

**責務**: 状態永続化、キャッシング  
**制約**: TTL管理（60分キャッシュ標準）

### Layer 4: Monitoring & Observability
```
┌──────────────────────────────┐
│ Observability Layer          │
│ - Token Usage Tracking       │
│ - Latency Monitoring         │
│ - Error Logging              │
│ - Cost Accounting            │
└──────────────────────────────┘
```

**責務**: パフォーマンス監視、コスト追跡  
**制約**: オーバーヘッド < 5%

## クラウドネイティブ展開アーキテクチャ

### マイクロサービス分割

```
┌──────────────────────────────────────────┐
│         API Gateway / Router             │
└────────────────┬───────────────────────────┘
                 │
    ┌────────┬───┼───┬────────┐
    ▼        ▼   ▼   ▼        ▼
┌────┐ ┌────┐ ┌──┐ ┌─────┐ ┌──────┐
│ CA │ │ SR │ │PO│ │ MAO │ │Infra │
└────┘ └────┘ └──┘ └─────┘ └──────┘
  (CodeAnalyst)
  (SecurityReviewer)
  (PerformanceOptimizer)
  (MultiAgentOrchestrator)
  (InfrastructureManager)

各マイクロサービス:
- 独立したコンテキストウィンドウ
- 専用スケーリング設定
- ヘルスチェック + サーキットブレーカ
```

### スケーリング戦略

```yaml
水平スケーリング:
  - CodeAnalyst: 高I/O, CPU少 → Pod数増加
  - PerformanceOptimizer: CPU集約 → 計算リソース増加
  - SecurityReviewer: バランス型 → 標準スケーリング

垂直スケーリング:
  - メモリ制約: 大規模ファイル分析時
  - コンテキスト不足: Agent委譲 + 再実行
```

## プロンプト&キャッシング戦略

### キャッシング層構造

```
┌──────────────────────────────┐
│ L1 Cache (Request内)         │ ← すべてのリクエスト
├──────────────────────────────┤
│ L2 Cache (60分TTL)           │ ← システムプロンプト
│ - Stable System Prompt       │
│ - Common Knowledge Base      │
├──────────────────────────────┤
│ L3 Cache (Session)           │ ← セッションコンテキスト
│ - Conversation History       │
│ - User Profile               │
├──────────────────────────────┤
│ L4 Cache (External Store)    │ ← 永続ストア
│ - Shared Knowledge (Redis)   │
│ - Temporal Data (DB)         │
└──────────────────────────────┘
```

### キャッシング実装チェックリスト

```yaml
システムプロンプト:
  - 動的部分を外部化（タイムスタンプ等）
  - 変更頻度: 月1回以下が目安
  - キャッシュ対象化 YES
  
ユーザー入力:
  - 毎回変わる → キャッシュ不可
  - コンテキスト内で繰り返し参照 → L1キャッシュ
  
リファレンス資料:
  - ドキュメント、コードリファレンス → L2キャッシュ
  - 更新頻度: 週1回以上なら再評価
  
セッション状態:
  - 会話履歴 → L3キャッシュ
  - TTL: 24時間
```

### プロンプトキャッシング設定例

```json
{
  "messages": [
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "[System Prompt - Cached]",
          "cache_control": {"type": "ephemeral"}
        },
        {
          "type": "text",
          "text": "[Dynamic Input]"
        }
      ]
    }
  ],
  "cache_control": {
    "max_tokens_to_evict": 0,
    "min_cache_creation_input_tokens": 1024,
    "ttl_seconds": 3600
  }
}
```

## セキュリティアーキテクチャ

### 防御層（Defense-in-Depth）

```
┌─────────────────────────────────────┐
│ Layer 1: Input Validation           │
│ - Type checking, Length limits      │
│ - Allowlist-based filtering         │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ Layer 2: Tool Sandbox               │
│ - Capability-based Security         │
│ - Permission Model (explicit grant) │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ Layer 3: Output Sanitization        │
│ - Injection Prevention               │
│ - PII Masking                       │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│ Layer 4: Audit & Logging            │
│ - Action Tracking                   │
│ - Anomaly Detection                 │
└─────────────────────────────────────┘
```

### OWASP Top 10 への対策

| Top 10 | 対策 | 実装チェック |
|--------|------|-----------|
| A1: Injection | パラメータ化・エスケープ | SQLAlchemy ORM / Parameterized Queries |
| A2: Broken Auth | 認証・認可の厳密化 | OAuth 2.0 / JWT |
| A3: Sensitive Data | 暗号化・マスキング | AES-256 / PII detection |
| A4: XML External Entities | XML解析の制限 | XML library config |
| A5: Broken Access Control | 権限チェック | RBAC/ABAC |
| A6: Security Misconfiguration | 定期監査 | Automated scanning |
| A7: XSS | Output encoding | Context-aware encoding |
| A8: Insecure Deserialization | Trusted input only | Type validation |
| A9: Using Vulnerable Components | Dependency audit | Trivy / Snyk |
| A10: Logging & Monitoring | 監査ログ | ELK stack / Splunk |

## パフォーマンス最適化ガイドライン

### レイテンシ最適化

```
┌─────────────┬────────────┐
│ 最適化施策   │ 期待削減  │
├─────────────┼────────────┤
│ プロンプトキャッシング │ 60% |
│ 並列実行  │ 40% |
│ コンテキスト圧縮 │ 30% |
│ 非同期ストリーミング │ 50% |
│ ロードバランシング │ 25% |
└─────────────┴────────────┘
```

### コスト最適化

```
施策1: キャッシング活用
  基本: 毎リクエスト 1,000トークン × $15/1M = $0.015
  キャッシュ後: 100トークン × $1.5/1M = $0.00015
  削減: 99%

施策2: モデル選択の最適化
  Opus（複雑）: $15/1M input
  Sonnet（バランス）: $3/1M input
  Haiku（軽量）: $0.80/1M input
  適切に使い分けで 70% コスト削減

施策3: コンテキスト利用率最適化
  充填率 80% → 平均 40% への改善
  トークン削減: 50%
```

## 監視・可観測性（Observability）

### メトリクス定義

```yaml
Primary Metrics:
  - P99 Latency: < 5秒 (標準)
  - Token Efficiency: > 70%
  - Cache Hit Rate: > 60%
  - Error Rate: < 0.5%

Secondary Metrics:
  - Cost per Request: トレンド監視
  - Agent Success Rate: > 95%
  - Context Utilization: 20-60% (最適)
```

### アラート設定例

```yaml
アラート1: Cache Hit Rate低下
  threshold: < 40%
  duration: 15分
  action: キャッシュ戦略見直し

アラート2: P99 Latency増加
  threshold: > 10秒
  duration: 5分
  action: スケーリング検討

アラート3: エラーレート上昇
  threshold: > 2%
  duration: 5分
  action: インシデント対応
```

---

**バージョン**: 1.0  
**最終更新**: 2026-04-27
