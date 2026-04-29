# SECURITY.md - セキュリティベストプラクティス

## セキュリティ哲学

Claude Code のセキュリティは **多層防御（Defense in Depth）** で実現します。

```
Layer 1: 権限最小化原則（Principle of Least Privilege）
  ├─ 必要なツールのみ有効化
  ├─ 不要な MCP サーバーは削除
  └─ read-only を優先

Layer 2: 入力検証（Input Validation）
  ├─ ユーザー入力 → 常に検証
  ├─ 外部 API → 型チェック
  └─ ファイル読込 → 所有権確認

Layer 3: セキュア・コーディング（Secure Coding）
  ├─ SQL injection 回避
  ├─ XSS 回避
  └─ Path traversal 回避

Layer 4: 環境管理（Environment Management）
  ├─ .env ファイル = 本番 secret として扱う
  ├─ 認証情報は環境変数のみ
  └─ Git に secret を commit しない
```

---

## 権限最小化ガイドライン

### 初期設定（最も制限的）

`~/.claude/settings.json` で全 hooks・MCP を無効化：

```json
{
  "hooks": {
    "enabled": false
  },
  "mcpServers": {}
}
```

### 段階的な権限追加（許可リスト方式）

必要なツールのみ「許可リストに追加」する方式：

```json
{
  "hooks": {
    "enabled": true,
    "allowedHooks": [
      "PreToolUse",     // ツール実行前チェック
      "PostToolUse"     // ツール実行後のlint等
    ]
  },
  "mcpServers": {
    "github": {
      "command": "mcp-server-github",
      "env": { "GITHUB_TOKEN": "${GITHUB_TOKEN}" }
    },
    "memory": {
      "type": "memory"
    }
  },
  "bash": {
    "sandbox": true,    // 常に true
    "blocklist": [
      "rm -rf /",
      "sudo",
      ":(){:|:&};:"     // Fork bomb
    ]
  }
}
```

### 環境変数とシークレット管理

**禁止事項**:
- ❌ API キー、パスワード、トークンを .env ファイルで管理（テンプレートのみ共有）
- ❌ Git リポジトリに認証情報をコミット
- ❌ README、ブログ記事に secret を埋込
- ❌ Claude コマンドラインに生パスワード入力

**推奨事項**:
- ✅ 本番 secret は環境変数のみ
- ✅ ローカル開発用は `.env.local` に（.gitignore に登録）
- ✅ CI/CD は Secret Manager（GitHub Actions secrets 等）を使用
- ✅ secret rotation を 3 ヶ月ごとに実施

```bash
# ✅ 正しい使い方
export GITHUB_TOKEN="ghp_xxxxx"
export DATABASE_URL="postgres://..."

# ❌ 間違った使い方
export API_KEY="sk-1234"  # ← git add .env してコミット
```

---

## 依存関係管理とセキュリティ

### 初期スキャン

新規プロジェクト開始時：

```bash
# Node.js
npm audit
# → 高リスク脆弱性があれば修正

# Python
pip audit
# → 同様に修正

# Go
go list -json -m all | nancy sleuth
```

### 継続的スキャン

**月 1 回**（以上）の定期実行：

```bash
npm audit
npm outdated  # 更新可能なパッケージ一覧

# または Dependabot を GitHub で有効化
# Settings → Code security → Dependabot alerts: ON
```

### 安全な更新ポリシー

❌ **危険な更新方法**:
```bash
npm audit fix --force  # メジャーバージョンアップで破損リスク
npm update             # すべてのパッケージを一括更新
```

✅ **安全な更新方法**:
```bash
# Step 1: 更新内容を確認
npm outdated

# Step 2: マイナー・パッチバージョンのみ更新
npm update --save

# Step 3: ローカルテスト実行
npm test

# Step 4: Git diff で変更確認
git diff package-lock.json | head -50

# Step 5: コミット
git commit -m "chore: bump dependencies (minor versions)"

# Step 6: 本番環境で動作確認
npm start  # ローカルで確認
# → 問題なければ本番デプロイ
```

### 新規ライブラリ導入時の審査チェックリスト

```
新規パッケージ追加時は以下を確認：

□ NPM メタデータ確認
  ├─ Star 数: 100+ 推奨
  ├─ 最終更新: 6 ヶ月以内
  ├─ オープンイシュー数: 50 以下
  └─ 保守者が実在するか

□ セキュリティスキャン
  ├─ npm audit で既知脆弱性なし
  ├─ CVSS スコア 7.0 以上の issue がないか
  └─ 依存関係が少ない（<20 個推奨）

□ ライセンス確認
  ├─ MIT, Apache 2.0, BSD, GPL ← 許容範囲
  ├─ AGPL → チームで判断
  └─ 独自/不明瞭 → 追加禁止

□ コード品質
  ├─ TypeScript 型定義あり（.d.ts）
  ├─ 単体テストあり
  ├─ リポジトリに .gitignore がある
  └─ README が詳細

□ 本体での動作確認
  ├─ ローカルで npm install + npm test
  ├─ 予期しない副作用がないか
  └─ パフォーマンス低下がないか
```

