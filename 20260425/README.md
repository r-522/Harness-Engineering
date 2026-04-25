# ハーネスエンジニアリング設定ガイド 2026年4月25日版

## 概要

このディレクトリは、2026年4月最新のAIエージェント・オーケストレーション技術に基づいた、**エンタープライズグレード**の「ハーネスエンジニアリング設定」です。Anthropicの最新研究成果と過去24時間のコミュニティトレンドを反映した構成になっています。

**生成日**: 2026-04-25  
**情報鮮度**: 過去24時間以内  
**対応Claude モデル**: Claude Opus 4.7, Sonnet 4.6, Haiku 4.5  

## 前バージョンからの主な変更点 (20260422 → 20260425)

### 1. **Guided & Sensored Agents の導入**
- **Guides（事前制御）**: エージェントの行動を事前に予測・導向
- **Sensors（事後制御）**: 実行後の自己補正メカニズム
- Orchestratorエージェントに双方向フィードバック機構を統合

### 2. **Generator / Evaluator 分離設計**
- GAN（生成的対抗ネットワーク）に着想を得た設計パターン
- Generatorエージェント: 高速・創造的な仮説生成
- Evaluatorエージェント: 厳密な品質検査
- 両者が協調して精度と創造性のバランスを実現

### 3. **Context Buffer 最適化（33K Token対応）**
- Claude Code 2026年4月更新に対応（45K→33Kトークン）
- `/compact` コマンドの活用ガイド追加
- プロンプトキャッシング（1時間TTL）の実装例

### 4. **Bounded & Deterministic Workflows**
- 無限ループ防止：明確な終了条件と段階制御
- Phase-Gating: 各フェーズ完了後の人間レビュー
- 超大規模エージェントスワームの否定（3-5エージェント推奨）

### 5. **Coaching / Adaptation Loop**
- エージェント間の学習メカニズム
- タスク実行後の自動フィードバック集約
- パフォーマンス指標の動的調整

## このセットアップが対象とするユースケース

### 推奨される用途

- **エンタープライズレベルのマルチエージェントシステム構築**  
  複数の特殊化されたエージェント（Generator、Evaluator、Reasoner、Retriever、Actioner、Validator）を協調させる必要がある組織

- **AIエージェント開発チーム** （規模：5名〜50名）
  エージェントの動作をコード化し、PR・レビュープロセスに組み込みたいチーム

- **本番環境でのAIシステム運用**  
  セキュリティ、ガバナンス、監視が重要な組織

- **高精度が求められるタスク**  
  金融、医療、法律分野など、エラーコストが高い領域

### 対象外のユースケース

- 個人の簡単なプロトタイプ実験（←単一エージェントで十分）
- リアルタイム応答性が最優先（←遅延許容度が低い）
- 単一のジェネラリストエージェント運用

## 構成ファイル解説

### 1. **CLAUDE.md**  
プロジェクト固有のコーディング規約と動作制限。AIエージェントが従うべきローカルルール。

### 2. **AGENT.md**  
エージェントの人格定義、思考プロセス、ツール使用時の振る舞い、スキル定義。Generator/Evaluatorパターンを含む最新の役割分担。

### 3. **DESIGN.md**  
設計原則とアーキテクチャガイドライン。Guide/Sensor制御、Bounded Workflows、MCP統合について。

### 4. **ORCHESTRATION.md**  
エージェント間のオーケストレーション戦略。Plan-Do-Evaluate ループと Adaptive Orchestration の実装。

### 5. **CONTEXT_OPTIMIZATION.md** ✨ **NEW**  
33Kトークンバッファの効率的な活用、プロンプトキャッシング、Sub-Agentによる文脈制御。

### 6. **GUIDED_SENSING.md** ✨ **NEW**  
Guided（事前予測的制御）とSensored（事後フィードバック）メカニズムの詳細実装。

## 2026年4月の最新トレンド反映

### 1. **ハーネスエンジニアリングの進化**

```
Evolution Path:
Prompt Eng. → Context Eng. → Harness Eng.

Harness Eng. = Context Eng. + Infrastructure + Evaluation + Adaptation
```

ハーネスエンジニアリングはプロンプト最適化を超え、エージェントの**環境・制御システム・フィードバックループ**全体を設計する分野へ進化。

**重要**: インフラストラクチャ設定だけで性能が5%以上変動する（Anthropic 2026研究報告）

### 2. **Guided & Sensored Agents**

- **Guide（事前制御）**: エージェントが取るであろう行動を事前予測し、不適切な経路を防ぐ
- **Sensor（事後制御）**: 実行後に自己観測し、誤りに気づき自己補正

この双方向制御により、エージェントの自律性と安全性の両立が可能に。

### 3. **Generator / Evaluator 分離**

```
Generator Agent:  高速、創造的、仮説生成に特化
           ↓
Evaluator Agent:  厳密、検証・品質チェック
           ↓
Feedback Loop:    Generatorの次回試行に反映
```

GAN（生成的対抗ネットワーク）の原理をエージェント設計に適用。

