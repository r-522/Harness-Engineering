# CLAUDE.md - プロジェクト設定 & チーム共有ルール

**対象**: Harness Engineering チーム  
**スコープ**: プロジェクト層（`.claude/CLAUDE.md`）  
**最終更新**: 2026-05-07  
**Claude Model**: claude-opus-4-7  

---

## 1. プロジェクト概要

**プロジェクト名**: Harness Engineering - AI Agent Orchestration Platform  
**言語スタック**: Python 3.10+, TypeScript, YAML  
**フレームワーク**: Claude Code, Claude Opus 4.7, Subagent SDK  
**チーム規模**: 小規模チーム（〜5名）  
**エージェント対象**: 上級者（Plan mode、Context最適化、Orchestration必須）  

**キー機能**:
- Subagent並列実行（最大3並列）
- MCP Server統合（AWS、GCP、カスタムツール）
- Context Window最適化（1Mトークン環境、30%以下目標使用率）

---

## 2. コーディング規約

### 命名規則

| 言語 | 変数・関数 | クラス | 定数 | ファイル |
|---|---|---|---|---|
| Python | `snake_case` | `PascalCase` | `UPPER_SNAKE_CASE` | `snake_case.py` |
| TypeScript | `camelCase` | `PascalCase` | `UPPER_SNAKE_CASE` | `kebab-case.ts` |
| Bash | `snake_case` | N/A | `UPPER_SNAKE_CASE` | `kebab-case.sh` |
| Subagent | `{role}_{objective}` | N/A | N/A | `agent-{name}.md` |

**例**:
- ✅ `process_webhook_payload()`, `DatabaseConnection`
- ❌ `processWebhookPayload()`, `database_connection`（Python内）

### コード品質

**Python**
```bash
ruff check . && black . && pytest tests/ --cov=src --cov-fail-under=80
```

**TypeScript**
```bash
eslint . && prettier --write . && npm test -- --coverage --coverageThreshold='{"global":{"branches":80}}'
```

**共通**
- コメント: WHYのみ（WHAT は命名で表現）
- 関数長: 30行以下推奨
- 循環複雑度: ≤ 10

### Type Safety

- Python: `mypy --strict` 必須（型ヒント必須）
- TypeScript: `strict: true` in `tsconfig.json`

---

## 3. プロジェクト構成・ディレクトリ

```
project-root/
├── .claude/
│   ├── CLAUDE.md (このファイル)
│   ├── AGENT.md
│   ├── DESIGN.md
│   ├── SKILLS.md
│   └── MCP-INTEGRATION.md
├── src/
│   ├── agents/ (Subagent定義)
│   ├── skills/ (カスタムスキル)
│   ├── mcp/ (MCP統合)
│   ├── core/ (共通ライブラリ)
│   └── api/ (API エンドポイント)
├── tests/
│   ├── agents/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/
├── config/
│   ├── settings.yaml
│   ├── agents.yaml
│   └── mcp-servers.yaml
├── docs/
│   ├── ARCHITECTURE.md
│   ├── API.md
│   └── TROUBLESHOOTING.md
└── README.md
```

---

## 4. ワークフロー & チームプロセス

### コミット規約

```
<type>: <subject>

<body>

Refs: #<issue-number>
```

**type** (必須): `feat` | `fix` | `refactor` | `test` | `docs` | `perf` | `ci`

**例**:
```
feat: Add async subagent execution for parallel workflows

Implement background task support allowing multiple agents to run
concurrently while main session continues. Adds queue management
and event-based coordination.

Refs: #42
```

### ブランチ戦略

| ブランチ | 用途 | 保護 | デプロイ |
|---|---|---|---|
| `main` | 本番環境 | ✅ | 自動デプロイ |
| `develop` | 統合ブランチ | ✅ | ステージング |
| `feature/*` | 新機能 | ❌ | マージテスト |
| `agent/*` | Subagent追加 | ❌ | マージテスト |
| `fix/*` | バグ修正 | ❌ | マージテスト |

### Pull Request ルール

1. **プランニング**: Plan modeで詳細計画を立案（必須）
2. **実装**: Plan承認後にコード実装
3. **レビュー**: 同期レビュー（チーム内最小1名）
4. **テスト**: 全テスト・Linter通過が必須（CI自動チェック）
5. **マージ**: リードの最終承認後にマージ

---

## 5. Subagent & Orchestration ルール

### 使用シーン

**並列実行を使う** ✅:
- 複数の独立研究タスク
- 外部API呼び出し（複数バッチ）
- コンテキスト分離が必要な作業
- 大規模ファイル解析（複数ファイル並列処理）

**直列実行を使う** ✅:
- 依存関係が強いステップ
- 単一ファイル編集
- デバッグ・検証フェーズ
- 状態管理が必要な作業

