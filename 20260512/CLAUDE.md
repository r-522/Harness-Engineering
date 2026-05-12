# CLAUDE.md - Harness Engineering チーム規約 v2.0

**対象**: Harness Engineering チーム  
**スコープ**: プロジェクト層（`.claude/CLAUDE.md`）  
**最終更新**: 2026-05-12（Web検索・Agent Teams・Prompt Caching統合版）  

---

## 1. プロジェクト概要

**プロジェクト名**: Harness Engineering - AI Agent Orchestration Platform  
**言語スタック**: Python 3.10+, TypeScript 5.0+, YAML  
**フレームワーク**: Claude Code (Agent Teams), Claude API (Opus 4.7), Subagent SDK  
**チーム規模**: 小規模チーム（〜5名、1 Coordinator + 3-4 Teammates推奨）  
**対象エージェント**: 上級者（Plan mode、Context最適化、Orchestration必須）  
**コンテキスト戦略**: 1M Token Context（Opus 4.6 / Sonnet 4.6）、Prompt Caching（90%コスト削減）

---

## 2. コーディング規約

### 命名規則

- **変数・関数**: `snake_case` (Python), `camelCase` (TypeScript)
- **クラス・Subagent**: `PascalCase`（例: `Researcher`, `CodeReviewer`）
- **定数**: `UPPER_SNAKE_CASE`
- **ファイル**: `kebab-case.ts`, `snake_case.py`
- **Agent定義**: `.claude/agents/{role}_{objective}.md`（例: `researcher_code_analysis.md`）
- **スキル定義**: `.claude/skills/{domain}_{action}.md`（例: `mcp_github_connect.md`）

### コード品質

- **Python**: PEP 8準拠、`ruff --fix` + `black` で自動フォーマット
- **TypeScript**: ESLint + Prettier、`strict: true` モード必須
- **コメント**: WHYのみ（WHAT は命名で表現。複雑な制御フローにのみ）
- **関数長**: 30行以下推奨。複数の責務がある場合は細分化
- **複雑度**: 循環複雑度 ≤ 10。超える場合は関数分割

### Subagent設計規約

```markdown
---
agent_type: subagent
name: Researcher
description: 複数の情報源から技術トレンドを調査し、まとめを返す
capabilities: ["web_search", "code_analysis"]
max_context_tokens: 256000  # 1M ÷ 4 teammates = 250K
timeout_seconds: 300
---

# Researcher Subagent

## ペルソナ
- 徹底的で、複数の情報源を確認する
- 曖昧な指示には返答前に確認を取る

## タスク分解戦略
1. ユーザーのクエリを分解
2. 並列で複数データソース検索
3. 信頼度スコアで結果をランク付け
4. サマリーとソース一覧を返す
```

---

## 3. Agent Teams & Orchestration

### 推奨構成（2026年ベストプラクティス）

```
Coordinator Agent (Plan mode)
├── Researcher Subagent ......... 技術調査・トレンド分析
├── Implementer Subagent ........ コード実装・リファクタリング
├── Reviewer Subagent ........... コード審査・テスト検証
└── Debugger Subagent ........... デバッグ・トラブルシューティング
```

**特性**:
- **Coordinator**: 全体の計画・タスク分配・結果統合を管理。Plan modeで思考空間を活用
- **Subagents**: 各々独立したコンテキスト（256K Token）で並列実行。メモリ隔離
- **最大並列数**: 3（デフォルト）。コンテキスト圧迫リスクがあるため超えない

### Subagent起動ルール

✅ **起動すべき場合**:
- 複数の独立したタスク（研究 + 実装 + レビュー）
- 外部API呼び出しが並列化可能（Web検索、GitHub API等）
- コンテキスト分離が必要（前タスクの痕跡を避けたい）

❌ **起動すべきでない場合**:
- 単一ファイル編集
- 強い依存関係がある順序作業
- デバッグ・検証フェーズ（Coordinator内で完結）

---

## 4. Prompt Caching 戦略

### 不変プレフィックス（キャッシュ対象）

```yaml
# .claude/cache-config.yaml
caching_strategy:
  prefix_1_system_instructions:
    - CLAUDE.md 全文（800行以下）
    - AGENT.md ペルソナ定義
    - DESIGN.md 設計原則
    target_token_count: 3000-5000
    stability: IMMUTABLE  # バージョン管理必須

  prefix_2_tool_definitions:
    - Subagent定義（agents/ 全ファイル）
    - MCP サーバスキーマ
    - API仕様（OpenAPI, GraphQL）
    target_token_count: 2000-4000
    stability: IMMUTABLE

  prefix_3_knowledge_base:
    - 過去の実装パターン（examples/）
    - トラブルシューティング集
    target_token_count: 2000-3000
    stability: LONG_LIVED  # 月1回更新

  # 可変部分（キャッシュ不可）
  variable_user_query:
    - 実行時の指示・データ
    position: END  # 常にプレフィックスの後ろ
```