### 4. **Bounded Deterministic Workflows**

- 固定パターンのマルチエージェントスワーム（100+エージェント）は廃止トレンド
- 代わりに **3-5個の専門化エージェント** と **段階的フェーズゲーティング** へシフト
- 各フェーズで人間レビューを挟み、エスカレーション可能な設計

### 5. **Context Buffer 最適化（April 2026 Update）**

- Claude Code バッファ削減: 45K → 33K トークン
- `/compact` コマンドで冗長なコンテキストを自動削除
- プロンプトキャッシング: 1時間TTLで同一クエリの再利用

### 6. **Sub-Agent による文脈隔離**

Dispatcherエージェントは Sub-Agent の**最終結果のみ**参照。  
中間出力の検査や強制パイプラインは、検証が必要な時のみ。

## 実装の推奨流れ

### フェーズ1：基礎設定（1-2週間）
1. CLAUDE.md で規約を定義
2. DESIGN.md でアーキテクチャ決定
3. 1個のGenerator + 1個のEvaluator で簡単な検証

### フェーズ2：拡張（2-4週間）
1. Reasoner、Retriever、Actioner エージェント追加
2. ORCHESTRATION.md の段階制御を実装
3. Guide/Sensor メカニズム統合

### フェーズ3：最適化（継続的）
1. Context最適化ガイド実践
2. パフォーマンス測定と調整
3. Human feedback ループの統合

## 参考資料 & 最新トレンドソース

### Harness Engineering 関連

- [Martin Fowler - Harness engineering for coding agent users](https://martinfowler.com/articles/harness-engineering.html)
- [Medium - From Prompt Engineering to Harness Engineering (Steven Cen, Mar 2026)](https://medium.com/@cenrunzhe/from-prompt-engineering-to-harness-engineering-the-layer-that-makes-ai-agents-actually-work-466fe0489fbe)
- [Addy Osmani - Agent Harness Engineering](https://addyosmani.com/blog/agent-harness-engineering/)
- [Awesome Harness Engineering - GitHub](https://github.com/ai-boost/awesome-harness-engineering)

### Multi-Agent Orchestration

- [Claude Code Docs - Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams)
- [Claude API Docs - Managed Agents Multi-agent](https://platform.claude.com/docs/en/managed-agents/multi-agent)
- [Anthropic Engineering - Scaling Managed Agents](https://www.anthropic.com/engineering/managed-agents)

### April 2026 Updates

- [Claude Code Context Buffer: 33K-45K Token Problem](https://claudefa.st/blog/guide/mechanics/context-buffer-management)
- [Master Claude Code 2026: Context Walls & Usage Limits](https://medium.com/@rameshkannanyt0078/master-claude-code-in-2026-a-developers-guide-to-beating-usage-limits-context-walls-be27f4463450)
- [Claude Code Context Discipline: CLAUDE.md, Memory, MCPs, and Subagents](https://techtaek.com/claude-code-context-discipline-memory-mcp-subagents-2026/)

## ファイル構造

```
20260425/
├── README.md（このファイル）
├── CLAUDE.md（プロジェクト規約・動作制限）
├── AGENT.md（エージェント定義・Generator/Evaluator）
├── DESIGN.md（設計原則・アーキテクチャ）
├── ORCHESTRATION.md（オーケストレーション戦略）
├── CONTEXT_OPTIMIZATION.md（33Kトークン最適化）
└── GUIDED_SENSING.md（Guide/Sensor制御メカニズム）
```

## 使い方

### 推奨ステップ

1. **CLAUDE.md を読む** → チームで合意
2. **DESIGN.md を読む** → アーキテクチャ決定
3. **AGENT.md を確認** → エージェント役割分担
4. **最初は Generator + Evaluator で開始** → 段階的拡張
5. **ORCHESTRATION.md で段階制御を実装**
6. **CONTEXT_OPTIMIZATION.md で最適化を開始**
7. **GUIDED_SENSING.md で高度な制御を統合**

### リポジトリ統合

```bash
# このファイル群をプロジェクトルートにコピー
cp -r 20260425/* /path/to/your/project/harness-config/

# バージョン管理に追加
git add harness-config/
git commit -m "Adopt Harness Engineering 2026-04-25 configuration"
```

## よくある質問

**Q: 前バージョン（20260422）との互換性は？**  
A: 99%互換。主な変更は Generator/Evaluator の役割追加と Context最適化ガイド。既存のエージェント設定はそのまま機能します。

**Q: このセットアップで何個のエージェントが最適？**  
A: 3-5個が推奨。Orchestrator + (Generator, Evaluator, Reasoner, Retriever, Actioner) の中から必要なものを選択。

**Q: Guide/Sensor制御はいつ有効にすべき？**  
A: 本番環境または高精度が求められるタスク。プロトタイプ段階では不要。

**Q: Context最適化（33K token）の実装難度は？**  
A: 低い。提供テンプレートをコピペで対応可能。

---

**最終更新**: 2026-04-25  
**情報源の鮮度**: 過去24時間以内  
**対象Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

