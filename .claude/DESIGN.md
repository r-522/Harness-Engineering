# DESIGN.md - 設計原則 & アーキテクチャ

**対象**: AIエージェント・オーケストレーション設計  
**基準年**: 2026-05  
**対応パターン**: 8 Canonical AI Agent Patterns (参考: Monday.com AI Agent Architecture)

---

## 1. 適用設計原則

### SOLID 原則（エージェント拡張性）

| 原則 | 適用内容 |
|---|---|
| **S** - Single Responsibility | 各Subagent は単一の責務（研究・実行・分析） |
| **O** - Open/Closed | 新Subagent追加で機能拡張、既存は変更しない |
| **L** - Liskov Substitution | 任意のWorkerはOrchestratorの指示に従える |
| **I** - Interface Segregation | Tool定義は機能グループ化（WebSearch, Bash等） |
| **D** - Dependency Inversion | Subagent は抽象的な Task に依存 |

### DRY (Don't Repeat Yourself)

```markdown
❌ 反例: 各Subagentが同じAPI呼び出しロジックを記述
✅ 推奨: MCP サーバで統一 & スキル経由で共有
```

**実装**: `MCP-INTEGRATION.md` の「スキルライブラリ」セクション参照

### KISS (Keep It Simple, Stupid)

```markdown
❌ 過度な最適化: 10並列、複雑なデータマージロジック
✅ 推奨: デフォルト3並列、単純な結果集約
```

---

## 2. アーキテクチャパターン (2026年8標準)

Claude Code + Subagent環境での適用:

### Pattern 1: ReAct (Reasoning + Acting)

**シーン**: 単一エージェント、逐次タスク  
**Flow**: Thought → Action → Observation → ... → Final Answer

```
例: ドキュメント読み込み → 分析 → レポート作成
Orchestrator直列処理 (Subagent不要)
```

### Pattern 2: Tool Use

**シーン**: MCP統合、外部API活用  
**Flow**: Plan → Tool Selection → Execute → Observe

```
例: WebSearch + Read + 分析
Subagent内で Tool Use → Orchestratorが結果統合
```

### Pattern 3: Planning (Split-and-Merge)

**シーン**: 複雑な複合タスク、並列実行可能  
**Flow**: 計画 → サブタスク分配 → 並列実行 → マージ

```
例: 5つの競合製品を並列分析 → 比較表生成
Orchestrator: 5つのリサーチタスク生成
3つのWorker: 並列実行
Orchestrator: 結果マージ
```

### Pattern 4: Multi-Agent Collaboration

**シーン**: エージェント間の協調が必要  
**Flow**: 共有タスク リスト → 各agent自律的に実行 → 状態同期

```
例: 大規模コードベース複数セクション分析
各analyzer: 割り当てセクションを分析
Orchestrator: 進捗監視 & 結果集約
```

### Pattern 5: Orchestrator-Worker (推奨)

**シーン**: 階層的な制御構造  
**Flow**: 中央指揮 → 実行者 → 結果上申

```
本構成の主流パターン
Orchestrator: タスク分解、スケジューリング、統合
Workers: 実行、報告
推奨使用シーン: 90%
```

### Pattern 6: Reflection

**シーン**: 結果評価・改善ループ  
**Flow**: 実行 → 評価 → 改善 → 再実行

```
例: テスト失敗 → 原因分析 → コード修正 → 再テスト
Orchestrator: 実行 → 結果品質判定
Worker: 修正 → 再実行
```

### Pattern 7: Evaluator-Optimizer

**シーン**: 品質保証、パフォーマンス最適化  
**Flow**: 実行 → 評価 → 最適化レコメンデーション → 実行

```
例: Subagent実行後、パフォーマンス評価 → 並列度調整
Evaluator: コスト分析、Token効率測定
Optimizer: 次回実行戦略提案
```

### Pattern 8: Collaborative Multi-Agent Teams

**シーン**: 完全分散、チーム的実行  
**Flow**: 共有goal → agent自律決定 → 結果同期

```
例: システム監視エージェント複数が並列監視
各agent: 割り当て領域を監視
共有状態: アラート キューイング
※ Orchestrator-Workerより複雑、小規模チームは推奨外
```

---

## 3. ディレクトリ構成ガイド

```
src/
├── agents/
│   ├── orchestrator.py (主指揮官)
│   ├── researcher.py (リサーチWorker)
│   ├── executor.py (実行Worker)
│   ├── analyzer.py (分析Worker)
│   └── __init__.py
├── skills/
│   ├── web_research.py (Web検索スキル)
│   ├── code_analysis.py (コード分析スキル)
│   ├── report_generation.py (レポート生成)
│   └── __init__.py
├── mcp/
│   ├── server_registry.py (MCP サーバ管理)
│   ├── tool_mapping.py (Tool ↔ Skill マッピング)
│   └── __init__.py
├── core/
│   ├── task_queue.py (Task定義・キューイング)
│   ├── context_manager.py (Context予算管理)
│   ├── error_handler.py (エラーハンドリング)
│   └── metrics.py (パフォーマンス計測)
└── utils/
    ├── logging.py
    └── config.py

tests/
├── agents/
│   ├── test_orchestrator.py
│   ├── test_workers.py
│   └── test_subagent_spawn.py
├── integration/
│   ├── test_mcp_connection.py
│   ├── test_skill_execution.py
│   └── test_parallel_execution.py
└── fixtures/
    ├── sample_tasks.yaml
    ├── mock_mcp_servers.py
    └── test_data.json
```

