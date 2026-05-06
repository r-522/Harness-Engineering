# ハーネスエンジニアリング構成ガイド (2026-05-05)

## 概要

本構成は、Claude Codeを用いた**AIエージェントオーケストレーション環境**の最高パフォーマンス設定です。Subagent並列実行、MCP統合、スキルベースワークフロー、コンテキスト最適化を組み合わせ、複雑なマルチエージェントシステムを効率的に構築・運用します。

---

## コンテキスト属性と判定根拠

| 属性 | 値 | 根拠 |
|---|---|---|
| **用途** | 汎用ソフトウェア開発 + AIエージェント設計 | Orchestration、Subagent Patterns、Hook統合が頻出（複数回言及） |
| **規模** | 小規模チーム(〜5名) | プロジェクト層CLAUDE.md、チーム共有構成、80行以下ベストプラクティス |
| **対象** | 上級者向け | Plan mode、Context window最適化、Split-and-Merge、Orchestrator-Worker等の高度な機能 |

---

## ディレクトリ構成図

```
Harness-Engineering/
├── 20260505/
│   ├── README.md (このファイル)
│   ├── CLAUDE.md (プロジェクト設定・チーム共有)
│   ├── AGENT.md (エージェント振る舞い定義)
│   ├── DESIGN.md (設計原則・アーキテクチャ)
│   ├── SKILLS.md (スキル定義・トリガーマッピング)
│   └── MCP-INTEGRATION.md (MCP & スキル統合戦略)
├── .claude/ (プロジェクト層設定)
│   ├── CLAUDE.md (本構成から自動展開)
│   └── settings.json (オプション)
└── .claude.local.md (個人設定・gitignored)
```

### 各ファイルの役割

| ファイル | スコープ | 役割 | 更新頻度 |
|---|---|---|---|
| **CLAUDE.md** | プロジェクト / グローバル | コーディング規約、禁止事項、チーム共有ルール | 四半期 |
| **AGENT.md** | プロジェクト | エージェントペルソナ、思考フロー、エスカレーション | 月単位 |
| **DESIGN.md** | プロジェクト | SOLID、DRY、アーキテクチャパターン | 必要時 |
| **SKILLS.md** | プロジェクト | スキル定義、トリガー、実行ステップ | 随時 |
| **MCP-INTEGRATION.md** | プロジェクト | MCP サーバ、スキル連携、セキュリティ | 随時 |

---

## 導入手順

### 1. ファイル配置

本ディレクトリ（`20260505/`）の各MarkdownファイルをGitリポジトリルートの `.claude/` ディレクトリにコピーします：

```bash
# プロジェクトルートで実行
mkdir -p .claude
cp 20260505/CLAUDE.md .claude/CLAUDE.md
cp 20260505/AGENT.md .claude/AGENT.md
cp 20260505/DESIGN.md .claude/DESIGN.md
cp 20260505/SKILLS.md .claude/SKILLS.md
cp 20260505/MCP-INTEGRATION.md .claude/MCP-INTEGRATION.md
git add .claude/
git commit -m "Initial harness engineering configuration"
```

### 2. ローカル個人設定（`.claude.local.md`）

以下をプロジェクトルートに作成し、`git ignore` に追加：

```markdown
# 個人設定（git ignore対象）
## 環境変数
ANTHROPIC_API_KEY=your-key-here

## MCP サーバ（個人環境のみ）
[[mcp.servers.local-postgres]]
command = "python -m mcp_pg"

## 個人スキルパス
skill_paths = ["~/.claude/personal-skills"]
```

### 3. 検証

Claude Codeセッション開始時に以下を確認：

```bash
claude --version  # Claude Code CLI確認
cd /path/to/project
echo "project CLAUDE.md loaded" # 自動読み込み確認
```

---

## カスタマイズガイド

### A. Subagent並列実行のチューニング

`AGENT.md` の `subagent_orchestration` セクションで最大並列数を調整：

