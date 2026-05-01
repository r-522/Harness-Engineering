# Claude Code ハーネスエンジニアリング設定パック 2026

**生成日時**: 2026年4月30日 | **情報収集期間**: 過去24時間以内

## 概要

Claude Codeの最新ベストプラクティスに基づいた、**小規模チーム向けの汎用ソフトウェア開発プロジェクト**用設定ドキュメント群です。本パックは、Web上の最新トレンド分析から自動生成され、プロンプトキャッシング、MCP統合、サブエージェント連携、コンテキスト管理の最適化を実装しています。

---

## 属性と判定根拠

| 属性 | 選択値 | 判定根拠 |
|---|---|---|
| **用途** | 汎用ソフトウェア開発 | テンプレート・プロジェクト構成・チーム共有設定が言及の中心 |
| **規模** | 小規模チーム(〜5名) | settings.json と settings.local.json の個人・チーム分離が標準パターン |
| **対象者** | 中級者向け | ベストプラクティス集・実践的パターン・上級テクニック混在 |

---

## ディレクトリ構成と役割

```
20260430/
├── README.md                 ← 本ファイル（構成・導入ガイド）
├── CLAUDE.md                 ← プロジェクト規約・コーディング制約
├── AGENT.md                  ← エージェント定義・思考プロセス
├── DESIGN.md                 ← 設計原則・アーキテクチャガイド
├── CONTEXT.md                ← コンテキストウィンドウ管理戦略
├── MCP.md                    ← MCP サーバー統合ガイド
├── CACHING.md                ← プロンプトキャッシング実装パターン
└── HOOKS.md                  ← hooks.json 定義・自動化ルール
```

| ファイル | 用途 | サイズ目安 |
|---|---|---|
| **CLAUDE.md** | プロジェクト固有の規約・禁止事項・テスト方針 | 60行以下（推奨） |
| **AGENT.md** | ペルソナ・思考プロセス・スキル定義・エラーハンドリング | 実装詳細ベース |
| **DESIGN.md** | SOLID原則・DRY・設計パターン・ディレクトリ構成ルール | ガイドライン形式 |
| **CONTEXT.md** | コンテキストウィンドウ容量管理・セッション設計 | 実装チェックリスト |
| **MCP.md** | MCP サーバー設定・インライン定義・権限管理 | 実装例付き |
| **CACHING.md** | キャッシュ戦略・TTL・費用最適化・キャッシュヒット率 | パターン集 |
| **HOOKS.md** | Pre/Post hooks・自動化トリガー・スクリプト例 | 実装テンプレート |

---

## 導入手順

### 1. プロジェクトへの統合

```bash
# ブランチ切り替え
git checkout claude/keen-curie-xtGUj

# ドキュメントのコピー
cp -r 20260430/* /path/to/your/project/

# CLAUDE.md の初期化（自動生成オプション）
# 手動で以下を実行してもよい：
# cd /path/to/your/project && /init

# コミット
git add CLAUDE.md AGENT.md DESIGN.md .claude/ settings.json
git commit -m "chore: Add Claude Code harness configuration (2026 best practices)"
```

### 2. 既存プロジェクトへの適用

