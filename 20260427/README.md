# ハーネスエンジニアリング設定キット 2026-04-27

## 概要

このディレクトリは、**最新のAIエージェント実行環境**の設定ファイル群です。2026年4月時点の業界ベストプラクティスと、Anthropic Claude APIの最適化戦略に基づいています。

## 適用対象

- **用途**: マルチエージェント協調システム、複雑なコード解析・生成タスク、AI駆動の研究システム
- **規模**: 個人開発者〜中規模チーム向け
- **主要フレームワーク**: Claude Code、Anthropic SDK、カスタムエージェント統合

## このキットに含まれるもの

| ファイル | 説明 |
|---------|------|
| **CLAUDE.md** | Claude Codeの動作制約、コーディング規約、セキュリティ要件 |
| **AGENT.md** | エージェントの人格定義、推論プロセス、ツール使用戦略、SKILLS定義 |
| **DESIGN.md** | ハーネスアーキテクチャの設計原則、最適化ガイドライン |
| **ORCHESTRATION.md** | マルチエージェント協調パターン、ワークフロー設計 |
| **PERFORMANCE.md** | プロンプトキャッシング、コンテキスト管理、コスト最適化 |
| **SECURITY.md** | セキュリティレビュー基準、インジェクション対策、認証フロー |

## 2026年4月時点の主要トレンド（情報収集: 2026-04-27、過去24時間以内）

### 1. **Plan Mode による思考の構造化**
- タスク実行をPlan（計画）フェーズとAction（実装）フェーズに分離
- 計画段階で失敗のリスクを早期検出・修正
- 不確実な課題は Plan Mode でユーザー確認を取得
- 複数の実行パスを並列検討（コンテキスト汚染軽減）

### 2. **コンテキスト管理が最優先課題**
- LLMのパフォーマンスはコンテキストウィンドウ充満度に直線相関（実測: 60%超で性能10-20%低下）
- サブエージェント分離により、各エージェントが新鮮なコンテキストで動作
- RTK（Response Token Kompressor）ツール活用で出力圧縮
- システムプロンプト・共通ライブラリは優先的にキャッシング対象化（推奨: 60分TTL明示設定）

### 3. **ハーネスエンジニアリングがモデル選択を上回る**
- モデル自体の性能差 (5%) < ハーネス設計 (5-10%の性能差生成)
- 「環境工学」：APIやコードベース自体をAI可読性に最適化
- 決定性とフェイルセーフ性が設計の中心（CAAF - Convergent AI Agent Framework）
- 型指定、関数シグネチャ、変数命名がAI理解度を大幅向上

### 4. **マルチエージェント協調の5つの標準パターン**
Anthropic推奨の協調パターン（複雑度順）：
1. **Orchestrator-Subagent**: 階層的タスク委譲（推奨スタート、低オーバーヘッド）
2. **Agent Teams**: 永続的メンバーシップ、ドメイン知識の蓄積
3. **Generator-Verifier**: 出力生成と検証の分離（品質二重チェック）
4. **Shared State**: 永続ストア（Redis/DB）を通じた非同期協調
5. **Message Bus**: イベント駆動型パイプライン（リアルタイム処理向け）

### 5. **Skills と Hooks による自動化の成熟**
- **Skills**: 週1回以上発動する定期タスクの自動化（トリガー: パターンマッチング）
- **Skill Collaboration Pattern**: スキル間の出力チェーニング（前スキルの出力 → 次スキルの入力）
- **Hooks**: コミット前・後の自動処理（テスト実行、秘密情報スキャン）
- 推奨: CLAUDE.md に明示的に skills セクション記述

## ファイル構造と使用方法

### スタートアップ
1. `CLAUDE.md`をプロジェクトルートにコピー（`.claude/CLAUDE.md`）
2. `AGENT.md`の人格定義をカスタマイズ
3. `ORCHESTRATION.md`から適切なパターンを選択
4. `PERFORMANCE.md`でコスト最適化パラメータを設定

### ランタイム
- `DESIGN.md`をチーム内で共有し、設計方針を統一
- `SECURITY.md`をコード審査時に参照
- パフォーマンスボトルネック発見時は`PERFORMANCE.md`の最適化チェックリストを実行

## 参考資料

