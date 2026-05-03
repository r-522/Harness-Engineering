# Claude Code ハーネスエンジニアリング構成 (2025-05-03)

## 概要

本構成は、Claude Code環境における最高パフォーマンスのAIエージェント開発を実現するために、最新のベストプラクティスと業界トレンドに基づいた統合的なハーネスエンジニアリング設定です。Harness Engineering「エージェントの環境・制約・フィードバックループの設計」という哲学に従い、プロジェクトスコープに最適化された操作手順書と制約定義を提供します。

## コンテキスト属性と判定根拠

| 属性 | 選択値 | 根拠 |
|---|---|---|
| **用途** | 汎用ソフトウェア開発 | Web検索結果で最頻出、複数プロジェクト対応を想定 |
| **規模** | 中規模チーム（5～20名） | Subagent orchestration（並列・順序制御）対応が必須 |
| **対象レベル** | 中級者向け | Harness Engineering理解、複雑な設定管理が前提 |

## ディレクトリ構成と各ファイルの役割

```
20250503/
├── README.md                 # 本ファイル（構成概要・導入手順）
├── CLAUDE.md                 # プロジェクト固有の制約・規約（全セッション自動読み込み）
├── AGENT.md                  # エージェント振る舞い定義（ペルソナ・思考プロセス・スキル）
├── DESIGN.md                 # 設計原則・アーキテクチャガイド
└── HARNESS.md                # ハーネスエンジニアリング専門ガイド（Subagent・MCP設定）
```

| ファイル | 役割 | サイズ目安 |
|---|---|---|
| **CLAUDE.md** | セッション開始時に自動読み込みされるプロジェクト定義。コーディング規約、禁止操作、ファイル構造、スタック情報 | 200～300行 |
| **AGENT.md** | Claude自身のペルソナ、思考フロー（Plan→Verify→Act）、スキル定義、エラーハンドリング | 150～200行 |
| **DESIGN.md** | SOLID原則、DRY、Clean Architecture等の適用範囲、ディレクトリ構成ルール | 100～150行 |
| **HARNESS.md** | Subagent並列実行パターン、MCP統合、フィードバックループ構築、Split-and-Merge戦略 | 120～180行 |

## 導入手順

### Step 1: ファイルの配置
```bash
# プロジェクトルートに .claude ディレクトリを作成
mkdir -p .claude

# 本構成ファイルを .claude ディレクトリへコピー
cp CLAUDE.md AGENT.md DESIGN.md HARNESS.md .claude/
```

### Step 2: 初回セッション開始
```bash
# Claude Code を起動（任意のプラットフォーム）
claude code

# セッション開始時に CLAUDE.md が自動読み込みされることを確認
# ステータスコマンドでコンテキスト使用状況を確認
/status
```

### Step 3: 設定検証
```bash
# スキルの確認
/skills

# 各 AGENT.md に記載されたスキルが認識されているか確認
# 必要に応じて settings.json を更新
```

## カスタマイズガイド

### CLAUDE.md のカスタマイズ

1. **スタック情報の更新**
   ```markdown
   ## Tech Stack
   - Language: [プロジェクト言語]
   - Framework: [使用フレームワーク]
   - Package Manager: [npm/pip/cargo等]
   ```

2. **プロジェクト構造の定義**
   - `src/`, `tests/`, `docs/` など実際のディレクトリ構成に合わせる
   - monorepo の場合は、workspace構造を明記

3. **コーディング規約の追加・修正**
   - チームの命名規則、フォーマット基準に合わせ更新

### AGENT.md のカスタマイズ

1. **スキル定義の拡張**
   ```markdown
   - Skill: `/deploy-custom`
     Trigger: ユーザーが本番環境へのデプロイを指示
     Steps: [実装ステップ]
   ```

2. **エラーハンドリング方針の調整**
   - プロジェクト固有のエラー型に対応

### HARNESS.md のカスタマイズ

