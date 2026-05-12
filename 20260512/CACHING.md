# CACHING.md - Prompt Caching 実装ガイド

**対象**: Prompt Caching 戦略、キャッシュ管理、コスト最適化  
**作成日**: 2026-05-12  
**基準**: Claude API Prompt Caching（90%コスト削減、80%遅延削減）

---

## 1. Prompt Caching とは

### 基本概念

**Prompt Caching** = API リクエストの静的プレフィックスをキャッシュし、再利用時の入力トークン数とレイテンシを削減

```
リクエスト1
├─ Static Prefix (Cache): システム命令・ツール定義（3000 tokens）
├─ Static Prefix (Cache): API スキーマ（2500 tokens）
└─ Variable: User Query（100 tokens）
   [API Result: キャッシュ作成、入力トークン = 5600]

リクエスト2 (同じプレフィックス)
├─ Static Prefix (Cache REUSED): システム命令・ツール定義（0 tokens）
├─ Static Prefix (Cache REUSED): API スキーマ（0 tokens）
└─ Variable: User Query（100 tokens）
   [API Result: キャッシュヒット、入力トークン = 100 + キャッシュ参照料]
   
→ **効果**: 5600 → 1200 tokens (78% 削減)
```

### 数値目標（Anthropic公式）

| 指標 | 達成値 | 備考 |
|---|---|---|
| **入力トークンコスト削減** | 最大 90% | キャッシュヒット率80%の場合 |
| **Time-to-first-token 削減** | 最大 80% | キャッシュプリフィックス有効時 |
| **キャッシュヒット率** | 60%+ | 実運用での目安 |
| **最小キャッシュサイズ** | 1024 tokens | 未満はキャッシュ対象外 |

---

## 2. Harness Engineering における Caching 戦略

### 2.1 3層キャッシング構造

```
Layer 1: System Prompt & Agent Definition (IMMUTABLE, 毎回ヒット)
├─ .claude/CLAUDE.md 要約（800行 → 3000 tokens）
├─ .claude/AGENT.md ペルソナ定義（2000 tokens）
├─ .claude/DESIGN.md パターン（1500 tokens）
├─ 合計: ~6500 tokens
└─ キャッシュヒット率: 95%+ (毎回同じ)

Layer 2: Tool Definition & API Schema (LONG_LIVED, 月1更新)
├─ Subagent インターフェース定義（2500 tokens）
├─ MCP Server スキーマ（1500 tokens）
├─ OpenAPI / GraphQL 仕様（1000 tokens）
├─ 合計: ~5000 tokens
└─ キャッシュヒット率: 80%+ (週単位で変わらない)

Layer 3: Knowledge Base & Examples (MEDIUM_LIVED, 月1-2回更新)
├─ パターン集（examples/*.py, *.ts）（2000 tokens）
├─ トラブルシューティング集（1500 tokens）
├─ パフォーマンス最適化ガイド（1000 tokens）
├─ 合計: ~4500 tokens
└─ キャッシュヒット率: 60%+ (徐々に更新)

[Variable Section] User Query & Session Context (毎回異なる)
└─ ユーザー指示（100-500 tokens）
└─ キャッシュなし
```

**全体キャッシュヒット率の計算**:
```
Hit Rate = (Cached Tokens) / (Total Input Tokens)
         = (6500 + 5000 + 4500) / (6500 + 5000 + 4500 + 300)
         = 16000 / 16300
         ≈ 98%（理想値）

実運用: 60-70% (Layer 3 の更新リスク、キャッシュキー管理の複雑さを考慮)
```

### 2.2 実装例（Python SDK）

