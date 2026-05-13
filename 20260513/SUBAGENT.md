# SUBAGENT.md - Subagent Orchestration & 並列パターン

**対象**: マルチエージェント調整・Context最適化  
**最終更新**: 2026-05-13

---

## 1. Subagent とは

**Subagent**: Claude Code で複数の独立した AI Agent を並列・直列実行し、大規模タスクを分散処理する仕組み。

### 単一 Agent vs Subagent

| 特性 | **単一 Agent** | **Subagent（複数）** |
|------|--------------|-------------------|
| Context | 600k tokens | 100k tokens × N 個 |
| 実行時間 | 長い（T分） | 短い（T/N分、並列時） |
| 復旧 | 全体失敗 | 1つの失敗 = その Agent のみ再試 |
| 複雑度 | シンプル（単一ロジック） | 複雑（複数ロジック + 統合） |

**判定基準**: Context 見積もり > 400k tokens なら Subagent 分割を検討

---

## 2. Orchestration パターン

### パターン1: 並列実行（Parallel）

**図解**:
```
Main Agent
    ↓
 [PLAN]
    ↓
┌────────────────────────────┐
│ Spawn parallel subagents   │
└────────────────────────────┘
    ↓
  ┌─────────┬──────────┬──────────┐
  ▼         ▼          ▼          ▼
Agent-A  Agent-B   Agent-C    Agent-D
  │         │        │          │
  ↓ (10min) ↓        ↓ (10min)  ↓
Result-A  Result-B Result-C   Result-D
  └─────────┴────────┴──────────┘
    ↓
 [VERIFY]
    ↓
Integrated Result
```

**実装例（Python）**:
```python
import asyncio
from src.agents import agent_a, agent_b, agent_c

async def parallel_orchestration(task):
    # PLAN: 分割
    subtask_a, subtask_b, subtask_c = decompose(task)
    
    # ACT: 並列実行（3つ同時）
    results = await asyncio.gather(
        agent_a.run(subtask_a),
        agent_b.run(subtask_b),
        agent_c.run(subtask_c)
    )
    
    # VERIFY: 統合
    return integrate(*results)
```

**使用シーン**:
- 複数の独立研究タスク（各5〜10分）
- 複数ドメインの並列実装（Backend, Frontend 同時）
- バッチ処理（複数API呼び出し並列化）

**メリット**:
- 全体実行時間: T → T/3（理論値）
- Context 利用効率: 300k tokens × 3 Agent

**デメリット**:
- 統合ロジックの複雑度向上
- デバッグが難しい（複数 stderr）

---

### パターン2: 直列実行（Sequential）

**図解**:
```
Main Agent
    ↓
Agent-A (10min) → Result-A
    ↓
[Validation]
    ↓
Agent-B (5min) → Result-B
    ↓
[Validation]
    ↓
Agent-C (3min) → Result-C
```

**実装例**:
```python
async def sequential_orchestration(task):
    # Step 1
    result_a = await agent_a.run(task)
    if not validate(result_a):
        raise ValueError("Agent-A failed validation")
    
    # Step 2 (Step 1 の結果に依存)
    task_b = prepare_task_b(result_a)
    result_b = await agent_b.run(task_b)
    
    # Step 3
    task_c = prepare_task_c(result_b)
    result_c = await agent_c.run(task_c)
    
    return result_c
```

**使用シーン**:
- 前の Agent 出力が次の入力になる場合（Data Pipeline）
- 単一ファイル編集（コンフリクト回避）
- デバッグ・検証フェーズ

**メリット**:
- 実装がシンプル（1ステップずつ進む）
- デバッグしやすい（各ステップで検証）

**デメリット**:
- 全体実行時間が長い（T = T1 + T2 + T3）

---

### パターン3: ハイブリッド（Parallel + Sequential）

