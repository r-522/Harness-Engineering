# HARNESS.md - Harness構成・Tool Registry・Context（総括版）

**有効期間**: 2026-05-10 〜  **行数**: 128行 / 上限 130行

---

## 1. 構成図 — Agent = Model + Harness

```
┌────────────────────────────────────────────┐
│            Harness (Agent Engine)          │
│ ┌──────────────┐  ┌──────────────────────┐ │
│ │ System Prompt│  │   Tool Registry      │ │
│ │  ≤80 lines   │  │   (定義 + 権限)      │ │
│ └──────────────┘  └──────────────────────┘ │
│ ┌──────────────┐  ┌──────────────────────┐ │
│ │ Context Mgmt │  │   Execution Engine   │ │
│ │  / Caching   │  │   (sandbox+timeout)  │ │
│ └──────────────┘  └──────────────────────┘ │
│ ┌────────────────────────────────────────┐ │
│ │ Feedforward + Feedback Sensors         │ │
│ │ (CLAUDE.md / tests / lint / coverage)  │ │
│ └────────────────────────────────────────┘ │
└────────────────────────────────────────────┘
                    ↓
            Claude Opus 4.7 API
```

Harness は Model に「仕事をさせる」全インフラ — 制約・道具・反省能力。

---

## 2. System Prompt 設計原則

- **80行以下**（実測スイートスポット 50–60行）
- 冒頭に Mission・Constraints・Phase の3塊
- Tools は名前と1行説明のみ（詳細はRegistryへ）
- 末尾に Self-Check テンプレ
- 完成版は `MASTER_PROMPT.md` を参照

---

## 3. Tool Registry

```yaml
read_file:        {category: safe,       perm: read_files}
edit_file:        {category: cautious,   perm: write_files}
write_file:       {category: cautious,   perm: write_files}
run_tests:        {category: safe,       perm: execute_code}
lint:             {category: safe,       perm: execute_code}
git_commit:       {category: cautious,   perm: git_write}
git_push:         {category: restricted, perm: git_write,
                   blocked_flags: ["--force", "-f"]}
mcp_call:         {category: restricted, perm: mcp,
                   allowlist: [registered_servers_only]}
git_reset_hard:   {category: dangerous,  perm: none, requires_user_approval: true}
rm_rf:            {category: dangerous,  perm: none, requires_user_approval: true}
```

### Tool 定義スキーマ

```python
@dataclass
class ToolDefinition:
    name: str
    description: str
    input_schema: dict
    required_permissions: list[str]
    timeout_seconds: int = 300
    max_retries: int = 3
    output_schema: dict | None = None
```

### 実行フロー

```
Request → Permission Check → Sandboxed Execute → Schema Validate → Return
                ↓ deny                              ↓ mismatch
            Escalate                              Retry / Escalate
```

### Retry ポリシー

- 指数バックオフ: 2s, 4s, 8s（最大3回）
- 4回目失敗で context snapshot 添付して human escalation
- 同一エラークラス3連続でループ停止

---

## 4. Context Engineering

### Window 配分（200k）

```
Initial    : System(3k) + CLAUDE(4k) + AGENT(3k) + Task(2k) = 12k
Working    : Code(40k) + Tests(10k) + Errors(5k) + Feedback(3k) = 58k
Target Cap : 80k (40%)
```

### 閾値とアクション

| 使用率 | 名称 | アクション |
|---|---|---|
| ≤40% | Safe | 通常運用 |
| 40–60% | Warn | 圧縮・Subagent委譲検討 |
| ≥60% | Danger | 計画ミス多発、即圧縮 |
| ≥90% | Force | Session 切替必須 |
| ≥300k累計 | Rot | 出力品質劣化、新Session |

### 圧縮戦略

```python
# 200倍圧縮の例
before = "FAILED test_auth.py::test_login (AssertionError at line 42 ...)"  # ~2k
after  = "FAILED: test_login (cov 75%)"                                      # ~10
```

- 中間テスト出力 → PASS/FAIL のみ
- Verbose ログ → key error のみ
- 完了タスク → 1行サマリ
- 大ファイル参照 → path のみ（全文埋込禁止）
- 子 Subagent 出力 → 親は要約のみ取込

### Prompt Caching（4ブロック）

```
[1] System Prompt (静的)        ← cache_control: ephemeral
[2] CLAUDE.md (静的)            ← cache_control: ephemeral
[3] AGENT.md  (静的)            ← cache_control: ephemeral
[4] 会話履歴 (動的)
```

90%+ ヒット率で実コスト 1/10。

---

## 5. Feedback Loops

```
Feedforward (予防): CLAUDE.md整合 / 計画整合 / Lint pre-check
   ↓
Action (実装)
   ↓
Feedback (検証): tests / coverage / type-check / SOLID review
   ↓
Self-correct (≤3回) → Peer Review → User Escalation
```

センサー出力は schema 検証してから Agent に返す。
失敗時は **原因分析 → 対策 → 再試行** の3点セットを残す。

---

## 6. Production Deployment Check

```markdown
- [ ] System Prompt ≤80行
- [ ] Tools 全件 Registry 登録 / 権限カテゴリ確定
- [ ] Coverage ≥80%, all PASS
- [ ] CLAUDE.md チーム合意済
- [ ] Subagent ごとに Context 予算割当
- [ ] DAG サイクル検出 OK
- [ ] L1–L3 エラーハンドリング実装済
- [ ] OWASP top10 対応
- [ ] README + API docs 完備
```

---

**次**: `ORCHESTRATION.md`（Split-and-Merge / Agent Teams）