```python
# src/core/prompt_cache.py
from anthropic import Anthropic

class PromptCacheManager:
    def __init__(self):
        self.client = Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
        self.cache_config = self._load_cache_config()
    
    def _load_cache_config(self) -> dict:
        """Load cache configuration from config/cache-config.yaml"""
        with open("config/cache-config.yaml") as f:
            return yaml.safe_load(f)
    
    def build_cached_prompt(self, user_query: str) -> tuple[str, dict]:
        """
        Build prompt with cache-optimized structure.
        
        Returns:
            (full_prompt, cache_metadata)
        """
        
        # Layer 1: System Instructions (IMMUTABLE)
        system_prompt = self._build_system_prompt()
        cache_control_layer1 = {"type": "ephemeral"}  # キャッシュ制御ポイント
        
        # Layer 2: Tool Definitions (LONG_LIVED)
        tool_definitions = self._build_tool_definitions()
        cache_control_layer2 = {"type": "ephemeral"}
        
        # Layer 3: Knowledge Base (MEDIUM_LIVED)
        knowledge_base = self._build_knowledge_base()
        cache_control_layer3 = {"type": "ephemeral"}
        
        # [Variable] User Query
        variable_section = f"""

## User Request
{user_query}

[Execution Time: {datetime.now().isoformat()}]
"""
        
        full_prompt = (
            system_prompt +
            tool_definitions +
            knowledge_base +
            variable_section
        )
        
        cache_metadata = {
            "layer1_tokens": len(system_prompt.split()),
            "layer2_tokens": len(tool_definitions.split()),
            "layer3_tokens": len(knowledge_base.split()),
            "variable_tokens": len(variable_section.split()),
            "total_cached_tokens": (
                len(system_prompt.split()) +
                len(tool_definitions.split()) +
                len(knowledge_base.split())
            )
        }
        
        return full_prompt, cache_metadata
    
    def call_claude_with_caching(self, user_query: str) -> dict:
        """
        Call Claude API with prompt caching enabled.
        """
        full_prompt, cache_metadata = self.build_cached_prompt(user_query)
        
        response = self.client.messages.create(
            model="claude-opus-4-7",
            max_tokens=4096,
            system=full_prompt,  # Prefix caching automatically enabled
            messages=[
                {"role": "user", "content": user_query}
            ]
        )
        
        # キャッシュ統計を抽出
        cache_stats = {
            "cache_creation_input_tokens": response.usage.cache_creation_input_tokens,
            "cache_read_input_tokens": response.usage.cache_read_input_tokens,
            "input_tokens": response.usage.input_tokens,
            "output_tokens": response.usage.output_tokens,
        }
        
        # キャッシュヒット率の計算
        cache_hit_rate = (
            cache_stats["cache_read_input_tokens"] /
            (cache_stats["cache_read_input_tokens"] + cache_stats["input_tokens"])
            if (cache_stats["cache_read_input_tokens"] + cache_stats["input_tokens"]) > 0
            else 0
        )
        
        return {
            "content": response.content[0].text,
            "cache_stats": cache_stats,
            "cache_hit_rate": cache_hit_rate,
            "metadata": cache_metadata
        }
    
    def _build_system_prompt(self) -> str:
        """Load and build Layer 1: System Prompt (IMMUTABLE)"""
        system_parts = []
        
        # CLAUDE.md 要約
        with open(".claude/CLAUDE.md") as f:
            claude_md = f.read()
            system_parts.append(f"## Project Rules\n{claude_md[:2000]}...")  # 要約
        
        # AGENT.md ペルソナ
        with open(".claude/AGENT.md") as f:
            agent_md = f.read()
            system_parts.append(f"## Agent Personas\n{agent_md[:1500]}...")
        
        # DESIGN.md パターン
        with open(".claude/DESIGN.md") as f:
            design_md = f.read()
            system_parts.append(f"## Design Patterns\n{design_md[:1000]}...")
        
        return "\n\n".join(system_parts)
    
    def _build_tool_definitions(self) -> str:
        """Load and build Layer 2: Tool Definitions (LONG_LIVED)"""
        # Subagent インターフェース定義
        return """
## Subagent Interfaces
[Researcher interface...]
[Implementer interface...]
[Reviewer interface...]

## MCP Server Schemas
[GitHub MCP schema...]
[Slack MCP schema...]
"""
    
    def _build_knowledge_base(self) -> str:
        """Load and build Layer 3: Knowledge Base (MEDIUM_LIVED)"""
        kb_parts = []
        
        # examples/ ディレクトリ内のファイルを読み込み
        for example_file in Path("examples/").glob("*.py"):
            with open(example_file) as f:
                kb_parts.append(f"### {example_file.name}\n{f.read()[:500]}...")
        
        return "\n\n".join(kb_parts)
```

---

## 3. キャッシュ無効化の落とし穴（⚠️ 重要）

### 3.1 タイムスタンプ挿入による落とし穴

```python
# ❌ 悪い例: タイムスタンプが early に挿入される
def build_bad_prompt(user_query: str) -> str:
    system_prompt = f"""
[System Instructions]
You are a helpful assistant.

[Current Timestamp: {datetime.now()}]  # ← 毎回異なる！
[Today's Date: {date.today()}]

[Rest of prompt...]
"""
    # 問題: システムプレフィックスが毎回異なるため、キャッシュヒット率 = 0%
    # 期待値: 90% 削減 → 実現値: 0% (フルコスト)
    return system_prompt

# ✅ 良い例: タイムスタンプを末尾に移動
def build_good_prompt(user_query: str) -> str:
    system_prompt = """
[System Instructions - IMMUTABLE]
You are a helpful assistant.

[Tool Definitions - IMMUTABLE]
...
"""
    
    variable_section = f"""

## Current Context
[Timestamp: {datetime.now()}]
[Date: {date.today()}]

## User Query
{user_query}
"""
    
    # system_prompt (不変) はキャッシュ有効
    # variable_section (可変) のみ毎回新規処理
    # 期待値: 85% 削減を達成
    return system_prompt + variable_section
```

