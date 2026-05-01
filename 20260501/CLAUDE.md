# CLAUDE.md - プロジェクト構成・コーディング規約・制約定義

## プロジェクト概要

**目的**: ハーネスエンジニアリング環境の最適化と、Claude Code エージェントのパフォーマンス検証

**スタック**: 
- 主言語: JavaScript/TypeScript（Node.js）
- 補助言語: Python（スクリプト・データ分析）
- テストフレームワーク: Jest、Pytest
- ビルドツール: npm/pnpm
- VCS: Git（主流ブランチ: main）

---

## コーディング規約

### 命名規則

| 対象 | 規則 | 例 |
|---|---|---|
| **変数** | camelCase | `const userCount = 42;` |
| **定数** | UPPER_SNAKE_CASE | `const MAX_RETRIES = 3;` |
| **関数** | camelCase (動詞＋目的語) | `function getUserData() {}` |
| **クラス** | PascalCase | `class UserManager {}` |
| **ファイル** | kebab-case（.js/.ts） | `user-manager.ts`, `api-client.ts` |
| **ディレクトリ** | kebab-case | `src/api/`, `tests/unit/` |
| **テストファイル** | `*.test.ts` / `*.spec.ts` | `user-manager.test.ts` |

### コード品質

1. **型安全性**: TypeScript `strict: true` 必須。`any` は禁止、`unknown` 使用
2. **テストカバレッジ**: 新機能は 80% 以上のカバレッジ。バグ修正は最低 1 テストケース
3. **関数の長さ**: 1 関数は 40 行以下が目安。超える場合は分割を検討
4. **複雑度**: 循環複雑度（Cyclomatic Complexity）は 10 以下
5. **コメント**: WHY が非自明な場合のみ。WHAT を説明するコメント不可

### フォーマッティング

- **Formatter**: Prettier（`.prettierrc.json` に準拠）
- **Linter**: ESLint + TypeScript-ESLint
- **行末**: LF（Unix形式）、末尾改行必須
- **インデント**: スペース 2 文字
- **行の最大長**: 120 文字（コメントは 100 文字推奨）

---

## 禁止事項・動作制限

### 絶対禁止

1. **`git reset --hard`, `git push --force`**: 明示的な指示なく実行禁止
2. **`.env`, `secrets.json`, `credentials.json` をコミット**: 秘密情報の漏洩防止
3. **Node.js `require()` の動的実行**: `require(variable)` は禁止、セキュリティリスク
4. **`eval()` または `Function()` コンストラクタ**: コード注入リスク排除
5. **`any` 型の使用**: TypeScript strict modeでは禁止
6. **未テストのコミット**: `npm test` が成功するまでコミット不可
7. **直接的な本番環境への操作**: `NODE_ENV=production` での実行は実験的なみ

### 条件付き制限

| 操作 | 条件 | 代替案 |
|---|---|---|
| 大規模ファイル削除（> 100MB） | 事前合意必須 | `git rm`, `.gitignore` 追加後コミット |
| 依存関係の破壊的更新 | メジャーバージョン変更時は計画書作成 | 段階的マイグレーション計画 |
| データベーススキーマ変更 | 本番環境では migration file 必須 | Backward-compatible migration |

---

## ファイル編集・テスト・ビルド実行ポリシー

### 既存ファイルの編集

**原則**: 確認なく編集可能。ただし以下は必ず確認

1. **アーキテクチャに影響する変更**: ディレクトリ構造の大幅改編、エクスポート仕様の変更
2. **セキュリティ関連**: Auth ロジック、権限管理、秘密情報処理
3. **CI/CD パイプライン**: `.github/workflows/`, `Dockerfile`, `docker-compose.yml`
4. **設定ファイル**: `tsconfig.json`, `.eslintrc.json`, `jest.config.js`

### 新規ファイル作成

**原則**: 明示的な指示がない限り作成不可。ただし以下は除外：

- テストファイル（`*.test.ts`, `*.spec.ts`）
- 型定義ファイル（`*.d.ts`）
- 機能実装時の必要なモジュール（既存ファイルのリファクタリング結果）

### テスト実行

**必須タイミング**:
1. コミット前: `npm test` で全テスト成功
2. ビルド前: `npm run lint` でエラーなし
3. PR作成前: `npm run build` が完了

