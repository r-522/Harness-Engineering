# Claude Code ハーネスエンジニアリング ベストプラクティス設定集

## 概要

このディレクトリは、Claude Code（Claude Agent SDK）を用いた **汎用ソフトウェア開発** の最高パフォーマンス環境を実現するための設定ファイル群です。中規模チーム（5～20名）向けに、コンテキスト管理、Subagentオーケストレーション、Hooks自動化を統合した構成を提供します。

**核となる3つの価値観**:
1. **コンテキストウィンドウの最適化**: メモリ使用を戦略的に管理し、Claude の性能劣化を最小化
2. **Subagent並行実行**: 複数の専門エージェントを同時稼働させ、処理時間を削減
3. **Hooks によるワークフロー自動化**: 設定駆動の工程管理で人的介入を最小化

---

## ディレクトリ構成と役割

| ファイル | 用途 | スコープ |
|---|---|---|
| **CLAUDE.md** | プロジェクト共有ルール（15行以内推奨） | `.claude/CLAUDE.md` / チーム全体 |
| **AGENT.md** | エージェント思考プロセス・ペルソナ定義 | プロジェクト固有（共有） |
| **DESIGN.md** | 設計原則・アーキテクチャパターン | プロジェクト固有（共有） |
| **SKILLS.md** | Skillsの実装ガイドと一覧 | `.claude/skills/` ディレクトリ構成 |
| **HOOKS.md** | Hooks自動化シナリオと実装例 | `settings.json` / `settings.local.json` |
| **settings.json** | チーム共有の設定（Hooks・権限・環境変数） | プロジェクト共有 |
| **settings.local.json** | 個人用ローカルオーバーライド | 本人のみ（`.gitignore` 推奨） |

---

## 導入手順

### 1. ディレクトリ構造の初期化

```bash
# プロジェクトルートで実行
mkdir -p .claude/skills
mkdir -p .claude/hooks

# 本ファイル群をコピー
cp CLAUDE.md .claude/CLAUDE.md
cp settings.json .claude/settings.json
```

### 2. CLAUDE.md の検証（チーム全員）

`.claude/CLAUDE.md` は **自動的に全セッションに読み込まれます**。以下を確認：
- プロジェクト固有の禁止事項は記述されているか
- コーディング規約は明示されているか
- 行数が15行以内か（過剰な情報はAGENT.md等に分離）

### 3. settings.json の設定（チーム共有）

```bash
# .claude/settings.json を編集
# 1. Hooks セクション: チーム全体の自動化タスク
# 2. Permissions: 共通して必要な権限
# 3. Environment: 開発環境変数

# Git でコミット（チーム共有）
git add .claude/settings.json
```

### 4. settings.local.json の個人設定

```bash
# ~/.claude/settings.local.json（ユーザー個人）
# または .claude/settings.local.json（プロジェクト個人用、.gitignore 推奨）

# プロジェクト固有の個人設定はこちらに記述
# 例: ローカルツールパス、個人用環境変数、テーマ設定
```

### 5. Skillsの登録

```bash
# .claude/skills/ に SKILL.md ファイルを配置
# 例: .claude/skills/code-review.md
#    .claude/skills/performance-audit.md

# Claude が自動的に検出・実行します
```

---

## カスタマイズガイド

### A. コンテキストウィンドウの危機的管理

**問題**: Claude の性能は context fill に比例して劣化
**対策**:
1. CLAUDE.md は **最小限**（プロジェクト全体に適用される項目のみ）
2. ドメイン固有ガイドは別ファイル参照: `# See AGENT.md for agent behavior`
3. 過去の失敗ログは**セッション終了時に削除**（hook: `SessionEnd`）

### B. Subagent 設計パターン

複数エージェントを **並行実行** して処理を加速：

```
Main Agent
├─ Code Review Agent (style/security check)
├─ Test Coverage Agent (coverage analysis)
└─ Documentation Agent (doc generation)
```

各 Subagent は専用の SKILL.md/PROMPT を持つ。詳細は `AGENT.md` 参照。

### C. Hooks 自動化の実装

設定駆動で以下を自動実行：
- **PostToolUse**: Prettier, Black, ESLint の自動フォーマット
- **PreToolUse**: Git status 確認、権限チェック
- **SessionEnd**: 一時ファイル削除、ログ統合

詳細は `HOOKS.md` で具体例を提供。

### D. チームスケーリング