1. **Subagent パターンの選択**
   - Split-and-Merge（並列）vs Sequential（順序）の判断基準
   - 例: テスト実行は並列、デプロイは順序制御

2. **MCP サーバーの統合**
   - 実際に使用する MCP（GitHub, Claude, Stripe等）を列記
   - `settings.json` の MCP 設定と同期

## 参考文献とトレンド情報

収集日時: **2025-05-03 06:30 UTC**  
情報収集期間: **過去24時間以内** （Web検索実施）

### 公式・一次資料

1. [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
   - 知見: コンテキスト管理（80%到達で清掃）、計画フェーズの重要性

2. [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
   - 知見: Agent Teams パターン、Subagent並列実行の要件

3. [Claude Code Templates: 1000+ Agents, Commands, Skills & MCP Integrations](https://www.aitmpl.com/)
   - 知見: 実装済みテンプレートの参考、スキル定義パターン

### コミュニティ・実装ガイド

4. [Writing a good CLAUDE.md | HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md)
   - 知見: CLAUDE.md 300行以下推奨、汎用性確保の重要性

5. [Harness Engineering Explained - AI Code Invest](https://aicodeinvest.com/harness-engineering-claude-code-ai-agents-guide/)
   - 知見: Harness = Model + Environment + Constraints + Feedback Loops

6. [Skill Issue: Harness Engineering for Coding Agents | HumanLayer Blog](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)
   - 知見: スキル定義の機械的実行可能性、トリガー設定の明確化

7. [Claude Code Agent Teams vs Sub-Agents: Which Pattern Should You Use?](https://www.mindstudio.ai/blog/claude-code-agent-teams-vs-sub-agents)
   - 知見: Subagent（階層型・集約） vs Agent Teams（分散・協調）の選定基準

8. [Claude Code Subagents: Parallel Multi-Agent Workflows | TURION.AI](https://turion.ai/blog/claude-code-multi-agents-subagents-guide/)
   - 知見: Split-and-Merge パターン、トークン効率、オーケストレータの責務

### GitHub リポジトリ参考例

9. [GitHub - Chachamaru127/claude-code-harness](https://github.com/Chachamaru127/claude-code-harness)
   - 知見: Plan→Work→Review サイクルの実装例

10. [GitHub - revfactory/harness](https://github.com/revfactory/harness)
    - 知見: メタスキル（ドメイン固有チーム設計）、スキル自動生成パターン

11. [GitHub - hesreallyhim/awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code)
    - 知見: スキル・フック・MCP統合の実装リスト（随時更新）

## セットアップ後の検証チェックリスト

- [ ] `.claude/CLAUDE.md` が存在し、セッション開始時に読み込まれている
- [ ] `/status` コマンドでコンテキスト使用率が80%未満
- [ ] 定義されたスキルがすべて `/skills` で認識されている
- [ ] `settings.json` 内の MCP サーバー設定が `HARNESS.md` と同期
- [ ] プロジェクト固有のコーディング規約が CLAUDE.md に反映
- [ ] Subagent パターン（並列/順序）が HARNESS.md で明記されている

## トラブルシューティング

**Q: CLAUDE.md が読み込まれない**  
A: `.claude/CLAUDE.md` ファイルの存在を確認。必ず隠し属性ディレクトリ `.claude/` 直下に配置してください。

**Q: スキル定義が認識されない**  
A: AGENT.md の スキル名（`/skill-name`）が `settings.json` の hooks 定義と一致しているか確認。

**Q: Subagent 並列実行が動作しない**  
A: HARNESS.md の「Split-and-Merge パターン」セクションを参照。複数の `Agent()` 呼び出しが同一メッセージ内に含まれているか確認。

---

**最終更新**: 2025-05-03  
**情報収集期間**: 過去24時間以内（Anthropic公式、GitHub Star100以上リポジトリ、HumanLayer等の信頼ソース）
