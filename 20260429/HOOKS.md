# HOOKS.md - クラウドワークフロー自動化ガイド

## Hooks とは

**Claude Code Hooks** は、特定のイベント発火時に自動的にシェルコマンドを実行する機構です。

```
イベント発火
  ↓
Hooks 判定
  ├─ PreToolUse: ツール実行「前」
  ├─ PostToolUse: ツール実行「後」
  ├─ PreTask: タスク開始「前」
  └─ Stop: セッション終了「時」
  ↓
条件マッチ
  ↓
シェルコマンド実行
```

**メリット**:
- ✅ 手動の linting・テスト実行を自動化
- ✅ ファイル編集後に自動フォーマット
- ✅ git commit 前の検査を自動実行

---

## Hooks の種類と実装方法

### Hook タイプ一覧

| Hook | トリガー | 用途 |
|---|---|---|
| **PreToolUse** | Claude がツール実行「前」 | 実行前チェック（dry-run等） |
| **PostToolUse** | Claude がツール実行「後」 | 実行後の自動テスト・フォーマット |
| **PreTask** | タスク開始時 | 環境準備、デバッグ情報表示 |
| **Stop** | セッション終了時 | クリーンアップ、ログ保存 |

### 設定ファイル

Hooks は `~/.claude/settings.json` に記述：

```json
{
  "hooks": {
    "enabled": true,
    "PreToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.ts",
        "command": "echo 'Editing TypeScript file: $FILE'"
      }
    ],
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.{ts,js}",
        "command": "npx prettier --write $FILE && npm run type-check"
      }
    ],
    "Stop": [
      {
        "command": "git status"
      }
    ]
  }
}
```

---

## 実装パターン

### Pattern 1: ファイル編集後の自動フォーマット

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.ts",
        "command": "npx prettier --write $FILE && npx eslint --fix $FILE"
      }
    ]
  }
}
```

**動作フロー**:
```
Claude がファイルを Edit
    ↓
PostToolUse Hook トリガー
    ↓
prettier → code format
    ↓
eslint --fix → lint 自動修正
    ↓
ユーザーに結果報告
```

**効果**: 手動でフォーマットを実行する手間が削減

### Pattern 2: テストファイル変更時の自動テスト実行

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "**/*.test.ts",
        "command": "npm test -- --testPathPattern=$(basename $FILE) --watch=false"
      }
    ]
  }
}
```

**動作フロー**:
```
Claude がテストファイルを編集
    ↓
PostToolUse Hook トリガー
    ↓
該当するテストを実行
    ↓
テスト結果（pass/fail）を表示
    ↓
テスト失敗 → Claude が修正を試みる
```

### Pattern 3: Commit 前のチェック

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Bash"],
        "pattern": "git commit",
        "command": "npm run type-check && npm test && echo '✅ All checks passed'"
      }
    ]
  }
}
```

**セキュリティ機能**:
```bash
# Secret 検出
git diff --cached | grep -E 'password|api.key|secret' && {
  echo "❌ Secret found in commit"
  exit 1
}
```

### Pattern 4: 複数ファイル編集時の関連テスト自動実行

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/services/**/*.ts",
        "command": "npm test -- --testPathPattern='services' --watch=false --passWithNoTests"
      }
    ]
  }
}
```

### Pattern 5: セッション終了時のクリーンアップ

```json
{
  "hooks": {
    "Stop": [
      {
        "command": "echo '=== Session Summary ===' && git status && echo '=== Uncommitted Changes ===' && git diff --stat"
      }
    ]
  }
}
```

**効果**: セッション終了時に自動で Git 状態を表示し、コミット漏れを防止

---

## チーム向け Hooks 設定

### プロジェクト共有設定（.claude/settings.json）

```json
{
  "hooks": {
    "enabled": true,
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.{ts,tsx}",
        "description": "TypeScript format and lint",
        "command": "npx prettier --write $FILE && npx eslint --fix $FILE"
      },
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.{py}",
        "description": "Python format with black",
        "command": "black $FILE && flake8 $FILE"
      },
      {
        "tools": ["Edit"],
        "pattern": "**/*.json",
        "description": "JSON format",
        "command": "npx prettier --write $FILE"
      }
    ]
  }
}
```

**チーム導入フロー**:
```
1. リーダーが .claude/settings.json を作成
2. Git に commit
3. チームメンバーが clone して自動適用
4. 全員が同じ Hooks で統一
```

### 個人用上書き設定（~/.claude/settings.json）

個人の開発環境が異なる場合、ローカル設定で上書き可能：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.ts",
        "command": "my-custom-formatter $FILE"  // 個人用フォーマッター
      }
    ]
  }
}
```

---

## Subagent との並行実行

### Subagent でも Hooks を適用

```
親エージェント（Hooks 有効）
    ↓
Subagent A（ファイル編集）
    ↓
PostToolUse Hook トリガー
    ↓
