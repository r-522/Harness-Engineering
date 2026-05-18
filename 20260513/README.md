# Claude Code ハーネスエンジニアリング設定パッケージ v1.0

**生成日時**: 2026-05-13  
**対象**: 小規模～中規模開発チーム（上級者向け）  
**情報収集期間**: 過去7日間（2026年4月末～5月13日）

---

## 📋 概要

このパッケージは、Claude Code のポテンシャルを最大化するための「ハーネスエンジニアリング」設定ファイル群です。以下の3つの目標を実現します：

1. **コンテキスト最適化**: トークン予算を効率的に管理し、400k〜600k トークンウィンドウ内での最大パフォーマンス発揮
2. **マルチエージェント調整**: Subagent並列実行とMCP統合を通じた分散タスク処理
3. **チーム開発標準化**: 小〜中規模チーム（5〜20名）における一貫した設定と指標

---

## 🎯 判定根拠

| 属性 | 決定値 | 根拠 |
|------|------|------|
| **用途** | 汎用ソフトウェア開発 | CLAUDE.md、Plan mode、Code quality governance が最頻出（8件以上） |
| **規模** | 小規模～中規模チーム(5〜20名) | Orchestration、複数Agent協調、チームプロセスの言及が多数 |
| **対象** | 上級者向け | Subagent、MCP統合、Context最適化が中核テーマ。初心者向けではない |

---

## 📁 ディレクトリ構成と各ファイルの役割

```
project-root/
├── .claude/
│   ├── CLAUDE.md              # プロジェクト固有制約・コーディング規約（80行以下推奨）
│   ├── AGENT.md               # エージェント振る舞い・ペルソナ・思考プロセス
│   ├── DESIGN.md              # 設計原則・アーキテクチャパターン
│   ├── MCP.md                 # MCP統合・サーバー選定・接続指針
│   ├── SUBAGENT.md            # Subagent orchestration・並列パターン
│   ├── SKILLS.md              # Skills定義・トリガー条件・実装例
│   └── settings.json          # Claude Code環境設定（hooks、permissions）
├── src/
│   ├── agents/                # Subagent定義（role_objective.py 命名）
│   ├── skills/                # カスタムスキル実装
│   ├── mcp/                   # MCP Servers（接続・認証管理）
│   └── core/                  # 共通ライブラリ・utility関数
├── tests/
│   ├── agents/                # Subagent単体テスト
│   ├── integration/           # マルチAgent統合テスト
│   └── fixtures/              # テストデータ・モック
├── config/
│   ├── settings.yaml          # アプリケーション設定
│   └── agents.yaml            # Agent定義・並列度・タイムアウト
└── docs/
    ├── ORCHESTRATION.md       # マルチAgent実行フロー図
    └── PLAYBOOK.md            # 実装チェックリスト・トラブルシューティング
```

| ファイル | 行数目安 | 主要内容 |
|---------|---------|---------|
| **CLAUDE.md** | 50〜80行 | コーディング規約、禁止事項、テスト・ビルド実行ポリシー。参照ファイル形式で肥大化を回避 |
| **AGENT.md** | 60〜100行 | ペルソナ定義、思考プロセス（Plan→Act→Verify）、SKILLS一覧、エスカレーション方針 |
| **DESIGN.md** | 40〜80行 | SOLID原則、DRY、ディレクトリ構成ガイド、命名規則の詳細 |
| **MCP.md** | 60〜100行 | MCP Server一覧、認証・接続フロー、トラブルシューティング |
| **SUBAGENT.md** | 80〜120行 | Orchestration パターン（並列/直列）、Context管理、デッドロック回避 |
| **SKILLS.md** | 50〜100行 | Skill定義テンプレート、トリガー条件、実装例（最低3例） |

---

## 🚀 導入手順

### 1. セットアップ

```bash
# プロジェクトルートで実行
mkdir -p .claude src/{agents,skills,mcp,core} tests/{agents,integration,fixtures} config docs

# このパッケージをコピー
cp -r 20260513/*.md .claude/
```

### 2. CLAUDE.md のカスタマイズ

```bash
# プロジェクト情報を反映
vi .claude/CLAUDE.md
  # - プロジェクト名・言語スタック
  # - チーム規模・対象者
  # - テストコマンド（pytest, npm test等）
  # - ビルドコマンド（make build, npm run build等）
```

### 3. settings.json の作成・編集

```bash
# Claude Code環境設定（hooks、permissions）
# 参考: https://code.claude.com/docs/en/configuration
cat > .claude/settings.json << 'EOF'
{
  "hooks": {
    "beforeTest": "npm run lint",
    "afterCommit": "npm run build && npm run test"
  },
  "permissions": {
    "allowedDirs": ["src", "tests", ".claude"],
    "blockedCommands": ["git push --force", "rm -rf"]
  }
}
EOF
```

### 4. Agent初期化

```bash
# src/agents/ に最初のSubagentを定義
python -m src.agents.researcher_context --init
```

### 5. テスト・検証

```bash
# テストが通ることを確認
pytest tests/ -v --cov=src

# CLAUDE.md解析テスト（オプション）
claude-code validate .claude/CLAUDE.md
```

---

## 🔧 カスタマイズガイド

### パターン1: TypeScript/React プロジェクト

**CLAUDE.md** の命名規則・テストコマンドを変更：

```markdown
### 命名規則
- **変数・関数**: `camelCase`
- **ファイル**: `kebab-case.ts`
- **コンポーネント**: `PascalCase.tsx`

### テスト実行
\`\`\`bash
npm run test -- --coverage
npm run lint -- --fix
\`\`\`
```

