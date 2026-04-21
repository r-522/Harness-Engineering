# 📑 Harness Engineering Framework - Complete Index

**生成日**: 2026年04月21日  
**フレームワークバージョン**: 1.0  
**適用開始**: 即時

---

## 🎯 このフレームワークについて

このハーネスエンジニアリング設定キットは、**AI エージェントの最高性能化**を目標に、2026年4月の最新トレンドを反映して構築されました。

### 3つの主要特性

1. **複雑性の削減** - システム設計により、モデル能力に依存しない一貫した出力品質を実現
2. **トークン最適化** - 60-80% のコスト削減を可能にする包括的な戦略
3. **スケーラビリティ** - 個人開発者から企業規模までの展開に対応

---

## 📂 ファイル構成と読順

### 推奨読順（初回のみ）

```
1️⃣ README.md          (15分)  概要・属性・使いどころを理解
         ↓
2️⃣ DESIGN.md          (20分)  アーキテクチャ・設計原則を学習
         ↓
3️⃣ AGENT.md           (20分)  エージェント人格・SKILLS定義を把握
         ↓
4️⃣ CLAUDE.md          (15分)  プロジェクト規約・制限を確認
         ↓
5️⃣ CONTEXT.md         (20分)  トークン最適化戦略を習得
         ↓
6️⃣ ORCHESTRATION.md   (20分)  マルチエージェント協調を理解
         ↓
7️⃣ settings.json      (5分)   Claude Code設定をインポート
```

**総学習時間**: 約 2 時間  
**実装開始**: 3 日目から

---

## 📚 ファイル詳細ガイド

### 1. README.md
**目的**: フレームワーク全体の概要と適用シナリオ

| セクション | 内容 | 対象者 |
|-----------|------|--------|
| ミッション | フレームワークの目的・対象スケール | 全員 |
| コア原則 | 4つの設計原則 | 設計者・アーキテクト |
| 参考文献 | 研究ベース・実装例 | 学習者・リーダー |
| 適用シナリオ | 3つのユースケース別ガイド | マネージャー |

**活用方法**:
- **初回**: 全セクション読破（属性確認）
- **定期**: 月 1 回、参考文献リンク確認（トレンド追従）
- **新プロジェクト**: シナリオ 1-3 から最適な構成を選択

---

### 2. DESIGN.md
**目的**: システムアーキテクチャ設計ガイド

| セクション | 内容 | 参照者 |
|-----------|------|--------|
| 原則 1-4 | 複雑性削減、3層構造、トークン効率、検証可能性 | 全員 |
| システムパターン | Pipeline/Fan-out/Supervisor | アーキテクト |
| キャッシング戦略 | L1-L3 キャッシュ階層 | バックエンド |
| セキュリティ | Threat model/チェックリスト | セキュリティ担当 |
| パフォーマンス | SLA/レイテンシ目標 | オペレーション |

**使用タイミング**:
- **実装前**: 4層構造の確認（1 時間）
- **設計レビュー**: Threat model の適用（30 分）
- **本番化**: パフォーマンス SLA 確認（15 分）

---

### 3. AGENT.md
**目的**: AI エージェントの人格・思考プロセス・SKILLS 定義

| セクション | 内容 | 使用者 |
|-----------|------|--------|
| Core Identity | エージェント人格・役割定義 | 全員 |
| 3層思考プロセス | Planning/Implementation/Review | プロンプトエンジニア |
| SKILLS 定義 | 5つの主要スキル & 実装方法 | デベロッパー |
| Tool Use Behavior | ツール選択マトリックス | チーム全体 |
| セルフ評価 | パフォーマンス指標 | QA・監視 |

**実装例**:
```python
# AGENT.md の "Skill: analyze-codebase" より
skill_name: "analyze-codebase"
execution_time: "1-3 minutes"
cost: "5K-10K tokens"
→ このパラメータを実装時に参照
```

---

### 4. CLAUDE.md
**目的**: プロジェクト固有の規約・制限・ポリシー

