# DESIGN.md - Harness Engineering 設計原則 & アーキテクチャ

**対象**: 設計パターン、アーキテクチャガイド、コンポーネント設計  
**作成日**: 2026-05-12  
**バージョン**: 2.0 (Agent Teams対応)

---

## 1. 適用する設計原則

### SOLID 原則

| 原則 | 適用方針 | 例 |
|---|---|---|
| **S** (Single) | 各クラス/Subagent は1つの責務のみ | Researcher = 検索のみ、Implementer = 実装のみ |
| **O** (Open/Closed) | 拡張に開く、変更に閉じる | 新Subagent追加は容易、既存Subagent変更は最小 |
| **L** (Liskov) | 派生クラスは基底と入れ替え可能 | すべてのSubagent は同じ Response フォーマット |
| **I** (Interface) | 不要な依存を強制しない | Researcher は Implementer のインタフェースを知らない |
| **D** (Dependency) | 抽象に依存、具象に依存しない | Coordinator は Subagent 抽象 に依存、個別実装に依存しない |

### DRY (Don't Repeat Yourself) 原則

- **複数回出現** (3回以上)→ 共通化 (util, base class, skill)
- **1-2回出現** → 共通化しない（過度な抽象化を避ける）
- **ペアプログラミング不可の場合** → ドキュメント化 (.md ファイル)

### YAGNI (You Aren't Gonna Need It)

- **仕様にない機能** → 実装しない (e.g., 「将来マイクロサービス化したときのために...」は NO)
- **假想の拡張ポイント** → 必要になった時点で実装
- **複雑さの判定** → 循環複雑度 ≤ 10 を超えたら分割検討

---

## 2. アーキテクチャパターン

### 2.1 Agent Teams Orchestration パターン

```
┌─────────────────────────────────────────────────────┐
│           Coordinator (Orchestrator)                │
│  - Plan mode で全タスク先読み                         │
│  - Subagent 並列スケジュール                         │
│  - 結果統合・コンフリクト解決                         │
└────────────┬────────────────────────────────────────┘
             │
    ┌────────┴────────┬──────────────┬──────────────┐
    ↓                 ↓              ↓              ↓
┌─────────┐  ┌──────────────┐ ┌────────────┐ ┌──────────┐
│Researcher│  │ Implementer  │ │  Reviewer  │ │ Debugger │
│(256K)   │  │  (256K)      │ │  (256K)    │ │ (256K)   │
│         │  │              │ │            │ │          │
│ Web     │  │ Code TDD     │ │ Security   │ │ Root     │
│ Search  │  │ Refactor     │ │ Test       │ │ Cause    │
│         │  │              │ │ Perf       │ │ Analysis │
└────┬────┘  └──────┬───────┘ └────┬───────┘ └────┬─────┘
     │              │              │              │
     └──────────────┼──────────────┴──────────────┘
                    │
            ┌───────▼────────┐
            │ Aggregation &  │
            │ Conflict Fix   │
            └───────┬────────┘
                    │
            ┌───────▼────────┐
            │ Final Result   │
            │ to User        │
            └────────────────┘
```

**特性**:
- **独立性**: 各Subagent は独立したコンテキスト・メモリで動作
- **並列度**: MAX 3 (Context budget 制約)
- **通信**: JSON Report + Metadata のみ
- **失敗処理**: Subagent失敗 → Debugger 呼び出し

### 2.2 Prompt Caching レイアリング

```
Request
    ↓
┌────────────────────────────────────────────┐
│ [Cache PREFIX 1] システム命令 + Agent定義   │ ✓ 不変・キャッシュ可
│ - CLAUDE.md 要約 (3000 tokens)             │
│ - AGENT.md Persona (2000 tokens)           │
│ - DESIGN.md Patterns (1500 tokens)         │
│ └─ CACHE HIT RATE: 95%+ (毎回同じ)        │
└────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────┐
│ [Cache PREFIX 2] ツール定義 + API スキーマ  │ ✓ 長期不変・週1更新
│ - Subagent interface specs (2500 tokens)   │
│ - MCP server schemas (1500 tokens)         │
│ - API examples (1000 tokens)               │
│ └─ CACHE HIT RATE: 80%+ (週単位)          │
└────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────┐
│ [Cache PREFIX 3] 知識ベース                 │ ◐ 中期・月1更新
│ - Pattern examples (examples/*.py)         │
│ - Troubleshooting guide (2000 tokens)      │
│ └─ CACHE HIT RATE: 60%+ (月単位)          │
└────────────────────────────────────────────┘
        ↓
┌────────────────────────────────────────────┐
│ [可変部分] ユーザー クエリ + コンテキスト    │ ✗ キャッシュ不可
│ - User instructions                        │
│ - Real-time data                           │
│ - Session context                          │
└────────────────────────────────────────────┘
        ↓
    API Response
    (キャッシュ統計付き)
```

**設計則**:
1. **不変プレフィックス**: システム命令・ツール定義を最初に（3000-5000 tokens）
2. **長期プレフィックス**: API スキーマを次に（2000-4000 tokens）
3. **可変部分**: ユーザークエリを末尾に配置（キャッシュ無効化リスク最小）
4. **タイムスタンプ**: 絶対に early に挿入しない

