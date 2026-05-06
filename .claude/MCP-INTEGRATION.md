# MCP-INTEGRATION.md - MCP & スキル統合戦略

**対象**: Model Context Protocol (MCP) サーバ統合  
**基準**: Claude Skills は ワークフロー、MCP は 接続を責務分離  
**最終更新**: 2026-05-05

---

## 1. MCP vs Skills 機能分離の原則

### MCP Server の役割

**定義**: 外部システム・サービスへの **セキュアなアクセスポイント**

```
MCP Server
  ├─ ツール提供（Tool definitions）
  │  └─ WebSearch, Read, Bash, Custom API接続
  ├─ 認証管理（API キー、OAuth)
  ├─ レート制限・エラーハンドリング
  └─ 利用ログ・監査
```

### Claude Skill の役割

**定義**: MCP上のツールを組み合わせた **ワークフローロジック**

```
Skill
  ├─ Tool の組み合わせ（Step定義）
  ├─ パラメータの加工
  ├─ 結果の統合・フォーマット
  └─ ドメインロジック（業務知識）
```

### 関係図

```
┌──────────────────────────────────┐
│       Skill Layer (Workflow)      │
│  - Step 1: Search                 │
│  - Step 2: Parse Results          │
│  - Step 3: Generate Report        │
└──────────────┬───────────────────┘
               │
┌──────────────▼───────────────────┐
│     MCP Server (Connectivity)     │
│  - WebSearch Tool                 │
│  - Read File Tool                 │
│  - Authentication                 │
└──────────────┬───────────────────┘
               │
┌──────────────▼───────────────────┐
│   External Systems (Runtime)      │
│  - Google Search API              │
│  - File System                    │
│  - Database                       │
└───────────────────────────────────┘
```

---

## 2. 推奨 MCP サーバ カタログ

### Group A: 標準組み込み MCP (Anthropic 公式)

| MCP Server | Tools Provided | 推奨用途 | 認証 |
|---|---|---|---|
| **web-search** | WebSearch, WebFetch | Web検索、記事閲覧 | API key (Google) |
| **filesystem** | Read, Write, Edit | ファイル操作 | OS権限 |
| **bash** | Bash command execution | シェルコマンド | OS権限 |
| **brave-search** | BraveSearch | プライベート検索代替 | API key |

**デフォルト設定** (`.claude.local.md`):
```yaml
[[mcp.servers.web-search]]
command = "mcp-server-web-search"
env = { GOOGLE_API_KEY = "$GOOGLE_API_KEY" }

[[mcp.servers.filesystem]]
command = "mcp-server-filesystem"

[[mcp.servers.bash]]
command = "mcp-server-bash"
```

---

### Group B: 開発環境連携 MCP (推奨追加)

| MCP Server | Tools Provided | 推奨用途 | Source |
|---|---|---|---|
| **github** | PR/Issue read, Create PR | GitHub統合 | Anthropic |
| **postgres** | SQL query execute | DB操作 | Community |
| **docker** | Container control | Docker操作 | Community |
| **git** | Commit, Branch, Log | Git操作 | Community |

**推奨構成** (小規模チーム):
```yaml
[[mcp.servers.github]]
command = "mcp-server-github"
env = { GITHUB_TOKEN = "$GITHUB_TOKEN" }

[[mcp.servers.git]]
command = "mcp-server-git"
args = ["--repo=/home/user/Harness-Engineering"]
```

---

### Group C: カスタム MCP (プロジェクト固有)

#### Example 1: 社内API連携

```yaml
[[mcp.servers.internal-api]]
command = "python -m mcp_internal_api"
env = {
  API_ENDPOINT = "https://api.internal.company.com",
  API_KEY = "$INTERNAL_API_KEY"
}
```

**提供Tool**:
```
- get_feature_flags(feature_name: str) → Dict
- log_event(event_name: str, metadata: Dict) → bool
- query_metrics(metric_key: str, time_range: str) → TimeSeries
```

