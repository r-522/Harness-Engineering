# 20260510_summary - 最強ハーネスプロンプト総括版

**対象**: Harness Engineering チーム / Claude Code + Anthropic SDK (Opus 4.7)
**位置付け**: 20260421〜20260508 の全成果（CLAUDE/AGENT/DESIGN/HARNESS/ORCHESTRATION/HOOKS/MCP/SECURITY/CONTEXT/SKILLS/CACHING/GOVERNANCE/PERFORMANCE）の総括版
**有効期間**: 2026-05-10 〜
**設計原則**: Agent = Model + Harness。Harnessこそが性能差を生む。

---

## 0. ファイル構成（最小・正規セット）

| ファイル | 役割 | 行数上限 |
|---|---|---|
| `README.md` | 全体ガイド・読み順 | 100 |
| `MASTER_PROMPT.md` | **最強の System Prompt 統括版** | 80 |
| `CLAUDE.md` | プロジェクト規約 | 100 |
| `AGENT.md` | Agentペルソナ・思考プロセス | 120 |
| `HARNESS.md` | Harness構成・Tool Registry・Context | 130 |
| `ORCHESTRATION.md` | Subagent分割統治・並列実行 | 130 |
| `OPERATIONS.md` | Hooks/MCP/Security/Cachingの統合運用 | 120 |

---

## 1. 読み順（新規参加者）

1. **README.md** (このファイル) — 全体像
2. **MASTER_PROMPT.md** — System Prompt として直接利用可能
3. **CLAUDE.md** — プロジェクト固有規約
4. **AGENT.md** — どのペルソナで動くか
5. **HARNESS.md** — Tool/Context/Feedback の詳細
6. **ORCHESTRATION.md** — 並列実行が必要な時
7. **OPERATIONS.md** — Hooks/MCP/Security/Caching

---

## 2. 「最強」と呼ぶ理由（過去版からの統合学習）

| 過去版 | 採用した知見 |
|---|---|
| 20260421 INDEX/CONTEXT | Context予算明示・40%安全域 |
| 20260423 GOVERNANCE | 禁止事項3層（厳禁/要レビュー/自動許可） |
| 20260425 GUIDED_SENSING | Feedforward + Feedback 構造 |
| 20260427 PERFORMANCE/SECURITY | OWASP top10 + Tool権限カテゴリ |
| 20260429-30 HOOKS/MCP/CACHING | Prompt Caching + Hook トリガー |
| 20260507 SKILLS | Skill 8分類とトリガー表 |
| 20260508 HARNESS/ORCHESTRATION | System Prompt 60行原則・DAG分割 |

→ これら全てを **80行 System Prompt + 6本ドキュメント** に圧縮。

---

## 3. 即時起動手順

```bash
# 1. .claude/ にシンボリックリンクで配置
ln -sf $(pwd)/20260510_summary/CLAUDE.md      .claude/CLAUDE.md
ln -sf $(pwd)/20260510_summary/AGENT.md       .claude/AGENT.md
ln -sf $(pwd)/20260510_summary/HARNESS.md     .claude/HARNESS.md
ln -sf $(pwd)/20260510_summary/ORCHESTRATION.md .claude/ORCHESTRATION.md
ln -sf $(pwd)/20260510_summary/OPERATIONS.md  .claude/OPERATIONS.md

# 2. Plan Mode で起動
claude-code --plan
> /init
> "Adopt MASTER_PROMPT.md as system prompt. Read CLAUDE.md and AGENT.md."
```

---

## 4. 成功条件（Definition of Done）

- [ ] System Prompt が 80行以下
- [ ] Context 使用率を常時報告（≤40% 安全域）
- [ ] Test カバレッジ ≥80%
- [ ] CLAUDE.md 違反 0
- [ ] Tool 権限カテゴリ全件登録
- [ ] Plan → Act → Verify を全タスクで実施

---

**次に読むファイル**: `MASTER_PROMPT.md`（System Promptとしてそのまま貼付可能）