### Anthropic 公式リソース（最新確認: 2026-04-27）
- [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices) - Plan Mode、Context Management、Skills活用
- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents) - Subagent設計と権限管理
- [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams) - マルチエージェント協調パターン
- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills) - Skills定義とトリガー
- [Introduction to subagents - Anthropic Skilljar](https://anthropic.skilljar.com/introduction-to-subagents) - Subagent基礎

### コミュニティ・エンタープライズ実装例（2026年4月）
- [GitHub - shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice) - CLAUDE.md テンプレート、orchestration workflow実装
- [GitHub - abhishekray07/claude-md-templates](https://github.com/abhishekray07/claude-md-templates) - CLAUDE.mdテンプレート集
- [GitHub - davila7/claude-code-templates](https://github.com/davila7/claude-code-templates) - CLI設定ツール
- [GitHub - VoltAgent/awesome-claude-code-subagents](https://github.com/VoltAgent/awesome-claude-code-subagents) - 100+ Subagentパターン
- [GitHub - trailofbits/claude-code-config](https://github.com/trailofbits/claude-code-config) - セキュリティ設定テンプレート

### 最新ベストプラクティス記事（2026年4月 24-27日）
- [Claude Code Best Practices for Better Coding 2026 - Evartology Substack](https://evartology.substack.com/p/claude-code-best-practices-for-better-coding-may-2026) - Plan Mode、Context污染対策、Skills自動化
- [Claude Code Tips I Wish I'd Had From Day One - Marmelab](https://marmelab.com/blog/2026/04/24/claude-code-tips-i-wish-id-had-from-day-one.html) - 実践的なキャッシング、RTKツール活用
- [50 Claude Code Tips and Best Practices For Daily Use - Builder.io](https://www.builder.io/blog/claude-code-tips-best-practices) - 50個の実装パターン
- [Claude Code Best Practices: Lessons From Real Projects - RanTheBuilder](https://ranthebuilder.cloud/blog/claude-code-best-practices-lessons-from-real-projects/) - 実プロジェクト事例

### Skills と Hooks の実装ガイド
- [The Complete Guide to Building Skills for Claude - Anthropic Resources](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf) - Skills設計原則、skill collaboration pattern
- [Claude Code Advanced Best Practices: 11 Practical Techniques for Hooks, Subagents & Context Management - SmartScope](https://smartscope.blog/en/generative-ai/claude/claude-code-best-practices-advanced-2026/) - Hooks、Context最適化、TTL設定
- [What Is Claude Code's Skill Collaboration Pattern? - MindStudio](https://www.mindstudio.ai/blog/claude-code-skill-collaboration-pattern) - Skill チェーニング、自動トリガー
- [Writing a good CLAUDE.md - HumanLayer Blog](https://www.humanlayer.dev/blog/writing-a-good-claude-md) - CLAUDE.md 設計原則

### マルチエージェント協調パターン（Orchestration）
- [Orchestrate teams of Claude Code sessions - Claude Code Docs](https://code.claude.com/docs/en/agent-teams) - 公式パターン定義（Orchestrator-Subagent、Agent Teams、Generator-Verifier等）
- [The Code Agent Orchestra - Addy Osmani](https://addyosmani.com/blog/code-agent-orchestra/) - マルチエージェント協調の設計原則
- [Claude Code Sub-Agents: Parallel vs Sequential Patterns - ClaudeFAST](https://claudefa.st/blog/guide/agents/sub-agent-best-practices) - 並列実行戦略、パターン比較
- [Claude Code Subagents and Main-Agent Coordination - Medium](https://medium.com/@richardhightower/claude-code-subagents-and-main-agent-coordination-a-complete-guide-to-ai-agent-delegation-patterns-a4f88ae8f46c) - Delegation戦略、hub-and-spoke パターン
- [Best practices for Claude Code subagents - PubNub Blog](https://www.pubnub.com/blog/best-practices-for-claude-code-sub-agents/) - 権限管理、context isolation
- [5 Claude Code Agentic Workflow Patterns - MindStudio](https://www.mindstudio.ai/blog/claude-code-agentic-workflow-patterns) - 5つのワークフローパターン（Sequential, Operator, Split-and-Merge, Teams, Headless）
- [Multi-Agent Orchestration: Running 10+ Claude Instances in Parallel - DEV Community](https://dev.to/bredmond1019/multi-agent-orchestration-running-10-claude-instances-in-parallel-part-3-29da) - 大規模並列実行

### 高度な最適化（プロンプトキャッシング、コンテキスト管理）
- [Dive into Claude Code: The Design Space of Today's and Future AI Agent Systems - arXiv](https://arxiv.org/html/2604.14228v1) - 学術的フレームワーク、設計空間分析
- [Claude Certification Guide — Architect Foundations Exam](https://claudecertificationguide.com/learn/1-agentic-architecture/1-2-orchestration-patterns) - 認定資格基準、検証項目

## 属性と情報収集詳細

| 属性 | 値 |
|-----|-----|
| **生成日時** | 2026-04-27 |
| **情報収集期間** | **過去24時間以内**（2026-04-27 00:00 〜 06:00 JST） |
| **検索対象** | Anthropic公式ドキュメント、GitHub（Stars 100+）、Medium、Substack、DEV Community |
| **言及度の高い主要トピック** | Plan Mode、Context Management、Subagent Orchestration、Skills & Hooks、CLAUDE.md設定、Environment Engineering |
| **推奨Claude Model** | claude-opus-4.7（複雑度高）/ claude-sonnet-4.6（バランス）/ claude-haiku-4.5（軽量タスク） |
| **対応プラットフォーム** | Claude Code (Web/Desktop), Anthropic SDK, Claude API |
| **バージョン** | 1.0 (April 27, 2026) |
| **ファイル形式** | Markdown + YAML（スキル定義） |

## ライセンス

このドキュメント群はパブリックドメインです。自由に改変・配布できます。

---

**最終更新**: 2026-04-27