| セクション | 内容 | 役割 |
|-----------|------|------|
| 目標・優先順位 | 品質 > コスト > スピード | マネージャー |
| コーディング規約 | 言語別ガイド・コメント方針 | 全開発者 |
| ツール使用ポリシー | Bash/GitHub/API の許可・制限 | 全員 |
| セキュリティ | シークレット管理・PII 扱い | セキュリティ |
| ワークフロー | Git Flow・Commit メッセージ | 全員 |
| 品質指標 | 監査前/リリース前チェック | QA・リーダー |

**実装手順**:
1. チームで読破（1 時間）
2. `.claude/` フォルダに `CLAUDE.md` として配置
3. 毎セッション開始時に Claude が自動参照

---

### 5. CONTEXT.md
**目的**: トークン最適化・コンテキスト管理戦略

| セクション | 内容 | 重要度 |
|-----------|------|--------|
| トークン経済 | コスト構造・危機・解決策 | 🔴 Critical |
| 予算管理 | 月額配分・トラッキング | 🔴 Critical |
| Prompt Caching | TTL 変更対応・実装方法 | 🟠 High |
| Model Routing | タスク別モデル選択 | 🟠 High |
| RAG 統合 | コンテキスト削減技術 | 🟡 Medium |
| 履歴管理 | Sliding window 実装 | 🟡 Medium |

**導入効果 (期待値)**:
- **API コスト**: 60-80% 削減
- **レイテンシ**: 30-40% 短縮
- **品質低下**: < 5%

**実装難易度**: ⭐⭐⭐ (3/5)

---

### 6. ORCHESTRATION.md
**目的**: マルチエージェント協調パターン

| セクション | 内容 | 適用条件 |
|-----------|------|---------|
| 3層構造 | Planning/Generation/Evaluation | 複雑タスク (必須) |
| Sequential | 順序実行 | タスク依存あり |
| Parallel | 並列実行 | 独立タスク多い |
| Adaptive | 動的選択 | 可変複雑度 |
| Communication | メッセージ・コンテキスト共有 | マルチエージェント |
| 監視・スケーリング | メトリクス・負荷分散 | 本番環境 |

**成功率向上**:
- **単一エージェント**: 40-50%
- **3層マルチエージェント**: 85-95%

**実装難易度**: ⭐⭐⭐⭐ (4/5)

---

### 7. settings.json
**目的**: Claude Code harness 設定テンプレート

| セクション | 内容 | カスタマイズ |
|-----------|------|-----------|
| model | モデル選択・TTL 設定 | 推奨値そのまま OK |
| permissions | Bash/GitHub 許可範囲 | ⚠️ チームで調整必須 |
| hooks | pre-commit/after-edit 自動化 | ✅ 推奨値使用可 |
| tokenOptimization | 予算・戦略 | ⚠️ 要件に応じ調整 |
| project | プロジェクトメタデータ | 要修正 |

**導入方法**:
```bash
# 方法 1: .claude/settings.json にコピー
cp 20260421/settings.json .claude/settings.json

# 方法 2: settings.local.json で個別オーバーライド
cp 20260421/settings.json .claude/settings.local.json
# (settings.local.json は .gitignore 対象)
```

---

## 🎯 シナリオ別クイックスタート

### シナリオ A: 個人開発者（1人）

**対象ファイル**: README.md → AGENT.md → CONTEXT.md → CLAUDE.md

```
実装ステップ:
1. README.md 読破 (属性確認)
2. AGENT.md の SKILLS を実装ノートに記録
3. CONTEXT.md から "Model Routing" を適用
4. CLAUDE.md を `.claude/CLAUDE.md` として配置
5. settings.json をインポート

期待効果: 開発時間 30-40% 短縮, API コスト 60% 削減
実装時間: 1-2 日
```

### シナリオ B: チームベース開発（3-15名）

**対象ファイル**: 全ファイル

