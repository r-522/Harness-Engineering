# HOOKS.md - Hook 自動化シナリオ・実装ガイド

## I. Hook の本質

### 定義
**Hook** = Claude Code のライフサイクルの特定ポイントで自動実行される shell command / HTTP endpoint / LLM prompt。

### 特徴
- **設定駆動**: Code 不要（`settings.json` で定義）
- **イベントベース**: 21 のライフサイクルイベント対応（2026年3月時点）
- **複数ハンドラ型**: Command / HTTP / Prompt / Agent（4種類）
- **Async 実行**: 非同期処理対応

---

## II. Hook ライフサイクルイベント（21個）

### Cadence 1: Per-Session（セッション単位）

| イベント | タイミング | 用途 |
|---|---|---|
| **SessionStart** | 新セッション開始時 | 環境初期化、cache clear、git status check |
| **SessionEnd** | セッション終了時 | Log 圧縮、一時ファイル削除、audit trail 保存 |

### Cadence 2: Per-Turn（ユーザーターン単位）

| イベント | タイミング | 用途 |
|---|---|---|
| **UserPromptSubmit** | ユーザー入力送信時 | 入力検証、敏感情報チェック、request logging |
| **Stop** | Claude 応答完了時 | 出力レビュー、secret scan、summary 生成 |
| **StopFailure** | エラーで停止時 | Error log、root cause analysis、ユーザー通知 |

### Cadence 3: Per-Tool（Tool 呼び出し単位）

| イベント | タイミング | 用途 |
|---|---|---|
| **PreToolUse** | Tool 実行**前** | 権限チェック、入力検証、dry-run |
| **PostToolUse** | Tool 実行**後** | 出力フォーマット、secret masking、logging |
| **ToolUseFailure** | Tool 失敗時 | エラー通知、リトライ、代替 tool 試行 |
| **ToolResult** | Tool 結果受信時 | 結果キャッシュ、データ変換 |

### Cadence 4: Agentic Loop（エージェント反復）

| イベント | タイミング | 用途 |
|---|---|---|
| **ThinkingStart** | Claude 思考開始時 | Plan logging |
| **ThinkingEnd** | Claude 思考完了時 | Decision capture |
| **ToolSelection** | Tool 選択時 | Tool 選択ログ、recommendation |
| **ToolExecution** | Tool 実行中 | Progress monitoring |
| **ResultProcessing** | 結果処理中 | Data transform |

### Cadence 5: Subagent / Context

| イベント | タイミング | 用途 |
|---|---|---|
| **SubagentSpawn** | Subagent 起動時 | Spawn log、context isolation check |
| **SubagentComplete** | Subagent 完了時 | Result aggregation |
| **ContextWarning** | Context 70%+ 到達時 | Alert、context compression trigger |
| **ContextExceeded** | Context full 時 | Emergency cleanup |

### Cadence 6: Error / Recovery

| イベント | タイミング | 用途 |
|---|---|---|
| **ErrorDetected** | エラー検出時 | Error classification、recovery strategy |
| **RecoveryAttempt** | リカバリ試行時 | Retry log、fallback 実行 |
| **RecoveryFailed** | リカバリ失敗時 | Escalation、human alert |

---

## III. Hook ハンドラ型（4種類）

