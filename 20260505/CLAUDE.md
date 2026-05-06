# CLAUDE.md - プロジェクト設定 & チーム共有ルール

**対象**: Harness Engineering チーム  
**スコープ**: プロジェクト層（`.claude/CLAUDE.md` 推奨）  
**最終更新**: 2026-05-05  
**行数**: 73行（ベストプラクティス 80行以下）

---

## 1. プロジェクト概要

**プロジェクト名**: Harness Engineering - AI Agent Orchestration Platform  
**言語スタック**: Python 3.10+, TypeScript, YAML  
**フレームワーク**: Claude Code, Claude API (Opus 4.7), Subagent SDK  
**チーム規模**: 小規模チーム（〜5名）  
**エージェント対象**: 上級者（Plan mode、Context最適化、Orchestration必須）

---

## 2. コーディング規約

### 命名規則

- **変数・関数**: `snake_case` (Python), `camelCase` (TypeScript)
- **クラス**: `PascalCase`
- **定数**: `UPPER_SNAKE_CASE`
- **ファイル**: `kebab-case.ts`, `snake_case.py`
- **Subagent**: `{role}_{objective}` (例: `researcher_literature`, `optimizer_cost`)

### コード品質

- **Python**: PEP 8準拠、`ruff` + `black` で自動フォーマット
- **TypeScript**: ESLint + Prettier、`strict` mode必須
- **コメント**: WHYのみ（WHAT は命名で表現）
- **関数長**: 30行以下推奨
- **複雑度**: 循環複雑度 ≤ 10

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
│   └── core/ (共通ライブラリ)
├── tests/
│   ├── agents/
│   ├── integration/
│   └── fixtures/
├── config/
│   ├── settings.yaml
│   └── agents.yaml
└── docs/
```

---

## 4. ワークフロー & チームプロセス

### コミット規約

```
<type>: <subject>

<body>

Refs: #<issue-number>
```

**type**: `feat`, `fix`, `refactor`, `test`, `docs`, `perf`

### ブランチ戦略

- `main`: 本番環境（保護ブランチ）
- `develop`: 統合ブランチ
- `feature/*`: 新機能
- `agent/*`: Subagent追加
- `fix/*`: バグ修正

### Pull Request ルール

- 必ず Plan mode（`.claude/AGENT.md` の `plan_mode` セクション参照）で整理した上で実装
- 同期レビュー: チーム内最小1名
- CI/CD: 全テスト・Linter通過が必須

---

## 5. Subagent & Orchestration ルール

### 使用シーン

**並列実行を使う**:
- 複数の独立研究タスク
- 外部API呼び出し（複数バッチ）
- コンテキスト分離が必要な作業

**直列実行を使う**:
- 依存関係が強いステップ
- 単一ファイル編集
- デバッグ・検証フェーズ

### 最大並列数

デフォルト: **3並列**  
コンテキスト予算: トークン消費量 ÷ 3（1subagent当たり目安）

---

## 6. 禁止事項 & 動作制限

### 🚫 厳禁

- [ ] `git push --force` （事前レビュー必須）
- [ ] 本番APIキーをコード内に埋め込み（`.env` / `.local.md` へ）
- [ ] Subagent数 > 10（コンテキスト枯渇リスク）
- [ ] MCP サーバ無許可接続（セキュリティレビュー必須）
- [ ] Context window残量 < 10% での新Subagent起動

### ⚠️ 要事前レビュー

- [ ] 新しいMCP統合
- [ ] エージェント振る舞いの大規模変更
- [ ] パッケージ追加（依存関係確認）
- [ ] API仕様変更

### ✅ 自動許可

- [ ] ローカルファイル編集
- [ ] テスト実行
- [ ] ドキュメント更新（コードに影響なし）
- [ ] `.local.md` への個人設定記入

---

## 7. ファイル編集・テスト・ビルドポリシー

### テスト実行

```bash
# 単体テスト
pytest tests/ -v --cov=src

# 統合テスト（Subagent）
python -m pytest tests/integration/ -v -k "orchestration"

# E2E（MCP接続含む）
pytest tests/e2e/ --mcp-live
```

**通過基準**: カバレッジ ≥ 80%, 全テスト PASS

### ビルド & デプロイ

```bash
# ローカル検証
make build && make test

# ステージング
make deploy-staging

# 本番（保護ブランチ、自動デプロイ）
git tag v1.x.x && git push origin v1.x.x
```

---

## 8. 参考文献 & リソース

- **Claude Code Docs**: https://code.claude.com/docs/en/best-practices
- **Subagent Orchestration**: https://claudefa.st/blog/guide/agents/sub-agent-best-practices
- **AI Agent Patterns**: https://monday.com/blog/ai-agents/ai-agent-architecture/

---

## 9. Contact & Escalation

- **チーム技術リード**: `@team-lead` (GitHub)
- **セキュリティ懸念**: `security@company.internal`
- **MCP統合サポート**: `@mcp-owner`
