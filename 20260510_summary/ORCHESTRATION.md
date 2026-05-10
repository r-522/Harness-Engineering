# ORCHESTRATION.md - Subagent 分割統治（総括版）

**有効期間**: 2026-05-10 〜  **行数**: 126行 / 上限 130行

---

## 1. パターン選択

| パターン | 規模 | 並列度 | 特徴 |
|---|---|---|---|
| **Split-and-Merge** | 小 | 3 | Orchestrator が分割→Workerが実装→Mergeで統合（推奨） |
| **Agent Teams** | 中 | 5–7 | 共有 `tasks.yaml` で自律協調 |
| Hierarchical | 大 | 多階層 | Context消費過大・デバッグ困難（**非推奨**） |

**第一選択は Split-and-Merge**。Team が必要なら明示的にPlanで宣言。

---

## 2. Split-and-Merge 4フェーズ

```
Phase 1: Plan      依存DAG / Token予算 / リスク
Phase 2: Dispatch  Subagent生成 / タスク割当 / 並列実行開始
Phase 3: Monitor   Context監視 / Failure検出 / Escalation判定
Phase 4: Merge     全完了待ち / 出力統合 / 最終Verify
```

### タスク分割テンプレ

```yaml
tasks:
  A: {desc: "認証ロジック",   deps: [],     assignee: w1, tokens: 40k}
  B: {desc: "API統合",         deps: [A],    assignee: w1, tokens: 30k}
  C: {desc: "ユニットテスト",  deps: [B],    assignee: w2, tokens: 35k}
  D: {desc: "E2Eテスト",       deps: [C],    assignee: w2, tokens: 25k}
  E: {desc: "ドキュメント",    deps: [A, B], assignee: w3, tokens: 20k}

execution:
  parallel:   [[A], [E_after_AB]]
  sequential: [A → B → C → D]
```

### Subagent Config

```python
SubagentConfig(
    name="worker_1",
    role="implementation_expert",
    context_budget=60_000,
    skills=["python_coding", "api_integration", "git"],
    task_ids=["A", "B"],
    instructions="Plan に厳密準拠。逸脱は即報告。",
)
```

---

## 3. Orchestrator Tool Set（4種）

```yaml
analyze_task:
  in:  task_description
  out: split_plan + dependency_graph
split_and_assign:
  in:  split_plan
  out: subagent_configs + assignments
monitor_execution:
  in:  subagent_status_list
  out: progress_report + escalation_alerts
merge_results:
  in:  all_subagent_outputs
  out: integrated_deliverable + quality_report
```

Orchestrator は **コードを書かない**。これを破ると Context が爆発する。

---

## 4. Agent Teams パターン（必要な時のみ）

```
Team Lead
├─ Planner   → tasks.yaml 書き込み
├─ Impl-1    → tasks.yaml 取得→実装
├─ Impl-2    → 同上
└─ Tester    → tasks.yaml 検証
```

Task Store（`.claude/tasks.yaml`）:

```yaml
tasks:
  - id: T1
    status: pending|in_progress|done|blocked
    owner: impl_1
    deps: []
    artifacts: [src/auth.py, tests/test_auth.py]
```

ロック方式: 楽観的更新（git commit 単位）。衝突時は Team Lead が再割当。

---

## 5. 並列度・予算ガバナンス

| 規模 | 並列 | 1agent予算 | 合計予算 |
|---|---|---|---|
| 小 | 3 | 60k | 180k |
| 中 | 5 | 50k | 250k |
| 大 | 7 | 40k | 280k |

ハードキャップ: 並列 10 / Context 残 <10% で新Agent起動禁止。

---

## 6. Monitor フェーズの 5指標

1. Context使用率（各Agent / 累計）
2. Tool失敗率（>10% で警告）
3. Test PASS率（<100% で停止）
4. 経過時間（予算の 1.5× で escalate）
5. Drift 検出（Plan からの逸脱量）

```python
if any(metric.exceeds_budget()):
    orchestrator.escalate(level=2 if recoverable else 3)
```

---

## 7. Merge の3ルール

1. **Cherry-pick のみ** — 全コピー禁止（Context汚染）
2. **テスト合流** — 各 Worker のテストを統合 suite で再実行
3. **Diff レビュー** — Reviewer が SOLID / セキュリティ最終確認

```bash
git checkout integration
for w in worker_1 worker_2 worker_3; do
  git cherry-pick $(git log $w --format=%H | reverse)
done
pytest tests/ --cov=src   # 統合カバレッジ ≥80%
```

---

## 8. アンチパターン

- 1 Orchestrator 配下 >7 Subagent → 監視不能
- Subagent 間で同一ファイルを編集 → merge地獄
- Plan を省いていきなり並列起動 → DAG崩壊
- 子の出力を親が全文取り込み → Context爆発
- 「とりあえず Subagent」依存 → 自分で考えなくなる

---

## 9. デバッグ時は直列に戻す

並列で再現しないバグは Worker 1 個で逐次再実行。
Context は捨てて新 Session で出直すほうが速い。

---

**次**: `OPERATIONS.md`（Hooks / MCP / Security / Caching の実運用）
