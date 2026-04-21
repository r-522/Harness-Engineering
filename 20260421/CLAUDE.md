# CLAUDE.md - Project Conventions & Operational Constraints

**Generated**: April 21, 2026  
**Harness Engineering Specification Version**: 1.0

## 📋 プロジェクト概要

このファイルは、Claude Code を用いて開発を行う際の**プロジェクト固有ルール・制限・コンテキスト管理方針**を定義します。

エージェント実行時に毎セッション参照され、モデルの意思決定・ツール使用・コード生成に直接影響を与えます。

## 🎯 目標と優先順位

### Primary Goals
1. **高品質コード生成** - エラー率 <3%, テストカバレッジ >85%
2. **トークン効率** - API コスト 60-80% 削減維持
3. **チーム協調** - コード一貫性 >95%, PR レビュー時間 <2時間

### Secondary Goals
4. セキュリティ・コンプライアンス維持
5. ドキュメント完全性
6. パフォーマンス最適化

## 📝 コーディング規約

### ファイル・命名規則

```
- ファイル名: snake_case (JavaScript/TypeScript でも統一)
- クラス名: PascalCase
- 関数名: camelCase
- 定数: UPPER_SNAKE_CASE
- プライベートメンバー: _prefixWithUnderscore
```

### 言語別ガイドライン

#### Python
- バージョン: 3.10+
- スタイル: PEP 8 + Black formatter
- 型注釈: 100% 必須（関数シグネチャ）
- ドキュメント: Google style docstrings（1行のみ、詳細なコメント不可）

#### TypeScript / JavaScript
- ES2022 以上
- 型チェック: `strict: true`
- フォーマッター: Prettier
- テスト: Jest or Vitest
- 命名: camelCase（HTML attributes は除く）

#### Go
- フォーマッター: gofmt
- 型安全性: 全関数に戻り値型を明示
- エラーハンドリング: 常に nil チェック

### コメント方針

**ルール**: コメントは WHY に専念。WHAT や HOW は実装方法で表現。

```python
# ✅ Good: 理由を述べる
# Skip cache on first call because model context differs significantly
if is_first_call and has_cache_control:
    cache_control = None

# ❌ Bad: WHAT を説明している
# Check if this is the first call and remove cache control
if is_first_call and has_cache_control:
    cache_control = None
```

### エラーハンドリング

**方針**: システム内部のエラーは信頼。外部入力（API、ユーザー入力）のみバリデーション。

```python
# ✅ Good
result = json.loads(response_body)  # 内部 API なら安全

# ❌ Bad: 不要なチェック
if isinstance(result, dict):  # 内部コードなら不要
    process(result)
```

## 🔧 ツール使用ポリシー

### Bash コマンド

**許可**:
- `git` コマンド（push/pull/commit）
- `npm`, `pip`, `go mod`（パッケージ管理）
- テスト・ビルド実行（Jest, pytest, go test）
- ファイル操作（mkdir, rm, mv）

**制限（事前確認必須）**:
- データベース操作（DROP, TRUNCATE）
- サーバー操作（restart, deploy）
- 外部 API 呼び出し（制限レート・実行回数）
- `--force`, `--no-verify` フラグ

**禁止**:
- `sudo` コマンド
- 暗号資格の直接操作
- システムワイドな設定変更

### GitHub MCP Tool

**許可**:
- PR 作成・更新（レビュー待ち）
- Issue 作成・更新
- コメント投稿（技術的な説明）

**制限（事前確認必須）**:
- PR マージ（ユーザーが指示するまで待機）
- ブランチ削除
- Issue クローズ（理由説明後）

**禁止**:
- 無関連な Issue への返信
- 自動マージ設定変更
- リリース 公開

### API コール

**トークン予算**: 月 500K プロンプト・トークン（キャッシュ活用で延長）

**コスト最適化ルール**:
```
- システムプロンプト: 必ずキャッシュ対象に
  (cache_control: { "type": "ephemeral" } または "type": "prefill")
- コンテキスト: 最小化（不要なファイル・履歴は除外）
- モデル選択: 
  * 計画フェーズ: Claude Haiku
  * 実装フェーズ: Claude Sonnet
  * 複雑な推論: Claude Opus
```

## 🛡️ セキュリティ・コンプライアンス

### シークレット管理