### 2.3 エラーハンドリング・リカバリーパターン

```
Task実行
    ↓
[Subagent実行]
    ├─ SUCCESS (5s以内)
    │  └→ 結果をCoordinatorに返す
    │
    ├─ TIMEOUT (>600s)
    │  └→ Context 圧迫の可能性 → Debugger 起動
    │     Debugger: "なぜ遅い?" → ログ分析 → 最適化提案
    │
    ├─ CONTEXT_OVERFLOW (>256K token)
    │  └→ タスク分割 or Coordinator で処理分割
    │     (e.g., "3 sources × 3 parallel" → "1 source × 3 sequential")
    │
    └─ ERROR (Exception, API Failure)
       ├─ Known Error (e.g., GitHub API 403 Forbidden)
       │  └→ Debugger: "なぜ403?" → Token期限切れ/権限不足 → 対応提案
       │
       └─ Unknown Error (e.g., Segmentation fault)
          └→ Debugger: Root Cause Analysis
             1. Reproduce with minimal steps
             2. Generate 3+ hypotheses
             3. Test each hypothesis
             4. Return: Root Cause + Fix
```

---

## 3. コンポーネント設計ガイド

### 3.1 Subagent インターフェース（統一仕様）

```python
# 全Subagent が実装すべき Response フォーマット
@dataclass
class SubagentResponse:
    subagent_name: str          # e.g., "Researcher"
    status: Literal["SUCCESS", "ERROR", "TIMEOUT"]
    execution_time_ms: int
    context_used_tokens: int
    
    # SUCCESS の場合
    result: dict[str, Any] | None
    findings: list[str] | None
    
    # ERROR / TIMEOUT の場合
    error_message: str | None
    error_type: str | None
    
    # Metadata
    metadata: dict = field(default_factory=dict)
    # - cache_hit_rate: float (0.0-1.0)
    # - api_calls: int
    # - parallel_tasks: int
    
    def to_json(self) -> str:
        return json.dumps(asdict(self), default=str)
```

### 3.2 Skill インターフェース（SKILLS.md）

```markdown
---
skill_name: example_skill
description: "What this skill does"
triggers:
  - "keyword1..."
  - "pattern2..."
target_subagent: [Researcher, Implementer]  # Optional
---

# Skill: Example

## When to Use
- Condition 1
- Condition 2

## Steps
1. Input parsing
2. Validation
3. Execution
4. Output formatting

## Checklist
- [ ] Check 1
- [ ] Check 2
```

### 3.3 MCP Server インテグレーション

```yaml
# config/mcp-servers.yaml
mcp_servers:
  github:
    endpoint: "https://mcp.github.com"
    auth_method: "oauth"
    scope: ["repo:read", "issues:write"]
    rate_limit: 100  # requests/minute
    timeout_sec: 10
    fallback_enabled: true
    security_review:
      reviewed_by: "Security Team"
      review_date: "2026-05-01"
      approval: true
    
  slack:
    endpoint: "https://slack.com/api"
    auth_method: "token"
    token_env: "SLACK_BOT_TOKEN"
    scope: ["chat:write", "users:read"]
    monitoring:
      error_alert_threshold: 5  # alerts per hour
      dashboard: "https://datadog.com/..."
```

---

## 4. ディレクトリ構成と責務分離

```
src/
├── agents/                      # Subagent 実装（SDK層）
│   ├── base_agent.py           # 基底クラス (SingleResponsibility)
│   ├── coordinator.py          # Coordinator logic
│   ├── researcher.py           # Researcher impl
│   ├── implementer.py          # Implementer impl
│   └── reviewer.py             # Reviewer impl
│
├── orchestration/              # Orchestration ロジック
│   ├── subagent_runner.py      # Subagent 実行エンジン
│   ├── result_aggregator.py    # 結果統合・競合解決
│   ├── task_scheduler.py       # 並列スケジュール管理
│   └── error_handler.py        # エラーハンドリング
│
├── skills/                      # Skill 実装
│   ├── web_search.py           # Web検索スキル
│   ├── code_review.py          # コード審査スキル
│   └── mcp_integration.py      # MCP 接続スキル
│
├── mcp/                        # MCP サーバ接続
│   ├── github_client.py        # GitHub MCP
│   ├── slack_client.py         # Slack MCP
│   └── mcp_router.py           # MCP ルーター
│
└── core/                       # 共通ライブラリ
    ├── prompt_cache.py         # Caching ロジック
    ├── context_manager.py      # Context 管理
    ├── logger.py               # ログ管理
    └── metrics.py              # メトリクス収集
```

