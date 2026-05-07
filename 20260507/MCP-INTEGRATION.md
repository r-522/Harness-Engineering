# MCP-INTEGRATION.md - MCP Server統合ガイド

**対象**: Harness Engineering のMCP連携  
**更新日**: 2026-05-07  
**参考**: https://github.com/barkain/claude-code-workflow-orchestration

---

## 1. MCPの基本概念

### 定義

**MCP (Model Context Protocol)** = Claude と外部システムをセキュアに接続するプロトコル

```
Claude Code
    │
    ├─→ [ MCP Client ]
    │       │
    │       ├─→ [TCP/HTTP]
    │       │
    │       └─→ MCP Server
    │           (AWS SDK, GCP Tools, Custom APIs)
    │
    └─→ Tool Chain (Read/Edit/Bash)
```

### MCP vs Tool Chain（何を使い分けるか）

| 操作 | Tool Chain | MCP |
|---|---|---|
| ローカルファイル読み込み | ✅ Read | ❌ |
| AWS S3へのアップロード | ❌ | ✅ AWS MCP Server |
| GitHubリポジトリ操作 | ✅ Bash (gh CLI) | ✅ GitHub MCP Server |
| コード編集 | ✅ Edit | ❌ |
| 外部API呼び出し | ⚠️ Bash (curl) | ✅ Custom MCP |
| データベースクエリ | ❌ | ✅ Database MCP |

---

## 2. MCP Server接続フロー

### セットアップ手順

```mermaid
step 1
Step 1: MCP Server の選定
    ↓ (CLAUDE.md の セキュリティレビュー項目確認)
Step 2: 認証情報の準備
    ↓ (.env.local に API_KEY 等を記入)
Step 3: config/mcp-servers.yaml に接続設定を記述
    ↓
Step 4: mcp-healthcheck スキル実行
    ↓ (接続確認)
Step 5: テストコード実装
    ↓
Step 6: 本運用開始
```

### config/mcp-servers.yaml （テンプレート）

```yaml
# MCP Server設定ファイル
# 認証情報は .env.local から読み込む

mcp_servers:
  # =========================
  # AWS Tools
  # =========================
  aws:
    enabled: true
    name: AWS CLI Tools
    type: executable
    command: aws
    args:
      - --version
    environment:
      - AWS_ACCESS_KEY_ID: "${AWS_ACCESS_KEY_ID}"
      - AWS_SECRET_ACCESS_KEY: "${AWS_SECRET_ACCESS_KEY}"
      - AWS_REGION: us-east-1
    timeout: 30
    allowed_operations:
      - s3:GetObject
      - s3:PutObject
      - ec2:DescribeInstances
      - lambda:ListFunctions
    blocked_operations:
      - iam:DeleteUser
      - iam:AttachUserPolicy

  # =========================
  # GCP Tools
  # =========================
  gcp:
    enabled: false  # 未使用の場合は false
    name: Google Cloud Tools
    type: rest_api
    endpoint: https://cloudresourcemanager.googleapis.com/v1
    headers:
      Authorization: "Bearer ${GCP_ACCESS_TOKEN}"
    timeout: 30
    allowed_operations:
      - projects:list
      - compute:instances:list

  # =========================
  # GitHub
  # =========================
  github:
    enabled: true
    name: GitHub MCP Server
    type: executable
    command: gh
    environment:
      - GITHUB_TOKEN: "${GITHUB_TOKEN}"
    allowed_operations:
      - api
      - pr:view
      - issue:list
    blocked_operations:
      - repo:delete

  # =========================
  # Custom Database MCP
  # =========================
  database:
    enabled: true
    name: PostgreSQL MCP Server
    type: tcp
    host: db.example.com
    port: 5432
    authentication:
      method: password
      username: claude_user
      password: "${DB_PASSWORD}"
    timeout: 10
    allowed_schemas:
      - public
      - analytics
    blocked_schemas:
      - internal
      - secrets

# グローバル設定
global:
  max_concurrent_connections: 3
  connection_timeout: 15
  retry_strategy:
    max_attempts: 3
    backoff_ms: [100, 500, 2000]  # 指数バックオフ
  health_check_interval: 300  # 5分ごと
```

---

## 3. セキュリティ・認証管理

### 認証情報の安全な管理

```bash
# ✅ 推奨: .env.local （.gitignore に登録）
.env.local:
  export AWS_ACCESS_KEY_ID="AKIA..."
  export AWS_SECRET_ACCESS_KEY="***"
  export GITHUB_TOKEN="ghp_***"
  export DB_PASSWORD="***"

# ❌ 禁止: コード内に埋め込み
# ❌ 禁止: CLAUDE.md に記載

# 読み込み方法
source .env.local
export $(cat .env.local | xargs)
```

### アクセス制御

