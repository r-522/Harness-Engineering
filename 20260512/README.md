# ハーネスエンジニアリング設定ガイド 2026

**生成日**: 2026-05-12  
**情報収集期間**: 過去24時間  
**対象プロジェクト**: Harness Engineering - AI Agent Orchestration Platform

---

## 概要

Harness-Engineering プロジェクト向けに、2026年5月の最新ベストプラクティスを統合した **Agent Teams・Subagent・Prompt Caching** 対応の設定ファイル群です。1M Token Context、並列Agent実行、Prompt Cachingによるコスト90%削減を実現するハーネスを構築します。

---

## 属性判定と根拠

| 属性 | 判定結果 | 判定根拠 |
|---|---|---|
| **用途** | AIエージェント・オーケストレーション開発 | Agent Teams、Multi-agent Orchestration、Subagent並列実行が中核。Claude Code Agent Teamsで3-5名の協調エージェント推奨 |
| **規模** | 小規模チーム（〜5名） | プロジェクト既存CLAUDE.mdで確認。Agent Teamsベストプラクティスと一致 |
| **対象** | 上級者向け | Prompt Caching、MCP統合、Context最適化、Subagent管理、並列実行制御が必須。Plan modeでの設計が前提 |

### 2026年新展開のキーポイント

1. **1M Token Context**: Opus 4.6 / Sonnet 4.6 で追加料金なしで利用可能（3月時点）
2. **Agent Teams**: 3-5名のCoordinator + Teammates 構成による垂直スライス開発
3. **Prompt Caching**: システム命令・ツール定義を不変プレフィックスとして配置し、**コスト90%削減・遅延80%削減** を実現
4. **Skills 組織展開**: Skills.md による組織レベルの自動デプロイ・更新管理
5. **MCP Apps**: 2026年1月から、MCP ツールが会話内でインタラクティブUI（ChatGPT/Claude/VS Code互換）をレンダリング可能

---

## ファイル構成と役割

| ファイル | 役割 | 対象読者 |
|---|---|---|
| **CLAUDE.md** | プロジェクト制約・コーディング規約・禁止事項 | 全メンバー（初回読み必須） |
| **AGENT.md** | Claude/Subagent ペルソナ・思考フロー・スキル定義 | 上級者・Orchestration担当者 |
| **DESIGN.md** | 設計原則（SOLID/DRY）・アーキテクチャパターン | エージェント設計担当者 |
| **ORCHESTRATION.md** | Agent Teams/Subagent/並列実行の詳細ガイド | Orchestration設計者 |
| **CACHING.md** | Prompt Caching 戦略・キャッシュ管理・CI/CD検証 | API実装者・パフォーマンス担当 |

---

## ディレクトリ構成（推奨レイアウト）

```
project-root/
├── .claude/
│   ├── CLAUDE.md ................... プロジェクト制約・コード規約
│   ├── AGENT.md .................... エージェント定義・ペルソナ
│   ├── DESIGN.md ................... 設計原則・アーキテクチャ
│   ├── agents/
│   │   ├── orchestrator.md ......... Coordinator agent 定義
│   │   ├── researcher.md .......... 情報収集 subagent
│   │   ├── implementer.md ......... 実装 subagent
│   │   ├── reviewer.md ............ 審査 subagent
│   │   └── debugger.md ............ デバッグ subagent
│   ├── skills/
│   │   ├── prompt-caching.md ...... キャッシング戦略スキル
│   │   ├── mcp-integration.md ..... MCP接続スキル
│   │   └── orchestration.md ....... オーケストレーションスキル
│   └── hooks/
│       ├── pre-commit.sh .......... Linter・型チェック
│       └── pre-push.sh ............ CI/CD検証
├── src/
│   ├── agents/ .................... Subagent実装
│   ├── skills/ .................... カスタムスキル
│   ├── mcp/ ....................... MCP統合
│   └── core/ ...................... 共通ライブラリ
├── config/
│   ├── settings.yaml .............. プロジェクト設定
│   ├── agents.yaml ................ Agent Teams定義
│   └── cache-config.yaml .......... Prompt Cache戦略
└── tests/
    ├── integration/ ............... Subagent並列実行テスト
    └── caching/ ................... キャッシュヒット率検証
```

---

## 導入手順（クイックスタート）

### 1. 設定ファイルの配置
```bash
# 20260512 ディレクトリから .claude/ へコピー
cp -r /path/to/20260512/* /path/to/Harness-Engineering/.claude/
```

### 2. 既存CLAUDE.mdとマージ
```bash
# 既存の CLAUDE.md をバックアップ
cp .claude/CLAUDE.md .claude/CLAUDE.md.backup

# 新しい CLAUDE.md を適用（最新トレンドを反映）
# コンフリクトの有無を確認し、重要な既存ルールは保持
```

### 3. Agent Teams の初期化
```bash
# orchestrator.md を読み込み、Coordinator agent として起動
# 同時に researcher, implementer, reviewer subagent を定義

# settings.yaml で並列実行数を設定
# max_parallel_subagents: 3
# context_per_agent: 256000  # 1M context を均等分配
```

### 4. Prompt Caching の有効化
```bash
# cache-config.yaml で以下を設定
# - 不変プレフィックス: システム命令・ツール定義
# - キャッシュキー管理戦略
# - CI/CD でキャッシュヒット率を検証（閾値: 60% 以上）
```