#### Example 2: カスタム分析エンジン

```yaml
[[mcp.servers.analysis-engine]]
command = "python -m mcp_analysis"
args = ["--config=/path/to/config.yaml"]
```

**提供Tool**:
```
- analyze_code_complexity(file_path: str) → CodeMetrics
- suggest_refactoring(file_path: str) → List[Suggestion]
- predict_performance(code_snippet: str) → PerformanceEstimate
```

---

## 3. Skill ↔ MCP マッピングマトリクス

### 対応表

| Skill | Primary MCP | Secondary MCP | Tool Chain |
|---|---|---|---|
| **web_research_parallel** | web-search | filesystem | WebSearch → Save → Read |
| **code_pattern_discovery** | bash, filesystem | (none) | Bash grep → Read → Report |
| **parallel_testing_suite** | bash | github | Bash pytest → GitHub check status |
| **code_generation_templates** | filesystem | (none) | Read template → Write file → Bash lint |
| **code_quality_audit** | bash | internal-api | Bash linting → Log metrics API |
| **markdown_report_generation** | filesystem | (none) | Read sources → Write report |

---

## 4. セキュリティ & ガバナンス

### 認証情報の管理

**❌ 禁止**:
```python
# Subagentに認証情報を渡さない
subagent_task = {
    "api_key": "sk-xxx",  # ← 危険
    "endpoint": "https://api..."
}
```

**✅ 推奨**:
```
Subagent
  ↓ (タスク指示のみ)
Orchestrator (認証情報保有)
  ↓ (Tool呼び出し、認証挿入)
MCP Server
  ↓ (セキュアアクセス)
外部API
```

### 権限スコープ

```yaml
# ~/.claude/settings.json (グローバル)
mcp_permissions:
  web-search: allow  # 外部アクセス許可
  bash: require      # 実行時に毎回確認
  github: require    # PR作成は確認
  internal-api: allow # 社内API信頼

# .claude/CLAUDE.md (プロジェクト層)
# bash実行 = チームレビュー前提
# github/PR = PR テンプレート確認
```

### 監査ログ

```yaml
# 各MCP実行後、自動記録されるべき情報
mcp_audit_log:
  - timestamp: 2026-05-05T14:23:45Z
    server: web-search
    tool: WebSearch
    query: "Claude Code best practices"
    result_count: 10
    cost: $0.001
    user: "@agent-name"
```

---

## 5. パフォーマンス & コスト最適化

### Token コスト見積もり

| MCP Tool | Avg Input | Avg Output | Cost/Call |
|---|---|---|---|
| **WebSearch** | 200 | 800 | $0.005 |
| **WebFetch** | 500 | 3000 | $0.015 |
| **Read file** | 100 | 2000 | $0.010 |
| **Bash cmd** | 200 | 500 | $0.003 |
| **GitHub API** | 150 | 1500 | $0.008 |

### 最適化戦略

```markdown
## Strategy 1: Caching
- 同一検索クエリ ≤ 24時間 → キャッシュ使用
- 大規模ファイル Read → JSON/CSV 抽出版を再利用

## Strategy 2: Batching
- 複数の小さなBashコマンド → 1つのスクリプトにまとめる
- 複数WebSearch → 1つの Explore subagent へ集約

## Strategy 3: Model Selection
- 単純Tool呼び出し → Sonnet (安価)
- 複雑フロー調整 → Opus (推論)

## Estimated Cost (3並列 × 3 workers × 1 research task)
- WebSearch x 3: $0.015
- Read x 9: $0.090
- Bash x 6: $0.018
- Total: ~$0.12 per research execution
```

---

## 6. 新しい MCP サーバ追加手順

### チェックリスト