### 3.2 ツール定義の順序固定化

```python
# ❌ 悪い例: ツール定義の順序が毎回異なる
def build_bad_tools() -> str:
    tools = {
        "web_search": {...},
        "code_edit": {...},
        "file_create": {...},
    }
    # dict の順序は実装に依存、プレフィックスが毎回変わる可能性
    return str(tools)

# ✅ 良い例: 順序を固定化
def build_good_tools() -> str:
    tools_in_order = [
        ("code_edit", {...}),
        ("file_create", {...}),
        ("web_search", {...}),  # アルファベット順に固定
    ]
    # 毎回同じ順序で生成 → キャッシュ有効
    return "\n".join(f"{name}: {spec}" for name, spec in tools_in_order)
```

### 3.3 UUIDs・乱数の挿入

```python
# ❌ 悪い例: UUIDs をプレフィックスに含める
def build_bad_context() -> str:
    session_id = str(uuid.uuid4())  # 毎回異なる
    return f"""
[Session ID: {session_id}]
[Rest of prompt...]
"""

# ✅ 良い例: UUIDs は可変セクションに
def build_good_context() -> str:
    immutable_prefix = """
[System Instructions]
...
"""
    
    variable_section = f"""
[Session ID: {str(uuid.uuid4())}]
[User Query: ...]
"""
    
    return immutable_prefix + variable_section
```

---

## 4. CI/CD での キャッシュヒット率監視

### 4.1 テスト実装例

```python
# tests/caching/test_cache_hit_rate.py
import pytest
from src.core.prompt_cache import PromptCacheManager

class TestPromptCachingHitRate:
    @pytest.fixture
    def cache_manager(self):
        return PromptCacheManager()
    
    def test_cache_hit_rate_above_60_percent(self, cache_manager):
        """
        Verify prompt caching achieves ≥60% hit rate
        (industry benchmark from Anthropic).
        """
        hit_rates = []
        
        # 同じプレフィックスで10回リクエスト
        for i in range(10):
            response = cache_manager.call_claude_with_caching(
                user_query=f"Task {i}: Explain Python list comprehension"
            )
            hit_rates.append(response['cache_hit_rate'])
        
        average_hit_rate = sum(hit_rates) / len(hit_rates)
        assert average_hit_rate >= 0.60, (
            f"Cache hit rate {average_hit_rate:.1%} < 60%\n"
            f"Per-request rates: {[f'{r:.1%}' for r in hit_rates]}"
        )
    
    def test_cache_reduces_input_cost_by_80_percent(self, cache_manager):
        """Verify 80% cost reduction vs. non-cached baseline."""
        
        # キャッシュなし (baseline)
        baseline_cost = 5600  # tokens (estimated)
        
        # キャッシュあり (actual)
        response = cache_manager.call_claude_with_caching(
            user_query="Explain Prompt Caching"
        )
        
        actual_input_cost = (
            response['cache_stats']['cache_read_input_tokens'] +
            response['cache_stats']['input_tokens']
        )
        
        # Prompt Caching の削減率を計算
        # 参考: 長いプレフィックスはキャッシュ参照料が安い
        cost_reduction = (baseline_cost - actual_input_cost) / baseline_cost
        
        assert cost_reduction >= 0.80, (
            f"Cost reduction {cost_reduction:.1%} < 80%\n"
            f"Baseline: {baseline_cost}, Actual: {actual_input_cost}"
        )
    
    def test_no_timestamp_in_immutable_prefix(self, cache_manager):
        """Verify no timestamps corrupt the cache prefix."""
        
        prompt1, metadata1 = cache_manager.build_cached_prompt(
            "Task 1"
        )
        prompt2, metadata2 = cache_manager.build_cached_prompt(
            "Task 2"
        )
        
        # 不変部分（Layer 1-3）が完全に一致
        cached_tokens_1 = metadata1["total_cached_tokens"]
        cached_tokens_2 = metadata2["total_cached_tokens"]
        
        assert cached_tokens_1 == cached_tokens_2, (
            f"Cached prefix differs: {cached_tokens_1} vs {cached_tokens_2}\n"
            "Likely cause: timestamp or UUID in prefix"
        )
    
    @pytest.mark.benchmark
    def test_latency_reduction_80_percent(self, cache_manager):
        """Measure time-to-first-token reduction."""
        import time
        
        # 1回目（キャッシュ作成）
        start = time.time()
        cache_manager.call_claude_with_caching("Query 1")
        time_with_cache_creation = time.time() - start
        
        # 2回目（キャッシュヒット）
        start = time.time()
        cache_manager.call_claude_with_caching("Query 2")
        time_with_cache_hit = time.time() - start
        
        # Latency 削減率
        latency_reduction = (
            (time_with_cache_creation - time_with_cache_hit) /
            time_with_cache_creation
        )
        
        assert latency_reduction >= 0.60, (  # 60% 削減は確実に達成
            f"Latency reduction {latency_reduction:.1%} < 60%"
        )
```