- **絶対禁止**: API キー、パスワード、トークンの commit
- **チェック**: すべての commit 前に git hooks で secret scanning 実行
- **.env ファイル**: `.gitignore` に必ず登録
- **公開リポジトリ**: credentials を `settings.local.json` に（.gitignore 対象）

### データ取扱

- PII（個人識別情報）: ログ出力禁止
- 本番データ: 開発環境では必ずマスク
- ストレージ: 暗号化（保存時・転送時）

## 📊 コンテキスト管理方針

### システムプロンプト構成

エージェント実行時のプロンプト構成:

```
1. Core Instructions (キャッシュ対象)
   ├── Mission & Goals
   ├── Coding Standards
   ├── Tool Usage Policy
   └── Security Guidelines

2. Project Context (新規追加可)
   ├── Tech Stack
   ├── Architecture Overview
   └── Key Dependencies

3. Conversation History (キャッシュ対象外)
   └── Recent messages + artifacts
```

### コンテキストウィンドウ制限

- **最大入力トークン**: 170K（Claude 3 Opus）
- **安全マージン**: 20K 予約（応答用）
- **実効最大**: 150K（プロンプト+履歴）

**最適化方法**:
- 不要なファイル履歴は削除
- テスト結果は要約版を保持
- 大規模ファイルは 段落ごとに分割参照

## 🔄 ワークフロー

### タスク実行フロー

```
1. Planning (Haiku)
   - タスク分解
   - 依存関係分析
   - 実装順序決定

2. Implementation (Sonnet)
   - コード生成
   - ユニットテスト
   - インテグレーション

3. Review & Optimization (Opus)
   - コード品質検査
   - パフォーマンス測定
   - 本番対応性確認

4. Deployment (Sonnet)
   - CI/CD テスト実行
   - ドキュメント更新
   - Git push
```

### Git ワークフロー

**ブランチ戦略**: Git Flow

```
main (プロダクション)
  ↑
├── release/* (リリース準備)
│
develop (統合ブランチ)
  ↑
├── feature/* (機能開発)
├── bugfix/* (バグ修正)
└── hotfix/* (緊急修正)
```

**Commit メッセージ規格**:

```
<type>(<scope>): <subject> (50文字以内)

<body> (72文字以内のラッピング)

<footer> (参照 Issue など)

type: feat|fix|refactor|docs|test|chore
scope: api|frontend|backend|infra など
```

## 📈 品質指標・チェックリスト

### コード審査前チェック

- [ ] 型チェック合格（TypeScript: `tsc --noEmit`）
- [ ] 単体テスト 合格率 100%
- [ ] カバレッジ >85%
- [ ] Linter エラー 0
- [ ] Secret scanning クリア
- [ ] API コスト見積 < 月額予算

### リリース前チェック

- [ ] インテグレーション テスト 合格
- [ ] エッジケース テスト 完了
- [ ] パフォーマンス ベンチマーク OK
- [ ] セキュリティ 監査 完了
- [ ] ドキュメント 最新化
- [ ] リリースノート 作成

## 🔐 マルチセッション通継ぎ

### セッション終了時に保存すべき情報

```json
{
  "context_summary": "現在のタスク進捗",
  "key_decisions": ["決定1", "決定2"],
  "pending_tasks": ["タスク1", "タスク2"],
  "api_costs_ytd": "月額使用量",
  "branch_state": "現在の git branch"
}
```

### セッション開始時の初期化

1. CLAUDE.md を読む
2. 前回の context_summary から復帰
3. pending_tasks を確認
4. API コスト予算を確認

## ⚠️ 制限事項・注意点

### 実装不可な要件

- **リアルタイム処理**: 5秒以上の待機時間が必要な場合は不可
- **バージョン管理**: 3世代以上前の依存関係サポート不可
- **外部サービス統合**: API ドキュメント未公開なサービス不可

### エスカレーション基準

以下の場合は **即座にユーザーに相談**:

1. 実装コスト > 2時間の複雑タスク
2. セキュリティに関わる判断が必要
3. 設計方針の重大な変更
4. リソース制限超過の可能性

## 📞 サポート・相談

**問題が生じた場合**:

1. ユーザーに状況報告
2. 代替案 2-3 個を提示
3. 推奨案を明記
4. 実装は ユーザーの指示を待つ

---

**最終更新**: 2026年04月21日  
**承認者**: Harness Engineering Team  
**適用開始**: 即時
