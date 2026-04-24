# AI エージェント最高パフォーマンス環境 - 20260424版

## ミッション概要
このディレクトリは、2026年4月24日時点における**AI エージェント（特に Claude Code）の最高パフォーマンス環境**を実現するための統合設定ファイル群です。Harness Engineering と AI Orchestration の最新ベストプラクティスを基に、構成されています。

## 対象ユース・ケース
- **適用対象**: Web 開発、データ分析、研究開発、ソフトウェア開ンジニアリング全般
- **最適規模**: 個人開発者 ~ チーム規模（10-50人程度）
- **実装難易度**: 中程度（設定ファイルの読み込みと理解が必要）

## 情報源と鮮度
- **収集日時**: 2026年4月24日
- **情報源**: HumanLayer, Anthropic Engineering Blog, Medium, GitHub, 技術ニュースサイト
- **参考文献**:
  - [Harness Engineering for Coding Agents - HumanLayer](https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents)
  - [12 Agentic Harness Patterns - Medium](https://medium.com/@simranjeetsingh1497/agent-harness-12-agentic-harness-patterns-from-claude-code-5505b7c239c4)
  - [Effective Harnesses for Long-Running Agents - Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
  - [The Definitive Guide to Agent Harness Engineering - Medium](https://engineeratheart.medium.com/the-definitive-guide-to-agent-harness-engineering-5f5edf25fd73)
  - [Agent Skills Overview - Claude API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)

## ディレクトリ構成
```
20260424/
├── README.md              # このファイル：構成解説、属性、参考文献、使いどころ
├── CLAUDE.md              # プロジェクト固有のコーディング規約、動作制限
├── AGENT.md               # AI人格、思考プロセス、ツール使用の振る舞い、SKILLS定義
├── DESIGN.md              # 設計原則、アーキテクチャガイドライン
├── HARNESS.md             # Harness Engineering 実装パターン
└── ORCHESTRATION.md       # Multi-Agent Orchestration 戦略
```

## 核心原則（2026年最新知見）

### 1. Context が Everything
**従来の誤解**: プロンプト文言に何時間も費やす  
**2026年の現実**: Context Structure が長期タスク全体の一貫性を決定する  
**実装**: CLAUDE.md で context を厳密に構造化

### 2. Tool Count の最適化
- **スイート・スポット**: 8-15 tools
- **危険域**: 50+ tools（attention fragmentation により機能停止レベルの低下）
- **パターン**: Tools は必要に応じて動的に有効化

### 3. Prompt Engineering → Orchestration へのパラダイム・シフト
**2025年**: 単一タスク用の完璧なプロンプトを磨く  
**2026年**: Multiple Agents が連携する Workflow 設計が主要スキル  
**実装**: ORCHESTRATION.md を参照

### 4. Model Routing（動的選択）
```
- Haiku:   簡単な質問、構文ヘルプ、単純な編集（高速）
- Sonnet:  複雑なリファクタリング、アーキテクチャ判断
- Opus:    研究開発、複雑な推論が必要な場面
```

### 5. Hooks によるオートメーション
- Pre-commit hooks: lint, format, type check
- Post-tool-use hooks: 自動フォーマット、品質チェック
- Session hooks: Context initialization, cleanup

## 実装ロードマップ

### Phase 1: Foundation（日1）
1. CLAUDE.md を読む（5分）
2. AGENT.md の SKILLS 定義を確認（5分）
3. Pre-commit hooks を設定（10分）
4. DESIGN.md のアーキテクチャパターンを確認（10分）

### Phase 2: Integration（日2-3）
1. HARNESS.md のパターンを プロジェクトに適用
2. Tools 数を監査・最適化
3. Context structure を実装
4. Multi-agent workflow の設計（必要に応じて）

### Phase 3: Validation（日4+）
1. パフォーマンス測定（token usage, latency）
2. エージェント failures の分析
3. Feedback loops の構築

## Key Metrics

| メトリクス | 目標値 | 検証方法 |
|-----------|-------|--------|
| Tools 数 | 8-15 個 | `grep -c "tool"` CLAUDE.md |
| CLAUDE.md 行数 | < 50 | `wc -l` CLAUDE.md |
| Context compression 率 | 20-30% | Token usage logs |
| Agent success rate | > 85% | Task completion logs |
| Average response latency | < 5s | Performance monitoring |

## 警告と制限

- **Context Window**: Haiku 4.5 は 200K tokens、Sonnet は 200K tokens、Opus は 200K tokens。Context 圧縮は必須。
- **Rate Limiting**: API rate limits に注意。大規模プロジェクトは token caching を有効化。
- **Tool Permissions**: セキュリティ上、tools アクセスは明示的に許可してください。

## トラブルシューティング

**Q: Agent がツール選択に失敗する**  
A: Tools 数が多すぎる可能性。10-15 個に削減して再試行。

**Q: Context がすぐに満杯になる**  
A: CLAUDE.md の context structure を見直し。Compression hooks を設定。

**Q: レスポンスが遅い**  
A: Model を Haiku に変更するか、/compact を実行。

## サポートとフィードバック

- 問題報告: GitHub Issues
- 拡張提案: Pull Requests
- コミュニティ: Claude Code Discord

---

**生成日時**: 2026年4月24日  
**バージョン**: 1.0  
**ステータス**: 本番環境対応（Production Ready）
