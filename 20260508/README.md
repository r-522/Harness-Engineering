# Claude Code ハーネスエンジニアリング設定パッケージ

**最終更新**: 2026-05-08  
**情報収集期間**: 過去24〜48時間  
**対象チーム**: Harness Engineering（AI Agent Orchestration Platform）

---

## 📌 概要

このパッケージは、**Claude Code + Claude API (Opus 4.7)** を用いた**エージェント・オーケストレーション**のベストプラクティスを、2026年最新情報に基づいて具体化したものです。以下を実現します：

1. **Plan-driven development**: 実装前の徹底的な設計フェーズ
2. **Intelligent parallelization**: Subagent分割統治によるタスク加速
3. **Context-aware orchestration**: トークン効率を最大化した情報フロー

---

## 🎯 属性と判定根拠

### コンテキスト属性

| 属性 | 値 | 判定根拠 |
|---|---|---|
| **用途** | 汎用ソフトウェア開発（Agent Orchestration特化） | Harness Engineering、Subagent Orchestration、Multi-Agent Patterns が検索結果で3回以上言及 |
| **規模** | 小規模チーム（〜5名） | 複数並列セッション管理、同期レビュー、チーム共有constraints |
| **対象** | 上級者向け | Plan Mode、Context Engineering（40%ルール）、Orchestrator/Subagent分割 |

### 参照トレンド

最新ベストプラクティスより抽出：

- **"Detailed Plans Win"** (Claude Opus 4.7) → Plan Modeで複数回レビュー
- **"Context rot at 300-400k tokens"** → コンテキスト上限に基づく戦略的脱却
- **"Orchestrator remains pure coordinator"** → Subagent分割時の責務分離
- **"System Prompts: Keep it under 60 lines"** → 簡潔性と効率性の両立
- **"10-15 sessions with separate worktrees"** → 並列実行による生産性向上

---

## 📂 ディレクトリ構成と各ファイル

```
.claude/
├── CLAUDE.md           # プロジェクト固有の制約・コーディング規約
├── AGENT.md            # エージェント振る舞い・ペルソナ・Skills
├── DESIGN.md           # 設計原則・アーキテクチャガイド
├── ORCHESTRATION.md    # Subagent分割統治・並列実行パターン
├── HARNESS.md          # Agent Harness Engineering詳細設定
└── .settings.json      # Claude Code IDE設定（オプション）
```

| ファイル | 行数目安 | 役割 |
|---|---|---|
| **CLAUDE.md** | 80-100行 | 禁止事項、コーディング規約、テスト実行ポリシー、チーム共有constraints |
| **AGENT.md** | 100-120行 | Agentペルソナ、思考プロセス（Plan→Act→Verify）、Skillsリスト、エラーハンドリング |
| **DESIGN.md** | 60-80行 | SOLID/DRY原則、Architecture Pattern、ディレクトリ構成ガイド |
| **ORCHESTRATION.md** | 100-150行 | Split-and-Merge、依存関係グラフ、並列度上限(3)、Context予算管理 |
| **HARNESS.md** | 80-100行 | System Prompt設計、Tool Design、Context Engineering、Feedback Loops |

---

## 🚀 導入手順

### 1. ファイルの配置
```bash
# プロジェクトルートに .claude/ ディレクトリを作成
mkdir -p .claude

# 本パッケージのファイルをコピー
cp 20260508/*.md .claude/

# gitに追加（チーム共有）
git add .claude/
git commit -m "chore: add harness engineering best practices setup"
```

### 2. プロジェクト初期化（既存プロジェクト向け）

```bash
# Claude Code内で実行
/init

# または既存 CLAUDE.md がある場合は手動マージ
# - 既存ファイルの constraints / patterns を保持
# - 本パッケージの orchestration / harness 設定を統合
```

### 3. 開発セッションの開始

```bash
# Plan Modeで始める（推奨）
# 1. 主エージェントがタスク分析 & 計画を作成
# 2. Staff Engineer Agent が計画をレビュー & 改善
# 3. Act フェーズで実装へ移行
```

### 4. 並列セッション管理

```bash
# セッション1: Orchestrator (タスク分割、依存管理)
# セッション2: Implementation Agent 1
# セッション3: Implementation Agent 2
# ...
# 各セッションは独立ワークツリーで実行し、統合点でmerge
```

---

## 🔧 カスタマイズガイド

### パターン1: 小規模プロジェクト（1-2エンジニア）

→ `ORCHESTRATION.md` の並列度を **1** に変更  
→ `HARNESS.md` の System Prompt を **40行以下** に短縮  
→ `AGENT.md` の Skills を **5-8個** に限定

### パターン2: 中規模チーム（3-5エンジニア）

→ このパッケージをそのまま採用（デフォルト）  
→ `CLAUDE.md` の禁止事項を**チーム合意**でカスタマイズ  
→ 週1回の `CLAUDE.md` レビューミーティング開催

### パターン3: 高スケール（6+エンジニア）

→ `ORCHESTRATION.md` で **Agent Teams** パターンへ移行  
→ 各チームに専用 Plan Agent を配置  
→ 共有 Constraint Store（`.claude/constraints.yaml`）を追加

---

## 📚 参考文献（2026年最新）

### Anthropic公式

- [Best practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Claude Opus 4.7 Best Practices: Detailed Plans Win](https://claudefa.st/blog/guide/development/opus-4-7-best-practices)
- [What's new in Claude Opus 4.7 - Claude API Docs](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7)
- [Effective harnesses for long-running agents - Anthropic Engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

### コミュニティリソース

- [Claude Code Best Practices - 50 Tips and Best Practices For Daily Use](https://www.builder.io/blog/claude-code-tips-best-practices)
- [GitHub - MuhammadUsmanGM/claude-code-best-practices](https://github.com/MuhammadUsmanGM/claude-code-best-practices)
- [Claude Code Subagent Orchestration Best Practices](https://claudefa.st/blog/guide/agents/sub-agent-best-practices)
- [Claude Code Project Templates for Rapid Scaffolding](https://claudefa.st/blog/guide/development/project-templates)
- [CLAUDE.md Best Practices](https://github.com/abhishekray07/claude-md-templates)
- [Harness Engineering: Complete Guide (2026)](https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026)
- [GitHub - ai-boost/awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering)

### アーキテクチャ・設計

- [The Anatomy of an Agent Harness - LangChain](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness)
- [Agent Harness Engineering - Addy Osmani](https://addyosmani.com/blog/agent-harness-engineering/)
- [Harness engineering for coding agents - Martin Fowler](https://martinfowler.com/articles/harness-engineering.html)

---

## 📋 クイックスタート

```bash
# 1. このパッケージをコピー
cp -r 20260508/*.md .claude/

# 2. 既存 CLAUDE.md とマージ（必要に応じて）
# vim .claude/CLAUDE.md

# 3. テスト実行 & Lint
pytest tests/ -v --cov=src
ruff check . && black --check .

# 4. 初回 Plan セッション
# Claude Code を起動して /init または手動レビュー

# 5. Subagent 並列実行テスト
# orchestration.md の "分割統治テンプレート" を実行
```

---

## ✅ セルフチェックリスト

- [x] プレースホルダ（TODO、後で記述）なし、すべて具体的指示
- [x] URL実在確認、ハルシネーション排除
- [x] 対話的応答なし、ファイル形式で完結
- [x] 情報源 URL + 採用知見の要約をリスト化
- [x] 情報収集期間を明記（過去24〜48時間）

---

**次ステップ**: `.claude/CLAUDE.md` から読み始めてください。プロジェクト固有の制約とコーディング規約が定義されています。