### 最大並列数

| チーム規模 | 推奨並列数 | 理由 |
|---|---|---|
| 小規模（〜5名） | 2-3 | コンテキスト予算・管理負荷 |
| 中規模（〜20名） | 3-5 | マシンリソース・コスト |
| 大規模（組織） | 5-10 | 分散実行・キューイング必須 |

**デフォルト: 3並列**（1subagent当たりコンテキスト予算は全体÷3）

### コンテキスト配分

1Mトークンモデルでの推奨配分:

```yaml
システムプロンプト + CLAUDE.md:     150k (15%)
ユーザーメッセージ:                 250k (25%)
過去会話履歴:                       200k (20%)
Subagent ワークスペース:            250k (25%)
バッファ（予約）:                   150k (15%)
```

→ **使用率30%目標**でセッション開始

---

## 6. 禁止事項 & 動作制限

### 🚫 厳禁

- [ ] `git push --force` （事前レビュー必須、mainへの絶対禁止）
- [ ] 本番APIキーをコード内に埋め込み（`.env.local` / `.secrets.yml` へ）
- [ ] Subagent数 > 10（コンテキスト枯渇リスク、要Plan）
- [ ] MCP Server無許可接続（セキュリティレビュー必須）
- [ ] Context window残量 < 10% での新Subagent起動
- [ ] `pytest --no-cov` での本番マージ（カバレッジ80%必須）
- [ ] 型チェック失敗のまま実装完了宣言

### ⚠️ 要事前レビュー（PR作成前にチーム相談）

- [ ] 新しいMCP統合（セキュリティレビュー含む）
- [ ] エージェント振る舞いの大規模変更（>100行変更）
- [ ] パッケージ追加（依存関係・セキュリティ確認）
- [ ] API仕様変更（互換性確認）
- [ ] コンテキスト予算の>20%変更

### ✅ 自動許可（事前レビュー不要）

- [ ] ローカルファイル編集（テスト含む）
- [ ] テスト実行
- [ ] ドキュメント更新（コードに影響なし）
- [ ] `.local.md` への個人設定記入
- [ ] Linter・型チェック修正

---

## 7. ファイル編集・テスト・ビルドポリシー

### テスト実行（必須）

```bash
# 単体テスト
pytest tests/ -v --cov=src --cov-fail-under=80

# 統合テスト（Subagent）
python -m pytest tests/integration/ -v -k "orchestration"

# E2E（MCP接続含む）
pytest tests/e2e/ --mcp-live --timeout=30

# Linter・型チェック
ruff check . && mypy src/ --strict && black --check .
```

**通過基準**: 
- カバレッジ ≥ 80%
- 全テスト PASS
- 型チェック 0エラー
- Linter 0警告

### ビルド & デプロイ

```bash
# ローカル検証
make build && make test

# ステージング
make deploy-staging && make smoke-test

# 本番（保護ブランチ、自動デプロイ）
git tag v1.x.x && git push origin v1.x.x
```

**リリース条件**:
1. develop ブランチが安定（全テスト PASS）
2. セキュリティレビュー完了
3. リードによる承認取得
4. タグ作成・自動デプロイ実行

---

## 8. セキュリティ & シークレット管理

### APIキー・認証情報

**絶対禁止**:
```python
# ❌ 悪い例
api_key = "sk-proj-abc123xyz"
```

**推奨方法**:
```bash
# .env.local （.gitignore に登録）
export ANTHROPIC_API_KEY="sk-..."
export DATABASE_PASSWORD="***"

# or environment variables
export $(cat .env.local | xargs)
```

### MCP Server接続時

MCP統合ファイル（`config/mcp-servers.yaml`）にて:
- セキュリティレビュー完了後のみ有効化
- 定期監査（月1回）実施
- 一度の接続エラーで自動無効化

---

## 9. 参考文献 & リソース

- **Claude Code Docs**: https://code.claude.com/docs/en/best-practices
- **Subagent Orchestration**: https://code.claude.com/docs/en/sub-agents
- **Claude API Reference**: https://platform.claude.com/docs/en/about-claude/models/overview
- **Anthropic Blog**: https://www.anthropic.com/news/claude-opus-4-7

---

## 10. Contact & Escalation

| 役割 | 連絡先 | 対応範囲 |
|---|---|---|
| **チーム技術リード** | `@team-lead` (GitHub) | コード品質、設計方針 |
| **セキュリティ懸念** | `security@company.internal` | キー管理、MCP接続 |
| **MCP統合サポート** | `@mcp-owner` | MCP Server設定、トラブル |

---

**最終チェック**:
- [ ] プレースホルダなし
- [ ] URL実在確認済み
- [ ] 完全に自動実行可能

✅ **本CLAUDE.md は即座に適用可能です**