### 1. Command Handler

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "handlers": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\""
          }
        ]
      }
    ]
  }
}
```

**特徴**: Shell script 実行、環境変数アクセス可、同期実行

**環境変数**:
- `$CLAUDE_TOOL_NAME`: Tool 名
- `$CLAUDE_TOOL_INPUT_FILE_PATH`: ファイルパス
- `$CLAUDE_SESSION_ID`: Session ID
- `$CLAUDE_USER_NAME`: ユーザー名

### 2. HTTP Handler

```json
{
  "hooks": {
    "SessionEnd": [
      {
        "handler": {
          "type": "http",
          "method": "POST",
          "url": "https://logs.example.com/api/session-end",
          "headers": {
            "Authorization": "Bearer $AUDIT_TOKEN"
          },
          "body": {
            "session_id": "$CLAUDE_SESSION_ID",
            "duration": "$CLAUDE_SESSION_DURATION",
            "status": "completed"
          }
        }
      }
    ]
  }
}
```

**特徴**: 外部 API 連携、Webhook、async 実行可

### 3. Prompt Handler

```json
{
  "hooks": {
    "StopFailure": [
      {
        "handler": {
          "type": "prompt",
          "model": "claude-opus-4-7",
          "systemPrompt": "Analyze the error and suggest remediation",
          "userPrompt": "Error: $ERROR_MESSAGE\n\nContext: $CONTEXT_SNIPPET",
          "temperature": 0.3
        }
      }
    ]
  }
}
```

**特徴**: Claude API 呼び出し、Root cause analysis、推奨事項生成

### 4. Agent Handler

```json
{
  "hooks": {
    "ToolUseFailure": [
      {
        "handler": {
          "type": "agent",
          "prompt": "Recommend alternative approaches to handle this tool failure",
          "model": "claude-sonnet-4-6"
        }
      }
    ]
  }
}
```

**特徴**: Subagent 自動起動、独立 context、並行実行可

---

## IV. Hook 実装シナリオ（実装例）

### シナリオ 1: 自動フォーマット

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "handlers": [
          {
            "type": "command",
            "command": "if [[ \"$CLAUDE_TOOL_INPUT_FILE_PATH\" == *.ts ]]; then npx prettier --write \"$CLAUDE_TOOL_INPUT_FILE_PATH\"; fi"
          },
          {
            "type": "command",
            "command": "if [[ \"$CLAUDE_TOOL_INPUT_FILE_PATH\" == *.py ]]; then black \"$CLAUDE_TOOL_INPUT_FILE_PATH\"; fi"
          }
        ]
      }
    ]
  }
}
```

**効果**: Code write 毎に自動フォーマット（人的フォーマット不要）

### シナリオ 2: Git Pre-Flight Check

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "handlers": [
          {
            "type": "command",
            "command": "git status --porcelain | head -5 && echo '⚠️  Uncommitted changes detected'"
          }
        ]
      }
    ]
  }
}
```

**効果**: Git command 実行前に status 確認、accident 予防

### シナリオ 3: Secret Scanning

```json
{
  "hooks": {
    "Stop": [
      {
        "matcher": "UserPromptSubmit",
        "handlers": [
          {
            "type": "command",
            "command": "echo \"$CLAUDE_CONTEXT_SNIPPET\" | grep -E '(api_key|password|token|secret)' && echo '⚠️  Potential secret detected' || echo '✓ No secrets'"
          }
        ]
      }
    ]
  }
}
```

**効果**: セッション終了時に secret exposure チェック

### シナリオ 4: Context Warning

```json
{
  "hooks": {
    "ContextWarning": [
      {
        "matcher": "usage > 70%",
        "handlers": [
          {
            "type": "prompt",
            "systemPrompt": "Suggest context compression strategies",
            "userPrompt": "Context usage: 72%. Recommend compression methods to free up 20% space"
          }
        ]
      }
    ]
  }
}
```

**効果**: Context 危機時に自動提案

### シナリオ 5: Test Auto-Run

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit.*test",
        "handlers": [
          {
            "type": "command",
            "command": "npm test -- --testPathPattern=$(basename \"$CLAUDE_TOOL_INPUT_FILE_PATH\" .ts).test.ts"
          }
        ]
      }
    ]
  }
}
```

**効果**: テストファイル編集後に自動テスト実行

### シナリオ 6: Subagent Context Isolation Alert

```json
{
  "hooks": {
    "SubagentSpawn": [
      {
        "handler": {
          "type": "prompt",
          "systemPrompt": "Log subagent spawn event",
          "userPrompt": "Subagent spawned: $SUBAGENT_NAME. Context: $SUBAGENT_CONTEXT_SIZE tokens"
        }
      }
    ]
  }
}
```

**効果**: Subagent 起動の可視化・監視

### シナリオ 7: Session End Cleanup