**図解**:
```
┌─────────────────────────┐
│ Phase 1: Parallel (10min)│
├─────────────────────────┤
Agent-A (Research)  │  Agent-B (Implementation)
Result-A            │   Result-B
└─────────────────────────┘
    ↓ [Main validation]
┌─────────────────────────┐
│ Phase 2: Sequential (5min)│
├─────────────────────────┤
Agent-C (Security Review)
(Result-A + Result-B 参照)
└─────────────────────────┘
    ↓
Final Result
```

**実装例**:
```python
async def hybrid_orchestration(task):
    # Phase 1: 並列（独立タスク）
    research, impl = decompose(task)
    result_research, result_impl = await asyncio.gather(
        agent_research.run(research),
        agent_impl.run(impl)
    )
    
    # 中間検証
    if not validate_intermediate(result_research, result_impl):
        return None
    
    # Phase 2: 直列（統合）
    security_task = prepare_security_check(result_research, result_impl)
    result_security = await agent_security.run(security_task)
    
    return result_security
```

**使用シーン**: 大規模機能開発（最も一般的）
- Phase 1: 複数ドメイン（Backend, Frontend, DevOps）の並列開発
- Phase 2: 統合テスト・セキュリティレビュー

**実行時間**: 10 + 5 = 15分（完全直列なら 10 + 5 + 3 = 18分）

---

## 3. Context 管理 & 最適化

### Context 予算の計算

```
Total Context = 600k tokens (Opus 4.7)

[起動時必須ロード]
  - CLAUDE.md: 5k
  - AGENT.md: 4k
  - test files: 45k
  → Subtotal: 54k

[利用可能]
  - 600k - 54k = 546k tokens

[Subagent 分割案]
  - Agent-A (researcher): 150k
  - Agent-B (developer): 150k
  - Agent-C (tester): 100k
  - Main (orchestrator): 80k
  → Subtotal: 480k

[安全マージン]: 546k - 480k = 66k (12%)
```

### トークン削減テクニック

| テクニック | 削減量 | 実装例 |
|----------|------|------|
| CLAUDE.md を 50行以下に圧縮 | -2k tokens | 外部参照ファイル形式 |
| テストコードを tests/ に分離 | -45k tokens | `from tests import fixtures` 不使用 |
| 不要なコメント・空行削除 | -5k tokens | `grep -v "^#\|^$"` |
| ログ出力を削減 | -3k tokens | `logger.setLevel(WARNING)` |
| 重複コードを関数化 | -10k tokens | DRY原則 |

### Subagent Context 割り当ての最適化

```yaml
# config/agents.yaml
subagents:
  researcher_context:
    context_tokens: 150000      # 25% 多く割り当て（調査は複雑）
    max_output_tokens: 20000
    timeout_seconds: 180
  
  developer_impl:
    context_tokens: 100000
    max_output_tokens: 25000
    timeout_seconds: 300        # 実装は時間がかかる
  
  tester_e2e:
    context_tokens: 80000       # テストは軽め
    max_output_tokens: 15000
    timeout_seconds: 180
```

---

## 4. エラー処理 & リカバリー

### エラータイプと対応

| エラータイプ | 原因例 | リカバリー |
|------------|------|----------|
| **Context Overflow** | メモリ超過 | Subagent を 2つに分割 |
| **Timeout** | 処理が長い | タイムアウト延長 → 依然失敗なら分割 |
| **API Failure** | MCP接続失敗 | リトライ（最大3回） + Backoff |
| **Validation Failed** | 出力スキーマ不正 | Agent に仕様を再度提示 → 再実行 |

### Graceful Degradation（段階的縮退）

```python
async def orchestrate_with_fallback(task):
    try:
        # Phase 1: 3つのAgent並列実行
        return await parallel_orchestration(task)
    except ContextOverflow:
        logger.warning("Context overflow: Reducing to 2 agents")
        # Phase 2: 2つのAgent並列実行（Context削減）
        return await sequential_orchestration(task)
    except Exception as e:
        logger.error(f"Orchestration failed: {e}")
        # Fallback: ユーザーに報告
        raise EscalationRequired(str(e))
```