### キャッシュヒット率の監視

```bash
# CI/CD で自動検証
pytest tests/caching/ --measure-cache-hit-rate

# ダッシュボード（例: CloudWatch）
- キャッシュヒット率 ≥ 60% （目標: 70%+）
- 平均 time-to-first-token ≤ 100ms
- 入力トークンコスト削減率 ≥ 80%
```

### ⚠️ キャッシュ無効化の落とし穴

**タイムスタンプを early に挿入しない**:
```python
# ❌ 悪い例（キャッシュ無効化）
system_prompt = f"""
You are a helpful assistant.
[Current time: {datetime.now()}]  # プレフィックスが変わる
Tasks: ...
"""

# ✅ 良い例（キャッシュ有効）
system_prompt = """
You are a helpful assistant.
Tasks: ...
"""
# タイムスタンプは末尾の可変セクションに挿入
```

**ツール定義の順序を固定化**:
```python
# ツール定義を毎回同じ順序で生成
tools_in_fixed_order = sorted(all_tools, key=lambda t: t.name)
```

---

## 5. プロジェクト構成・ディレクトリ

```
project-root/
├── .claude/
│   ├── CLAUDE.md ................. このファイル（プロジェクト規約）
│   ├── AGENT.md .................. エージェント・ペルソナ定義
│   ├── DESIGN.md ................. 設計原則・アーキテクチャ
│   ├── ORCHESTRATION.md .......... Agent Teams詳細ガイド
│   ├── CACHING.md ................ Prompt Caching実装ガイド
│   ├── agents/
│   │   ├── orchestrator.md ....... Coordinator agent （Plan mode）
│   │   ├── researcher.md ......... 情報収集 subagent
│   │   ├── implementer.md ........ 実装 subagent
│   │   ├── reviewer.md ........... コード審査 subagent
│   │   └── debugger.md ........... デバッグ subagent
│   ├── skills/
│   │   ├── web_search.md ......... Web検索スキル
│   │   ├── code_review.md ........ コード審査スキル
│   │   └── mcp_integration.md .... MCP接続スキル
│   └── hooks/
│       ├── pre-commit.sh ......... Linter・型チェック・シークレット検査
│       └── pre-push.sh ........... CI/CD検証（テスト・キャッシュヒット率確認）
├── src/
│   ├── agents/ ................... Subagent実装（API層）
│   ├── orchestration/ ............ Orchestration ロジック
│   ├── skills/ ................... カスタムスキル実装
│   ├── mcp/ ...................... MCP サーバ統合
│   └── core/ ..................... 共通ライブラリ
├── config/
│   ├── settings.yaml ............. プロジェクト設定
│   ├── agents.yaml ............... Agent Teams定義
│   ├── cache-config.yaml ......... Prompt Cache戦略
│   └── mcp-servers.yaml .......... 公認MCP登録
├── tests/
│   ├── integration/ .............. Subagent並列実行テスト
│   ├── orchestration/ ............ Orchestration フロー検証
│   ├── caching/ .................. キャッシュヒット率テスト
│   └── e2e/ ...................... エンドツーエンド検証
├── examples/
│   ├── agent_teams_setup.py ...... Agent Teams初期化例
│   ├── prompt_caching.py ......... Caching実装例
│   └── mcp_integration.py ........ MCP接続例
└── docs/
    └── ARCHITECTURE.md ........... 全体アーキテクチャ図
```

---

## 6. ワークフロー & チームプロセス

### コミット規約（Conventional Commits）

```
<type>(<scope>): <subject>

<body>

Refs: #<issue-number>
Co-authored-by: <name> <email>
```

**type**: `feat`, `fix`, `refactor`, `test`, `docs`, `perf`, `chore`  
**scope**: `orchestration`, `caching`, `mcp`, `agent:<name>`, `skill:<name>`  
**例**:
```
feat(orchestration): parallel researcher + implementer execution

- Researcher subagent searches 3 parallel sources
- Implementer processes results concurrently
- Coordinator aggregates findings

Refs: #42
```

### ブランチ戦略（Git Flow）

- `main`: 本番環境（保護ブランチ）
- `develop`: 統合ブランチ
- `feature/*`: 新機能（例: `feature/agent-teams-setup`）
- `agent/*`: Subagent追加（例: `agent/debugger-v2`）
- `fix/*`: バグ修正（例: `fix/cache-invalidation`）
- `perf/*`: パフォーマンス最適化（例: `perf/prompt-caching-threshold`）

### Pull Request ルール