**コマンド一覧**:

```bash
npm test              # Jest テスト全実行
npm run test:watch   # Watch モード
npm run test:cov     # カバレッジレポート
npm run lint         # ESLint実行
npm run lint:fix     # 自動修正
npm run build        # TypeScript コンパイル
npm run type-check   # 型チェックのみ
```

### ビルド・デプロイ

- **ローカルビルド**: `npm run build` で出力は `dist/` または `build/`
- **本番ビルド**: `npm run build:prod` で最適化・ソースマップ除外
- **デプロイメント**: CI/CD パイプライン経由のみ（`.github/workflows/` 参照）
- **ホットリロード開発**: `npm run dev` で開発サーバー起動（テスト実行時は実行しない）

---

## Claude Code セッションでの振る舞い

### 初期化

セッション開始時に以下を自動実行：

```bash
npm install          # 依存関係確認
npm run type-check   # 型チェック
npm test -- --passWithNoTests  # テスト実行（テストがない場合はスキップ）
```

### 実装フロー

1. **PLAN**: タスク分析（5-10分）、ファイル構造の確認
2. **IMPLEMENT**: コード実装（テスト駆動開発推奨）
3. **TEST**: `npm test` で全テスト成功確認
4. **LINT**: `npm run lint:fix` で自動修正
5. **BUILD**: `npm run build` で TypeScript コンパイル成功確認
6. **COMMIT**: `git commit` で履歴に記録

### エラーハンドリング

- **テスト失敗**: 失敗原因を特定し、コード修正 → テスト再実行まで繰り返す
- **型エラー**: TypeScript strict mode の制約を守り、`any` を使わずに型定義
- **Lint エラー**: 自動修正 (`npm run lint:fix`) 推奨、不可の場合は理由をコメント
- **ビルド失敗**: 構文エラー、不足モジュール、設定エラーを段階的に解決

### コミット戦略

- **1 commit = 1 機能 / 1 バグ修正**: 論理的に独立したユニット
- **コミットメッセージ**: 命令形 + 詳細
  ```
  Fix: Handle null values in user profile validation
  
  - Add null checks before property access
  - Add test case for null input
  - Update user profile type definition
  ```

### セッション終了時

- [ ] 全テスト合格（`npm test`）
- [ ] Lint エラーなし（`npm run lint`）
- [ ] ビルド成功（`npm run build`）
- [ ] コミットメッセージが明確
- [ ] 未テストの機能がない

---

## 連携・エスカレーション

### 自己判断可能な範囲

- バグ修正（テスト付き）
- リファクタリング（動作保証）
- 小規模な機能追加（スコープ < 1 ファイル）
- 依存関係の minor/patch 更新

### 確認が必要な範囲

- **アーキテクチャ変更**: 大幅な構成変更、Design Pattern 導入
- **大規模機能**: スコープ > 5 ファイル、新しいライブラリ導入
- **セキュリティ**: Auth、権限管理、暗号化処理
- **本番環境への操作**: デプロイメント、DB migration

### 連携方法

確認が必要な場合、以下を提示：

1. **現在の状態**: 変更前のコード構造・テスト状況
2. **提案内容**: 実装方針、代替案
3. **判断根拠**: なぜその方針か、トレードオフ
4. **推定時間**: 実装に要する時間

---

## ツール・スキル連携

### 利用可能なスキル

| スキル | トリガー | 用途 |
|---|---|---|
| `/simplify` | コード品質向上時 | コード重複排除、効率化 |
| `/init` | 新規プロジェクト開始時 | CLAUDE.md テンプレート初期化 |
| `/review` | PR作成時 | コードレビュー自動実行 |
| `/loop` | 定期実行タスク時 | CI/CD 定期チェック |

### MCP サーバー

- **Git**: リポジトリ操作（コミット・ブランチ）
- **GitHub**: PR・Issue 管理、Code Review
- **Node.js REPL**: スクリプト検証（大規模実行は避ける）

---

## 参考リンク

- [Claude Code Best Practices](https://code.claude.com/docs/en/best-practices)
- [TypeScript Strict Mode](https://www.typescriptlang.org/tsconfig#strict)
- [Testing Library Best Practices](https://testing-library.com/docs/)
- [Conventional Commits](https://www.conventionalcommits.org/)