**チーム環境での承認プロセス**:
```
開発者 → package.json に追加
  ↓
Pull Request（チェックリスト添付）
  ↓
レビュアー：チェックリスト確認
  ↓
Approve → merge → npm install → テスト実行
  ↓
本番環境反映
```

---

## コード脆弱性パターンと回避方法

### SQL Injection

❌ **脆弱な例**:
```javascript
// 危険：ユーザー入力を直接埋込
const userId = req.query.id;
const sql = `SELECT * FROM users WHERE id = ${userId}`;
db.query(sql);
```

✅ **安全な例**:
```javascript
// 正解：Prepared statement + パラメータ化
const userId = req.query.id;
const sql = `SELECT * FROM users WHERE id = ?`;
db.query(sql, [userId]);  // パラメータは別途

// または Prisma/ORM を使用
const user = await prisma.user.findUnique({
  where: { id: userId }  // 自動的にサニタイズ
});
```

### Cross-Site Scripting (XSS)

❌ **脆弱な例（React）**:
```jsx
// 危険：innerHTML で HTML を直接埋込
const userContent = `<img src=x onerror="alert('xss')">`;
return <div dangerouslySetInnerHTML={{ __html: userContent }} />;
```

✅ **安全な例**:
```jsx
// 正解：React は自動的にエスケープ
const userContent = `<img src=x onerror="alert('xss')">`;
return <div>{userContent}</div>;  // テキストとして表示（無害化）

// または html-escaper ライブラリ
import { escape } from 'lodash';
return <div>{escape(userContent)}</div>;
```

### Path Traversal

❌ **脆弱な例**:
```javascript
// 危険：../../../ 等でディレクトリ外アクセス可能
const filePath = `./uploads/${req.query.filename}`;
fs.readFileSync(filePath);  // /etc/passwd も読める可能性
```

✅ **安全な例**:
```javascript
// 正解：ホワイトリスト + パス正規化
const path = require('path');
const allowedDir = '/var/uploads';
const filePath = path.join(allowedDir, req.query.filename);

// 許可ディレクトリ外へのアクセスをブロック
if (!filePath.startsWith(allowedDir)) {
  throw new Error("Access denied");
}

fs.readFileSync(filePath);
```

### Command Injection

❌ **脆弱な例**:
```javascript
// 危険：シェル変数をそのまま埋込
const userInput = req.body.filename;
exec(`convert ${userInput} output.jpg`);  // ; rm -rf / も実行可能
```

✅ **安全な例**:
```javascript
// 正解：execFile + 引数配列化（シェルパース回避）
const { execFile } = require('child_process');
execFile('convert', [userInput, 'output.jpg'], (err) => {
  // シェルメタ文字は自動的にエスケープ
});
```

---

## MCP サーバー審査フロー

### MCP サーバー追加時の判定フロー

```
新規 MCP サーバー要望
    ↓
┌─────────────────────────────┐
│ セキュリティレビュー        │
├─────────────────────────────┤
│ 1. 公式・信頼されるソース？  │
│    ├─ Anthropic 公式 → OK   │
│    ├─ GitHub Star 500+ → OK │
│    └─ 不明瞭 → NG           │
│                             │
│ 2. どのツールを公開する？    │
│    ├─ read-only API → LOW   │
│    ├─ write API → MEDIUM    │
│    └─ admin/delete → HIGH   │
│                             │
│ 3. 認証情報の管理方法は？    │
│    ├─ 環境変数 → OK         │
│    ├─ ハードコード → NG     │
│    └─ OAuth2 → OK           │
│                             │
│ 4. ネットワークリスク       │
│    ├─ 社内ネット限定 → OK   │
│    ├─ 外部 API → 要確認     │
│    └─ P2P/WebSocket → NG    │
└─────────────────────────────┘
    ↓
リスク判定
├─ Low → 即座に追加
├─ Medium → チーム合意後
└─ High → CTO/Security Officer 承認
```

