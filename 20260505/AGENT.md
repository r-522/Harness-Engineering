# AGENT.md - エージェント振る舞い定義

**対象**: Claude Subagent Orchestration Layer  
**対応モデル**: Claude Opus 4.7 (Orchestrator), Claude Sonnet 4.6 (Worker)  
**最終更新**: 2026-05-05

---

## 1. エージェントペルソナ

### Orchestrator Agent (主指揮官)

**役割**: タスク分解、Subagent編成、結果統合  
**思考スタイル**: 戦略的・分析的、トップダウン  
**トーン**: 指示的だが柔軟、コンテキスト喪失時は再確認  
**モデル**: Claude Opus 4.7 (最高推論能力)  
**コンテキスト予算**: 100K トークン（Subagent 3並列 × 30K）

### Worker Subagent (実行エージェント)

**役割**: 単一タスク実行、ツール利用、レポート作成  
**思考スタイル**: 実行志向・効率重視、ボトムアップ  
**トーン**: タスク中心、結果報告明確  
**モデル**: Claude Sonnet 4.6 (コスト最適化)  
**コンテキスト予算**: 30K トークン (Orchestratorからの指示 + 作業)

---

## 2. 思考プロセス (ReAct + Planning)

### フェーズ1: Plan (計画)

```
User Input
    ↓
[Orchestrator: Plan mode]
- タスク解析: 複雑度、依存関係判定
- 分解: 独立可能なサブタスクに分割
- 割り当て: Subagent種別を決定
- スケジューリング: 並列 vs 直列判定
    ↓
Plan Document (ユーザ確認待ち)
```

### フェーズ2: Act (実行)

```
Approved Plan
    ↓
[Orchestrator: Execution]
- Subagent Spawn (max 3並列)
- 監視: Context予算、エラー検出
- 調整: リアルタイム並列度調整
    ↓
[Worker Subagents: 並列実行]
- Tool Use (MCP統合)
- 部分結果キャッシュ
- エラーハンドリング (再試行 or Escalate)
```

### フェーズ3: Verify (検証)

```
Partial Results Collection
    ↓
[Orchestrator: Integration]
- 結果統合: データ整合性確認
- 品質検査: 期待値との照合
- コスト分析: Token消費量評価
    ↓
Final Output + Metadata Report
```

---

## 3. Subagent定義 & トリガーマッピング

### 3.1 リサーチ型 Subagent (`researcher`)

```yaml
name: researcher_parallel
trigger: 複数の独立した情報収集タスク
max_instances: 3
model: sonnet-4.6
tools:
  - WebSearch
  - WebFetch
context_budget: 25K tokens
timeout: 300s
error_strategy: retry_with_fallback
```

**実装例**:
```markdown
## Subagent: researcher_product_features
Trigger: 競合製品5社の機能比較タスク
Orchestrator: 5つの「製品X機能分析」タスクを作成
Worker: 各社の公式ドキュメント解析 → 構造化レポート返却
Merge: 統合比較表として集約
```

### 3.2 コード実行型 Subagent (`executor`)

```yaml
name: executor_testing
trigger: 複数ファイルのテスト実行 or ビルド並列化
max_instances: 2
model: sonnet-4.6
tools:
  - Bash
  - Read
  - Edit
context_budget: 30K tokens
timeout: 600s
error_strategy: fail_fast
```

**実装例**:
```markdown
## Subagent: executor_unit_test
Trigger: pytest.ini で定義された複数テストスイート
Distribution: 各テストディレクトリごとに並列化
Merge: JUnit XML 統合 → カバレッジレポート生成
```

### 3.3 分析型 Subagent (`analyzer`)

```yaml
name: analyzer_parallel_review
trigger: 大規模コード変更の複数セクション分析
max_instances: 3
model: sonnet-4.6
tools:
  - Read
  - Bash (grep, find)
context_budget: 28K tokens
timeout: 180s
error_strategy: skip_and_report
```

---

## 4. エラーハンドリング & エスカレーション方針

### エラー分類と対応

| エラー型 | 原因 | 対応 | 決定者 |
|---|---|---|---|
| **Context枯渇** | Subagent単体で>30K消費 | タスク分割 | Orchestrator自動判定 |
| **Tool失敗** | API限界、ネットワーク | 指数バックオフ (最大3回) | Subagent自動リトライ |
| **データ不整合** | Subagent間の結果矛盾 | Manual Verify or Rollback | Orchestrator確認 |
| **権限不足** | Tool実行拒否 | ユーザ許可プロンプト | UI (ユーザ手動) |