- [ ] **目的を明確化**: どのような機能が必要か
- [ ] **既存MCP確認**: 類似Tool がないか検索
- [ ] **実装方針決定**: 公式 SDK か自作か
- [ ] **セキュリティレビュー**: API キー管理、権限スコープ
- [ ] **テストシナリオ作成**: ローカルで動作確認
- [ ] **チーム内レビュー**: MCP-INTEGRATION.md に記入、同意を得る
- [ ] **本番昇格**: `.claude.local.md` → `.claude/settings.json`

### 実装テンプレート（Python MCP Server）

```python
# mcp_custom_server.py
import mcp

server = mcp.Server("custom-server")

@server.tool()
def query_custom_api(endpoint: str, params: dict) -> dict:
    """Custom APIへの安全なアクセスをラップ"""
    # 認証情報は MCP Server内部で管理
    auth_header = os.environ.get("CUSTOM_API_KEY")
    return requests.post(
        endpoint,
        headers={"Authorization": f"Bearer {auth_header}"},
        json=params
    ).json()

if __name__ == "__main__":
    server.run()
```

**設定** (`.claude.local.md`):
```yaml
[[mcp.servers.custom]]
command = "python -m mcp_custom_server"
env = { CUSTOM_API_KEY = "$CUSTOM_API_KEY" }
```

---

## 7. トラブルシューティング

### Issue 1: MCP サーバが起動しない

```bash
# 確認手順
1. MCP バイナリが存在するか
   which mcp-server-web-search

2. 環境変数が設定されているか
   echo $GOOGLE_API_KEY

3. ログを確認
   tail -f ~/.claude/mcp.log

4. 再起動
   pkill mcp-server-* && sleep 2 && claude restart
```

### Issue 2: Tool呼び出しが遅い

```markdown
**原因**: API レート制限 または ネットワーク遅延

**対策**:
1. Caching有効化 (同一クエリ24h以内再利用)
2. Batching: 複数Tool呼び出しを1つに集約
3. タイムアウト増加: request_timeout_ms を調整
4. フォールバック: 代替API登録
```

### Issue 3: 認証エラー

```markdown
**確認事項**:
1. API キーが正しいか (`echo $API_KEY`)
2. キーに必要権限があるか
3. キーが有効期限切れでないか
4. IP ホワイトリスト確認（社内API）
```

---

## 8. MCP ガバナンスポリシー

### 新規 MCP 導入の審査基準

| 基準 | チェック |
|---|---|
| **ビジネス価値** | プロジェクト目標にどう貢献するか |
| **セキュリティ** | 認証、権限スコープが適切か |
| **信頼性** | SLA、エラーハンドリングが明確か |
| **コスト** | 利用料、Token消費が見積もられているか |
| **運用性** | ドキュメント、サポート体制が整備されているか |

### 承認フロー

```
提案 (MCP-INTEGRATION.md にドラフト)
  ↓
テスト (ローカル環境で動作確認)
  ↓
レビュー (チーム + セキュリティ)
  ↓
本番昇格 (settings.json に登録)
  ↓
監視 (使用頻度、エラー率、コスト)
```

---

## 9. 参考リソース

- **MCP Specification**: https://modelcontextprotocol.io/
- **Claude Skills vs MCP**: https://intuitionlabs.ai/articles/claude-skills-vs-mcp
- **Extending Claude**: https://claude.com/blog/extending-claude-capabilities-with-skills-mcp-servers
- **MCP Servers Directory**: https://mcpmarket.com/

---

## 10. チェックリスト: 本構成への適用

本プロジェクトで導入すべき MCP 構成（推奨）:

- [ ] **web-search** (標準): Web検索研究
- [ ] **filesystem** (標準): ファイル操作
- [ ] **bash** (標準): シェルコマンド
- [ ] **github** (推奨): PR/Issue管理
- [ ] **git** (推奨): バージョン管理
- [ ] **internal-api** (カスタム): 社内API連携
- [ ] **analysis-engine** (カスタム): コード分析

**デプロイ完了目安**: 上記7サーバが `.claude.local.md` に登録され、各3回以上テスト実行されていること。
