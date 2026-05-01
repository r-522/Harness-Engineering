# HOOKS.md - イベント駆動型自動化ガイド

## Hooks 概要

**目的**: Claude Code のイベント（ファイル編集、ツール実行など）に対して、自動でシェルコマンドやプロンプト実行を割り当てる

**動作原理**: Event → HookMatcher → Condition Check → Commands Execute

**設定場所**: `.claude/settings.json` の `hooks` キー

---

## Hooks の 4 大イベント

### 1. PreToolUse（ツール実行前）

**トリガー**: Claude が任意のツール（Bash, Edit, Write等）を実行しようとする

**使用例**:
- ファイル編集前の型チェック
- Bash コマンド実行前の安全性確認
- セキュリティチェック

**設定例**:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": {
          "tool": "Edit"
        },
        "commands": [
          {
            "type": "shell",
            "command": "npm run type-check"
          }
        ]
      }
    ]
  }
}
```

**チェック内容**:
- TypeScript コンパイルエラーの検出
- 構文エラーの事前発見
- Lint 警告の確認

### 2. PostToolUse（ツール実行後）

**トリガー**: ツール実行完了後（成功・失敗問わず）

**使用例**:
- ファイル編集後のテスト実行
- ビルド検証
- ログ出力

**設定例**:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": {
          "tool": "Bash",
          "command": "npm run build"
        },
        "commands": [
          {
            "type": "shell",
            "command": "npm test"
          }
        ]
      }
    ]
  }
}
```

**実行フロー**:
1. `npm run build` 実行
2. ビルド完了
3. `npm test` 自動実行（PostToolUse）

### 3. Stop（セッション終了時）

**トリガー**: Claude Code セッション終了時

**使用例**:
- 未テストのコード検出・警告
- キャッシュクリア
- 最終チェックサム計算

**設定例**:

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": {},
        "commands": [
          {
            "type": "shell",
            "command": "npm test"
          },
          {
            "type": "shell",
            "command": "npm run lint"
          }
        ]
      }
    ]
  }
}
```

### 4. Notification（通知イベント）

**トリガー**: Claude が重要なイベント（エラー、成功）を通知

**使用例**:
- テスト失敗時の Slack 通知
- ビルド完了時のメール通知
- デプロイ成功時のログ記録

**設定例**:

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": {
          "type": "error"
        },
        "commands": [
          {
            "type": "shell",
            "command": "curl -X POST https://hooks.slack.com/services/... -d '{\"text\": \"Build failed\"}'"
          }
        ]
      }
    ]
  }
}
```

---

## Matcher 定義

### Matcher とは

Hooks を実行するかどうかを判定する条件。複数の条件を AND / OR で結合可能

### 主要な Matcher フィールド

| フィールド | 型 | 説明 | 例 |
|---|---|---|---|
| `tool` | string | ツール名 | `"Edit"`, `"Bash"`, `"Write"` |
| `command` | regex または string | 実行コマンド | `"npm test"`, `"/npm.*/` |
| `path` | regex または string | 対象ファイルパス | `"/src/.*\\.ts/"` |
| `type` | string | イベントタイプ | `"error"`, `"success"`, `"warning"` |
| `exitCode` | number | シェルの終了コード | `0` (成功), `1` (失敗) |

### Matcher 例

**例 1: TypeScript ファイル編集時のみ型チェック**

```json
{
  "matcher": {
    "tool": "Edit",
    "path": "/\\.ts$/"
  }
}
```

**例 2: テストコマンド失敗時のみリトライ**

```json
{
  "matcher": {
    "tool": "Bash",
    "command": "/npm test/",
    "exitCode": 1
  }
}
```

**例 3: ファイル削除操作時の確認**

```json
{
  "matcher": {
    "tool": "Bash",
    "command": "/rm -rf/"
  }
}
```

---

## Commands 定義

### Command タイプ

| タイプ | 説明 | 実行例 |
|---|---|---|
| `shell` | シェルコマンド実行 | `npm test`, `git status` |
| `llm` | LLM プロンプト実行 | 検査・分析プロンプト |
| `webhook` | HTTP ウェブフック呼び出し | Slack 通知、GitHub Actions トリガー |
| `mcp` | MCP ツール呼び出し | Git コマンド、GitHub API操作 |

### Shell Command

```json
{
  "type": "shell",
  "command": "npm test",
  "timeout": 30000,
  "failOnError": true
}
```

**パラメータ**:
- `command`: 実行するコマンド
- `timeout`: タイムアウト時間（ミリ秒）、デフォルト 60000
- `failOnError`: エラー時に Hook チェーンを停止するか

### LLM Prompt

```json
{
  "type": "llm",
  "prompt": "このコードにセキュリティ問題がないかレビューしてください。発見された問題があれば、修正案を提案してください。",
  "model": "claude-opus-4-7"
}
```

**使用例**: コードレビュー、セキュリティチェック、デバッグ支援

### Webhook