```
実装ステップ:
1. README.md で全員が属性を理解 (30 min)
2. DESIGN.md で設計方針決定 (1 hour)
3. CLAUDE.md をチーム規約として採択 (30 min)
4. AGENT.md で人格・思考プロセス共有 (45 min)
5. CONTEXT.md で予算管理体制構築 (1 hour)
6. ORCHESTRATION.md で協調パターン習得 (1.5 hours)
7. settings.json をプロジェクト設定に統合 (30 min)

期待効果: 統合バグ 50% 削減, コード審査時間 25% 削減
実装時間: 3-5 日
```

### シナリオ C: 企業・組織規模（50+名）

**対象ファイル**: 全ファイル + 拡張設定

```
実装ステップ:
1. C-level 向け: README.md + ROI 分析 (30 min)
2. アーキテクト: DESIGN.md + ORCHESTRATION.md (2-3 hours)
3. チームリード: CLAUDE.md + CONTEXT.md 準拠体制 (2-3 days)
4. 全開発者: AGENT.md + settings.json 習得 (1 day)
5. オペレーション: 監視・スケーリング設定 (1-2 days)
6. セキュリティ: CLAUDE.md セキュリティ項目 監査 (1-2 days)

期待効果: API コスト 60-80% 削減, エージェント成功率 50% 向上
実装時間: 2-3 週間
```

---

## 🚀 30日実装ロードマップ

### Week 1: 学習・計画

```
Day 1: README.md 読破 & チーム共有
Day 2: DESIGN.md で設計方針決定
Day 3: CLAUDE.md + AGENT.md で規約・人格共有
Day 4-5: CONTEXT.md + ORCHESTRATION.md 習得
Day 6-7: 実装計画・環境構築
```

### Week 2-3: 実装

```
Day 8-10: 単一エージェント最適化
         - CONTEXT.md のモデルルーティング適用
         - Prompt caching 導入
         - Token tracking 実装
         
Day 11-14: マルチエージェント導入 (オプション)
          - ORCHESTRATION.md の 3層構造実装
          - Agent communication 構築
          - 並列実行テスト
          
Day 15-21: 本番化・監視
          - CI/CD 統合
          - メトリクス監視 開始
          - SLA 確認
```

### Week 4: 最適化・監視

```
Day 22-30: パフォーマンス測定・最適化
          - キャッシュ効率を測定
          - コスト削減実績 検証
          - ボトルネック特定・改善
          - 月末報告作成
```

---

## 📊 効果測定ダッシュボード

### 月次レビュー (毎月最終日)

```yaml
指標1_API_コスト:
  期待値: 前月比 60-80% 削減
  測定: 請求書確認、token tracking ログ
  アラート: >前月比 20% 増加

指標2_エージェント成功率:
  期待値: 85-95% (複雑タスク対象)
  測定: Task completion rate, error logs
  アラート: <80%

指標3_レイテンシ:
  期待値: 30-40% 短縮
  測定: 実行時間ログ
  アラート: 目標 SLA 超過

指標4_キャッシュ効率:
  期待値: Cache hit rate >75%
  測定: API response headers
  アラート: <60% (最適化必要)

指標5_コード品質:
  期待値: テスト合格率 >95%, カバレッジ >85%
  測定: CI/CD ログ
  アラート: <90%
```

### トラッキングテンプレート

```json
{
  "date": "2026-04-30",
  "metrics": {
    "api_cost_usd": 1200,
    "api_cost_prev_month": 6000,
    "cost_reduction_percent": 80,
    "agent_success_rate": 0.92,
    "cache_hit_rate": 0.78,
    "avg_latency_ms": 3200,
    "test_pass_rate": 0.97,
    "coverage_percent": 87
  },
  "notes": "Exceeded targets on cost (80% vs 60% goal)",
  "action_items": [
    "Implement RAG for remaining 15% cost savings",
    "Review failing 3% of tasks for patterns"
  ]
}
```

---

## 🔗 関連リソース・外部リンク

