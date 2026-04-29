# Claude Code ハーネスエンジニアリング設定 (2026-04-29)

## 概要

このディレクトリは、Claude Codeを最高パフォーマンスで運用するための総合設定ファイル群です。
コンテキストウィンドウの効率的な管理、セキュリティベストプラクティス、MCP統合、自動化フローの構築を通じて、
個人〜小規模チーム開発における生産性を最大化します。

---

## コンテキスト属性と判定根拠

| 属性 | 選択 | 根拠 |
|---|---|---|
| **用途** | 汎用ソフトウェア開発 | Web/CLI/DS案件すべてに対応可能な設定方針を採用 |
| **規模** | 小規模チーム（〜5名） | 最新トレンド検索で「小規模チーム」向けベストプラクティスが多く言及される |
| **対象** | 中級者向け | CLAUDE.md基本から hooks・subagent 応用まで段階的に学習可能な構成 |
| **情報収集期間** | 過去24時間 | 最新記事 2026年4月24日を含む、Anthropic公式ドキュメント一次ソース確保 |

---

## ディレクトリ構成と各ファイル役割

```
/20260429/
├── README.md           ← このファイル（概要・導入手順）
├── CLAUDE.md           ← プロジェクト制約・コーディング規約
├── AGENT.md            ← エージェントの思考プロセス・スキル定義
├── DESIGN.md           ← 設計原則・アーキテクチャガイド
├── CONTEXT.md          ← コンテキストウィンドウ最適化ガイド
├── SECURITY.md         ← セキュリティベストプラクティス
├── MCP.md              ← MCP サーバー統合ガイド
└── HOOKS.md            ← フック・自動化ワークフロー実装
```

| ファイル | 役割 | 主な内容 |
|---|---|---|
| **CLAUDE.md** | プロジェクト固有の制約・規約 | コーディング規約、禁止操作、テスト・ビルドポリシー |
| **AGENT.md** | エージェント振る舞い定義 | ペルソナ、思考プロセス、スキル、エラーハンドリング |
| **DESIGN.md** | 設計原則とアーキテクチャ | SOLID原則、ディレクトリ構成、パターンガイド |
| **CONTEXT.md** | トークン管理最適化 | コンテキスト監視、/compact活用、チェックリスト |
| **SECURITY.md** | セキュリティ強化方針 | 依存関係管理、MCP審査、権限最小化 |
| **MCP.md** | MCP サーバー統合 | 推奨サーバー、設定例、トラブルシューティング |
| **HOOKS.md** | 自動化フロー実装 | hooks種別、実装例、チーム協働パターン |

---

## 導入手順

### 1. ファイルのコピー
```bash
# リポジトリの .claude/ ディレクトリにコピー
cp -r 20260429/* /path/to/your/project/.claude/
```

### 2. プロジェクト固有のカスタマイズ
各ファイルの `[PROJECT_NAME]`, `[TECH_STACK]`, `[TEAM_SIZE]` プレースホルダを実際の値に置き換えてください（本設定ではプレースホルダなし、全て具体値で記述）。

### 3. CLAUDE.md の動作確認
```bash
# Claude Code を起動し、/context コマンドで CLAUDE.md 読み込みを確認
claude-code
> /context
```

### 4. Hooks の登録（必要に応じて）
`HOOKS.md` の実装例に従い、 `~/.claude/settings.json` に hooks セクションを追加してください。

### 5. MCP サーバーの追加（チーム環境）
`MCP.md` のサーバーリストから必要なものを選定し、 `.claude/mcp.json` に登録してください。

---

## カスタマイズガイド

### ユースケース別アプローチ

#### Web 開発チーム向け
1. CLAUDE.md に Node.js/TypeScript 版のフォーマット規約を記述
2. MCP.md から GitHub, npm, Prettier MCP を選定
3. HOOKS.md で PostToolUse → lint + format フローを実装

#### データ分析・研究開発
1. CLAUDE.md に Jupyter, Pandas 操作規約を追加
2. MCP.md から Notion, SQLite MCP を選定
3. CONTEXT.md の「大規模データセット読込」セクションを参照

#### DevOps・インフラストラクチャ
1. CLAUDE.md に Terraform, Docker 操作禁止事項を強調
2. SECURITY.md の「権限最小化」セクションを徹底
3. HOOKS.md で Plan モード強制トリガーを設定