```yaml
# CLAUDE.md の禁止事項として記載
Security Rules:
  - [ ] MCP Server への接続は必ずセキュリティレビュー実施
  - [ ] API_KEY は絶対にコード・ドキュメントに埋め込まない
  - [ ] allowed_operations は最小権限で設定
  - [ ] 月1回の定期監査を実施

# 監査ログ出力
audit_log:
  location: logs/mcp_access.log
  format: |
    [timestamp] operation user_id resource status result
    2026-05-07 14:32:15 s3:GetObject bot@harness OK 200ms
```

---

## 4. 統合テスト（MCP接続確認）

### テストコード例

```python
# tests/integration/test_mcp_integration.py
import pytest
from src.mcp.aws_tools import S3Tool
from src.mcp.github_tools import GitHubTool
from src.mcp.database_tools import DatabaseTool

class TestMCPIntegration:
    """MCP Server接続テスト"""

    @pytest.fixture
    def aws_tool(self):
        return S3Tool()

    @pytest.fixture
    def github_tool(self):
        return GitHubTool()

    def test_aws_s3_connection(self, aws_tool):
        """AWS S3接続確認"""
        result = aws_tool.list_buckets()
        assert result.success == True
        assert isinstance(result.buckets, list)

    def test_github_api_access(self, github_tool):
        """GitHub API アクセス確認"""
        result = github_tool.list_repos(org="harness-engineering")
        assert result.success == True
        assert len(result.repos) > 0

    def test_database_query(self):
        """データベースクエリ実行"""
        db = DatabaseTool()
        result = db.query("SELECT COUNT(*) FROM tasks")
        assert result.success == True
        assert result.count > 0

    @pytest.mark.timeout(5)
    def test_mcp_latency(self, aws_tool):
        """MCP レスポンス時間確認（<5秒）"""
        start = time.time()
        aws_tool.list_buckets()
        elapsed = time.time() - start
        assert elapsed < 5, f"Latency {elapsed}s exceeds 5s threshold"

    def test_connection_failure_handling(self, aws_tool):
        """接続失敗時のハンドリング"""
        # 接続を意図的に失敗させる
        aws_tool.endpoint = "invalid-endpoint"
        
        with pytest.raises(ConnectionError):
            aws_tool.list_buckets()
```

### テスト実行コマンド

```bash
# MCP接続テストのみ実行
pytest tests/integration/ -k "mcp" -v

# 本番環境でのE2Eテスト（MCP Live）
pytest tests/e2e/ --mcp-live --timeout=30

# 全統合テスト実行
pytest tests/integration/ -v --cov=src/mcp
```

---

## 5. トラブルシューティング

### Issue 1: MCP Server接続失敗

**症状**: `ConnectionRefusedError` または `TimeoutError`

**原因と対応**:

```bash
# 1. 接続確認
telnet mcp-server.example.com 5432
→ Connection refused: ネットワーク疎通がない
  → VPN/ファイアウォール設定確認

# 2. 認証確認
echo $AWS_ACCESS_KEY_ID
→ 空文字列: 環境変数が設定されていない
  → source .env.local 実行

# 3. MCP Server のステータス確認
/mcp-healthcheck スキル実行

# 4. ログ確認
tail -f logs/mcp_connection.log
```

### Issue 2: API レスポンス遅延

**症状**: `SlowResponseWarning` または タイムアウト

**対応**:

```yaml
# config/mcp-servers.yaml で timeout を増加
aws:
  timeout: 60  # 30秒 → 60秒に変更
```

**パフォーマンス最適化**:

```python
# キャッシングを導入
from functools import lru_cache

@lru_cache(maxsize=100)
def get_aws_resource(resource_id):
    """AWS リソース情報キャッシュ"""
    return aws_tool.get_resource(resource_id)

# キャッシュTTL: 5分
cache_ttl = 300
```

### Issue 3: 権限エラー（AccessDenied）

**症状**: `AccessDeniedException` または `PermissionDenied`

**原因確認**:

```bash
# 1. IAM ポリシー確認
aws iam get-user
→ ユーザーに必要な権限がない

# 2. allowed_operations 確認
config/mcp-servers.yaml で操作が許可されているか確認

# 3. トークン有効性確認
gh auth status
→ Token expired: 新しいトークンを生成
```

---

## 6. MCP Server の種類別ガイド

### A. AWS Tools

**設定例**:
```yaml
aws:
  enabled: true
  type: executable
  command: aws
  allowed_operations:
    - s3:GetObject
    - s3:PutObject
    - ec2:DescribeInstances
    - lambda:Invoke
```

**使用シーン**:
- S3バケットへのデータアップロード
- EC2インスタンス管理
- Lambda関数実行
- CloudWatch ログ閲覧

**テスト例**:
```python
def test_aws_lambda_invoke():
    aws = AWSTool()
    result = aws.invoke_lambda(
        function_name="harness-orchestrator",
        payload={"task_id": "123"}
    )
    assert result.success
```

