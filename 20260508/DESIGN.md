# DESIGN.md - 設計原則 & アーキテクチャガイド

**対象**: ハーネスエンジニアリング・プロジェクト  
**フレームワーク**: Claude Code Agent Orchestration  
**有効期間**: 2026-05-08 〜  
**行数**: 75行（制限: 80行以下）

---

## 1. 適用設計原則

### SOLID 原則

| 原則 | 適用内容 | 例 |
|---|---|---|
| **S**ingle Responsibility | 各 Agent・Tool は単一責務 | Plan Agent は計画のみ、Impl Agentは実装のみ |
| **O**pen/Closed | 拡張に開く、修正に閉じる | Tool追加は簡単、既存Tool修正は慎重 |
| **L**iskov Substitution | Subagent は互換性を保つ | Worker1, Worker2 は同じ interface |
| **I**nterface Segregation | 小さなインターフェース | Agent → Tool は明確な入出力仕様 |
| **D**ependency Inversion | 抽象に依存 | Tool.execute() の実装ではなくinterface |

### DRY / KISS 原則
- **DRY**: 同じロジック 3回以上なら関数化
- **KISS**: 複雑度 ≤ 10, 関数 ≤ 30行, CLAUDE.md ≤ 100行
- **YAGNI**: 推測機能は実装しない（実フィードバック後に追加）

### Orchestration 設計
- **Orchestrator**: 計画・分割・統合のみ（実装は Subagent に委譲）
- **Explicit**: 依存関係は明示的に指定（自動最適化に頼らない）
- **Async-friendly**: Tool完了を待たず、次の依存タスクへ

---

## 2. アーキテクチャパターン

### Split-and-Merge パターン

```
┌─────────────────────────────────┐
│  Orchestrator (Plan Agent)      │
│  - 大タスク分割                  │
│  - 依存グラフ生成                │
│  - 結果の統合                    │
└────────┬────────────────────────┘
         │
    ┌────┴────┬──────────┐
    │          │          │
    ▼          ▼          ▼
┌─────────┐ ┌──────────┐ ┌────────┐
│ Worker1 │ │ Worker2  │ │Worker3 │
│ (実装)  │ │(テスト)  │ │(Doc)   │
└────┬────┘ └────┬─────┘ └───┬────┘
     │           │           │
     └───────────┼───────────┘
                 │
            ┌────▼─────┐
            │ Merge     │
            │ 結果出力  │
            └──────────┘
```

### 依存関係グラフ（DAG）
```yaml
tasks:
  task_1:
    name: "環境準備"
    dependencies: []  # 最初に実行
    
  task_2:
    name: "設計ドキュメント作成"
    dependencies: [task_1]
    
  task_3a:
    name: "実装 Part A"
    dependencies: [task_2]
    parallel_with: [task_3b]  # 並列可能
    
  task_3b:
    name: "実装 Part B"
    dependencies: [task_2]
    parallel_with: [task_3a]
    
  task_4:
    name: "統合テスト"
    dependencies: [task_3a, task_3b]  # 両方完了待ち
    
  task_5:
    name: "デプロイ"
    dependencies: [task_4]  # 最後
```

---

## 3. ディレクトリ構成 & 責務分離

```
src/
├── agents/
│   ├── plan_director.py        # 計画・分割・統合
│   ├── impl_worker.py          # 実装処理
│   └── test_worker.py          # テスト実行
│
├── orchestrators/
│   ├── task_splitter.py        # タスク分割ロジック
│   ├── dependency_graph.py     # DAG管理
│   └── context_manager.py      # Token予算管理
│
├── tools/
│   ├── file_tools.py           # ファイル読み書き
│   ├── test_tools.py           # テスト実行
│   └── git_tools.py            # Git操作
│
└── core/
    ├── agent_base.py           # Agent基底クラス
    ├── tool_interface.py       # Tool interface
    └── config.py               # グローバル設定
```

### 責務マッピング

