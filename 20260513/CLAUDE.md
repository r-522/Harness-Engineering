# CLAUDE.md - Claude Code プロジェクト設定

**対象**: 小規模～中規模開発チーム（上級者）  
**スコープ**: プロジェクト層  
**最終更新**: 2026-05-13

---

## 1. プロジェクト概要

**プロジェクト名**: [プロジェクト名を記入]  
**言語スタック**: [例: Python 3.10+, TypeScript, YAML] / または参照ファイル: `config/tech-stack.md`  
**フレームワーク**: Claude Code, Claude API (Opus 4.7), Subagent SDK  
**チーム規模**: 小規模～中規模（5〜20名）  
**エージェント対象**: 上級者（Plan mode、Context最適化、Orchestration必須）

---

## 2. コーディング規約

### 命名規則

- **変数・関数**: `snake_case` (Python), `camelCase` (TypeScript)
- **クラス**: `PascalCase`
- **定数**: `UPPER_SNAKE_CASE`
- **ファイル**: `kebab-case.ts`, `snake_case.py`
- **Subagent**: `{role}_{objective}` (例: `researcher_context`, `optimizer_cost`)

### コード品質

- **Python**: PEP 8準拠、`ruff` + `black` で自動フォーマット
- **TypeScript**: ESLint + Prettier、`strict` mode必須
- **コメント**: **WHYのみ**（WHATは命名で表現。参照: src/examples/*.py の行番号）
- **関数長**: 30行以下推奨（複雑ロジックは関数分割）
- **循環複雑度**: ≤ 10（mcabe ツールで検査）

### ファイル編集のルール

- **単一責任原則**: 1ファイル = 1責任 / 関数分割が必要なら split する
- **参照ファイル形式**: CLAUDE.md に大きなコード例を埋め込まない。`src/examples/naming-convention.py:15-30` の形式で参照
- **リネーム操作**: 未使用の変数を削除する際、コメント残置（`// removed`）は不要。完全に削除

---

## 3. 禁止事項 & 動作制限

### 🚫 厳禁

- [ ] `git push --force`（事前レビュー必須）
- [ ] 本番 API キー、OAuth tokens をコード内に埋め込み（`.env.local` / `.secrets.local.json` へ）
- [ ] Subagent 数 > 10（コンテキスト枯渇リスク）
- [ ] 無許可の MCP サーバ接続（セキュリティレビュー必須）
- [ ] Context window 残量 < 10% での新 Subagent 起動
- [ ] 破壊的操作の無確認実行（`rm -rf`, `git reset --hard`, branch 削除）

### ⚠️ 要事前レビュー

- [ ] 新しい MCP 統合（セキュリティレビュー必須）
- [ ] エージェント振る舞いの大規模変更（Orchestration ロジック変更）
- [ ] パッケージ追加（依存関係・セキュリティ確認）
- [ ] API 仕様変更、外部サービス新規接続

### ✅ 自動許可（事前許可なし）

- [ ] ローカルファイル編集（.gitignore 対象外）
- [ ] テスト実行（`pytest`, `npm test` など）
- [ ] ドキュメント更新（コードに影響なし）
- [ ] `.env.local`, `.local.md` への個人設定記入

---

## 4. ファイル編集・テスト・ビルドポリシー

### テスト実行

```bash
# 単体テスト（推奨: pytest で 80% カバレッジ）
pytest tests/ -v --cov=src --cov-fail-under=80

# 統合テスト（Subagent 並列度確認）
pytest tests/integration/ -v -k "orchestration"

# E2E テスト（MCP 接続含む）
pytest tests/e2e/ --mcp-live
```

**通過基準**: カバレッジ ≥ 80%, 全テスト PASS, Lint エラー 0

### ビルド & ローカル検証

```bash
# ローカル検証（コミット前）
make build && make test && make lint

# ステージング デプロイ前
make deploy-staging

# 本番 デプロイ（保護ブランチ、自動CI/CD）
git tag v1.x.x && git push origin v1.x.x
```

### 破壊的操作のポリシー

**判定基準**: 下記いずれかに該当する場合、**事前に UserAskQuestion 等で確認を取得**

1. ローカルの未コミット変更を削除する（`git checkout .`, `git reset --hard`）
2. リモートブランチを強制上書き（`git push --force`）
3. ファイル / ディレクトリの削除（`rm`, `rm -rf`, MCP GitHub ツール）
4. 30行以上のコード削除

---

## 5. Context Management（トークン予算）

- **デフォルト Context Window**: 600k tokens（Opus 4.7）
- **推奨上限**: 400k tokens（以降、パフォーマンス低下リスク）
- **Subagent 1つあたり**: ~100k tokens（CLAUDE.md, テストコード含む）
- **最大並列 Subagent 数**: 3（200k tokens = max parallel 3）

**最適化テクニック**:
- CLAUDE.md は 80行以下、外部参照ファイル形式を使う
- テストコードは tests/ に分離（ロード対象外）
- 不要な中間変数、ログ行を削除

---

## 6. Commit ルール

```
<type>: <subject>

<body>

Refs: #<issue-number>
```

**type**: `feat`, `fix`, `refactor`, `test`, `docs`, `perf`

**例**:
```
feat: Add researcher subagent for context analysis

Implements parallel agent orchestration for literature review.
Reduces context window by 35% vs sequential approach.

Refs: #42
```

---

## 7. Pull Request ルール

- **必須**: Plan mode（.claude/AGENT.md の `plan_mode` セクション）で整理した上で実装
- **同期レビュー**: チーム内最小 1名（セキュリティレビューが必須な場合は 2名）
- **自動チェック**: CI/CD で全テスト・Linter PASS が必須

**例外**: ドキュメント更新、`.local.md` 個人設定は レビュー不要

---

## 8. MCP & Subagent 接続ルール

参照: `.claude/MCP.md`, `.claude/SUBAGENT.md`

- **MCP Server**: 参照ファイル（`src/mcp/servers.yaml`）で一元管理
- **Subagent**: `src/agents/{role}_{objective}.py` 命名 + `config/agents.yaml` に定義
- **並列度**: デフォルト 3、Context 不足時は 2 に低下
- **タイムアウト**: 180秒（API 呼び出しが多い場合は 300秒）

---

## 参考

- Claude Code Docs: https://code.claude.com/docs/en/overview
- Subagent Best Practices: https://claudefa.st/blog/guide/agents/sub-agent-best-practices
- MCP Introduction: https://anthropic.skilljar.com/introduction-to-model-context-protocol
