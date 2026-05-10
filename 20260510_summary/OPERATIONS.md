# OPERATIONS.md - Hooks / MCP / Security / Caching 運用（総括版）

**有効期間**: 2026-05-10 〜  **行数**: 118行 / 上限 120行

---

## 1. Hooks（settings.json）

Claude Code の Hooks は、ハーネスが Agent ではなく **harness 自身** にやらせるべき
自動化の入り口。Agent の memory に「毎回 X しろ」と書いても守られない。

### 推奨フック

```jsonc
{
  "hooks": {
    "PreToolUse": [
      // 危険コマンドのブロック
      { "matcher": "Bash",
        "hooks": [{ "type": "command",
          "command": ".claude/hooks/block-dangerous.sh" }] }
    ],
    "PostToolUse": [
      // 編集後 lint 自動
      { "matcher": "Edit|Write",
        "hooks": [{ "type": "command",
          "command": "ruff check ${CLAUDE_FILE_PATHS:-.} || true" }] }
    ],
    "SessionStart": [
      { "hooks": [{ "type": "command",
          "command": ".claude/hooks/load-context.sh" }] }
    ],
    "Stop": [
      { "hooks": [{ "type": "command",
          "command": ".claude/hooks/self-check-reminder.sh" }] }
    ]
  }
}
```

### `block-dangerous.sh` 例

```bash
#!/usr/bin/env bash
input=$(cat)
cmd=$(jq -r '.tool_input.command' <<<"$input")
if grep -qE 'git push.* (--force|-f)\b|rm -rf /|reset --hard' <<<"$cmd"; then
  echo '{"decision":"block","reason":"Dangerous command blocked"}'
  exit 0
fi
```

---

## 2. MCP 統合

### 接続ポリシー

- Allowlist 方式。新規接続は **Security Review 必須**
- 認証情報は `.env` のみ。リポジトリにコミット禁止
- 各 MCP サーバーに category（safe/cautious/restricted）を割当

### `.mcp.json` テンプレ

```jsonc
{
  "mcpServers": {
    "github":   { "command": "mcp-github",   "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" } },
    "internal": { "command": "mcp-internal", "scope": "project" }
  }
}
```

### MCP 利用時の Self-Check 追記

- [ ] 呼び出した MCP は allowlist 内か
- [ ] 機密情報を MCP 経由で外部公開していないか
- [ ] レート制限・タイムアウト設定済みか

---

## 3. Security（OWASP top10 を Harness で防ぐ）

| 項目 | Harness 側の備え |
|---|---|
| Injection | Tool 入力 schema 検証 / shell quoting |
| Broken Auth | API キーは `.env` / hooks で push 時にスキャン |
| Sensitive Data | `git secret-scan` を pre-commit hook で強制 |
| XSS | TS strict + sanitization util 強制 |
| Misconfig | `settings.json` を CI で lint |
| Vulnerable Deps | `pip-audit` / `npm audit` を CI 必須 |
| Logging | 機密情報マスキング util を共通化 |
| SSRF | MCP allowlist + outbound proxy |
| Deserialization | `yaml.safe_load` 強制 |
| Components | 依存追加は ⚠️ 要レビュー |

### Pre-commit Secret Scan

```bash
# .claude/hooks/scan-secrets.sh
git diff --cached | grep -E 'sk-[a-zA-Z0-9]{20,}|AKIA[0-9A-Z]{16}' && {
  echo "secret detected"; exit 2; }
```

---

## 4. Prompt Caching

- 4ブロック構成で 90%+ ヒット率を狙う
- 静的ブロック（System / CLAUDE / AGENT）に `cache_control: ephemeral`
- 動的ブロック（会話履歴）はキャッシュしない
- TTL: 5分（標準）/ 1時間（heavy session）

```python
messages = [
  {"role": "system",
   "content": [{"type": "text", "text": SYSTEM,
                "cache_control": {"type": "ephemeral"}}]},
  {"role": "user",
   "content": [{"type": "text", "text": CLAUDE_MD,
                "cache_control": {"type": "ephemeral"}},
               {"type": "text", "text": AGENT_MD,
                "cache_control": {"type": "ephemeral"}},
               {"type": "text", "text": user_turn}]},
]
```

実コスト目安: cache hit で input 1/10。長セッションほど効く。

---

## 5. Observability

最低限取るメトリクス:
- Token 使用率（per turn / cumulative）
- Tool 失敗率 / retry 回数
- テスト PASS 率 / カバレッジ推移
- Subagent ごとの所要時間・予算消化率
- Cache hit rate

ダッシュボード or `.claude/metrics.jsonl` への追記で十分。
**測れないものは改善できない。**

---

## 6. Runbook（障害時）

| 症状 | 一次対応 |
|---|---|
| Context Rot | 新 Session / 要約引継ぎ |
| Tool 連続失敗 | retry 停止 → snapshot 添付で escalate |
| MCP タイムアウト | 該当 MCP を一時 disable / fallback |
| カバレッジ低下 | PR ブロック / Reviewer 召喚 |
| Subagent デッドロック | 直列再実行 / DAG 再生成 |

---

**最後に**: 5本のドキュメントを `.claude/` に配置し、`MASTER_PROMPT.md` を System Prompt として起動すれば、最強のハーネスが立ち上がる。