**責務分離の原則**:
- **agents/**: ペルソナ・思考ロジック定義のみ
- **orchestration/**: Subagent 間の調整・並列実行管理
- **skills/**: 再利用可能なワークフロー
- **mcp/**: 外部システム接続・認証
- **core/**: インフラストラクチャ・テクニカル

---

## 5. パフォーマンス最適化パターン

### 5.1 Caching の最適化

```python
# ✅ GOOD: 不変プレフィックスを先に配置
system_instructions = """
[SYSTEM PROMPT - 3000 tokens, IMMUTABLE]
You are Coordinator Agent...
[Tool Definitions - 2500 tokens, IMMUTABLE]
Subagent Interface...
"""  # ← 固定

knowledge_base = """
[KNOWLEDGE BASE - 2000 tokens, LONG_LIVED]
Patterns and Examples...
"""  # ← 月1更新

user_query = f"""
[USER QUERY - variable]
{user_input}
[Execution Time: {now}]
"""  # ← 毎回変わる

full_prompt = system_instructions + knowledge_base + user_query
```

**効果**:
- Cache Hit Rate: 70%+ 達成
- Input Token Cost: 80% 削減
- Time-to-first-token: 100ms 以下

### 5.2 Context Budget 管理

```python
# 1M context を最適配分
TOTAL_CONTEXT = 1_000_000

# Subagent 配分
COORDINATOR_BUDGET = 300_000      # 30%
RESEARCHER_BUDGET = 250_000       # 25%
IMPLEMENTER_BUDGET = 250_000      # 25%
REVIEWER_BUDGET = 250_000         # 25%
BUFFER = 200_000                  # 20% (Safety buffer)

# 実行時監視
def check_context_health():
    if used_tokens > BUDGET * 0.9:  # 90%以上使用
        logger.warning(f"Context pressure: {used_tokens}/{BUDGET}")
        # 対応: タスク分割 / Subagent 呼び出し中止
```

### 5.3 並列度の制御

```python
# 3並列を超えない（Context 制約）
MAX_CONCURRENT_SUBAGENTS = 3

async def run_subagents_parallel(tasks: list[Task]) -> list[Result]:
    """Ensure max 3 concurrent, queue others."""
    semaphore = asyncio.Semaphore(MAX_CONCURRENT_SUBAGENTS)
    
    async def bounded_task(task):
        async with semaphore:
            return await execute_subagent(task)
    
    return await asyncio.gather(*[bounded_task(t) for t in tasks])
```

---

## 6. テスト設計

### テストピラミッド

```
        ▲
       /|\
      / | \    E2E (5%)
     /  |  \   - MCP接続テスト
    /   |   \  - 全Subagent統合
   /----|----\
  /     |     \ Integration (20%)
 /      |      \ - Subagent並列実行
/-------|-------\- Orchestration フロー
       Unit (75%)
    - Subagent個別
    - Skill 個別
    - Cache Hit Rate
```

### テスト例（Python）

```python
# tests/caching/test_prompt_cache_hit_rate.py
def test_cache_hit_rate_above_60_percent():
    """Verify prompt caching achieves ≥60% hit rate."""
    cache_manager = PromptCacheManager()
    
    # 同じプレフィックスを10回実行
    for i in range(10):
        result = call_claude_api(
            system_instructions=IMMUTABLE_PROMPT,
            user_query=f"Query {i}"
        )
    
    hit_rate = cache_manager.get_hit_rate()
    assert hit_rate >= 0.60, f"Cache hit rate {hit_rate} < 0.60"
    
    # Metadata も検証
    assert result.metadata['cache_hit_rate'] >= 0.60
    assert result.metadata['input_tokens_cached'] > 0
```

---

## 7. 監視・ダッシュボード

### キー メトリクス

| メトリクス | 目標値 | 測定方法 | Alert閾値 |
|---|---|---|---|
| キャッシュヒット率 | 70%+ | CI/CD test run | <60% |
| Subagent応答時間 | <5s | Execution log | >10s |
| Context使用率 | <90% | Runtime monitor | >95% |
| テストカバレッジ | ≥80% | pytest --cov | <75% |
| エラー率 | <1% | CloudWatch | >5% |

### ダッシュボード実装例

```yaml
# .github/workflows/metrics-dashboard.yml
on: [push, workflow_dispatch]
jobs:
  metrics:
    runs-on: ubuntu-latest
    steps:
      - name: Measure Cache Hit Rate
        run: |
          pytest tests/caching/ --json=metrics.json
      - name: Push to Datadog
        env:
          DATADOG_API_KEY: ${{ secrets.DATADOG_API_KEY }}
        run: |
          curl -X POST "https://api.datadoghq.com/api/v1/series" \
            -H "DD-API-KEY: ${DATADOG_API_KEY}" \
            -d @metrics.json
```

---

## チェックリスト（設計レビュー）

- [ ] SOLID 原則が適用されているか（特にSingleResponsibility）
- [ ] DRY vs YAGNI のバランスが取れているか
- [ ] Subagent インターフェースが統一されているか
- [ ] Prompt Caching レイアリングが最適化されているか
- [ ] 並列度が3以下に制限されているか
- [ ] エラーハンドリングが Debugger を自動で呼び出しているか
- [ ] テストカバレッジが ≥80% か
- [ ] キャッシュヒット率が ≥60% か（CI/CDで監視）

---

**次回更新**: 2026-06-12（実装フィードバック反映）
