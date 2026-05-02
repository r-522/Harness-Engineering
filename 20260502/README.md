# Harness-Engineering: AI Agent Orchestration Harness (2026-05-02)

## 概要

本構成は、Claude CodeにおけるAIエージェント最高パフォーマンス環境の実現を目指します。Subagent Teams パターン、Skills ベースの段階的読み込み、フック駆動のワークフロー自動化により、組織規模での複数エージェント協調実行を実現します。

**キーベネフィット**:
- Context 最適化（Skills による遅延読み込み）
- Subagent Teams による分散オーケストレーション
- Hook 駆動の自動ワークフロー管理

---

## 属性と判定根拠

| 属性 | 値 | 根拠 |
|---|---|---|
| **用途** | AIオーケストレーション・DevOps | Skills、Subagent、Agent Teams、Hook による自動化が中心。実装、テスト、デプロイの協調が必要 |
| **規模** | 組織 | 「最高峰スペシャリスト」向け。複数チーム、複数プロジェクト対応。 |
| **対象** | 上級者向け | 自律判断・補完が求められ、複雑なワークフロー管理が必須。 |

**判定期間**: 過去24時間（2026-05-02 時点）

---

## ディレクトリ構成とファイル役割

```
20260502/
├── README.md              # 本ファイル。構成目的、導入手順、参考文献
├── CLAUDE.md              # プロジェクト固有の制約・規約（50-70行）
├── AGENT.md               # エージェントペルソナ、思考フロー、スキル定義
├── DESIGN.md              # 設計原則、アーキテクチャパターン、ディレクトリガイド
├── HOOKS.md               # Hook 定義とトリガー仕様
├── settings.json          # .claude/ ツール権限設定、環境変数
└── [追加参考ファイル]      # 必要に応じて追加
```

### 各ファイルの役割

| ファイル | 用途 | 対象者 | キー情報 |
|---|---|---|---|
| **README.md** | 構成全体の説明・オンボーディング | PM、全メンバー | 判定根拠、導入手順、カスタマイズガイド |
| **CLAUDE.md** | Claude への session 制約・規約 | Claude（自動読み込み） | コーディング規約、禁止事項、テストポリシー（50-70行推奨） |
| **AGENT.md** | エージェント行動定義 | 上級エンジニア | ペルソナ、思考フロー、スキル定義、エスカレーション |
| **DESIGN.md** | 設計原則・アーキテクチャ | アーキテクト | SOLID、DRY、ディレクトリ構成、コンポーネント分類 |
| **HOOKS.md** | ワークフロー自動化 | DevOps、自動化担当 | Hook トリガー、実行スクリプト、エラーハンドリング |
| **settings.json** | 環境・ツール権限 | システム管理者 | 権限マッピング、MCP サーバー設定、Skill 登録 |

---

## 導入手順

### 1. ファイルの配置
```bash
cp -r 20260502/* .claude/
# または、個別ファイルをプロジェクトルートにコピー
```

### 2. CLAUDE.md の検証
```bash
# ファイルサイズを確認（推奨 50-70行）
wc -l .claude/CLAUDE.md
```

### 3. settings.json の環境に合わせてカスタマイズ
```bash
# 権限設定を確認
cat .claude/settings.json | jq '.permissions'
```

### 4. Skill の登録と動作確認
```bash
# Claude Code で /skills コマンドを実行し、登録済みスキルを確認
```

### 5. Hook の有効化
```bash
# settings.json に hook を追加後、プロジェクトを reload
# Claude Code リスタート
```

---

## カスタマイズガイド

### CLAUDE.md の縮小（オーバーヘッド削減）
1. 汎用ルール（80行以上）は DESIGN.md に移動
2. Skill トリガー条件は AGENT.md に記述
3. session-specific constraint のみ残す

### Subagent Teams の拡張
1. `AGENT.md` の `## SKILLS` セクションに新スキル定義を追加
2. `HOOKS.md` に新しいワークフロートリガーを追加
3. `settings.json` に skill の MCP 登録を追加

### Hook のカスタマイズ
**例**: Pre-commit フック追加
```json
{
  "hooks": {
    "pre-commit": {
      "command": "npm run lint && npm run type-check",
      "stages": ["staged-files"]
    }
  }
}
```

---

## 参考文献

| 情報源 | URL | 採用知見 | 参照日 |
|---|---|---|---|
| Writing a good CLAUDE.md | https://www.humanlayer.dev/blog/writing-a-good-claude-md | CLAUDE.md は 50-70行がベスト。@imports で外部ファイル参照可能 | 2026-05-02 |
| Best Practices for Claude Code - Docs | https://code.claude.com/docs/en/best-practices | Skill はトリガーベースで遅延読み込み。Context bloat 回避 | 2026-05-02 |
| Orchestrate teams of Claude Code | https://code.claude.com/docs/en/agent-teams | Agent Teams パターンが Subagent より推奨。共有タスク状態管理 | 2026-05-02 |
| Claude Code: Hooks, Subagents, Skills Guide | https://ofox.ai/blog/claude-code-hooks-subagents-skills-complete-guide-2026/ | Hook + Skill + Subagent の組み合わせで自動化ワークフロー構築 | 2026-05-02 |
| Dive into Claude Code | https://github.com/VILA-Lab/Dive-into-Claude-Code | 328 設定ファイル分析。協調パターンと SOLID 原則の適用事例 | 2026-05-02 |
| Everything Claude Code | https://github.com/affaan-m/everything-claude-code | Agent harness、セキュリティ、研究優先開発フレームワーク | 2026-05-02 |
| Claude Code Best Practices 2025 | https://github.com/Grayhat76/claude-code-resources/blob/main/claude-code-best-practices-2025.md | Skills、Memory、Cost Optimization の最新動向 | 2026-05-02 |
| Claude Code Ultimate Guide | https://github.com/FlorianBruniaux/claude-code-ultimate-guide | Production-ready templates、agentic workflows、quizzes | 2026-05-02 |

---

## 更新履歴

- **2026-05-02**: 初版作成。Organization 規模、上級者向け構成。Agent Teams パターン、Skill 遅延読み込み、Hook 自動化を中心に設計。

