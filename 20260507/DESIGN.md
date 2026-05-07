# DESIGN.md - 設計原則・アーキテクチャ

**スコープ**: Harness Engineering プロジェクト全体  
**更新日**: 2026-05-07  

---

## 1. 設計原則

### SOLID 原則（適用）

| 原則 | 説明 | Harness Engineering での実装 |
|---|---|---|
| **S** (Single Responsibility) | 各モジュールは1つの責任のみ | Subagent: 1ロール = 1責任（Researcher/Implementer等） |
| **O** (Open/Closed) | 拡張に開く、修正に閉じる | MCP Server追加時は既存コード変更なし（インターフェース定義） |
| **L** (Liskov Substitution) | 子は親の機能を完全置換可能 | BaseSubagent → 各専門Subagent継承 |
| **I** (Interface Segregation) | 不要なインターフェース依存を避ける | Tool permissions: 最小権限の原則 |
| **D** (Dependency Inversion) | 具体依存を抽象依存に | MCP Server選択は実行時に決定（設定ファイル） |

### 追加原則

**DRY (Don't Repeat Yourself)**
```python
# ❌ 悪い例：
if validation_failed:
    log_error("Validation failed")
    send_alert("Validation failed")
    ...

# ✅ 良い例：
def handle_validation_failure():
    log_and_alert("Validation failed")

# 3回目以降は共通関数化
```

**YAGNI (You Aren't Gonna Need It)**
- 仕様にない機能は実装しない
- 「将来必要になるかも」は理由にしない
- 実装は必要になった時点で

**KISS (Keep It Simple, Stupid)**
- ネストの深さ ≤ 3層
- 関数長 ≤ 30行
- 循環複雑度 ≤ 10

---

## 2. アーキテクチャパターン

### 全体アーキテクチャ図

```
┌──────────────────────────────────────────────────┐
│          Main Agent (Claude Opus 4.7)            │
│                                                  │
│  ┌─────────────────────────────────────────┐   │
│  │    Orchestration Layer                  │   │
│  │  - Task Decomposition                   │   │
│  │  - Subagent Scheduling                  │   │
│  │  - Result Aggregation                   │   │
│  └──────────────┬──────────────────────────┘   │
│                 │                               │
└─────────────────┼───────────────────────────────┘
                  │
      ┌───────────┼───────────┬──────────────┐
      ▼           ▼           ▼              ▼
   Researcher  Implementer Reviewer      Debugger
   Subagent   Subagent   Subagent       Subagent
      │           │          │             │
      └───────────┼──────────┴─────────────┘
                  │
      ┌───────────┴──────────┬────────────────┐
      ▼                      ▼                ▼
   Tool Chain         MCP Server         External APIs
  (Read/Edit/Bash)   (AWS/GCP/Custom)    (GitHub/Slack)
```

### コンポーネント責務

| コンポーネント | 責務 | 依存先 |
|---|---|---|
| **Main Agent** | 全体オーケストレーション | Subagent群 |
| **Researcher** | 情報収集、分析 | Read/WebFetch/Search |
| **Implementer** | コード実装、修正 | Edit/Write/Bash |
| **Reviewer** | 品質検証、テスト | Bash/Read |
| **Debugger** | エラー分析、デバッグ | Bash/Read/MCP |
| **Tool Chain** | ファイル操作、実行 | OS/Filesystem |
| **MCP Server** | 外部API統合 | Cloud Platform SDKs |

---

## 3. ディレクトリ構成ガイド

### 最適な構成

```
project-root/
│
├── .claude/                    # ← エージェント設定
│   ├── CLAUDE.md              # プロジェクト制約
│   ├── AGENT.md               # エージェント定義
│   ├── DESIGN.md              # 本ファイル
│   ├── SKILLS.md              # スキル定義
│   ├── MCP-INTEGRATION.md      # MCP設定ガイド
│   └── DISCOVERIES.md          # チーム学習記録
│
├── src/                        # ← メインコード
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base.py            # BaseSubagent クラス
│   │   ├── researcher.py      # Researcher実装
│   │   ├── implementer.py     # Implementer実装
│   │   └── ...
│   │
│   ├── skills/                # ← カスタムスキル
│   │   ├── orchestrate_parallel.md
│   │   ├── context_optimize.md
│   │   └── scripts/
│   │       ├── parallel_runner.py
│   │       └── ...
│   │
│   ├── mcp/                   # ← MCP統合
│   │   ├── __init__.py
│   │   ├── base_server.py     # 基本サーバー
│   │   ├── aws_tools.py
│   │   ├── gcp_tools.py
│   │   └── custom_tools.py
│   │
│   ├── core/                  # ← 共通ライブラリ
│   │   ├── __init__.py
│   │   ├── config.py          # 設定管理
│   │   ├── context.py         # コンテキスト管理
│   │   ├── orchestrator.py    # オーケストレーター
│   │   ├── logger.py          # ロギング
│   │   └── errors.py          # カスタム例外
│   │
│   └── api/                   # ← API層
│       ├── __init__.py
│       ├── main.py            # メインエンドポイント
│       ├── routes/
│       │   ├── agents.py
│       │   ├── tasks.py
│       │   └── ...
│       └── models.py          # Request/Responseモデル
│
├── tests/                     # ← テスト
│   ├── unit/
│   │   ├── test_agents.py
│   │   ├── test_skills.py
│   │   └── ...
│   │
│   ├── integration/
│   │   ├── test_orchestration.py
│   │   ├── test_mcp_integration.py
│   │   └── ...
│   │
│   ├── e2e/
│   │   ├── test_full_workflow.py
│   │   └── ...
│   │
│   └── fixtures/
│       ├── sample_tasks.json
│       ├── mock_mcp_responses.yaml
│       └── ...
│
├── config/                    # ← 設定ファイル
│   ├── settings.yaml          # アプリケーション設定
│   ├── agents.yaml            # Subagent設定
│   ├── mcp-servers.yaml       # MCP Server設定
│   └── development.yaml       # 開発環境設定
│
├── docs/                      # ← ドキュメント
│   ├── ARCHITECTURE.md        # アーキテクチャ詳細
│   ├── API.md                 # API仕様書
│   ├── DEPLOYMENT.md          # デプロイ手順
│   ├── TROUBLESHOOTING.md     # トラブル対応
│   └── ONBOARDING.md          # 新メンバー向け
│
├── scripts/                   # ← ユーティリティ
│   ├── setup.sh               # 環境構築
│   ├── test.sh                # テスト実行
│   ├── deploy.sh              # デプロイ
│   └── ...
│
├── Makefile                   # ← タスク自動化
├── pyproject.toml             # ← Python メタデータ
├── requirements.txt           # ← 依存パッケージ
├── .gitignore
├── .env.example               # ← 環境変数テンプレート
├── README.md
└── LICENSE
```

### ディレクトリ命名規則

| パターン | 用途 | 例 |
|---|---|---|
| `_{name}` | プライベート（内部専用） | `_internal.py` |
| `{name}s` | 複数形（複数ファイル格納） | `agents/`, `tests/` |
| `test_{name}.py` | テストファイル | `test_orchestrator.py` |
| `{name}.py` | モジュール | `config.py` |

---

## 4. データフロー & 状態管理

### リクエスト処理フロー

```
User Request
    │
    ▼
┌─────────────────────────────┐
│ Main Agent: UNDERSTAND      │
│ - 意図解析（3行サマリ）      │
│ - 必要情報確認              │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│ Main Agent: PLAN            │
│ - タスク分解（DAG生成）      │
│ - 依存関係確認              │
│ - リソース予測              │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│ Decide: 並列 or 直列         │
└────┬─────────────────────┬───┘
     │                     │
  並列 (1-3個)          直列
     │                     │
     ▼                     ▼
┌──────────────┐  ┌──────────────────┐
│ Subagent群   │  │ Sequential chain │
│ 同時実行     │  │ Task1→Task2→...  │
│ (async)      │  │                  │
└──────┬───────┘  └────────┬─────────┘
       │                   │
       └────────┬──────────┘
                │
                ▼
┌──────────────────────────────┐
│ Result Aggregation           │
│ - 結果の統合                 │
│ - 矛盾検出・解決             │
│ - 最終ドキュメント生成       │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│ Reviewer: VERIFY             │
│ - テスト実行                 │
│ - 型チェック                 │
│ - Linter確認                 │
└────────────┬─────────────────┘
             │
     ┌───────┴────────┐
     │                │
    PASS             FAIL
     │                │
     ▼                ▼
Success         Debugger Fix
     │           (修正試行)
     │           × max 2回
     │                │
     └────────┬───────┘
              │
              ▼
        User Notification
```

### 状態遷移図（Task）

```
┌─────────────┐
│   PENDING   │ ← 新規タスク
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  PLANNING   │ ← 計画策定中
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ EXECUTING   │ ← 実行中
└──┬──────┬───┘
   │      │
   │      └─────────┐
   │                │
   ▼                ▼
┌─────────────┐  ┌──────────────┐
│  VERIFYING  │  │ FAILED       │
└──────┬──────┘  │ (max retry)  │
       │         └──────────────┘
       ▼
┌─────────────┐
│ COMPLETED   │ ← 完了
└─────────────┘
```

---

## 5. エラーハンドリング & リカバリ

### エラー分類と対応

```python
# 1. Validation Error（入力エラー）
# → 即座に拒否、ユーザーに理由を説明
if not is_valid_input(task):
    raise ValidationError(f"Invalid input: {task}")

# 2. Transient Error（一時的エラー）
# → 自動リトライ（最大3回）
for attempt in range(3):
    try:
        result = call_external_api()
        break
    except NetworkError:
        if attempt < 2:
            await asyncio.sleep(2 ** attempt)  # 指数バックオフ
        else:
            raise

# 3. Persistent Error（永続的エラー）
# → ユーザー相談、エスカレーション
except PersistentError as e:
    ask_user(f"Cannot proceed: {e}")

# 4. Security Error（セキュリティ）
# → 即座に停止、詳細ログ記録
except SecurityError as e:
    escalate_to_security_lead(e)
    raise
```

---

## 6. パフォーマンス・スケーリング

### 最適化目標

| メトリック | 目標値 | 測定方法 |
|---|---|---|
| **Response Time** | <5s | `/time curl` |
| **Subagent Init** | <2s/agent | time Subagent起動 |
| **Context Usage** | ≤30% | token counter API |
| **Test Exec** | <60s (全テスト) | pytest --durations=10 |
| **MCP Latency** | <1s per call | mcp-server metrics |

### スケーリング戦略

**ボトルネック検出**:
```bash
# Profile実行
python -m cProfile -s cumtime main.py > profile.txt

# Context使用率監視
echo $CLAUDE_CONTEXT_USAGE_PERCENT

# MCP応答時間
curl -w "@curl-format.txt" https://mcp-server/health
```

**スケーリング判定**:
- 並列数 < 3 → 並列化
- テスト時間 > 60s → 並列テスト実装
- Context使用率 > 40% → Compaction戦略変更
- MCP Latency > 1s → キャッシング導入

---

## 7. セキュリティアーキテクチャ

### 最小権限の原則（Least Privilege）

```yaml
# tool_permissions.yaml
permissions:
  Read:
    allowed_paths:
      - src/**/*.py
      - tests/**/*.py
      - .claude/**/*.md
    denied_paths:
      - .env.local
      - .secrets/**
      - private/**
  
  Edit:
    allowed_paths: [src/**/*.py]
    denied_paths: [CLAUDE.md, package.json]  # 重要ファイルは保護
  
  Bash:
    allowed_commands:
      - pytest tests/
      - python -m mypy src/
      - git status
      - git diff
    denied_commands:
      - rm -rf /
      - git push --force
      - curl external-api
```

### MCP Server 検証フロー

```
MCP Server接続要求
    │
    ▼
┌────────────────────────────────┐
│ 1. セキュリティレビュー        │
│    - URL/ドメイン確認         │
│    - 認証スキーム確認         │
│    - 権限スコープ確認         │
└────────────┬───────────────────┘
             │
             ▼
┌────────────────────────────────┐
│ 2. 接続テスト                  │
│    - TCP接続確認              │
│    - TLS/SSL証明書検証        │
│    - 認証トークン有効性確認   │
└────────────┬───────────────────┘
             │
      ┌──────┴──────┐
      │             │
     PASS          FAIL
      │             │
      ▼             ▼
  有効化    → 管理者通知 + ログ
```

---

## 8. 監視・ロギング・メトリクス

### ログレベルと出力

```python
import logging

# CRITICAL: システム停止レベル
logging.critical("Database connection failed - system halt")

# ERROR: エラー（回復可能）
logging.error("Task failed after 3 retries")

# WARNING: 注意（動作続行）
logging.warning("Context usage at 75% - compaction recommended")

# INFO: 重要情報
logging.info("Task completed: feature_x in 45 seconds")

# DEBUG: 詳細情報（開発時のみ）
logging.debug(f"Subagent {agent_id} received task {task_id}")
```

### メトリクス収集

```yaml
# metrics.yaml
metrics:
  - name: subagent_execution_time
    type: histogram
    buckets: [1, 5, 10, 30, 60, 120]  # 秒
  
  - name: context_usage_percent
    type: gauge
    min: 0
    max: 100
  
  - name: task_success_rate
    type: counter
    labels: [task_type, subagent_role]
  
  - name: mcp_call_latency
    type: histogram
    buckets: [0.1, 0.5, 1.0, 5.0]  # 秒
```

---

## 9. テスタビリティ

### テスト層構成

```
Unit Tests (40%)        → 個別関数・クラス
├── test_agents.py
├── test_skills.py
└── test_core.py

Integration Tests (35%) → Subagent × Tool Chain
├── test_orchestration.py
├── test_mcp_integration.py
└── test_context_management.py

E2E Tests (25%)        → 全体フロー
├── test_full_workflow.py
└── test_production_scenario.py
```

### テスト記述ガイド

```python
# ✅ 良い例
def test_researcher_collects_multiple_sources():
    """Researcher が複数ソースから情報収集できる."""
    task = Task(query="Recent AI trends")
    researcher = ResearcherAgent()
    
    result = researcher.execute(task)
    
    assert len(result.sources) >= 3
    assert all(src.valid_url for src in result.sources)
    assert result.summary is not None

# ❌ 悪い例
def test_agent():
    """Test agent."""  # 説明不足
    agent = Agent()
    agent.execute()  # 検証なし
    assert True  # 意味のないアサーション
```

---

**最終チェック**:
- [ ] アーキテクチャ図は実装可能か？
- [ ] ディレクトリ構成に矛盾はないか？
- [ ] パフォーマンス目標は現実的か？

✅ **本DESIGN.mdは本実装の設計基盤として完成**