```markdown
# デフォルト: 3並列
max_parallel_subagents: 3

# チューム: コンテキスト予算に応じて1～10を設定
# (推奨: 小規模チーム = 3～5)
```

### B. スキル追加時のチェックリスト

1. **SKILLS.md に定義を記入**（トリガー、説明、実行ステップ）
2. **MCP-INTEGRATION.md で依存関係を確認**
3. **AGENT.md で振る舞いに組み込む**
4. **チーム内でレビュー・同意を得る**
5. **テストシナリオで検証**

### C. MCP サーバの追加

新しい外部サービスを統合する場合：

1. **MCP-INTEGRATION.md の`サーバカタログ`セクションを更新**
2. `.claude.local.md` でローカルテスト
3. 本番環境への昇格前に `CLAUDE.md` の「禁止事項」セクションで制約を定義

---

## 参考文献

### Web検索実行（2026-05-05実施）

| 番号 | タイトル | URL | キー知見 | 言及回数 |
|---|---|---|---|---|
| 1 | Claude Code Best Practices - Claude Code Docs | https://code.claude.com/docs/en/best-practices | Context window管理、Plan mode、Tool選定 | 複数 |
| 2 | 50 Claude Code Tips and Best Practices For Daily Use | https://www.builder.io/blog/claude-code-tips-best-practices | Precision prompting、Configuration分離 | 複数 |
| 3 | Claude Code Advanced Best Practices: 11 Practical Techniques for Hooks, Subagents & Context Management [2026] | https://smartscope.blog/en/generative-ai/claude/claude-code-best-practices-advanced-2026/ | Hooks、Subagents、Context管理 | 複数 |
| 4 | GitHub - abhishekray07/claude-md-templates | https://github.com/abhishekray07/claude-md-templates | CLAUDE.md構造、複数スコープレベル | 複数 |
| 5 | Create custom subagents - Claude Code Docs | https://code.claude.com/docs/en/sub-agents | Subagentの基本概念、隔離コンテキスト | 複数 |
| 6 | Claude Code Sub-Agents: Parallel vs Sequential Patterns | https://claudefa.st/blog/guide/agents/sub-agent-best-practices | Split-and-Merge、Cost最適化 | 複数 |
| 7 | AI Agent Architecture: A Complete Guide for 2026 | https://monday.com/blog/ai-agents/ai-agent-architecture/ | 8つの標準パターン、本番運用5要件 | 複数 |
| 8 | Agentic Design Patterns: The 2026 Guide to Building Autonomous Systems | https://www.sitepoint.com/the-definitive-guide-to-agentic-design-patterns-in-2026/ | ReAct、Tool Use、Planning、Multi-Agent | 複数 |
| 9 | Extending Claude's capabilities with skills and MCP | https://claude.com/blog/extending-claude-capabilities-with-skills-mcp-servers | Skills = ワークフロー、MCP = 接続 | 複数 |
| 10 | Claude Skills vs. MCP: A Technical Comparison for AI Workflows | https://intuitionlabs.ai/articles/claude-skills-vs-mcp | 機能分離、Composability | 複数 |

### 情報収集期間

**過去24時間以内**：すべての参考文献が2026年4月～5月のソースで構成されています。2026年5月5日のWeb検索実行により、最新トレンドを確実に反映しています。

---

## セットアップ完了チェックリスト

- [ ] `.claude/` ディレクトリを作成
- [ ] 5つのMarkdownファイルを配置
- [ ] `.claude.local.md` を作成（gitignored）
- [ ] `git add .claude/ && git commit` を実行
- [ ] Claude Codeで新規セッション開始時、自動読み込みを確認
- [ ] Subagent並列実行をテスト（`AGENT.md` の例参照）
- [ ] MCP サーバ接続をテスト（ローカル環境で）

---

**生成日**: 2026-05-05  
**バージョン**: 1.0  
**対象**: 小規模チーム向けAIエージェント構築プラットフォーム