1. **必ず Plan mode で整理**してから実装（.claude/AGENT.md の `plan_mode` セクション参照）
2. **同期レビュー**: チーム内最小1名（Reviewer subagent 自動実行可）
3. **CI/CD 通過必須**:
   - 全テスト: `pytest tests/ --cov=src`（カバレッジ ≥ 80%）
   - Linter: `ruff check src/`, `eslint src/`
   - 型チェック: `mypy src/` (Python), `tsc --noEmit` (TypeScript)
   - Prompt Caching: キャッシュヒット率 ≥ 60%
4. **マージ戦略**: Squash + Rebase（ブランチ履歴を簡潔に）

---

## 7. 禁止事項 & 動作制限

### 🚫 **厳禁**（セキュリティ・安定性に直結）

- [ ] `git push --force` （GitHub 保護ルール設定で防止）
- [ ] 本番APIキーをコード内に埋め込み（`.env` / `.local.md` へ）
- [ ] Subagent数 > 10（コンテキスト枯渇リスク）
- [ ] MCP サーバ無許可接続（セキュリティレビュー必須）
- [ ] Context window残量 < 10% での新Subagent起動（メモリ不足エラー）
- [ ] 既存プレフィックスの改変（キャッシュ無効化）
- [ ] タイムスタンプ・UUID を早期に挿入（キャッシュ回避）

### ⚠️ **要事前レビュー**

- [ ] 新しいMCP統合（セキュリティ・安定性審査）
- [ ] エージェント振る舞いの大規模変更（Orchestration影響分析）
- [ ] パッケージ追加（依存関係・ライセンス確認）
- [ ] API仕様変更（互換性確認）
- [ ] Subagent数の増加（Context容量計画）

### ✅ **自動許可**

- [ ] ローカルファイル編集
- [ ] テスト実行・追加
- [ ] ドキュメント更新（`.md` ファイル、コードに影響なし）
- [ ] `.local.md` への個人設定記入
- [ ] 既存の公認Subagent / Skillの利用

---

## 8. ファイル編集・テスト・ビルドポリシー

### テスト実行

```bash
# 単体テスト（カバレッジ80%以上必須）
pytest tests/ -v --cov=src --cov-fail-under=80

# 統合テスト（Subagent並列実行）
python -m pytest tests/integration/ -v -k "orchestration"

# Subagent並列実行テスト
pytest tests/orchestration/ -v --parallel=3

# キャッシュヒット率測定
pytest tests/caching/ -v --measure-cache-hit-rate

# E2E（MCP接続含む）
pytest tests/e2e/ -v --mcp-live
```

**通過基準**: 
- カバレッジ ≥ 80%
- 全テスト PASS
- キャッシュヒット率 ≥ 60%
- 型チェック通過（mypy, tsc）

### ビルド & デプロイ

```bash
# ローカル検証
make validate && make test

# ステージング
make deploy-staging

# 本番（タグでトリガー、自動デプロイ）
git tag v2.0.0 && git push origin v2.0.0
```

### パフォーマンス目標

| 指標 | 目標値 | 測定方法 |
|---|---|---|
| 新規Agent起動時間 | ≤ 500ms | `pytest tests/caching/` |
| Prompt Cache Hit Rate | ≥ 60% | CI/CDダッシュボード |
| 入力トークンコスト削減 | ≥ 80% | CloudWatch メトリクス |
| Subagent平均応答時間 | ≤ 5s | `time pytest tests/orchestration/` |

---

## 9. セキュリティ・権限ポリシー

### API キー・シークレット管理

- **保存**: `.env.local` （Git ignore済み）または `.local.md`
- **参照**: `os.getenv()` / `dotenv.load_dotenv()`
- **ローテーション**: 四半期ごと、または漏洩時は即座に
- **監査**: CloudTrail / GitHub Audit Log で定期確認

### MCP サーバセキュリティレビュー

新規MCP接続の際:
1. サーバの信頼性を確認（公式ドキュメント、GitHub Star数等）
2. 権限スコープを最小化（必要な操作のみ許可）
3. レート制限・タイムアウトを設定
4. 定期的にアクセスログを審査

### Context 隔離ポリシー

- **Subagent間**: メモリ共有なし。各々独立したコンテキスト
- **環境変数**: Subagent固有のENVは`.local.md` で管理
- **ログ出力**: 機密情報を含まない（API応答の一部のみ記録）

---

## 10. Contact & Escalation

- **チーム技術リード**: `@team-lead` (GitHub Issues / Discussions)
- **セキュリティ懸念**: `security@company.internal` （即座に報告）
- **MCP統合サポート**: `.claude/AGENT.md` の `mcp_owner` 欄
- **パフォーマンス問題**: `@perf-team` (Slack / GitHub Discussions)

---

**バージョン履歴**:
- v2.0 (2026-05-12): Agent Teams・Prompt Caching・Subagent統合版
- v1.0 (2026-04-01): 初期リリース