### 5. MCP サーバの登録
CACHING.md 内 "MCP統合" セクションで、組織で公認されたMCPサーバの接続手順を確認。セキュリティレビュー必須。

---

## カスタマイズガイド

### コンテキストに応じた調整

#### A. 組織規模が拡大した場合（5名 → 20名）
1. **CLAUDE.md**: "規模" セクションを `中規模チーム(〜20名)` に更新
2. **AGENT.md**: Subagent数を5 → 10 に増加、メモリ分離ルール強化
3. **settings.yaml**: `max_parallel_subagents` を 5 に引き上げ、キャッシュプール管理を導入
4. **ORCHESTRATION.md**: 複数Coordinator構成、DynamoDB等での状態管理を追加

#### B. 新しい言語スタック導入（Python/TypeScript → Go/Rust）
1. **CLAUDE.md**: "コーディング規約" に新言語セクション追加
2. **DESIGN.md**: 言語別のアーキテクチャパターン例を追加
3. **tests/**: 新言語向けのテスト例を追加

#### C. セキュリティ要件が高まった場合
1. **CLAUDE.md**: "禁止事項" に暗号化・監査ロギング要件を追加
2. **新規作成**: `SECURITY.md` ファイルを追加（MCP接続・API キー管理・コンテキスト隔離）
3. **hooks/**: セキュリティスキャン（secret scanning）を pre-push に統合

---

## 参考文献（2026-05-12 収集）

### 1. Claude Code Agent Teams & Orchestration
- **[Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)** — Agent Teams の公式ドキュメント。Coordinator・Teammates概念、inter-agent messaging、タスク分配戦略
- **[Claude Code Agent Teams: Setup & Usage Guide 2026](https://claudefa.st/blog/guide/agents/agent-teams)** — 3-5名構成の利点、垂直スライス開発パターン、メモリ隔離
- **[GitHub - shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)** — 実装例を含む CLAUDE.md テンプレート（Star 800+）
- **[GitHub - wshobson/agents](https://github.com/wshobson/agents)** — Multi-agent orchestration ライブラリ実装例

### 2. Subagent & SDK 統合
- **[Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)** — Subagent 定義・トリガー・権限管理
- **[Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)** — Anthropic公式エンジニアリングブログ。SDKベストプラクティス
- **[Subagents in the SDK - Claude API Docs](https://platform.claude.com/docs/en/agent-sdk/subagents)** — Python / TypeScript SDK での Subagent 実装
- **[Claude Code Advanced Patterns: Subagents, MCP, and Scaling](https://resources.anthropic.com/hubfs/Claude%20Code%20Advanced%20Patterns_%20Subagents,%20MCP,%20and%20Scaling%20to%20Real%20Codebases.pdf)** — PDF リソース

### 3. Prompt Caching & コスト最適化
- **[Prompt caching - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)** — 公式ドキュメント。90%コスト削減、80%遅延削減の実績値
- **[Prompt Caching for AI Agents - Medium (April 2026)](https://medium.com/@arvisionlab/prompt-caching-for-ai-agents-how-to-cut-cost-and-latency-without-breaking-context-245dc2502b4b)** — 最新事例。タイムスタンプによるキャッシュ無効化の落とし穴も記述
- **[Redis: Prompt caching vs semantic caching](https://redis.io/blog/prompt-caching-vs-semantic-caching-how-to-make-ai-agents-faster/)** — キャッシング戦略比較

### 4. Skills & MCP 統合
- **[Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)** — Skill 定義・トリガー・組織レベルデプロイ
- **[Extending Claude's capabilities with skills and MCP](https://claude.com/blog/extending-claude-capabilities-with-skills-mcp-servers)** — Claude Blog。Skills（内部プレイブック）vs MCP（神経系）の相補関係
- **[The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf)** — PDF 完全ガイド
- **[Claude Skills vs MCP: A Technical Comparison 2026](https://intuitionlabs.ai/articles/claude-skills-vs-mcp)** — ユースケース別の使い分けマトリックス

### 5. Context管理 & メモリ隔離
- **[Claude Code & Agent Memory: Best Practices for 2026](https://orchestrator.dev/blog/2026-04-06--claude-code-agent-memory-2026/)** — 1M Token Context の活用、Subagent間メモリ隔離、Context最適化手法
- **[Multiagent sessions - Claude API Docs](https://platform.claude.com/docs/en/managed-agents/multi-agent)** — API レベルでの並列Agent管理

---

## チェックリスト（導入完了判定）

- [ ] CLAUDE.md をすべてのメンバーが読了・サイン
- [ ] Agent Teams 構成を設定（Coordinator 1 + Teammates 3-4）
- [ ] 各 Subagent に独立した `agents/` ファイルを定義
- [ ] Prompt Caching を有効化し、キャッシュヒット率を CI/CD で監視（ダッシュボード設置）
- [ ] MCP サーバ（GitHub/Slack/Notion等）をセキュリティレビュー後に接続
- [ ] skills/ ディレクトリに 3+ の カスタムスキルを定義
- [ ] テスト: `pytest tests/integration/ -k orchestration` が全 PASS
- [ ] ドキュメント: README.md、AGENT.md、DESIGN.md を社内 Wiki に掲載

---

**更新方針**: 翌月（2026-06）に再度Web検索を実施し、新トレンドがあれば差分更新します。
