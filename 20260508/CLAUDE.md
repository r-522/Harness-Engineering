# CLAUDE.md - ハーネスエンジニアリング基本規約

**プロジェクト**: Harness Engineering - AI Agent Orchestration Platform  
**対象**: Claude Code + Claude API (Opus 4.7) 用統合配置  
**有効期間**: 2026-05-08 〜  
**行数**: 95行（制限: 100行以下）

---

## 1. プロジェクト概要

**言語**: Python 3.10+, TypeScript, YAML  
**フレームワーク**: Claude Code, Anthropic SDK (Opus 4.7), Subagent SDK  
**規模**: 小規模チーム（〜5名）、上級者向け  
**主用途**: エージェント・オーケストレーション、Harness Engineering

---

## 2. コーディング規約

### 命名規則
- **変数・関数**: `snake_case` (Python), `camelCase` (TypeScript)
- **クラス**: `PascalCase`
- **定数**: `UPPER_SNAKE_CASE`
- **ファイル**: `kebab-case.ts`, `snake_case.py`
- **Agent**: `{role}_{objective}` 例: `plan_director`, `impl_worker`

### コード品質
- **Python**: PEP 8準拠、`ruff` + `black` で自動フォーマット
- **TypeScript**: ESLint + Prettier、`strict` mode必須
- **コメント**: WHYのみ（WHAT は命名で表現）
- **関数長**: 30行以下推奨
- **複雑度**: 循環複雑度 ≤ 10

---

## 3. ディレクトリ構成

```
project-root/
├── .claude/
│   ├── CLAUDE.md (このファイル)
│   ├── AGENT.md
│   ├── DESIGN.md
│   ├── ORCHESTRATION.md
│   └── HARNESS.md
├── src/
│   ├── agents/      (Agent定義)
│   ├── orchestrators/ (Task分割・管理)
│   ├── tools/       (Tool実装)
│   └── core/        (共通ライブラリ)
├── tests/
│   ├── agents/
│   ├── integration/
│   └── e2e/
└── config/
    └── agents.yaml
```

---

## 4. ワークフロー & チームプロセス

### コミット規約
```
<type>(<scope>): <subject>

<body>

Refs: #<issue-number>
```
**type**: `feat`, `fix`, `refactor`, `test`, `docs`, `perf`, `chore`

### ブランチ戦略
- `main`: 本番環境（保護ブランチ）
- `develop`: 統合ブランチ
- `feature/*`: 新機能
- `agent/*`: Subagent追加
- `fix/*`: バグ修正
- `claude/*`: Claude Code開発ブランチ

### Pull Request ルール
- 必ず Plan Mode で設計後に実装
- チーム内最小1名の同期レビュー
- 全テスト・Linter通過が必須（CI/CD）

---

## 5. Subagent & Orchestration ルール

### 使用シーン
**並列実行**: 複数独立研究、外部API呼び出し、コンテキスト分離  
**直列実行**: 強い依存関係、単一ファイル編集、デバッグ・検証

### 並列度上限
- **デフォルト**: 3並列
- **コンテキスト予算**: 1 agent当たり 200k〜300k tokens
- **Context rot**: 40%以上使用で品質低下（監視必須）

### Orchestrator 責務
- 純粋なコーディネーター（計画・分割のみ）
- 各 Subagent への責務は明確に
- 依存関係グラフを明示的に指定

---

## 6. 禁止事項 & 動作制限

### 🚫 厳禁
- [ ] `git push --force` （事前レビュー必須）
- [ ] 本番APIキーをコード内に埋め込み（`.env` / `.local.md` へ）
- [ ] Subagent数 > 10（コンテキスト枯渇リスク）
- [ ] MCP サーバ無許可接続（セキュリティレビュー必須）
- [ ] Context残量 < 10% での新 Subagent 起動

### ⚠️ 要事前レビュー（チーム相談）
- [ ] 新しいMCP統合
- [ ] Agent振る舞いの大規模変更
- [ ] 依存パッケージ追加
- [ ] API仕様変更

### ✅ 自動許可（個人判断OK）
- [ ] ローカルファイル編集
- [ ] テスト実行
- [ ] ドキュメント更新（コード影響なし）
- [ ] `.local.md` への設定記入

---

## 7. テスト・ビルド・デプロイポリシー

### テスト実行（必須）
```bash
pytest tests/ -v --cov=src
```
**通過基準**: カバレッジ ≥ 80%, 全テスト PASS

### Linter実行
```bash
ruff check . && black --check .
```

### ビルド & デプロイ
```bash
make build && make test  # ローカル検証
make deploy-staging      # ステージング
git tag v1.x.x && git push origin v1.x.x  # 本番（自動デプロイ）
```

---

## 8. Context Management（重要）

### Token予算管理
- **Model**: Claude Opus 4.7（200k context window）
- **安全域**: 40%以下（80k tokens）推奨
- **危険域**: 60%以上（120k tokens）で計画ミス多発
- **Context rot**: 300-400k token消費で出力品質低下（Session切り替え必須）

### コンテキスト脱却戦略
1. 不要な履歴・ログを削除
2. 大規模ファイルは路径指定で参照（全文埋め込み禁止）
3. Subagent にタスク委譲してコンテキスト分散
4. Session 切り替え（新規ブランチ・新規Agentで再開）

---

## 9. Session & Worktree管理

### 並列セッション実行
```bash
# セッション1: Orchestrator
git worktree add ../work1 develop
cd ../work1 && claude-code

# セッション2: Implementation
git worktree add ../work2 develop
cd ../work2 && claude-code

# セッション3: Testing
git worktree add ../work3 develop
cd ../work3 && claude-code
```

### マージ & 衝突解決
- 各worktree で独立開発（同一ファイル編集禁止）
- 統合点で explicit rebase/merge
- テストを通してから main へ

---

## 10. Contact & Escalation

- **技術リード**: チーム相談（GitHub Issue）
- **セキュリティ懸念**: Security Review実施
- **MCP統合**: HARNESS.md 参照

---

**最後に**: `.claude/AGENT.md` でAgentのペルソナと思考プロセスを確認してください。