### エスカレーションフロー

```
Subagent Error
    ├─ Recoverable? (自動リトライ可能か)
    │  ├─ YES → 指数バックオフ x3 回
    │  └─ NO ↓
    ├─ Critical? (システム停止に直結するか)
    │  ├─ YES → Orchestrator即停止 & ユーザ通知
    │  └─ NO ↓
    └─ Escalate to User with Recommendation
       (代替案提示、手動介入オプション)
```

---

## 5. Context Window 最適化

### 予算配分（例: 100K token）

```
Orchestrator: 40K (タスク分解、統合、制御)
  ├─ 初期システムプロンプト: 2K
  ├─ チーム共有ルール(.claude/): 3K
  ├─ 実行履歴・監視: 15K
  └─ Subagent制御指示: 20K

Worker A (Subagent 1): 20K (独立タスク A)
Worker B (Subagent 2): 20K (独立タスク B)
Worker C (Subagent 3): 20K (独立タスク C)
```

### 圧縮戦略

1. **チーム規約分離**: 大量ドキュメントは `.claude.local.md` へ移動
2. **結果キャッシング**: Subagentの部分結果をファイルに永続化
3. **プロンプト圧縮**: 構造化YAML形式で指示を簡潔化
4. **世代管理**: 古い実行履歴は削除（保持期間: 最新5回分）

---

## 6. パフォーマンス指標

### Orchestration Success Metrics

| 指標 | 目標値 | 測定方法 |
|---|---|---|
| **Subagent成功率** | ≥ 95% | (成功数 / 送信数) |
| **平均応答時間** | ≤ 120秒 | Wall-clock 計測 |
| **Token効率** | ≥ 80% | (有効出力token / 入力token) |
| **Context予算消費** | ≤ 85% | (使用 / 割り当て) |
| **コスト削減率** | ≥ 40%* | (Opus単独 vs Orch+多Sonnet) |

*Sonnet使用による削減見積り（参考値）

### 失敗パターン分析

```bash
# Orchestration実行後、自動ログ出力
[orchestration-metrics]
- parallel_subagents: 3
- successful: 3
- failed: 0
- avg_context_used: 24.5K (budget: 30K)
- total_wall_time: 45.3s
- cost_estimate: $0.23 (Opus-only: $1.04)
```

---

## 7. 禁止事項・安全ガード

### 🚫 Subagent運用での禁止事項

- [ ] 最大並列数超過 (>10)
- [ ] Context残量 < 10% での新Subagent起動
- [ ] APIキー・認証情報のSubagent送信
- [ ] Subagent間の無制御データ共有（必ず Orchestrator経由）
- [ ] 無限再試行ループ（最大3回で打ち切り）

### ✅ 安全実装パターン

```python
# 例: 安全なSubagent並列化
orchestrator_safe = {
    "max_parallel": min(3, available_slots),
    "context_limit_per_agent": 30_000,
    "timeout_seconds": 300,
    "retry_policy": {
        "max_attempts": 3,
        "backoff_multiplier": 2,
        "max_backoff_seconds": 16
    },
    "escalation_threshold": 0.8  # Context 80% で警告
}
```

---

## 8. Plan Mode フロー（ユーザ確認）

### Orchestrator が Plan を提示する際の要素

```markdown
# Execution Plan (ユーザ確認待ち)

## Task Analysis
- **Complexity**: 中 (3つの独立タスク)
- **Est. Duration**: 45秒
- **Est. Cost**: $0.18

## Proposed Subagent Distribution
1. **researcher_A**: 製品X機能分析 (20K ctx)
2. **researcher_B**: 競合Y機能分析 (20K ctx)
3. **executor_C**: ベンチマーク実行 (20K ctx)

## Execution Strategy
- **Parallel**: A, B, C 同時起動
- **Merge**: 結果を統合表に集約
- **Fallback**: A失敗時は手動調査へ

## Risks & Mitigations
- ⚠️ API Rate Limit (WebSearch × 2) → スロットリング
- ✓ Context枯渇 → Orchestrator 40K確保で対応

**Proceed?** [Y/n]
```

---

## 9. 参考・拡張リソース

- **Orchestration Patterns**: `DESIGN.md` / アーキテクチャセクション
- **スキル定義**: `SKILLS.md`
- **MCP統合**: `MCP-INTEGRATION.md`
- **Team CLAUDE.md**: `.claude/CLAUDE.md`
