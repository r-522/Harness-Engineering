# Claude Code ハーネスエンジニアリング 構成パッケージ

**生成日時**: 2026-05-15  
**対象プロジェクト**: Harness Engineering - AI Agent Orchestration Platform  
**情報収集期間**: 過去24時間以内（Web最新トレンド採用）

---

## 📋 概要

このパッケージは、Claude Code による **AI エージェント・オーケストレーション** の最高パフォーマンス環境を構築するための設定ファイル群です。Anthropic 公式ドキュメント、GitHub ベストプラクティス、および2026年のコミュニティトレンドを統合し、以下の課題に対応します：

1. **Subagent 並列実行の最適化** - 複数の独立専門タスクを効率的に並列化
2. **Context Window 管理** - 限られた予算内で複数エージェント間の協調を実現
3. **Skill・MCP 統合** - 再利用可能なツールと外部システムの安全な統合

---

## 🎯 コンテキスト判定結果

| 属性 | 値 | 根拠 |
|---|---|---|
| **用途** | 汎用ソフトウェア開発（AI Agent Orchestration 重視） | Web検索で「AI Agent Architecture 2026」「Orchestration Patterns」の言及頻度が最高 |
| **規模** | 小規模チーム（〜5名） | プロジェクトスコープ、CLAUDE.md 行数制限から判定 |
| **対象** | 上級者向け | Plan mode、Context最適化、Subagent 多層構成が必須 |

---

## 📁 ディレクトリ構成と各ファイルの役割

```
.claude/
├── CLAUDE.md              # プロジェクト固有の制約・ルール（80行以下）
├── AGENT.md              # エージェント振る舞い定義・思考プロセス
├── DESIGN.md             # 設計原則・アーキテクチャパターン
├── SKILLS.md             # Skills 定義と トリガー条件
└── MCP-INTEGRATION.md    # MCP サーバー連携ガイド
```

| ファイル | サイズ目安 | 主な内容 | 読込時期 |
|---|---|---|---|
| **CLAUDE.md** | ≤ 80行 | コーディング規約、禁止事項、ワークフロー | セッション開始時（必須） |
| **AGENT.md** | 100-150行 | ペルソナ、思考プロセス、SKILLS リスト、エラーハンドリング | ユーザー初回質問時 |
| **DESIGN.md** | 80-120行 | SOLID原則、アーキテクチャパターン、ディレクトリ設計ガイド | 実装フェーズ開始時 |
| **SKILLS.md** | 100-200行 | Skills の定義、トリガー、実行手順、コスト最適化 | Skills 活用時 |
| **MCP-INTEGRATION.md** | 80-120行 | MCP サーバー一覧、安全な統合、トラブルシューティング | 外部連携が必要な場合 |

---

## 🚀 導入手順

### 1. ファイル配置
```bash
# 現在のプロジェクトディレクトリで実行
cp -r 20260515/* .claude/

# 確認
ls -la .claude/
# → CLAUDE.md, AGENT.md, DESIGN.md, SKILLS.md, MCP-INTEGRATION.md
```

### 2. .gitignore 設定（重要）
```bash
# .claude/.gitignore に以下を追加
.claude/*.local.md      # 個人設定・秘密情報
.claude/temp/           # 一時ファイル
```

### 3. セッション初期化
```bash
# Claude Code session を再起動
# → CLAUDE.md が自動読み込まされます
```

### 4. 動作確認
```bash
# AGENT.md で定義したペルソナが反映されているか確認
# /agents コマンドで Subagent リストを表示
```

---

## 🔧 カスタマイズガイド

### Scenario A: Web開発チーム向けのカスタマイズ
```
CLAUDE.md の修正項目：
- コーディング規約に「TypeScript strict mode」を追加
- 禁止事項に「console.log()本番コード混在」を追加
```

### Scenario B: データ分析・研究開発向けのカスタマイズ
```
AGENT.md の修正項目：
- SKILLS に「データ検証」「可視化」スキルを追加
- 思考プロセスに「仮説検証ループ」を明示
```

### Scenario C: 組織レベルでの展開
```
グローバル設定 (~/.claude/CLAUDE.md) と プロジェクト設定 (./.claude/CLAUDE.md) を分離：
- グローバル: 全社統一ルール（セキュリティ、API アクセス）
- プロジェクト: プロジェクト固有ルール（パッケージ管理、ディレクトリ構成）
```

---

## 📚 参考文献

### 公式ドキュメント
1. [Best practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
   - CLAUDE.md の80行制限、ファイル配置、セマンティック構造化の重要性を確認

2. [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
   - Subagent 定義（YAML frontmatter）、model/tools 設定

3. [Introduction to agent skills](https://anthropic.skilljar.com/introduction-to-agent-skills)
   - Skills の定義、ポータブル性、トリガー条件

### コミュニティベストプラクティス
4. [Writing a good CLAUDE.md | HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
   - 実践的な CLAUDE.md 編成法（コンテンツ選別、優先度付け）

5. [CLAUDE.md Best Practices: The 10-Section Template](https://blink.new/blog/claude-md-best-practices)
   - 10セクション・テンプレート（マルチプロジェクト対応）

6. [GitHub - MuhammadUsmanGM/claude-code-best-practices](https://github.com/MuhammadUsmanGM/claude-code-best-practices)
   - 多層 Subagent パターン、コスト最適化、実装例

### アーキテクチャ・パターン
7. [AI Agent Orchestration Patterns - Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
   - 5つの Orchestration パターン（Orchestrator-Worker, Swarm, Mesh, Hierarchical, Pipeline）

8. [AI Agent Architecture: A Complete Guide for 2026](https://monday.com/blog/ai-agents/ai-agent-architecture/)
   - 2026年の最新アーキテクチャトレンド、マルチエージェント設計原則

9. [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
   - Claude Code 内での Subagent オーケストレーション実装ガイド

### 統合・運用ガイド
10. [Understanding Skills, Agents, Subagents, and MCP | Colin McNamara](https://colinmcnamara.com/blog/understanding-skills-agents-and-subagents-and-mcp-in-claude-code)
   - Skills vs Subagent の判定基準、MCP 統合シーン

11. [GitHub - affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)
   - エージェント・ハーネス・パフォーマンス最適化（Skills、Memory、Security）

---

## 📌 更新履歴

| 日付 | 版 | 変更内容 | 情報源 |
|---|---|---|---|
| 2026-05-15 | 1.0 | 初版生成（5つの設定ファイル、11の参考文献） | Anthropic公式Docs + GitHub Stars100+ + 2026年Medium記事 |

---

## 💡 FAQ

**Q: CLAUDE.md が80行を超えた場合、どうすればよいですか？**  
A: 以下の順で優先度をつけて削減してください：
1. コメント・説明文を削る（ファイル/行参照に置き換え）
2. 補足は AGENT.md に移行
3. 詳細は各ファイルの参考文献セクションへリンク

**Q: Subagent を何個まで並列実行できますか？**  
A: 標準ガイドラインは **3並列**（Context予算 ÷ 3）です。詳細は AGENT.md の「最大並列数」セクションを参照。

**Q: MCP サーバーの追加時、どのレビュー プロセスが必要ですか？**  
A: セキュリティレビューが必須です。MCP-INTEGRATION.md の「安全な統合チェックリスト」を参照。

---

**🔒 Security Note**: この構成ファイルは `.claude/` ディレクトリに格納され、`.gitignore` で個人設定を除外可能です。本番 API キーはコードに埋め込まず、`.local.md` または環境変数で管理してください。

