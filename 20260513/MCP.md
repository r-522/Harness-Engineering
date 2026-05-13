# MCP.md - Model Context Protocol 統合ガイド

**対象**: MCP Server 接続・認証・トラブルシューティング  
**最終更新**: 2026-05-13

---

## 1. MCP とは

**Model Context Protocol (MCP)** は、Claude がツール・リソースへアクセスするための標準プロトコル。

### Skills との違い

| 特徴 | **Skills** | **MCP** |
|------|----------|--------|
| **役割** | AI の内部知識（HOW）| 外部リソースへのアクセス（WHAT） |
| **用途例** | コード品質チェック、計画立案 | GitHub API、AWS API、データベース |
| **トークン消費** | 30〜50 tokens（起動時のみ） | 動的（API 呼び出しごと） |
| **認証** | 不要（内部） | 必須（API キー等） |

**原則**: 「説明できる = Skill」「アクセス必要 = MCP」

---

## 2. 推奨 MCP Servers（2026年5月時点）

### 標準ドメイン別推奨

| ドメイン | MCP Server | 認証 | 優先度 |
|---------|-----------|------|------|
| **コード管理** | `github` | OAuth / Token | 🟠 High |
| **クラウド** | `aws-tools` | IAM Role / AccessKey | 🟠 High |
| **Web検索** | `search-web` | API Key | 🟢 Medium |
| **データベース** | `postgres`, `mongodb` | 接続文字列 | 🟢 Medium |
| **メッセージング** | `slack`, `discord` | Bot Token | 🔴 Low |
| **API Gateway** | `graphql-introspection` | 不要（スキーマ検証） | 🟢 Medium |

### 推奨インストール手順

```bash
# 1. MCP Servers をローカルインストール
pip install mcp-server-github
pip install mcp-server-aws

# 2. config/servers.yaml に定義
cat >> config/servers.yaml << 'EOF'
servers:
  github:
    type: github
    auth_method: token
    env_var: GITHUB_API_TOKEN
  aws:
    type: aws
    auth_method: iam
    env_var: AWS_PROFILE
EOF

# 3. .env.local に認証情報（.gitignore対象）
echo "GITHUB_API_TOKEN=ghp_xxxxxxxxxxxx" >> .env.local
echo "AWS_PROFILE=dev-profile" >> .env.local
```

---

## 3. 接続フロー（セットアップ）

### Step 1: 認証情報を取得

**GitHub**:
```bash
# https://github.com/settings/tokens で Personal Access Token 生成
# Scopes: repo, workflow, read:org 最小限

export GITHUB_API_TOKEN="ghp_xxxxxxxxxxxxxxx"
```

**AWS**:
```bash
# AWS CLI で認証済み設定を確認
aws configure list

# または IAM User の AccessKey を取得
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
```

### Step 2: MCP Server をローカルで起動（テスト用）

```bash
# 1. Docker で起動（オプション）
docker run -e GITHUB_TOKEN=$GITHUB_API_TOKEN \
  -p 3000:3000 \
  anthropic/mcp-server-github:latest

# 2. または Python で起動
python -m src.mcp.github_connector start

# 確認
curl http://localhost:3000/health  # 200 OK なら接続可
```

### Step 3: Claude Code 設定に MCP を登録

`.claude/settings.json`:
```json
{
  "mcp_servers": {
    "github": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "src.mcp.github_connector"],
      "env": {
        "GITHUB_API_TOKEN": "${GITHUB_API_TOKEN}"
      }
    },
    "aws": {
      "type": "stdio",
      "command": "mcp-server-aws",
      "env": {
        "AWS_PROFILE": "${AWS_PROFILE}"
      }
    }
  }
}
```

### Step 4: 接続テスト

```bash
# MCP 経由で GitHub API を試行
python -c "
import src.mcp.github_connector as gh
user = gh.get_user()
print(f'Authenticated as: {user.login}')
"

# 期待出力: Authenticated as: <username>
```

---

## 4. 各 MCP Server の詳細設定

### GitHub MCP Server

**使用例**: PR作成・コメント・リリース管理

```python
# src/mcp/github_connector.py
from mcp_server_github import GitHubMCP

connector = GitHubMCP(token=os.getenv("GITHUB_API_TOKEN"))

# 典型的な操作
pr = connector.create_pull_request(
    owner="r-522",
    repo="harness-engineering",
    title="feat: Add researcher agent",
    head="feature/researcher",
    base="develop"
)
```

**セキュリティ注意**:
- API Token は `.env.local` に（`.gitignore` で除外）
- Token スコープを最小限（repo, workflow のみ）
- 定期的に Token をローテーション（90日ごと）

### AWS Tools MCP Server

**使用例**: Lambda デプロイ・EC2 管理

```python
# src/mcp/aws_connector.py
from mcp_server_aws import AWSMCP

connector = AWSMCP(profile=os.getenv("AWS_PROFILE"))

# Lambda 関数をデプロイ
response = connector.deploy_lambda(
    function_name="data-processor",
    zip_path="build/lambda.zip",
    runtime="python3.10"
)
```

**推奨設定**:
- IAM Role に最小権限ポリシー（Lambda, EC2 のみ）
- Region を明示（デフォルト: us-east-1）
- Timeout: 300秒（API 呼び出しが長い場合）

