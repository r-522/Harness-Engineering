# CLAUDE.md - プロジェクト規約（総括版）

**プロジェクト**: Harness Engineering - AI Agent Orchestration Platform
**スタック**: Python 3.10+ / TypeScript (strict) / YAML
**モデル**: Claude Opus 4.7（200k context window）
**有効期間**: 2026-05-10 〜  **行数**: 96行 / 上限 100行

---

## 1. 命名規則

- 変数・関数: `snake_case` (Py) / `camelCase` (TS)
- クラス: `PascalCase` / 定数: `UPPER_SNAKE_CASE`
- ファイル: `snake_case.py` / `kebab-case.ts`
- Agent: `{role}_{objective}` 例: `plan_director`, `impl_worker_1`

## 2. コード品質

- Python: PEP 8 / `ruff` + `black` 自動整形
- TypeScript: ESLint + Prettier / `strict` 必須
- 関数 ≤30行 / 循環複雑度 ≤10
- コメントは WHY のみ（WHAT は命名で表現）
- 不要な error handling・後方互換 shim・予防的抽象化を書かない

## 3. ディレクトリ構成

```
project-root/
├── .claude/            # CLAUDE/AGENT/HARNESS/ORCHESTRATION/OPERATIONS.md
├── src/
│   ├── agents/         # Agent定義
│   ├── orchestrators/  # 分割・合流
│   ├── tools/          # Tool実装
│   └── core/           # 共通
├── tests/{unit,integration,e2e}/
└── config/{settings.yaml,agents.yaml}
```

## 4. コミット & ブランチ

```
<type>(<scope>): <subject>     type ∈ feat|fix|refactor|test|docs|perf|chore
```

- `main` 保護 / `develop` 統合 / `feature/*` `agent/*` `fix/*` `claude/*`
- PR は Plan Mode 経由・最小1名レビュー・全 CI PASS 必須

## 5. Subagent ルール

- 既定並列度 **3**、ハードキャップ **10**
- 1 Subagent あたり Context 予算 60〜80k tokens
- Orchestrator はコード書かない（Plan / Dispatch / Merge のみ）
- 並列実行: 独立タスク・外部API・Context分離
- 直列実行: 強依存・単一ファイル編集・デバッグ

## 6. 禁止 / 要レビュー / 自動許可

🚫 **厳禁**
- `git push --force` / `git reset --hard` (ユーザー承認なしで)
- 本番APIキーのハードコード（`.env` / `.local.md` 必須）
- Subagent >10 / MCP 無許可接続 / Context残 <10% で新Agent起動
- `--no-verify` `--no-gpg-sign` 等の安全装置スキップ

⚠️ **要レビュー**
- 新 MCP 統合 / Agent 大規模変更 / 依存パッケージ追加 / API仕様変更

✅ **自動許可**
- ローカルファイル編集 / テスト実行 / ドキュメント更新 / `.local.md`

## 7. テスト & デプロイ

```bash
pytest tests/ -v --cov=src           # カバレッジ ≥80%
ruff check . && black --check .      # Lint / Format
pytest tests/integration -k orch     # Subagent統合
pytest tests/e2e --mcp-live          # MCP接続込みE2E
```

```bash
make build && make test              # ローカル
make deploy-staging                  # ステージング
git tag v1.x.x && git push origin v1.x.x   # 本番（自動デプロイ）
```

## 8. Context 管理（重要）

| 領域 | 閾値 | アクション |
|---|---|---|
| Safe | ≤40% (80k) | 通常運用 |
| Warn | 40–60% | 圧縮検討・Subagent委譲 |
| Danger | ≥60% (120k) | 計画ミス多発、即圧縮 |
| Rot | ≥300k cumulative | Session 切替必須 |

圧縮戦略: ログ削除→要約化→大ファイルは path 参照→Subagent 委譲→Session 再起動。

## 9. Worktree 並列セッション

```bash
git worktree add ../w1 develop && cd ../w1 && claude-code   # Orchestrator
git worktree add ../w2 develop && cd ../w2 && claude-code   # Implementation
git worktree add ../w3 develop && cd ../w3 && claude-code   # Testing
```

同一ファイル編集禁止 / 統合点で explicit rebase / テスト通過後のみ main へ。

## 10. Escalation

- L1 自己訂正（同一エラー <3回）
- L2 Peer Review（別 Agent に相談）
- L3 ユーザー判断（規約変更・API変更・依存追加）

---

**次**: `AGENT.md` でペルソナを確認、`HARNESS.md` で Tool/Context、`ORCHESTRATION.md` で並列パターン。
