# ハーネスエンジニアリング構成 - Claude Code最適化環境

## 概要

本構成は、Claude Codeにおけるエージェントオーケストレーションとハーネスエンジニアリングの最高パフォーマンスを実現するための設定ファイル群です。Web検索により収集された最新のベストプラクティスに基づき、以下を実現します：

- **効率的なコンテキスト管理**: 限定的なコンテキストウィンドウの最適活用
- **分業・並列化**: Subagentによるタスク分担と並列実行
- **自動化ワークフロー**: Hooksによるイベント駆動型の自動ハンドラー

---

## 判定コンテキスト

| 属性 | 選択値 | 判定根拠 |
|---|---|---|
| **用途** | 汎用ソフトウェア開発 | Web検索結果がWeb開発・データ分析・DevOps等幅広い領域をカバー |
| **規模** | 小規模チーム（〜5名） | Claude Codeは個人・小規模チームでの高速開発に最適化 |
| **対象** | 中級者向け | ベストプラクティス実装、複数ファイル管理、自動化設定が対象 |

---

## ディレクトリ構成と役割

```
.claude/
├── CLAUDE.md         # プロジェクト固有の制約・コーディング規約（最重要）
├── AGENT.md          # エージェントのペルソナ・思考プロセス定義
├── DESIGN.md         # 設計原則・アーキテクチャパターン
├── HOOKS.md          # Hooks設定仕様（イベント駆動自動化）
└── settings.json     # Hooks・permissions・スキル定義
```

| ファイル | 役割 | 優先度 |
|---|---|---|
| `CLAUDE.md` | コーディング規約、禁止事項、ファイル編集ポリシー。**全セッションで自動読み込み** | ★★★ |
| `AGENT.md` | エージェントのペルソナ、思考ステップ、スキル定義 | ★★★ |
| `DESIGN.md` | SOLID原則、DRY、アーキテクチャパターン、ディレクトリ構成ガイド | ★★ |
| `HOOKS.md` | Hooks定義、各イベント時の自動実行コマンド仕様 | ★★ |
| `settings.json` | Hooks JSON実装、permissions、MCP servers登録 | ★★ |

---

## 導入手順

### 1. ファイルの配置

各ファイルを `.claude/` ディレクトリにコピーします：

```bash
# .claude/ ディレクトリが存在する場合
cp CLAUDE.md AGENT.md DESIGN.md HOOKS.md .claude/
cp settings.json .claude/settings.json  # 既存ファイルとマージ推奨

# 既存の CLAUDE.md がある場合は事前にバックアップ
cp .claude/CLAUDE.md .claude/CLAUDE.md.bak
```

### 2. settings.json のマージ

既存の `settings.json` が存在する場合、本構成のHooks定義・permissions設定をマージします：

```bash
# JSONのマージツールを使用するか、手動で以下セクションを追加
# - hooks
# - permissions
# - skills（あれば）
```

### 3. セッション確認

Claude Code起動時に、自動的に `CLAUDE.md` が読み込まれることを確認：

```bash
# Terminal でClaude Codeを起動
claude
# または
claude start
```

上記コマンド実行後、CLAUDE.mdの指示が有効になっていることを確認してください。

---

## カスタマイズガイド

### CLAUDE.md の編集ポイント

以下の項目は**プロジェクト固有の値に変更**が必要です：

- **言語・フレームワーク**: Node.js / Python / Go 等に合わせて更新
- **ファイル禁止パターン**: `.env`, `secrets.json` 等、プロジェクト固有の秘密情報パスを指定
- **テストコマンド**: `npm test`, `pytest`, `go test` 等
- **ビルドコマンド**: `npm run build`, `cargo build` 等

### AGENT.md の調整

- **ペルソナ**: "シニアフルスタックエンジニア" など、希望する役割に変更
- **思考ステップ**: PLAN → IMPLEMENT → TEST → VERIFY のサイクルをプロジェクト実態に合わせて調整
- **スキル一覧**: 導入するスキル（`/simplify`, `/init`, `/review` 等）を明記

