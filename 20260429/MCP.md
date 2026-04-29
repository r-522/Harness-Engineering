# MCP.md - Model Context Protocol サーバー統合ガイド

## MCP とは

**Model Context Protocol (MCP)** は、Claude Code が外部ツール・API・データベースにアクセスするための標準化プロトコルです。

```
Claude Code
    ↓
MCP クライアント層
    ↓
[MCP サーバー1] [MCP サーバー2] [MCP サーバー3]
    ↓              ↓              ↓
   GitHub        Notion        Slack
   API           API           API
```

**メリット**:
- ✅ ツール追加が簡単（CLI 1 行で可能）
- ✅ 組織固有の API も MCP サーバー化できる
- ✅ Tool Search で lazy loading（コンテキスト節約）
- ✅ セキュリティ境界が明確

---

## MCP サーバー設定スコープ

### 3 つのスコープレベル

```
グローバル（~/.claude/mcp.json）
  ├─ すべてのプロジェクト・セッションで有効
  ├─ 個人の汎用設定（GitHub など）
  └─ 権限: 管理者のみ設定可能

プロジェクト（.claude/mcp.json）
  ├─ 特定プロジェクトのみで有効
  ├─ プロジェクト固有ツール（社内 API 等）
  └─ 権限: プロジェクトメンバー設定可能

ローカル（.claude/mcp.local.json）
  ├─ 個人マシンローカルのみ
  ├─ テスト・開発用
  └─ Git に commit しない（.gitignore 登録）
```

### 推奨設定戦略

```
Layer 1: グローバル（最小限）
  ├─ github （GitHub API）
  └─ memory （セッションメモリ）

Layer 2: プロジェクト（プロジェクト固有）
  ├─ 社内 API サーバー
  ├─ Notion（ドキュメント管理）
  └─ Slack（チーム通知）

Layer 3: ローカル（開発用・テスト）
  ├─ local-api-mock （開発環境 API）
  └─ custom-tools （個人実験用）
```

---

## 推奨 MCP サーバー一覧

### Tier 1: 必須サーバー

#### 1. GitHub MCP

**目的**: GitHub API 統合（PR・Issue・Repository 操作）

```json
{
  "mcpServers": {
    "github": {
      "command": "uvx",
      "args": [
        "mcp-server-github"
      ],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}",
        "GITHUB_OWNER": "${GITHUB_OWNER}"
      }
    }
  }
}
```

**使用例**:
```
> create pull request
> list open issues
> add comment to issue #123
> merge pull request
```

**セットアップ**:
```bash
# 1. Personal Access Token 生成
# GitHub Settings → Developer settings → Personal access tokens

# 2. 環境変数設定
export GITHUB_TOKEN="ghp_xxxxx"
export GITHUB_OWNER="your-org"

# 3. Claude Code 起動
claude-code
```

#### 2. Memory MCP

**目的**: セッション間メモリ保存（重要な進捗を永続化）

**機能**:
```
/memory save    → セッション内容を記録
/memory load    → 前回のメモ復元
/memory clear   → メモ削除
```

**自動有効化**: Claude Code が自動的に含める（追加設定不要）

---

### Tier 2: 推奨サーバー（用途別）

#### 3. Notion MCP（ドキュメント・プロジェクト管理）

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["mcp-server-notion"],
      "env": {
        "NOTION_API_KEY": "${NOTION_API_KEY}"
      }
    }
  }
}
```

**セットアップ**:
```bash
# 1. Notion Integration 作成
# https://www.notion.so/my-integrations

# 2. API Key 取得 & 環境変数設定
export NOTION_API_KEY="secret_xxxxx"