```json
{
  "hooks": {
    "SessionEnd": [
      {
        "handler": {
          "type": "command",
          "command": "rm -f .claude/cache/*.tmp && find . -name '*.log' -mtime +7 -delete"
        }
      }
    ]
  }
}
```

**効果**: セッション終了時に一時ファイル・古ログ自動削除

---

## V. Hook Matcher パターン

### Matcher 構文

```json
"matcher": "Tool1|Tool2"              // Tool 名で OR 条件
"matcher": "*.ts"                     // File pattern（glob）
"matcher": "usage > 70%"              // Context usage 条件
"matcher": "error.*timeout"           // Regex pattern
```

### 複合 Matcher

```json
{
  "matcher": {
    "tools": ["Write", "Edit"],
    "filePattern": "*.{ts,js}",
    "context": "usage > 50%"
  }
}
```

---

## VI. Hook 設定ファイル構成

### .claude/settings.json（チーム共有）

```json
{
  "hooks": {
    "SessionStart": [ ... ],
    "PostToolUse": [ ... ],
    "SessionEnd": [ ... ]
  },
  "permissions": {
    "allowed_tools": ["Bash", "Edit", "Read"],
    "blocked_operations": ["git push --force", "rm -rf"]
  },
  "environment": {
    "AUDIT_TOKEN": "${AUDIT_TOKEN}",
    "DEBUG": "false"
  }
}
```

### .claude/settings.local.json（個人用、.gitignore）

```json
{
  "hooks": {
    "SessionStart": [
      {
        "handler": {
          "type": "command",
          "command": "echo 'Personal session init'"
        }
      }
    ]
  },
  "environment": {
    "DEBUG": "true"
  }
}
```

### 読み込み順序

```
1. ~/.claude/settings.json       (Global user)
2. .claude/settings.json         (Project shared)
3. .claude/settings.local.json   (Project personal)
→ マージ（ローカルが上書き）
```

---

## VII. Hook 開発チェックリスト

### 設計段階
- [ ] イベント選択（正しい timing か）
- [ ] Matcher 定義（scope 過不足ないか）
- [ ] ハンドラ型選択（Command/HTTP/Prompt/Agent）
- [ ] Error handling（失敗時の動作）
- [ ] Async vs Sync（ブロッキング判断）

### 実装段階
- [ ] 環境変数確認（`$CLAUDE_*` アクセス可能か）
- [ ] Command escape（Shell injection 対策）
- [ ] ローカルテスト（settings.local.json で試験）
- [ ] Performance（実行時間測定）

### チームレビュー
- [ ] ドキュメント完備
- [ ] Error message 明確
- [ ] .gitignore 設定（local 設定）

---

## VIII. Hook パフォーマンス・Context コスト

### Hook 実行のコスト（典型値）

| Hook 型 | 実行時間 | Context コスト |
|---|---|---|
| Command (simple) | 100-500ms | negligible |
| Command (heavy) | 1-5s | 1-3% |
| HTTP | 200-2000ms | 0.5-2% |
| Prompt | 2-10s | 3-8% |
| Agent | 5-30s | 10-20% |

### 最適化テクニック

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "handlers": [
          {
            "type": "command",
            "command": "npx prettier --write \"$FILE\" --no-color",
            "timeout": 5000,
            "background": true
          }
        ]
      }
    ]
  }
}
```

**オプション**:
- `timeout`: 実行タイムアウト（ms）
- `background`: 非同期実行（bool）
- `retry`: リトライ回数（int）

---

## IX. Hook トラブルシューティング

### Hook が実行されない

```bash
# 1. Hook definition 確認
cat .claude/settings.json | grep -A 20 "hooks"

# 2. Event matching 確認
# 実際の trigger が matcher と合致しているか

# 3. Debug mode
DEBUG=true claude-code  # Hook 実行ログ出力
```

### Hook 実行が遅い

```bash
# Async 実行に変更
"background": true

# Command を最適化
# 例: 不要な grep を削除、pipe 最小化

# Context cost 確認
# Prompt handler なら model downgrade（Haiku to Sonnet）
```

---

最終更新: 2026-05-06
参照: CLAUDE.md, AGENT.md, DESIGN.md