- **CLAUDE.md**: 既存ファイルがあれば、本パックの CLAUDE.md 内容をマージ（新規作成の場合はそのまま採用）
- **.claude/settings.json**: プロジェクトの環境に合わせて MCP サーバーを追加・削除
- **.claude/agents/**: サブエージェント定義を `AGENT.md` に従って実装
- **hooks.json**: `HOOKS.md` のテンプレートからチーム の自動化ニーズに合わせてカスタマイズ

### 3. チーム展開

```bash
# プロジェクトレベル設定をコミット
git commit --allow-empty -m "ci: Initialize Claude Code harness for team"
git push origin claude/keen-curie-xtGUj

# チームメンバーは以下で同期
git pull origin claude/keen-curie-xtGUj
# settings.local.json は .gitignore に入れている（個人設定保持）
```

---

## カスタマイズガイド

### シナリオ 1: コンテキスト容量が不足している場合

**症状**: 「context window approaching limit」の警告が頻繁に出現

**対応**:
1. `CONTEXT.md` の「セッション容量計算式」を確認
2. `/init` で CLAUDE.md を再生成し、不要なセクションを削除
3. 大規模ファイルの読み込みは `Explore` エージェントに委譲（AGENT.md 参照）
4. 古いメッセージの自動圧縮タイミングを `settings.json` で調整

### シナリオ 2: MCP サーバーが応答しない

**対応**:
1. `MCP.md` の「デバッグ手順」セクション実行
2. `settings.json` の mcpServers セクションから該当サーバーの定義を確認
3. `--enable-mcp-debug` フラグで詳細ログを出力
4. インライン定義 vs グローバル定義の使い分けを確認（AGENT.md）

### シナリオ 3: プロンプトキャッシュが効いていない

**対応**:
1. `CACHING.md` の「キャッシュヒット率診断」実行
2. `cache_control` メタデータの配置を確認（構造化データの前に配置すること）
3. キャッシュブレークポイント（「最後の変更ブロック」）の移動を監視
4. 1時間 TTL への切り替え検討（コスト vs パフォーマンストレード）

### シナリオ 4: Hooks が実行されない

**対応**:
1. `HOOKS.md` の「トリガー条件」リストで該当ルールを確認
2. `.claude/hooks.json` の JSON 構文を検証
3. `settings.json` の hookExecutionMode が「enabled」になっているか確認
4. 権限プロンプトが出ていないか確認（permission denied の場合は `settings.json` を更新）

---

## 参考文献

**基本設定・ベストプラクティス:**
- [Best Practices for Claude Code - Claude Code Docs](https://code.claude.com/docs/en/best-practices) — 公式設定ガイド（2026年対応）
- [Claude Code: Workflows and Best Practices 2026](https://smart-webtech.com/blog/claude-code-workflows-and-best-practices/) — 実践パターン集
- [50 Claude Code Tips and Best Practices For Daily Use](https://www.builder.io/blog/claude-code-tips-best-practices) — Builder.io による実用技法
- [Claude Code Tips I Wish I'd Had From Day One](https://marmelab.com/blog/2026/04/24/claude-code-tips-i-wish-id-had-from-day-one.html) — 実装者の知見（2026年4月24日）
- [Claude Code Advanced Best Practices: 11 Practical Techniques for Hooks, Subagents & Context Management [2026]](https://smartscope.blog/en/generative-ai/claude/claude-code-best-practices-advanced-2026/) — 上級技法解説

**CLAUDE.md テンプレート:**
- [GitHub - abhishekray07/claude-md-templates](https://github.com/abhishekray07/claude-md-templates) — テンプレート集（複数スタック対応）
- [How to Create the Perfect CLAUDE.md](https://www.gradually.ai/en/claude-md/) — 詳細ガイド
- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills) — スキル定義公式ドキュメント

**MCP & サブエージェント:**
- [Create custom subagents - Claude Code Docs](https://code.claude.com/docs/en/sub-agents) — 公式サブエージェント仕様
- [Install and Configure MCP Servers in Claude Code (2026)](https://systemprompt.io/guides/claude-code-mcp-servers-extensions) — MCP統合ガイド
- [Understanding Claude Code's Full Stack: MCP, Skills, Subagents, and Hooks Explained](https://alexop.dev/posts/understanding-claude-code-full-stack/) — アーキテクチャ解説
- [Building agents with the Claude Agent SDK](https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk) — Anthropic 公式エージェント設計

**プロンプトキャッシング:**
- [Prompt caching - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) — キャッシング公式仕様（2026年2月更新）
- [Claude Prompt Caching: Cut LLM API Costs 70%](https://jangwook.net/en/blog/en/claude-api-prompt-caching-cost-optimization-guide/) — コスト最適化パターン
- [Anthropic API Pricing in 2026: Complete Guide](https://www.finout.io/blog/anthropic-api-pricing) — 料金体系・キャッシュ費用

**ハーネス実装例:**
- [GitHub - Chachamaru127/claude-code-harness](https://github.com/Chachamaru127/claude-code-harness) — 公式ハーネス実装（Plan→Work→Review サイクル）
- [Everything Claude Code: Inside the 82K-Star Agent Harness](https://medium.com/@tentenco/everything-claude-code-inside-the-82k-star-agent-harness-thats-dividing-the-developer-community-4fe54feccbc1) — 業界事例分析（2026年3月）
- [GitHub - affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) — パフォーマンス最適化ハーネス

---

## トラブルシューティング早見表

| 現象 | 確認項目 | 参照ドキュメント |
|---|---|---|
| コンテキスト容量超過警告 | CLAUDE.md 行数、読み込みファイル数 | CONTEXT.md |
| MCP サーバー接続失敗 | settings.json の mcpServers 定義 | MCP.md |
| キャッシュヒット率 0% | cache_control フィールド配置、TTL 設定 | CACHING.md |
| Hooks が実行されない | hookExecutionMode、JSON 構文、権限 | HOOKS.md |
| サブエージェント起動失敗 | .claude/agents/ パス、YAML フロントマター | AGENT.md |

---

## よくある質問（FAQ）

**Q: CLAUDE.md の推奨行数は？**  
A: 60行以下（HumanLayer ベストプラクティス）。超過するとClaude が内容の一部を無視します。

**Q: settings.json と settings.local.json の違いは？**  
A: settings.json はリポジトリにコミット（チーム共有）、settings.local.json は .gitignore 入り（個人設定）。機密情報・個人の環境変数は .local.json に。

**Q: コンテキストウィンドウの容量は？**  
A: 200,000トークン。推奨は 60% 以下（120,000トークン）で運用開始時点で既に 20,000トークン消費。

**Q: プロンプトキャッシュはいつ自動適用される？**  
A: cache_control メタデータを付与した時点で自動適用。キャッシュブレークポイントは「最後の変更ブロック」から後方。

**Q: サブエージェントとスキルの違いは？**  
A: スキル = 機能（CLI呼び出し）、サブエージェント = 独立した思考エンティティ（独自コンテキスト＋権限）。

---

## ライセンスと更新

本設定パックは **CC0 1.0 Universal** (パブリックドメイン相当) の下で提供されます。自由に複製・改変・配布いただけます。

**更新ポリシー:**
- Claude Code の公式更新（モデルリリース・仕様変更）があった場合は随時更新
- 最新版は [GitHub リポジトリ](https://github.com/r-522/harness-engineering) を参照
- 本 README の「情報収集期間」は各版の鮮度を示します

---

**作成者**: Claude Code AI Assistant | **バージョン**: 1.0.0-2026.04.30