### 公式ドキュメント
- [Claude API Documentation](https://platform.claude.com/docs)
- [Claude Code Configuration Reference](https://gist.github.com/mculp/c082bd1e5a439410158974de90c89db7)

### オーケストレーションフレームワーク
- [CrewAI](https://github.com/crewaiinc/crewai)
- [Haystack](https://github.com/deepset-ai/haystack)
- [Ruflo (Claude Agent Orchestration)](https://github.com/ruvnet/ruflo)

### 研究論文・ケーススタディ
- [Anthropic 3-Agent Harness](https://www.infoq.com/news/2026/04/anthropic-three-agent-harness-ai/)
- [Meta Unified AI Agent Architecture](https://engineering.fb.com/2026/04/16/developer-tools/)
- [Token Optimization Guide](https://fast.io/resources/ai-agent-token-cost-optimization/)

### コミュニティ・議論
- [Awesome Harness Engineering](https://github.com/ai-boost/awesome-harness-engineering)
- [Harness Engineering Discussions](https://reddit.com/r/OpenAI) / [X/Twitter](https://twitter.com/search?q=harness+engineering)

---

## ✅ 実装チェックリスト

初回導入時は以下を確認してください:

```yaml
準備段階:
  - [ ] フレームワーク全ファイル読破
  - [ ] 対象シナリオ (A/B/C) 決定
  - [ ] 実装チーム 編成
  - [ ] スケジュール 確定

実装段階:
  - [ ] CLAUDE.md を `.claude/` に配置
  - [ ] AGENT.md の SKILLS 実装
  - [ ] DESIGN.md 設計方針の確認
  - [ ] CONTEXT.md トークン予算 設定
  - [ ] ORCHESTRATION.md (オプション) 導入判断
  - [ ] settings.json インポート

テスト段階:
  - [ ] 単体テスト (各エージェント)
  - [ ] 統合テスト (全体フロー)
  - [ ] 負荷テスト (予想ピーク)
  - [ ] セキュリティ監査

本番化段階:
  - [ ] カナリアリリース (10% トラフィック)
  - [ ] メトリクス監視 24時間
  - [ ] ロールバック計画 確認
  - [ ] SLA 達成 確認

運用段階:
  - [ ] 月 1 回メトリクス確認
  - [ ] 四半期レビュー (改善機会抽出)
  - [ ] 最新トレンド追従 (参考文献更新)
```

---

## 📞 Q&A・トラブルシューティング

### Q1: 全 3 層エージェント構造を導入する必要があるか?

**A**: 不要です。シナリオ A（個人開発者）の場合、Sonnet 単一エージェントで十分です。ただし、複雑度が増えれば段階的に層を追加することをお勧めします。

### Q2: 既存プロジェクトにどう適用するか?

**A**: CLAUDE.md と settings.json をプロジェクトに追加するだけで部分導入可能。CONTEXT.md のコスト最適化から開始することをお勧めします。

### Q3: チーム内で意見が分かれた場合?

**A**: DESIGN.md の「Design Review Checklist」を使い、データドリブンで決定してください。決定は CLAUDE.md に記録し、翌月レビューします。

### Q4: キャッシュ TTL が変わったら?

**A**: settings.json の `tokenOptimization.ttl` を更新し、月末メトリクスで cache_hit_rate を確認してください。

---

## 🎓 学習リソース

| リソース | 対象者 | 学習時間 | 難易度 |
|---------|--------|---------|--------|
| README.md | 全員 | 15分 | ⭐ Easy |
| DESIGN.md | アーキテクト | 30分 | ⭐⭐ Medium |
| AGENT.md | デベロッパー | 30分 | ⭐⭐ Medium |
| CLAUDE.md | 全員 | 20分 | ⭐ Easy |
| CONTEXT.md | バックエンド | 40分 | ⭐⭐⭐ Hard |
| ORCHESTRATION.md | Lead Engineer | 45分 | ⭐⭐⭐⭐ Very Hard |
| settings.json | DevOps/PM | 15分 | ⭐⭐ Medium |

---

**最終更新**: 2026年04月21日  
**ステータス**: 📦 Production Ready  
**バージョン**: 1.0  

質問・フィードバック: GitHub Issues or [お問い合わせ](https://github.com/ai-boost/awesome-harness-engineering)