**5人以下**: 本設定でそのまま利用可能
**6～20人**: `settings.json` に `permissions` セクション追加（企業ポリシー)
**20人以上**: 組織レベルの `~/.claude/settings.json` + プロジェクトレベルのオーバーライド

---

## 参考文献

### 公式ドキュメント・ブログ

| 日時 | ソース | 知見 |
|---|---|---|
| 2026-05-06 | [Best practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices) | Context management の重要性、Plan vs Act の分離 |
| 2026-05-05 | [Claude Code Advanced Best Practices: 11 Practical Techniques - SmartScope](https://smartscope.blog/en/generative-ai/claude/claude-code-best-practices-advanced-2026/) | 21 lifecycle events（March 2026 更新）、async hook execution |
| 2026-04-24 | [Claude Code Tips I Wish I'd Had From Day One](https://marmelab.com/blog/2026/04/24/claude-code-tips-i-wish-id-had-from-day-one.html) | Skill/Subagent 活用、精密指示の重要性 |
| 2026-05-06 | [Writing a good CLAUDE.md - HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md) | CLAUDE.md 設計原則（15行以内、Progressive Disclosure） |

### GitHub リポジトリ（Star数100以上）

| リポジトリ | 特徴 | 採用知見 |
|---|---|---|
| [shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice) | Vibe coding → Agentic engineering の体系化 | Agentic loop best practices |
| [davila7/claude-code-templates](https://github.com/davila7/claude-code-templates) | CLI 設定ツール・テンプレート集 | 3層設定階層（User/Project/Local） |
| [abhishekray07/claude-md-templates](https://github.com/abhishekray07/claude-md-templates) | CLAUDE.md テンプレート事例集 | 業界別テンプレート |
| [disler/claude-code-hooks-mastery](https://github.com/disler/claude-code-hooks-mastery) | Hook イベント全網羅 | 21 イベントと実装パターン |
| [centminmod/my-claude-code-setup](https://github.com/centminmod/my-claude-code-setup) | 実戦テンプレート＋メモリシステム | Session memory integration |

### Expert ブログ・Substack

| 著者・メディア | 記事 | 日時 | 主要知見 |
|---|---|---|---|
| Nader Dabit | [The Complete Guide to Building Agents with Claude Agent SDK](https://nader.substack.com/p/the-complete-guide-to-building-agents) | 2026-04 | Agent SDK 総論、Session/Memory 概念 |
| Colin McNamara | [Understanding Skills, Agents, Subagents, and MCP](https://colinmcnamara.com/blog/understanding-skills-agents-and-mcp-in-claude-code) | 2026-05 | Tool/Skill/Subagent の使い分け |
| KS Red | [Claude Agent SDK: Subagents, Sessions, Why It's Worth It](https://www.ksred.com/the-claude-agent-sdk-what-it-is-and-why-its-worth-understanding/) | 2026-04 | Subagent 並行実行、context isolation |
| unicodeveloper | [Claude Managed Agents: What It Actually Offers](https://medium.com/@unicodeveloper/claude-managed-agents-what-it-actually-offers-the-honest-pros-and-cons-and-how-to-run-agents-52369e5cff14) | 2026-04 | Managed Agents（Public Beta）、secure sandbox |

### 公式リリースノート

| 日時 | アップデート | インパクト |
|---|---|---|
| 2026-05-06 | Claude Code May 2026 Updates (Releasebot) | Latest hooks, agent SDK improvements |
| 2026-04-08 | Claude Managed Agents Public Beta | Infrastructure-as-a-service agent runtime（sandbox, permissions, tracing） |
| 2026-03-15 | 21 Lifecycle Events | Async hook execution 追加 |

---

## 情報収集期間

**過去24時間**（2026年4月中旬～2026年5月6日）

検索対象：
- Anthropic 公式ドキュメント
- GitHub（Star数100以上、`.claude/` 構成を持つリポジトリ）
- 技術ブログ・Substack（直近1ヶ月以内の記事）
- Reddit (r/ClaudeAI)、X (Twitter)（言及3件以上のトレンド）

---

## 注記

このディレクトリは **生成日時: 2026-05-06** の情報に基づいています。
Anthropic の最新リリースノート、GitHub 検索、技術コミュニティの知見から自動判定・生成された設定です。

**更新推奨**: 毎月1回、以下を確認してください：
1. `claude.com/docs` の更新（新 Hooks イベント等）
2. GitHub の新規 `.claude/` テンプレート
3. 使用中の Claude model version（現在: Haiku 4.5 / Sonnet 4.6 / Opus 4.7）