| ファイル | 責務 | 主要関数 |
|---|---|---|
| `agents/plan_director.py` | タスク分析・分割・統合 | `analyze()`, `split()`, `merge()` |
| `agents/impl_worker.py` | コード実装・テスト | `implement()`, `verify()` |
| `orchestrators/dependency_graph.py` | 依存関係管理 | `build_dag()`, `topological_sort()` |
| `tools/test_tools.py` | テスト実行・報告 | `run_tests()`, `get_coverage()` |

---

## 4. データフロー & コミュニケーション

### Agent間メッセージフォーマット

```python
@dataclass
class AgentMessage:
    sender: str              # "plan_director", "worker_1", ...
    receiver: str            # "worker_2", "main", ...
    message_type: str        # "task", "result", "error", "status"
    payload: dict            # 実際のデータ
    timestamp: str           # ISO 8601
    context_tokens_used: int # 親の context 使用率
```

### タスク定義スキーマ
```yaml
task:
  id: "task_001"
  name: "Feature A 実装"
  description: "..."
  agent_type: "implementation_worker"
  acceptance_criteria:
    - "Code coverage >= 80%"
    - "All tests pass"
    - "Lint errors = 0"
  dependencies: ["task_001"]
  estimated_tokens: 50000
  timeout_minutes: 30
```

---

## 5. エラーハンドリング & リカバリ

### エラー分類

| Level | 例 | 対応 |
|---|---|---|
| **1** | Tool実行失敗 | Retry 3回 → Level 2へ |
| **2** | Test failure | Code修正 + 再テスト |
| **3** | Architecture問題 | 再計画 (Plan Agent へ) |
| **4** | Policy違反 | エスカレーション |

### Retry戦略
```python
def execute_with_retry(tool_fn, max_retries=3):
    for attempt in range(max_retries):
        try:
            return tool_fn()
        except ToolError as e:
            if attempt < max_retries - 1:
                log.warning(f"Retry {attempt+1}/{max_retries}: {e}")
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                log.error("Max retries exceeded, escalate to Level 2")
                raise EscalationRequired(e)
```

---

## 6. テスト戦略

### テスト階層

| 階層 | 目的 | 対象 | カバレッジ |
|---|---|---|---|
| **Unit** | 関数の動作確認 | Tool, utility関数 | >= 90% |
| **Integration** | Agent間連携確認 | Orchestrator, Tool呼び出し | >= 70% |
| **E2E** | 全フロー確認 | 計画 → 実装 → テスト → デプロイ | >= 50% |

### テスト実行例
```bash
pytest tests/unit/ -v --cov=src/tools
pytest tests/integration/ -v -k "orchestration"
pytest tests/e2e/ -v --mcp-live
```

---

## 7. Context Engineering 原則

### トークン予算配分

```
総budget: 200k tokens
├── System Prompt: 5k
├── Code Review: 20k
├── Implementation: 100k
└── Testing & Integration: 75k
```

### 削減戦略
1. **Compression**: 履歴を要約（"直近10コミット → 1行サマリー")
2. **Delegation**: Subagent にタスク委譲
3. **Reference**: 大ファイルはパス参照（全文埋め込み禁止）
4. **Rotation**: Context 60%超えたら Session切り替え

---

## 8. セキュリティ設計

### 原則
- **Principle of Least Privilege**: 最小権限付与（Tool実行の許可制）
- **Defense in Depth**: 多層防御（入力検証、Tool sandboxing）
- **Audit Trail**: 全 Tool実行をログ記録

### 実装例
```python
class Tool:
    required_permissions = ["read_files", "execute_tests"]
    
    def execute(self, *args, **kwargs):
        if not check_permissions(required_permissions):
            raise PermissionDenied()
        log_audit(actor=self.agent_id, action=self.__name__)
        return self._run(*args, **kwargs)
```

---

**次**: `.claude/ORCHESTRATION.md` で Subagent 分割統治パターンを確認してください。