### 4.2 CI/CD パイプライン統合

```yaml
# .github/workflows/caching-validation.yml
name: Caching Validation

on: [push, pull_request]

jobs:
  caching_tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-benchmark
      
      - name: Run Caching Tests
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          pytest tests/caching/ -v \
            --benchmark-only \
            --json=caching-results.json
      
      - name: Check Cache Hit Rate
        run: |
          python scripts/validate_cache_hit_rate.py \
            --input=caching-results.json \
            --threshold=0.60
      
      - name: Upload to Dashboard
        if: success()
        run: |
          python scripts/upload_metrics.py \
            --metrics=caching-results.json \
            --datadog-api-key=${{ secrets.DATADOG_API_KEY }}
      
      - name: Fail if cache hit rate < 60%
        run: |
          python -c "
          import json
          with open('caching-results.json') as f:
              data = json.load(f)
              hit_rate = data.get('cache_hit_rate', 0)
              if hit_rate < 0.60:
                  exit(1)
          "
```

---

## 5. トラブルシューティング

### 5.1 キャッシュヒット率が低い場合

```
症状: Cache hit rate が 60% 未満

原因チェックリスト:
1. [ ] タイムスタンプが prefix に含まれているか
   → 修正: 可変セクションに移動
2. [ ] UUIDs / ランダム値が prefix に含まれているか
   → 修正: 可変セクションに移動
3. [ ] ツール定義の順序が毎回異なるか
   → 修正: アルファベット順に固定化
4. [ ] .claude/*.md ファイルが頻繁に更新されているか
   → 修正: 月1回のスケジュール更新に統一
5. [ ] キャッシュキー管理が不適切か
   → 修正: cache-config.yaml で bucket strategy を見直す

デバッグ方法:
$ python -c "
from src.core.prompt_cache import PromptCacheManager
mgr = PromptCacheManager()
r1, m1 = mgr.build_cached_prompt('Task 1')
r2, m2 = mgr.build_cached_prompt('Task 2')

print('Prefix 1 length:', len(r1))
print('Prefix 2 length:', len(r2))
print('Match:', r1[:5000] == r2[:5000])  # 最初の5000文字が一致?
"
```

### 5.2 キャッシュサイズが小さすぎる場合

```
症状: キャッシュが作成されない

原因: プレフィックスサイズが 1024 tokens未満

修正:
- Layer 1 (System Prompt) を拡張
  追加内容: API 仕様、パターン例、ガイドライン
- Layer 2 (Tool Definitions) を拡張
  追加内容: Subagent仕様、MCP スキーマ詳細
```

---

## 6. パフォーマンス目標表

| 指標 | 目標値 | 現状 | ギャップ |
|---|---|---|---|
| キャッシュヒット率 | ≥ 60% | 測定中 | — |
| 入力コスト削減 | ≥ 80% | 測定中 | — |
| Time-to-first-token削減 | ≥ 60% | 測定中 | — |
| プレフィックスサイズ | 16K tokens | 測定中 | — |
| キャッシュ参照料 | 10% input cost | 契約値 | — |

---

## 7. チェックリスト（Caching導入完了）

- [ ] `src/core/prompt_cache.py` 実装完了
- [ ] 3層キャッシング構造が設計・実装済み
- [ ] Layer 1 (システム命令) が完全に不変
- [ ] Layer 2 (ツール定義) が決定的順序で生成
- [ ] Layer 3 (知識ベース) が月1回の更新スケジュールで運用
- [ ] タイムスタンプ・UUID が可変セクションに配置
- [ ] キャッシュヒット率テストが pytest に統合済み
- [ ] CI/CD で自動測定・ダッシュボード監視が設定済み
- [ ] 目標値（60%+）に到達

---

**参考**:
- [Prompt caching - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [Prompt Caching for AI Agents - Medium (April 2026)](https://medium.com/@arvisionlab/prompt-caching-for-ai-agents-how-to-cut-cost-and-latency-without-breaking-context-245dc2502b4b)