---

## 4. コンポーネント間の責務分離

### 責務マトリクス

| コンポーネント | 責務 | 所有者 | Context予算 |
|---|---|---|---|
| **Orchestrator** | タスク分解、Subagent起動、結果統合 | Agent Layer | 40K |
| **Researcher Worker** | Web検索、ドキュメント解析 | Skill Layer | 20K |
| **Executor Worker** | Code execution、ファイル操作 | Skill Layer | 20K |
| **Analyzer Worker** | コード品質分析、Linting | Skill Layer | 20K |
| **MCP Server** | 外部接続、Tool提供 | Infrastructure | - |
| **Context Manager** | 予算管理、警告発行 | Core Layer | - |

---

## 5. エラーハンドリング・アーキテクチャ

### 層別エラー処理

```
Layer 1: Subagent (Worker)
  → 自動リトライ (max 3回)
  → ログ記録
  → 失敗時は上位へ escalate

Layer 2: Orchestrator (Task管理)
  → 失敗パターン認識
  → リカバリ戦略選択
  → タスク再分配

Layer 3: System (Harness)
  → Context枯渇警告
  → API限界警告
  → ユーザ通知
```

### 失敗サイクル

```python
max_retries = 3
backoff = [1, 2, 4]  # exponential

for attempt in range(max_retries):
    try:
        result = subagent.execute(task)
        return result
    except RecoverableError:
        if attempt < max_retries - 1:
            wait(backoff[attempt])
            continue
        else:
            escalate_to_orchestrator(task)
    except CriticalError:
        escalate_immediately(task)
```

---

## 6. スケーラビリティ考慮事項

### チームサイズ別の構成推奨

| チームサイズ | Subagent最大数 | Context予算 | 推奨パターン |
|---|---|---|---|
| 1～3名 | 3 | 100K | Pattern 5 (Orchestrator-Worker) |
| 4～5名 | 5 | 150K | Pattern 5 + Pattern 4 |
| 6～10名 | 8 | 200K | Pattern 8 (Teams) |
| 10+名 | 独立検討 | 要カスタマイズ | 複合・分散 |

**本構成対象**: 小規模チーム(3～5名) → Pattern 5 推奨

---

## 7. セキュリティ考慮事項

### データ隔離

```
Subagent A ──→ Isolated Context A (認証情報不可)
Subagent B ──→ Isolated Context B (認証情報不可)
Subagent C ──→ Isolated Context C (認証情報不可)
        ↓
  Orchestrator (認証情報 管理 ← .local.md)
        ↓
     MCP Server (セキュアアクセス)
```

### 禁止パターン

```
❌ Subagent ← APIキー（直接送信）
✅ Orchestrator → MCP Server ← APIキー（仲介）
```

---

## 8. 参考アーキテクチャ図

```
┌─────────────────────────────────────────────────────────┐
│                    User Input                            │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────┐
        │   ORCHESTRATOR AGENT (Opus)   │
        │   - Plan, Decompose, Merge    │
        │   - Context Budget: 40K       │
        └───┬───────────┬───────────┬───┘
            │           │           │
      ┌─────▼─┐  ┌─────▼─┐  ┌─────▼─┐
      │Worker │  │Worker │  │Worker │
      │A      │  │B      │  │C      │
      │(Sonnet)  │(Sonnet)  │(Sonnet)
      │Ctx:20K   │Ctx:20K   │Ctx:20K
      └────┬─────┴────┬─────┴────┬───┘
           │          │          │
      ┌────▼──────────▼──────────▼───┐
      │   MCP Server Layer            │
      │   - WebSearch, Bash, Read     │
      │   - Custom Skills             │
      └───────────────────────────────┘
           │
      ┌────▼──────────────────────────┐
      │  External Systems             │
      │  - APIs, Databases, Files     │
      └───────────────────────────────┘
```

---

## 9. テスト戦略

### テストピラミッド

```
        △ E2E Tests (Orchestration全体)
       / \
      /   \  Integration Tests (Subagent + MCP)
     /     \
    / Unit  \ Unit Tests (各Agent, Skill)
   /         \
  ───────────── 
```

**カバレッジ目標**: ≥ 80%

---

## 参考リソース

- **AI Agent Patterns Reference**: https://monday.com/blog/ai-agents/ai-agent-architecture/
- **Pattern Deep Dives**: https://www.sitepoint.com/the-definitive-guide-to-agentic-design-patterns-in-2026/
- **Team CLAUDE.md**: `.claude/CLAUDE.md` (コーディング規約)