---

### B. GitHub Tools

**設定例**:
```yaml
github:
  enabled: true
  type: executable
  command: gh
  allowed_operations:
    - api
    - pr:view
    - pr:create
    - issue:list
```

**使用シーン**:
- PR の作成・レビュー
- Issue の管理
- コードベース検索

**テスト例**:
```python
def test_github_create_pr():
    gh = GitHubTool()
    result = gh.create_pr(
        head="feature/new-skill",
        base="develop",
        title="Add new orchestration skill"
    )
    assert result.success
    assert result.pr_number > 0
```

---

### C. カスタムMCP Server（例：Database）

**実装例**:
```python
# src/mcp/database_tools.py
from src.core.mcp_base import BaseMCPServer
import psycopg2

class DatabaseMCPServer(BaseMCPServer):
    def __init__(self, host: str, port: int, database: str):
        self.conn = psycopg2.connect(
            host=host,
            port=port,
            database=database,
            user=os.getenv("DB_USER"),
            password=os.getenv("DB_PASSWORD")
        )
    
    def execute_query(self, sql: str, params=None):
        """SQL クエリ実行"""
        cursor = self.conn.cursor()
        try:
            cursor.execute(sql, params or ())
            return {
                "success": True,
                "rows": cursor.fetchall(),
                "columns": [desc[0] for desc in cursor.description]
            }
        except Exception as e:
            return {
                "success": False,
                "error": str(e)
            }
```

**使用例**:
```python
db = DatabaseMCPServer("db.example.com", 5432, "harness_db")
result = db.execute_query(
    "SELECT * FROM tasks WHERE status = %s",
    ["pending"]
)
```

---

## 7. Subagent × MCP 統合パターン

### パターン: Researcher Subagent が AWS データ収集

```markdown
## フロー図

User: "S3のログファイルを分析して"
    │
    ↓
Main Agent: orchestrate-parallel スキル起動
    │
    ├─→ Researcher_aws_logs_v1
    │   │
    │   ├─→ [MCP] AWS S3 接続
    │   │   └─→ logs/*.json リスト取得
    │   │
    │   └─→ [Tool Chain] Read でローカルに保存
    │       └─→ logs/ ディレクトリに展開
    │
    └─→ Researcher_analysis_v1
        │
        ├─→ [Tool Chain] Read でログ読み込み
        │
        └─→ 分析実施
            └─→ 結果をユーザーに報告
```

**実装スクリプト**:
```python
# src/agents/researcher.py
class ResearcherAgent:
    async def collect_aws_logs(self, bucket: str, prefix: str):
        # MCP Server経由でS3アクセス
        mcp = MCPClient(config="aws")
        objects = await mcp.list_objects(bucket, prefix)
        
        # 各オブジェクトをダウンロード
        for obj in objects:
            content = await mcp.get_object(bucket, obj)
            # ローカルに保存
            with open(f"logs/{obj}", "w") as f:
                f.write(content)
        
        return {"status": "collected", "count": len(objects)}
```

---

## 8. セキュリティベストプラクティス

### チェックリスト

- [ ] **認証**: すべてのMCP接続に認証が必須か確認
- [ ] **暗号化**: 通信がTLS/SSLで暗号化されているか確認
- [ ] **最小権限**: allowed_operations は本当に必要な操作のみか
- [ ] **監査ログ**: すべてのMCP操作が logs/mcp_access.log に記録されているか
- [ ] **秘密管理**: API KEYが .gitignore対象のファイルに入っているか
- [ ] **定期レビュー**: 月1回、接続設定と権限をレビュー
- [ ] **失敗時処理**: MCP接続エラー時のフォールバック機能があるか

### 監査スクリプト

```bash
#!/bin/bash
# scripts/audit_mcp_security.sh

echo "🔍 MCP Security Audit"
echo "====================="

# 1. コード内にAPIキーが埋め込まれていないか確認
echo "Checking for hardcoded API keys..."
grep -r "AKIA\|sk_live\|ghp_" src/ config/ && echo "❌ FOUND" || echo "✅ OK"

# 2. 認証情報が正しく管理されているか
echo "Checking .env.local..."
test -f .env.local && echo "✅ Found" || echo "❌ Missing"
test -f .gitignore && grep -q ".env.local" .gitignore && echo "✅ In .gitignore" || echo "❌ Not in .gitignore"

# 3. MCP接続設定の安全性確認
echo "Reviewing MCP server configuration..."
python scripts/validate_mcp_config.py

echo ""
echo "✅ Security Audit Complete"
```

---

**最終チェック**:
- [ ] すべてのMCP設定例は実行可能か？
- [ ] セキュリティガイドラインは明確か？
- [ ] テスト例は現実的か？

✅ **本MCP-INTEGRATION.md はMCP統合の完全ガイド**