### 段階的な有効化

- **Week 1**: CLAUDE.md + AGENT.md のみ有効化
- **Week 2**: DESIGN.md + CONTEXT.md を参照開始
- **Week 3**: SECURITY.md の権限設定を反映
- **Week 4**: MCP.md から最初の 2-3 サーバーを追加
- **Week 5+**: HOOKS.md で自動化フロー実装

---

## 参考文献

### Anthropic 公式ドキュメント
1. [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
2. [Security - Claude Code Docs](https://code.claude.com/docs/en/security)
3. [Connect Claude Code to tools via MCP - Claude Code Docs](https://code.claude.com/docs/en/mcp)
4. [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
5. [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)

### コミュニティリソース
6. [GitHub - shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice) ⭐ CLAUDE.md テンプレート
7. [GitHub - abhishekray07/claude-md-templates](https://github.com/abhishekray07/claude-md-templates)
8. [GitHub - trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config) ⭐ セキュリティ重視設定
9. [GitHub - hesreallyhim/awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) ⭐ skills/hooks リポジトリ
10. [GitHub - anthropics/claude-code-security-review](https://github.com/anthropics/claude-code-security-review)

### 専門家ブログ
11. [Claude Code: Workflows and Best Practices 2026 - smart-webtech.com](https://smart-webtech.com/blog/claude-code-workflows-and-best-practices/)
12. [Claude Code Tips I Wish I'd Had From Day One - marmelab.com](https://marmelab.com/blog/2026/04/24/claude-code-tips-i-wish-id-had-from-day-one.html) 🔴 *最新記事 2026-04-24*
13. [50 Claude Code Tips and Best Practices For Daily Use - builder.io](https://www.builder.io/blog/claude-code-tips-best-practices)
14. [Claude Code Advanced Best Practices: 11 Practical Techniques for Hooks, Subagents & Context Management [2026]](https://smartscope.blog/en/generative-ai/claude/claude-code-best-practices-advanced-2026/)
15. [Claude Code Sub-agents: The 90% Performance Gain Nobody Talks About](https://www.codewithseb.com/blog/claude-code-sub-agents-multi-agent-systems-guide)
16. [Writing a good CLAUDE.md - HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
17. [Claude Code Context Window: Optimize Your Token Usage](https://claudefa.st/blog/guide/mechanics/context-management)

### チーム・企業向けリソース
18. [How Anthropic teams use Claude Code](https://www-cdn.anthropic.com/58284b19e702b49db9302d5b6f135ad8871e7658.pdf)
19. [Claude Code Hooks: A Practical Guide to Workflow Automation - DataCamp](https://www.datacamp.com/tutorial/claude-code-hooks)
20. [Claude Code Team Best Practices: Standardize with CLAUDE.md, Hooks & GitHub Actions](https://smartscope.blog/en/generative-ai/claude/claude-code-creator-team-workflow-best-practices/)

---

## FAQ

**Q: 複数プロジェクトで異なる CLAUDE.md を使い分けたい場合は？**  
A: `~/.claude/CLAUDE.md` をグローバル基本ルール（15行以下）とし、各プロジェクトの `.claude/CLAUDE.md` でプロジェクト固有ルール（50行以下）を上書きします。

**Q: Hooks はどのタイミングで有効化するべき？**  
A: 本設定では Week 4 以降の段階的導入を推奨。最初は `/clear` と `/compact` でコンテキスト管理を習慣化してから、PostToolUse hooks で自動化を追加します。

**Q: 小規模チーム内で CLAUDE.md を共有する場合のベストプラクティスは？**  
A: `.claude/CLAUDE.md` をリポジトリにコミットし、pull request で変更を議論してから反映します。全員で同じ標準を従うことで 33% → 70% 以上の成功率向上が期待できます。

**Q: セキュリティ設定の推奨事項は？**  
A: SECURITY.md で「hooks すべて無効化」を初期状態とし、明示的に許可したもののみ有効化する「許可リスト方式」を採用しています。

---

**作成日**: 2026-04-29  
**情報源収集期間**: 過去 24 時間（最新トレンド対応）  
**推奨対象**: 汎用ソフトウェア開発チーム（個人〜5名規模）
