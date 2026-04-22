# ハーネスエンジニアリング設定ガイド 2026年4月版

## 概要

このディレクトリは、2026年最新のAIエージェント・オーケストレーション技術に基づいた、エンタープライズグレードの「ハーネスエンジニアリング設定」です。Anthropicの最新Claude APIと市場トレンドを反映した構成になっています。

**生成日**: 2026-04-22（情報鮮度：過去24時間以内）

## このセットアップが対象とするユースケース

### 推奨される用途
- **エンタープライズレベルのマルチエージェントシステム構築**
  - 複数の特殊化されたエージェント（推論エージェント、検索エージェント、アクションエージェント、検証エージェント）を協調させる必要がある組織
  
- **AIエージェント開発チーム** （規模：5名〜）
  - エージェントの動作をコード化し、PR・レビュープロセスに組み込みたいチーム
  - プロンプト、スキル、MCP設定をバージョン管理したいチーム
  
- **本番環境でのAIシステム運用**
  - セキュリティ、ガバナンス、監視が重要な組織
  - Human-on-the-loop な意思決定プロセスが必要なケース

### 対象外のユースケース
- 個人の簡単なプロトタイプ実験
- 単一のジェネラリストエージェントで十分な小規模タスク

## 構成ファイル解説

### 1. **CLAUDE.md** 
プロジェクト固有のコーディング規約と動作制限。AIエージェントが従うべきローカルルール。

### 2. **AGENT.md**
エージェントの人格定義、思考プロセス、ツール使用時の振る舞い、SKILLS定義。マルチエージェント設定に含まれるそれぞれの役割を明示。

### 3. **DESIGN.md**
設計原則とアーキテクチャガイドライン。特にマルチエージェント間の協調パターン（シーケンシャル、階層型、並列、イベント駆動）と、Model Context Protocol (MCP) 統合について。

### 4. **ORCHESTRATION.md**
エージェント間のオーケストレーション戦略。計画→実行→評価(Plan-Do-Evaluate) ループの構成と、adaptive orchestration の実装方法。

### 5. **HARNESS.md**
ハーネスエンジニアリングの具体的なツール設定。ガードレール、フィードバックループ、監視(Observability)層、セキュリティ設定。

## 2026年最新トレンドの反映

### 1. **ハーネスエンジニアリングの重要性**
- インフラストラクチャ設定が性能に5%以上の影響を与える（Anthropic 2026 Agentic Coding Trends Report）
- コードのようにスキル、プロンプト、MCP設定をバージョン管理
- PR・レビュープロセスで品質管理

### 2. **マルチエージェント・オーケストレーション**
- 汎用エージェント1つではなく、特殊化されたエージェントチーム
- 動的なオーケストレーション：固定パターンではなくタスク特性に応じて戦略を選択
- Human-on-the-loop への移行（2026年の高度なビジネスの特徴）

### 3. **コンテキスト・エンジニアリング**
- XML タグの時代は終了 → より自然な構造化
- 明示的な指示が最重要
- クエリをコンテキストの後に配置（モデルの注意機構の最適化）

### 4. **Adaptive Thinking と Sub-Agent**
- Claude Opus 4.7 の extended thinking が複雑推論を内部化
- Sub-agent による文脈制御：ディスパッチャーエージェントは sub-agent の最終結果のみ見る
- プロンプトチェーンは中間出力の検査や強制パイプラインが必要な時のみ

### 5. **Model Context Protocol (MCP)**
- エージェントと外部システムの統合標準
- セキュアで監視可能な接続が実現可能

### 6. **セキュリティ & ガバナンス**
- エージェント間の通信セキュリティ
- データアクセスの最小権限化
- 監査ログの完全性

## 実装の流れ

### フェーズ1：基礎設定（1-2週間）
1. CLAUDE.md で規約を定義
2. DESIGN.md でアーキテクチャ決定
3. 1-3 エージェントのシンプルなシステムで検証

