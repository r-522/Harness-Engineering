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

## 2026年4月時点の主要トレンド

### 1. **コンテキスト管理が最優先**
- LLMのパフォーマンスはコンテキストウィンドウ充満度に直線相関
- サブエージェント分離により、各エージェントが新鮮なコンテキストで動作
- システムプロンプトはキャッシング対象化（60分 → 5分TTLに変更）

### 2. **ハーネスがパフォーマンス変数の主要因**
- モデル自体の性能差より、ハーネス設計が5%以上の性能差を生成
- 環境エンジニアリング：APIやコードベース自体をAI可読性に最適化
- 決定性とフェイルセーフ性が設計の中心（CAAF - Convergent AI Agent Framework）

### 3. **マルチエージェント協調の収束パターン**
Anthropicが推奨する5つの協調パターン：
- **Orchestrator-Subagent**: 階層的タスク委譲（推奨スタート）
- **Agent Teams**: 永続的メンバーシップ、ドメイン知識の蓄積
- **Generator-Verifier**: 出力生成と検証の分離
- **Shared State**: 永続ストアを通じた非同期協調
- **Message Bus**: イベント駆動型パイプライン

### 4. **プロンプトキャッシングの革新**
- 2026年初旬の重大変更：TTL 60分 → 5分への短縮
- 適切に実装すれば月額数千ドルのコスト削減が可能
- アンチパターン：タイムスタンプをキャッシュ対象に含める（毎回キャッシュミス）

### 5. **システムベースのアプローチ**
- アドホックなプロンプティングから卒業
- `.claude/agents/`でスペシャライズドアシスタント定義
- SKILLsの自動トリガー（パターンマッチングベース）
- 実行自動化とワークフロー管理

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

### 公式ドキュメント
- [Claude Code Best Practices - Anthropic](https://code.claude.com/docs/en/best-practices)
- [Anthropic Agents](https://claude.com/solutions/agents)
- [Prompt Caching Documentation](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

### 2026年の重要な考察
- [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices)
- [GitHub - shanraisshan/claude-code-best-practice](https://github.com/shanraisshan/claude-code-best-practice)
- [10 Must-Have Skills for Claude - Medium](https://medium.com/@unicodeveloper/10-must-have-skills-for-claude-and-any-coding-agent-in-2026-b5451b013051)
- [Agent Harness Engineering: The Rise of the AI Control Plane - Medium](https://medium.com/@adnanmasood/agent-harness-engineering-the-rise-of-the-ai-control-plane-938ead884b1d)
- [The Art and Science of Harness Engineering - Frank's World](https://www.franksworld.com/2026/04/20/the-art-and-science-of-harness-engineering-redefining-ai-agent-performance/)
- [Prompt Caching in 2026: The 5-Minute TTL Change - DEV Community](https://dev.to/whoffagents/claude-prompt-caching-in-2026-the-5-minute-ttl-change-thats-costing-you-money-4363)
- [Anthropic Multi-Agent Coordination Framework](https://blockchain.news/news/anthropic-multi-agent-coordination-patterns-framework)
- [How we built our multi-agent research system - Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system)

### 認定オブジェクト
- [Awesome Harness Engineering - ai-boost/awesome-harness-engineering](https://github.com/ai-boost/awesome-harness-engineering)
- [VILA-Lab/Dive-into-Claude-Code](https://github.com/VILA-Lab/Dive-into-Claude-Code)

## 属性

| 属性 | 値 |
|-----|-----|
| **生成日時** | 2026-04-27 |
| **情報鮮度** | 過去24時間以内 |
| **推奨Claude Model** | claude-opus-4.7（複雑度高）/ claude-sonnet-4.6（バランス） |
| **対応プラットフォーム** | Claude Code (Web/Desktop), Anthropic SDK, GitHub Copilot |
| **バージョン** | 1.0 (April 2026) |

## ライセンス

このドキュメント群はパブリックドメインです。自由に改変・配布できます。

---

**最終更新**: 2026-04-27