### Web Search MCP Server

**使用例**: 技術トレンド調査、ドキュメント検索

```python
# src/skills/researcher_context.py
from mcp_server_search import SearchMCP

search = SearchMCP(api_key=os.getenv("SEARCH_API_KEY"))

results = search.query(
    q="Claude Code best practices 2026",
    num_results=10,
    recency_days=7  # 過去7日間
)
```

---

## 5. トラブルシューティング

### 問題: "MCP Server connection timeout"

**原因**: サーバーが起動していない / ネットワーク接続エラー

```bash
# 確認コマンド
lsof -i :3000                           # Port が使用中か確認
ps aux | grep mcp                       # サーバープロセス確認

# 解決
python -m src.mcp.github_connector start &  # バックグラウンド起動
```

### 問題: "Authentication failed: Invalid token"

**原因**: API Token が無効 / スコープ不足 / 期限切れ

```bash
# 確認コマンド
curl -H "Authorization: token $GITHUB_API_TOKEN" \
  https://api.github.com/user

# 200 OK なら有効。それ以外はトークンを再生成
# https://github.com/settings/tokens
```

### 問題: "Permission denied: user@example.com"

**原因**: IAM 権限不足

```bash
# AWS 権限確認
aws iam list-attached-user-policies --user-name $USER

# 必要なポリシー
# - AWSLambdaFullAccess
# - AmazonEC2ReadOnlyAccess  など
```

### 問題: "Rate limit exceeded"

**原因**: API 呼び出し頻度が上限を超えた

```bash
# GitHub API: 60 req/hour (認証なし), 5000 req/hour (認証済み)
# AWS API: Variable (サービスごと)

# 対策
# 1. Backoff 戦略を実装（Exponential Backoff）
# 2. API 呼び出しをキャッシュ（Redis, memcache）
# 3. 並列数を制限
```

---

## 6. MCP + Subagent 統合パターン

### パターン: リサーチ Agent が GitHub を利用

```python
# src/agents/researcher_context.py
from mcp_server_github import GitHubMCP

class ResearcherAgent:
    def __init__(self):
        self.github = GitHubMCP(token=...)
    
    def research_project(self, owner, repo):
        # MCP 経由で GitHub 情報を取得
        pr_list = self.github.list_pull_requests(owner, repo)
        commits = self.github.get_commits(owner, repo)
        
        # 分析
        analysis = self.analyze(pr_list, commits)
        return analysis
```

**Context 消費**:
- ResearcherAgent 起動時: 20k tokens (CLAUDE.md等)
- GitHub MCP API 呼び出し: ~1k tokens/call（API 応答ペイロード）
- 合計: 120k tokens/研究タスク

---

## 7. セキュリティ & ベストプラクティス

### 認証情報の管理

| 方式 | 推奨度 | 説明 |
|------|------|------|
| **.env.local** | 🟢 推奨 | ローカル開発用（.gitignore対象） |
| **環境変数** | 🟢 推奨 | CI/CD パイプライン用 |
| **Secrets Manager** | 🟠 推奨 | 本番環境（AWS Secrets Manager等） |
| **コードに直書き** | 🔴 厳禁 | 絶対にしない |

### API Token の安全性

```bash
# ❌ 危険
export GITHUB_API_TOKEN="ghp_abc123..."  # シェル履歴に残る

# ✅ 安全
echo "ghp_abc123..." > ~/.github-token   # ファイル化
chmod 600 ~/.github-token
export GITHUB_API_TOKEN=$(cat ~/.github-token)
```

### Scope の最小化

```python
# GitHub Token: 必要なスコープのみ
# - repo: リポジトリアクセス
# - workflow: CI/CD
# - ❌ admin:org_hook（不要なら選択しない）

# AWS IAM: ポリシーはリソース単位で制限
# ❌ "Resource": "*"
# ✅ "Resource": "arn:aws:lambda:*:ACCOUNT_ID:function/my-func"
```

---

## 8. MCP サーバーの本番デプロイ

### Docker で MCP Server を実行

```dockerfile
# Dockerfile.mcp
FROM python:3.10-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

ENV GITHUB_API_TOKEN=${GITHUB_API_TOKEN}
ENV AWS_PROFILE=${AWS_PROFILE}

CMD ["python", "-m", "src.mcp.github_connector"]
```

```bash
# ビルド・実行
docker build -f Dockerfile.mcp -t mcp-server .
docker run -e GITHUB_API_TOKEN=$GITHUB_API_TOKEN \
  -e AWS_PROFILE=$AWS_PROFILE \
  -p 3000:3000 \
  mcp-server
```

### 監視・ロギング

```python
# src/mcp/monitoring.py
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
))
logger.addHandler(handler)

# MCP 呼び出しをログ
logger.info(f"MCP call: github.create_pr(owner={owner}, repo={repo})")
```

---

## 参考

- MCP Introduction: https://anthropic.skilljar.com/introduction-to-model-context-protocol
- MCP GitHub Server: https://github.com/anthropics/mcp-servers
- Claude Skills vs MCP: https://intuitionlabs.ai/articles/claude-skills-vs-mcp