### フェーズ2：拡張（2-4週間）
1. ORCHESTRATION.md に基づき本格的なマルチエージェントシステムへ
2. MCP 統合
3. 監視・ガバナンス層の構築

### フェーズ3：最適化（継続的）
1. ハーネス設定の反復改善
2. パフォーマンス測定と調整
3. Human feedback ループの統合

## 参考資料 & トレンドソース

### 公式ドキュメント
- [Claude API Docs - Prompt Engineering Best Practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices)
- [Claude Docs - Prompt Engineering Overview](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview)

### AI Agent Orchestration
- [Best Practices for AI Agent Implementations: Enterprise Guide 2026](https://onereach.ai/blog/best-practices-for-ai-agent-implementations/)
- [Unlocking exponential value with AI agent orchestration - Deloitte](https://www.deloitte.com/us/en/insights/industry/technology/technology-media-and-telecom-predictions/2026/ai-agent-orchestration.html)
- [AI Agent Orchestration in 2026: What Enterprises Need to Know - Kanerika](https://kanerika.com/blogs/ai-agent-orchestration/)
- [Best practices for building agentic systems - InfoWorld](https://www.infoworld.com/article/4154570/best-practices-for-building-agentic-systems.html)
- [AI Agent Orchestration Patterns - Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [How to orchestrate AI agents - Gumloop](https://www.gumloop.com/blog/how-to-orchestrate-ai-agents)

### Harness Engineering
- [Harness engineering: Structured workflows for AI-assisted development - Red Hat Developer](https://developers.redhat.com/articles/2026/04/07/harness-engineering-structured-workflows-ai-assisted-development)
- [Harness engineering - leveraging Codex in an agent-first world - OpenAI](https://openai.com/index/harness-engineering/)
- [What Is Harness Engineering? Complete Guide for AI Agent Development (2026) - NxCode](https://www.nxcode.io/resources/news/what-is-harness-engineering-complete-guide-2026)
- [Skill Issue: Harness Engineering for Coding Agents - HumanLayer Blog](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)
- [Everything I Learned About Harness Engineering - Escape Tech](https://escape.tech/blog/everything-i-learned-about-harness-engineering-and-ai-factories-in-san-francisco-april-2026/)

### Multi-Agent Design Patterns
- [Multi-Agent Architecture Design Patterns - Medium](https://medium.com/@princekrampah/multi-agent-architecture-in-multi-agent-systems-multi-agent-system-design-patterns-langgraph-b92e934bf843)
- [Google's Eight Essential Multi-Agent Design Patterns - InfoQ](https://www.infoq.com/news/2026/01/multi-agent-design-patterns/)
- [Four Design Patterns for Event-Driven, Multi-Agent Systems - Confluent](https://www.confluent.io/blog/event-driven-multi-agent-systems/)

### Claude System Prompts & Latest Models
- [GitHub - Piebald-AI/claude-code-system-prompts](https://github.com/Piebald-AI/claude-code-system-prompts)
- [Changes in the system prompt between Claude Opus 4.6 and 4.7 - Simon Willison](https://simonwillison.net/2026/apr/18/opus-system-prompt/)
- [How Claude Code Builds a System Prompt](https://www.dbreunig.com/2026/04/04/how-claude-code-builds-a-system-prompt.html)

## ファイル構造

```
20260422/
├── README.md（このファイル）
├── CLAUDE.md（プロジェクト規約）
├── AGENT.md（エージェント定義）
├── DESIGN.md（設計原則）
├── ORCHESTRATION.md（オーケストレーション戦略）
└── HARNESS.md（ハーネス設定）
```

## 使い方

1. このファイル群を プロジェクトルートにコピー
2. チームで CLAUDE.md を確認し、合意する
3. 最初は小規模な 1-3 エージェント システムから開始
4. ORCHESTRATION.md の指針に従い段階的に拡張
5. 各ファイルは PR・レビュープロセスで管理

---

**最終更新**: 2026-04-22
**情報源の鮮度**: 直近24時間以内
**対象Claude モデル**: Claude Opus 4.7, Sonnet 4.6, Haiku 4.5