```json
{
  "type": "webhook",
  "url": "https://hooks.slack.com/services/YOUR/WEBHOOK/URL",
  "method": "POST",
  "payload": {
    "text": "ビルドが成功しました: {{timestamp}}"
  }
}
```

**テンプレート変数**:
- `{{timestamp}}`: タイムスタンプ
- `{{tool}}`: ツール名
- `{{command}}`: 実行コマンド
- `{{exitCode}}`: 終了コード

---

## 実装パターン

### パターン 1: 編集後の自動テスト

**目標**: TypeScript ファイル編集後、自動的に該当テストを実行

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": {
          "tool": "Edit",
          "path": "/src\\/.*\\.ts$/"
        },
        "commands": [
          {
            "type": "shell",
            "command": "npm run type-check"
          },
          {
            "type": "shell",
            "command": "npm test -- --testPathPattern={{path}}.test.ts"
          }
        ]
      }
    ]
  }
}
```

**フロー**:
1. Claude が `src/services/user.ts` を編集
2. PostToolUse イベント発火
3. `npm run type-check` 実行
4. `npm test -- --testPathPattern=src/services/user.ts.test.ts` 実行

### パターン 2: ビルド失敗時のスラック通知

**目標**: `npm run build` 失敗時、Slack で通知

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": {
          "tool": "Bash",
          "command": "npm run build",
          "exitCode": 1
        },
        "commands": [
          {
            "type": "webhook",
            "url": "https://hooks.slack.com/services/...",
            "payload": {
              "text": "❌ Build failed",
              "color": "danger"
            }
          }
        ]
      }
    ]
  }
}
```

### パターン 3: セッション終了時の最終チェック

**目標**: セッション終了前にテスト＆Lint 実行

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": {},
        "commands": [
          {
            "type": "shell",
            "command": "npm test"
          },
          {
            "type": "shell",
            "command": "npm run lint"
          },
          {
            "type": "shell",
            "command": "npm run build"
          }
        ]
      }
    ]
  }
}
```

### パターン 4: ファイル編集前の安全性確認

**目標**: 機密ファイル（`.env`, `secrets.json`）の編集を防止

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": {
          "tool": "Edit",
          "path": "/(\\.env|\\.env\\.local|secrets\\.json)$/"
        },
        "commands": [
          {
            "type": "llm",
            "prompt": "このファイルは機密情報を含む可能性があります。編集前に確認してください。本当に編集しますか？"
          }
        ]
      }
    ]
  }
}
```

### パターン 5: 保護されたディレクトリへの削除操作防止

**目標**: `node_modules/`, `.git/` の誤削除を防止

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": {
          "tool": "Bash",
          "command": "/rm -rf.*(node_modules|\\.git)/"
        },
        "commands": [
          {
            "type": "llm",
            "prompt": "重要なディレクトリの削除操作が検出されました。本当に削除しますか？"
          }
        ]
      }
    ]
  }
}
```

---

## Hooks 設定ガイド

### ベストプラクティス

1. **Matcher は具体的に**
   - `tool` と `path` または `command` の組み合わせで特定
   - 過度に広い Matcher は副作用を生む

2. **Command チェーンは短く**
   - 3 コマンド以上は不要
   - 複雑な処理は Shell Script ファイル化

3. **Timeout を設定**
   - 長時間処理は `timeout` 値を調整
   - デフォルト 60 秒は多くのテストに不足

4. **failOnError で制御**
   - 必須チェック: `failOnError: true`
   - オプション通知: `failOnError: false`

### トラブルシューティング

**Hooks が実行されない場合**:

1. `.claude/settings.json` の JSON 構文確認
2. Matcher 条件が正確か再確認
3. `tool` / `command` / `path` フィールド確認

**Hooks の動作確認**:

```bash
# Claude Code で Hooks 定義確認
# Settings → Hooks → View Configuration
```

**無限ループ防止**:

PostToolUse → Shell Command → ファイル変更 → 再度 PostToolUse のループを避ける

```json
// ❌ 悪い例: 無限ループの危険性
{
  "PostToolUse": [
    {
      "matcher": { "tool": "Bash" },
      "commands": [
        { "type": "shell", "command": "npm run lint:fix" }  // ← ファイル変更
      ]
    }
  ]
}

// ✅ 良い例: 上流で防止
{
  "PreToolUse": [
    {
      "matcher": { "tool": "Bash", "command": "npm run lint" },
      "commands": [
        { "type": "shell", "command": "npm run lint:fix" }
      ]
    }
  ]
}
```

---

## 参考文献

- [Claude Code settings - Claude Code Docs](https://code.claude.com/docs/en/settings)
- [Claude Code Hooks: Complete Guide to All 12 Lifecycle Events](https://claudefa.st/blog/tools/hooks/hooks-guide)
- [Claude Code settings.json Deep Dive (Part 3): The Hooks System](https://blog.vincentqiao.com/en/posts/claude-code-settings-hooks/)
- [Claude Code settings.json: Complete config guide (2026)](https://www.eesel.ai/blog/settings-json-claude-code)