---

## 5. デッドロック回避

### 問題: Agent A が Agent B の結果を待ち、Agent B が Agent A の結果を待つ

```
❌ 危険なパターン
Agent-A: "I need Result-B to proceed"
Agent-B: "I need Result-A to proceed"
→ 永遠に待機 (Deadlock)
```

### 解決策

1. **DAG (Directed Acyclic Graph) を事前に設計**
   ```
   Agent-A → Agent-B → Agent-C
   (線形フロー、循環依存なし)
   ```

2. **Timeout を設定**
   ```python
   result = await asyncio.wait_for(
       agent_b.run(task),
       timeout=180  # 3分
   )
   ```

3. **逆方向依存関係を禁止**
   ```
   ✅ OK: A → B → C
   ❌ NG: A ↔ B（相互依存）
   ```

---

## 6. 並列度の決定ルール

| Parallel Agent 数 | Context要件 | 適用シーン | リスク |
|-----------------|-----------|---------|------|
| **1** | 400k tokens | 単一タスク | なし |
| **2** | 300k + 150k | 2つの独立タスク | 低 |
| **3** | 150k × 3 | 複数ドメイン並列 | 中（Context管理） |
| **4** | 100k × 4 | 多数の軽量タスク | 高（統合複雑） |
| **> 5** | 不推奨 | ❌ | 非常に高（デバッグ困難） |

**推奨**: デフォルト 3、メモリ削減時は 2、軽量タスクなら最大 4

---

## 7. モニタリング & ロギング

### Orchestration ログの構造

```python
# src/core/logger.py
import logging
import json
from datetime import datetime

logger = logging.getLogger("orchestration")

def log_orchestration_step(step_name, agent_name, status, context_tokens):
    log_data = {
        "timestamp": datetime.now().isoformat(),
        "step": step_name,
        "agent": agent_name,
        "status": status,  # started, completed, failed
        "context_tokens": context_tokens,
        "remaining_tokens": 600000 - context_tokens
    }
    logger.info(json.dumps(log_data))

# 使用例
log_orchestration_step("phase1", "researcher_context", "completed", 120000)
```

### Orchestration メトリクス

```yaml
# config/monitoring.yaml
metrics:
  - execution_time_per_phase
  - context_tokens_used
  - subagent_parallel_factor
  - failure_rate_per_agent
  - integration_time
```

---

## 8. ベストプラクティス

### チェックリスト

- [ ] DAG（有向非巡回グラフ）で依存関係を可視化
- [ ] 各 Subagent の Context 割り当てを事前に見積もり
- [ ] Timeout を設定（リソースリーク防止）
- [ ] 並列度は最大 3（デフォルト）
- [ ] エラー時のリカバリーロジックを実装
- [ ] ログに timestamps を含める
- [ ] 統合ロジックは Main Agent に集約
- [ ] Subagent 間で状態を共有しない（独立性）

### 実装チェックリスト（新規 Subagent 追加時）

```python
# src/agents/new_agent.py

class NewSubagent:
    def __init__(self):
        self.context_budget = 100000  # ✅ 割り当て明示
        self.timeout = 180            # ✅ Timeout 設定
        self.logger = setup_logger()  # ✅ ロギング
    
    async def run(self, task):
        self.logger.info(f"Starting: {task}")  # ✅ ログ
        try:
            result = await self._process(task)
            self.logger.info(f"Completed: {result}")
            return result
        except Exception as e:
            self.logger.error(f"Failed: {e}")  # ✅ エラー処理
            raise
    
    async def _process(self, task):
        # 実装
        pass
```

---

## 参考

- Claude Code Sub-Agents: https://code.claude.com/docs/en/sub-agents
- Sub-Agent Best Practices: https://claudefa.st/blog/guide/agents/sub-agent-best-practices
- The Code Agent Orchestra: https://addyosmani.com/blog/code-agent-orchestra/