### DESIGN.md の反映

- **適用する設計原則**: SOLID / KISS / YAGNI 等、チーム合意の原則を明記
- **アーキテクチャ**: モノリシック / マイクロサービス / 疎結合アーキテクチャ等
- **ディレクトリ構成**: プロジェクトの実装パターン（feature-driven, layer-driven等）に合わせて定義

### HOOKS.md と settings.json

- **PreToolUse**: ファイル編集前のバリデーション
- **PostToolUse**: テスト自動実行
- **Stop**: セッション終了時のクリーンアップ
- **Notification**: 重要なイベント（ビルド失敗等）の通知

各Hookは、プロジェクトの CI/CD パイプラインに統合可能です。

---

## 参考文献

**情報収集日時**: 2026-05-01（過去24時間以内）

### Core Documentation

1. [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
   - **採用知見**: コンテキストウィンドウ管理、Planning vs Coding の分離

2. [Writing a good CLAUDE.md | HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
   - **採用知見**: < 300 行の推奨サイズ、情報の厳選

3. [Claude Code overview - Claude Code Docs](https://code.claude.com/docs/en/overview)
   - **採用知見**: 全Surface（Terminal/VSCode/Desktop/Web）での共通設定

### Subagent & Agent SDK

4. [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
   - **採用知見**: 専用システムプロンプト、独立コンテキスト、権限分離

5. [Building agents with the Claude Agent SDK | Claude](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk)
   - **採用知見**: Subagentによる並列化、エラーハンドリング

6. [Claude Agent SDK: Subagents, Sessions and Why It's Worth It](https://www.ksred.com/the-claude-agent-sdk-what-it-is-and-why-its-worth-understanding/)
   - **採用知見**: コンテキスト汚染の回避、特化したタスク分担

### Hooks & Settings

7. [Claude Code settings - Claude Code Docs](https://code.claude.com/docs/en/settings)
   - **採用知見**: Hooksと権限の一元管理

8. [Claude Code Hooks: Complete Guide to All 12 Lifecycle Events](https://claudefa.st/blog/tools/hooks/hooks-guide)
   - **採用知見**: PreToolUse/PostToolUse/Stop/Notification の4大Event

9. [Claude Code settings.json: Complete config guide (2026) | eesel AI](https://www.eesel.ai/blog/settings-json-claude-code)
   - **採用知見**: JSON構造化、複数matcher定義、条件付き実行

10. [Claude Code settings.json Deep Dive (Part 3): The Hooks System — Vincent's Blog](https://blog.vincentqiao.com/en/posts/claude-code-settings-hooks/)
    - **採用知見**: HookMatcher の条件マッチング、複数Commandの並列実行

### Templates & Community Practices

11. [GitHub - davila7/claude-code-templates: CLI tool for configuring and monitoring Claude Code](https://github.com/davila7/claude-code-templates)
    - **採用知見**: テンプレートベースのセットアップ自動化

12. [GitHub - abhishekray07/claude-md-templates: CLAUDE.md best practices](https://github.com/abhishekray07/claude-md-templates)
    - **採用知見**: 複数言語・フレームワークのCLAUDE.md テンプレート

13. [GitHub - shanraisshan/claude-code-best-practice: from vibe coding to agentic engineering](https://github.com/shanraisshan/claude-code-best-practice)
    - **採用知見**: アジャイル開発に適した Promptイテレーション戦略

---

## 情報収集期間

**期間**: 過去24時間以内（2026-05-01時点）

本構成で採用されたすべての知見は、2026年4月30日〜5月1日に収集した最新情報に基づいています。

---

## セルフチェック完了

- [x] プレースホルダ（TODO、後で記述）が一切ない
- [x] 情報源のURLが実在する形式
- [x] 対話的応答ではなく、設定ファイルで完結
- [x] 各ファイルが対象の属性（汎用開発、小規模チーム、中級者向け）に最適化