# 3. Notion Page と連携
# Integration を対象 page に招待
```

**使用例**:
```
> read notion page "Project Roadmap"
> update notion task "Implement authentication" to done
> list all notion databases
```

#### 4. Slack MCP（チーム通知・コラボレーション）

```json
{
  "mcpServers": {
    "slack": {
      "command": "npx",
      "args": ["mcp-server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}",
        "SLACK_CHANNEL": "${SLACK_CHANNEL}"
      }
    }
  }
}
```

**使用例**:
```
> send slack message "Deploy ready for review"
> post code snippet to #dev-channel
> notify team about test failures
```

#### 5. SQLite / PostgreSQL MCP（データベース操作）

```json
{
  "mcpServers": {
    "sqlite": {
      "command": "npx",
      "args": ["mcp-server-sqlite", "path/to/database.db"]
    },
    "postgres": {
      "command": "npx",
      "args": ["mcp-server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

**使用例**:
```
> query database: select count(*) from users
> insert test data into products table
> analyze query performance
```

**セキュリティ注意**:
- ✅ 読み取りのみ権限（本番 DB）
- ❌ write・delete 権限（開発/テスト環境のみ）

---

### Tier 3: 専門用途サーバー

#### 6. npm / Python Package Manager

**npm MCP**: package.json 操作、dependency 管理

```bash
> search npm package "lodash"
> check npm package security "axios@0.21.0"
> show outdated packages
```

#### 7. Docker MCP

**用途**: Docker コンテナ管理

```bash
> list docker containers
> build docker image
> run container with environment variables
```

#### 8. AWS / GCP / Azure MCP

**用途**: クラウドインフラ操作（リソース確認、デプロイ等）

**セキュリティ警告** ⚠️:
- AWS credentials は環境変数で管理（.env.local）
- 本番環境操作には 2 要素認証必須
- 権限を IAM で最小化

---

## MCP サーバーのインストール・設定

### 方法 1: npx による直接インストール（推奨）

```bash
# step 1: 設定ファイル作成
cat > ~/.claude/mcp.json << 'EOF'
{
  "mcpServers": {
    "github": {
      "command": "uvx",
      "args": ["mcp-server-github"]
    },
    "notion": {
      "command": "npx",
      "args": ["mcp-server-notion"]
    }
  }
}
EOF

# step 2: 環境変数設定
export GITHUB_TOKEN="ghp_xxxxx"
export NOTION_API_KEY="secret_xxxxx"

# step 3: Claude Code 再起動
claude-code
```

### 方法 2: CLI コマンド（簡易設定）

```bash
# GitHub サーバーを追加
claude-code mcp add github --env GITHUB_TOKEN=$GITHUB_TOKEN

# 設定確認
claude-code mcp list
```

### 方法 3: プロジェクト固有設定

```bash
# プロジェクトディレクトリで
mkdir -p .claude
cat > .claude/mcp.json << 'EOF'
{
  "mcpServers": {
    "internal-api": {
      "command": "node",
      "args": ["./mcp-servers/internal-api.js"]
    }
  }
}
EOF
```

---

## トラブルシューティング

### MCP サーバーが起動しない

```bash
# Step 1: 環境変数確認
echo $GITHUB_TOKEN
# → 空 = 設定漏れ

# Step 2: 設定ファイル文法確認
cat ~/.claude/mcp.json | jq .
# → syntax error があれば修正

# Step 3: Claude Code ログ確認
# Claude Code 内で /debug コマンド実行
> /debug

# Step 4: サーバー再起動
# Claude Code を再起動
```

### トークン制限エラー

```
Error: Rate limit exceeded (GitHub API)
↓
対策:
1. API Token の権限確認
2. API 呼び出し頻度を削減
3. キャッシング戦略を導入
```

### 権限不足エラー

```
Error: Permission denied (Notion API)
↓
対策:
1. Integration が対象 page に招待されているか確認
2. API Key の有効期限確認
3. 権限レベルを確認（read-only 等）
```

---

## 組織固有 MCP サーバーの開発

### カスタムサーバーテンプレート（Node.js）

```typescript
// mcp-servers/custom-api.js
const net = require('net');
const readline = require('readline');

const server = net.createServer((socket) => {
  const rl = readline.createInterface({ input: socket });

  socket.on('data', async (data) => {
    const request = JSON.parse(data.toString());

    // Tool Request: company-api
    if (request.method === 'tools/call') {
      const { name, arguments: args } = request.params;

      if (name === 'fetch-deployment-status') {
        const result = await getDeploymentStatus(args.environment);
        socket.write(JSON.stringify({
          jsonrpc: '2.0',
          id: request.id,
          result: { content: [{ type: 'text', text: result }] }
        }));
      }
    }
  });
});

async function getDeploymentStatus(env) {
  // 社内 API 呼び出し
  const response = await fetch(`https://internal-api.company.com/deploy/${env}`);
  return await response.text();
}

server.listen(3000);
```

### 設定に追加

```json
{
  "mcpServers": {
    "company-api": {
      "command": "node",
      "args": ["./mcp-servers/custom-api.js"],
      "scope": "project"
    }
  }
}
```

---

## 権限管理・セキュリティ

### MCP サーバー権限の最小化

```json
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["mcp-server-postgres"],
      "readOnly": true,          // 読み取りのみ
      "allowedOperations": [
        "SELECT",                 // 許可
        "EXPLAIN"                 // 許可
      ],
      "blockedOperations": [
        "DELETE",
        "DROP",
        "ALTER"                   // 禁止
      ]
    }
  }
}
```

### 環境別設定の分離

```bash
# 開発環境
export DATABASE_URL="postgres://dev-user@localhost/dev_db"

# テスト環境（本番権限なし）
export DATABASE_URL="postgres://test-user@testdb/test"

# 本番環境（ローカルでは設定しない）
# → CI/CD で環境変数に設定
```

---

## MCP サーバー監視・メトリクス

### Tool Search による確認

```
> /tools-info

GitHub MCP:
  ├─ search-code: 検索
  ├─ read-file: ファイル読込
  └─ create-pull-request: PR 作成

Notion MCP:
  ├─ read-page: ページ読込
  └─ update-page: ページ更新

（合計 N 個のツール利用可能）
```

### パフォーマンス監視

```bash
# MCP サーバーのレスポンス時間を測定
# /debug を実行
> /debug

MCP Performance:
├─ github: avg 200ms
├─ notion: avg 500ms (Notion API 遅延)
└─ memory: avg 50ms
```

---

## チェックリスト

### 初回セットアップ
- [ ] グローバル mcp.json に GitHub・Notion を設定
- [ ] 環境変数（TOKEN 等）を `.env.local` に記載
- [ ] `/tools-info` で確認・動作テスト

### 月次確認
- [ ] MCP サーバーの API Token 有効期限確認
- [ ] Rate limit エラーの有無確認
- [ ] 新規サーバー要望があれば SECURITY.md チェックリスト実施

### 四半期確認
- [ ] 使用していない MCP サーバーの削除検討
- [ ] 新規公開 MCP サーバーの評価

---

**最終更新**: 2026-04-29  
**参照すべきファイル**: SECURITY.md（MCP 審査フロー）  
**次のステップ**: HOOKS.md で自動化フローを学習