自動テスト実行
```

**注意**: Subagent は親の Hook を継承しない設計。各 Subagent で独立した hooks を設定する場合は、明示的に指定が必要。

### Subagent オーケストレーション + Hooks

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.ts",
        "command": "npm test -- --testPathPattern=$(basename $FILE) --watch=false; echo $?"
      }
    ]
  },
  "subagents": {
    "code-formatter": {
      "instructions": "You are a code formatter specialist",
      "tools": ["Edit", "Read"]
    },
    "test-runner": {
      "instructions": "You run tests and report results",
      "tools": ["Bash"]
    }
  }
}
```

---

## Hook パラメータと変数

### 利用可能な環境変数

```bash
$FILE            # 編集されたファイルパス
$TOOL            # 実行されたツール名（Edit, Bash等）
$TIMESTAMP       # Unix timestamp
$CLAUDE_SESSION  # セッション ID
$PROJECT_ROOT    # プロジェクトルートパス
```

### Hook 条件指定

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],              // 複数指定可
        "pattern": "src/**/*.ts",       // glob パターン
        "description": "説明（ログ表示用）",
        "command": "command here",
        "async": false,                 // true = バックグラウンド実行
        "failureMode": "warn"           // warn = エラーでも続行
      }
    ]
  }
}
```

---

## トラブルシューティング

### Hook が実行されない

```bash
# Step 1: 設定ファイル構文確認
cat ~/.claude/settings.json | jq '.hooks'

# Step 2: Pattern マッチ確認
# ファイルパスが glob パターンに合致しているか検証

# Step 3: Hooks が有効か確認
jq '.hooks.enabled' ~/.claude/settings.json
# → false なら true に変更

# Step 4: Claude Code 再起動
```

### Hook が遅く感じる

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.ts",
        "command": "fast-formatter $FILE",  // 高速化
        "async": true                       // 非同期実行
      }
    ]
  }
}
```

### Hook でエラーが発生

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.ts",
        "command": "prettier --write $FILE || true",  // || true でエラーを無視
        "failureMode": "warn"                          // または warn に設定
      }
    ]
  }
}
```

---

## ベストプラクティス

### Hook コマンド設計指針

✅ **良い例**:
```bash
# 高速・安全・単機能
npx prettier --write "$FILE"

# 複数ステップは && で連結
npx prettier --write "$FILE" && npx eslint --fix "$FILE"

# エラー時は明示的に処理
command-name "$FILE" || echo "⚠️ Warning: command failed"
```

❌ **避けるべき例**:
```bash
# 遅い・危険
find . -type f -name "*.ts" | xargs prettier  # 全ファイル処理は避ける

# 副作用が大きい
rm -rf node_modules  # 削除操作は危険

# サイレント失敗
command-name || true > /dev/null 2>&1  # エラーを隠さない
```

### Hook の粒度

```
粒度が小さすぎる（避ける）:
PostToolUse → 毎回 1 ファイルのテストで遅延

粒度が適切（推奨）:
PostToolUse → 変更したセクション関連のテストのみ

粒度が大きすぎる（避ける）:
PostToolUse → 全テストを実行（時間がかかりすぎ）
```

### Hook の監視・ログ

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.ts",
        "command": "echo '[Hook] Formatting $FILE at $(date)' && prettier --write '$FILE'",
        "description": "Log each hook execution"
      }
    ]
  }
}
```

---

## 高度な使用例

### Pattern: CI/CD パイプラインの模擬

```json
{
  "hooks": {
    "Stop": [
      {
        "description": "Run full CI pipeline before session ends",
        "command": "npm run lint && npm run type-check && npm test && npm run build"
      }
    ]
  }
}
```

### Pattern: 複数環境での Hook 切り替え

```bash
# 環境変数で Hook コマンドを変更
if [ "$ENV" = "production" ]; then
  FORMATTER="black --line-length=100"
else
  FORMATTER="black --line-length=120"
fi

# Hook に組込
cat > ~/.claude/settings.json << EOF
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.py",
        "command": "$FORMATTER \$FILE"
      }
    ]
  }
}
EOF
```

### Pattern: リアルタイム通知付き Hook

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "tools": ["Edit"],
        "pattern": "src/**/*.ts",
        "command": "npm test && curl -X POST https://hooks.slack.com/services/... -d '{\"text\":\"Tests passed\"}' || true"
      }
    ]
  }
}
```

---

## チェックリスト

### 初回設定
- [ ] `~/.claude/settings.json` で hooks を有効化
- [ ] 1-2 個の basic hook から始める（例: prettier）
- [ ] 手動実行と Hook 実行の結果を比較・確認

### 段階的拡張
- [ ] 基本 Hook（フォーマッタ）が安定
  → テスト実行 Hook を追加
- [ ] テスト Hook が安定
  → チーム設定（.claude/settings.json）を作成
- [ ] チーム適用完了
  → より複雑な Hook を検討（Slack 通知等）

### 月次レビュー
- [ ] Hook の実行時間を確認（1 秒以上かかるなら最適化）
- [ ] Hook エラーログを確認（失敗パターンの改善）
- [ ] チーム内で Hook 運用の feedback を収集

---

**最終更新**: 2026-04-29  
**参照すべきファイル**: SECURITY.md（hook の権限設定）  
**全体構成確認**: README.md に戻る