### MCP サーバー設定テンプレート

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
      },
      "scope": "project",  // 全プロジェクト共有
      "readOnly": false    // write 権限あり
    },
    "notion": {
      "command": "npx",
      "args": ["mcp-server-notion"],
      "env": {
        "NOTION_API_KEY": "${NOTION_API_KEY}"
      },
      "scope": "user",     // ユーザー個別設定
      "readOnly": true     // read-only
    }
  }
}
```

### 推奨 MCP サーバー一覧

| サーバー | 目的 | リスク | 推奨度 |
|---|---|---|---|
| **github** | GitHub API 統合 | Medium | ⭐⭐⭐ |
| **memory** | セッションメモリ | Low | ⭐⭐⭐ |
| **filesystem** | ローカルファイル | Medium | ⭐⭐ |
| **bash** | シェルコマンド | High | ⭐ |
| **npm** | Package manager | Medium | ⭐⭐ |
| **notion** | Notion 連携 | Low | ⭐⭐ |
| **slack** | Slack 連携 | Medium | ⭐⭐ |

**非推奨**:
- ❌ curl（HTTP リクエスト）→ WebFetch で代替
- ❌ eval, exec（任意コード実行）

---

## git セキュリティルール

### コミット・プッシュ前チェック

```bash
# Step 1: 意図しないファイルが staged になっていないか確認
git status

# Step 2: diff を確認（secret コミット防止）
git diff --cached | grep -E "(password|secret|key|token|api)" && echo "ERROR: Secret detected"

# Step 3: 大きなファイルの add を防止（LFS 未対応の場合）
git diff --cached --stat | awk '{print $NF, $1}' | grep -E "[0-9]M" && echo "WARN: Large file"

# Step 4: コミット実行
git commit -m "message"

# Step 5: プッシュ実行（--force は避ける）
git push origin branch-name
```

### pre-commit hooks 設定（チーム全体）

`.git/hooks/pre-commit` で自動チェック：

```bash
#!/bin/bash

# Secret detection
git diff --cached | grep -qE "(password=|api_key|secret_token)" && {
  echo "❌ Secret detected in commit"
  exit 1
}

# Lint check
npm run lint || exit 1

# Type check
npm run type-check || exit 1

echo "✅ Pre-commit checks passed"
exit 0
```

### ブランチ保護ルール（GitHub）

Settings → Branch protection rules:
- ✅ Require pull request reviews: **1 person minimum**
- ✅ Require status checks to pass: **npm test, type-check**
- ✅ Require branches to be up to date
- ❌ Allow force pushes: **OFF**
- ❌ Allow deletions: **OFF**

---

## ログ・監視・アラート

### ログに記録すべき情報

```
✅ 記録する：
├─ 認証試行（成功・失敗）
├─ 権限変更
├─ データアクセス（大量読込）
├─ API レート制限超過
└─ エラーメッセージ（一般的なもののみ）

❌ 記録してはいけない：
├─ パスワード・secret
├─ 個人情報（PII）
├─ クレジットカード番号
└─ 暗号化されていない通信内容
```

### ログレベルガイドライン

```
ERROR  : ログイン失敗、権限不足、脆弱性検知
WARN   : 異常なアクセスパターン、レート制限近接
INFO   : ユーザー操作（create/delete 等の audit log）
DEBUG  : 開発環境でのみ、本番では削除
```

---

## セキュリティインシデント対応

### 脆弱性発見時の対応フロー

```
脆弱性の発見
    ↓
    【重大度判定】
    ├─ Critical (CVSS >= 9.0)
    │  └─ 即座に対応・デプロイ
    ├─ High (CVSS 7-9)
    │  └─ 48 時間以内に対応
    └─ Medium/Low (CVSS < 7)
       └─ 定期更新ウィンドウで対応
    ↓
    【パッチ適用】
    ├─ 対象パッケージのアップデート
    ├─ ローカルテスト
    └─ 本番環境へのデプロイ
    ↓
    【ポストモーテム】
    ├─ 原因分析
    ├─ 再発防止策の実装
    └─ チーム全体への共有
```

### Anthropic へのセキュリティ報告

Claude Code に脆弱性を発見した場合：

1. **Anthropic HackerOne プログラムで報告**
   - https://hackerone.com/anthropic
   - 詳細な再現手順を含める
   - 公開 disclosure は待機

2. **一般的なセキュリティ issue（ライブラリ脆弱性等）**
   - npm/GitHub の Security Advisory を活用
   - チーム内で共有・対応

---

## チェックリスト

### 毎日実行
- [ ] git status で意図しないファイルがないか確認
- [ ] npm audit で脆弱性警告がないか確認

### 毎週実行
- [ ] 権限設定の見直し（MCP サーバー）
- [ ] git log で異常なコミットがないか確認

### 毎月実行
- [ ] 完全な依存関係スキャン（npm audit + pip audit）
- [ ] シークレット・認証情報が正しく管理されているか確認
- [ ] アクセスログの監視（不正アクセスの兆候）

### 四半期実行
- [ ] セキュリティトレーニング（チーム向け）
- [ ] ポリシー見直し（本ドキュメント更新）

---

**最終更新**: 2026-04-29  
**参照すべき関連ドキュメント**: CLAUDE.md（禁止事項）  
**次のステップ**: MCP.md でサーバー統合を学習