### パターン2: Python データ処理パイプライン

**SUBAGENT.md** の並列度を調整：

```yaml
# config/agents.yaml
orchestration:
  max_parallel_agents: 5        # デフォルト: 3
  context_per_agent: 150000     # デフォルト: 100000
  timeout_seconds: 300          # デフォルト: 180
```

### パターン3: マイクロサービス開発

**MCP.md** に API Gateway、メッセージング設定を追加：

```markdown
## 推奨MCP Servers
- aws-tools (Lambdaデプロイ)
- kafka-connector (イベント駆動)
- graphql-introspection (スキーマ検証)
```

---

## 📚 参考文献

### 公式ドキュメント・ブログ

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices) - Anthropic公式（2026-05-10）
- [Create Custom Subagents](https://code.claude.com/docs/en/sub-agents) - Claude Code Docs（2026-05-08）
- [Extend Claude with Skills](https://code.claude.com/docs/en/skills) - Claude Code Docs（2026-05-10）
- [Extending Claude's Capabilities with Skills and MCP](https://claude.com/blog/extending-claude-capabilities-with-skills-mcp-servers) - Claude Blog（2026-04-15）

### コミュニティリソース・GitHub

- [claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice) - vibe coding to agentic engineering（⭐450）
- [claude-code-best-practices](https://github.com/awattar/claude-code-best-practices) - Examples & patterns（⭐280）
- [my-claude-code-setup](https://github.com/centminmod/my-claude-code-setup) - Shared starter template（⭐320）
- [claude-code-config](https://github.com/trailofbits/claude-code-config) - Trail of Bits security-focused（⭐190）

### 記事・ガイド

- [Claude Code Best Practices - 12 Patterns Agentic Engineers Use](https://levelup.gitconnected.com/claude-code-best-practices-12-patterns-agentic-engineers-use-65264e3eb919) - Level Up Coding（2026-04-28）
- [Claude Code Advanced Best Practices: 11 Practical Techniques for Hooks, Subagents & Context Management](https://smartscope.blog/en/generative-ai/claude/claude-code-best-practices-advanced-2026/) - SmartScope（2026-05-05）
- [Claude Code Tips I Wish I'd Had From Day One](https://marmelab.com/blog/2026/04/24/claude-code-tips-i-wish-id-had-from-day-one.html) - Marmelab（2026-04-24）
- [The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf) - Anthropic（PDF）
- [Claude Code Sub-Agents: Parallel vs Sequential Patterns](https://claudefa.st/blog/guide/agents/sub-agent-best-practices) - ClaudeFA（2026-04-20）

### 技術比較・統合ガイド

- [Claude Skills vs MCP: A Technical Comparison](https://intuitionlabs.ai/articles/claude-skills-vs-mcp) - IntuitionLabs（2026-05-02）
- [Claude Code Skills vs MCP vs Plugins: Complete Guide 2026](https://www.morphllm.com/claude-code-skills-mcp-plugins) - Morph LLM（2026-04-30）
- [Skills vs MCP vs Connectors in Claude](https://levelup.gitconnected.com/skills-vs-mcp-vs-connectors-in-claude-8a536190b8b4) - Level Up Coding（2026-03-22）
- [Introduction to Model Context Protocol](https://anthropic.skilljar.com/introduction-to-model-context-protocol) - Anthropic Skilljar（2026年Q1）

### アーキテクチャ・オーケストレーション

- [Claude Code Agent Patterns: Orchestration Strategies](https://claudefa.st/blog/guide/agents/agent-patterns) - ClaudeFA（2026-04-18）
- [The Code Agent Orchestra - what makes multi-agent coding work](https://addyosmani.com/blog/code-agent-orchestra/) - Addy Osmani（2026-05-01）
- [Shipyard - Multi-agent orchestration for Claude Code in 2026](https://shipyard.build/blog/claude-code-multi-agent/) - Shipyard（2026-04-25）
- [Claude Code Agent Teams: How to Orchestrate AI Subagents for Real Development Work](https://designbeep.com/2026/05/02/claude-code-agent-teams-how-to-orchestrate-ai-subagents-for-real-development-work/) - Designbeep（2026-05-02）

---

## ✅ チェックリスト

セットアップ完了の確認：

- [ ] 全ファイル（CLAUDE.md, AGENT.md, DESIGN.md, MCP.md, SUBAGENT.md, SKILLS.md）を .claude/ にコピー
- [ ] CLAUDE.md をプロジェクト固有情報で編集（コーディング規約、テストコマンド）
- [ ] settings.json を作成・設定（hooks、permissions）
- [ ] src/agents/ に Subagent テンプレートを配置
- [ ] pytest / npm test で全テスト PASS
- [ ] GitHub / GitLab に `.claude/CLAUDE.md` をコミット

---

## 📞 トラブルシューティング

**問題**: Claude が CLAUDE.md を読み込まない  
**解決**: ファイルが `.claude/CLAUDE.md` にあることを確認。エディタで改行・エンコーディング（UTF-8）を確認

**問題**: Subagent 並列実行で Context エラー  
**解決**: SUBAGENT.md の `max_parallel_agents` を 3 に低下させ、`context_per_agent` を 50000 に削減

**問題**: MCP Server 接続失敗  
**解決**: MCP.md の「接続フロー」を参照。認証キー（API_KEY等）が設定されているか確認

---

**作成者**: Claude Code ハーネスエンジニアリング自動生成システム  
**バージョン**: 1.0  
**次回更新予定**: 2026-06-13
