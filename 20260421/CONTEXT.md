# CONTEXT.md - Token Optimization & Context Management Strategy

**Generated**: April 21, 2026  
**Framework**: AI Agent Context Engineering  
**Focus**: 60-80% cost reduction without quality loss

## 📊 The Token Economy in 2026

### 現状: コスト構造の危機

```
Average Project Costs:
  • Initial implementation: $500-2000 (5M-20M tokens)
  • Monthly maintenance: $200-1000 (2M-10M tokens)
  • Problem: 60-70% is wasted on poor context management

Root Causes:
  1. 不要なファイル全文を毎回送信
  2. 会話履歴が蓄積し続ける（quadratic growth）
  3. キャッシング戦略なし
  4. モデルルーティング未実装
  5. RAG 統合なし
```

**衝撃**: TTL 短縮（60m → 5m）により、2026年初頭で 30-60% のコスト増加

### 解決策: Context Engineering

```
定義: 各 LLM コール時に「最適な情報セット」をデリバリーすること
      → 不要な情報は完全に除外
      → 必要な情報は完全に包含
```

**実績**: 正しい Context Engineering で 60-80% コスト削減可能（品質維持）

## 🎯 Token 予算管理

### 月額予算配分（月 500K tokens 想定）

```
Planning Phase (15%)        = 75K tokens
  • Task decomposition
  • Architecture design
  • Risk assessment

Implementation Phase (60%)  = 300K tokens
  • Code generation
  • Testing
  • Integration

Review Phase (20%)          = 100K tokens
  • QA/validation
  • Performance tuning
  • Security audit

Reserve (5%)                = 25K tokens
  • Unexpected issues
  • Edge cases
  • Rework
```

### 使用量トラッキング

```python
# リアルタイムトラッキング
class TokenBudgetTracker:
    def __init__(self, monthly_limit=500_000):
        self.monthly_limit = monthly_limit
        self.used = {}  # phase -> token count
    
    def log_usage(self, phase: str, prompt_tokens: int, completion_tokens: int):
        cache_read_tokens = response.get("cache_read_input_tokens", 0)
        total = prompt_tokens + completion_tokens - (cache_read_tokens * 0.9)
        # キャッシュは 90% 割引
        
        if phase not in self.used:
            self.used[phase] = 0
        self.used[phase] += total
        
        remaining = self.monthly_limit - sum(self.used.values())
        if remaining < 50_000:
            alert_user(f"Warning: {remaining} tokens remaining")
    
    def report(self):
        return {
            "used_by_phase": self.used,
            "total_used": sum(self.used.values()),
            "remaining": self.monthly_limit - sum(self.used.values()),
            "cache_savings": self._estimate_cache_savings()
        }
```

## 💾 Prompt Caching Strategy（April 2026 版）

### TTL 変更への対応

```
Old Config (2025年末):
  cache_control:
    type: "ephemeral"
    ttl: 3600  # 60 minutes

New Config (2026年4月以降):
  cache_control:
    type: "ephemeral"
    ttl: 300  # 5 minutes (default)
  # 対応策: 明示的に 1h TTL を要求
  ttl_preference: "1h"  # Claude Haiku/Sonnet/Opus でサポート
```

### キャッシュ対象の設計

```
Best Candidates for Caching (90% discount):

1. System Instructions (8K tokens)
   ✅ Cache at top level
   ✅ Stable across all calls
   ✅ Reused 10-100 times
   Savings: 7.2K tokens per reuse

2. Project Context (20K tokens)
   ├── Architecture overview
   ├── Tech stack specification
   ├── Code style guide
   └── Security guidelines
   ✅ Cache when stable
   ✅ Reused per session
   Savings: 18K tokens per session

3. Code Libraries (15K tokens)
   ├── Utility functions
   ├── Common patterns
   └── Shared types
   ✅ Cache per module
   ✅ Rarely changes
   Savings: 13.5K per module

Bad Candidates (do NOT cache):

1. Conversation History
   ❌ Changes every turn
   ❌ Small overhead
   ❌ Caching ineffective

2. API Responses
   ❌ Time-sensitive
   ❌ Often different
   ❌ Not idempotent

3. User Input
   ❌ Unique per request
   ❌ High cardinality
   ❌ Cache pollution risk
```

### 実装例: 3層キャッシュ

```python
from anthropic import Anthropic

client = Anthropic()

class CachedContextAgent:
    def __init__(self):
        # L1: System prompt with cache
        self.system = {
            "type": "text",
            "text": COMPREHENSIVE_INSTRUCTIONS,
            "cache_control": {
                "type": "ephemeral",
                "ttl": "1h"  # 明示的に 1h 指定
            }
        }
    
    def call_with_project_context(self, user_query: str, project_files: list[str]):
        # L2: Project context (stable for session)
        project_context = {
            "type": "text",
            "text": f"PROJECT CONTEXT:\n{load_project_summary()}",
            "cache_control": {"type": "ephemeral"}
        }
        
        # L3: Conversation history (not cached, recent only)
        conversation = recent_messages[-5:]  # Last 5 only
        
        # API call with caching
        response = client.messages.create(
            model="claude-opus-4-7",
            max_tokens=1000,
            system=[
                self.system,  # キャッシュ対象
                project_context  # キャッシュ対象
            ],
            messages=[
                *conversation,  # キャッシュ対象外
                {"role": "user", "content": user_query}
            ]
        )
        
        # キャッシュ効率を報告
        cache_creation = response.usage.cache_creation_input_tokens
        cache_read = response.usage.cache_read_input_tokens
        print(f"Cache: Created {cache_creation} tokens, Read {cache_read} tokens")
        # Cache read は 90% 割引
        
        return response.content[0].text
```

## 🎯 Model Routing Strategy

### Cost-Performance Matrix

```
Task Type              Recommended Model    Cost    Quality
─────────────────────  ─────────────────    ─────   ────────
Planning/Analysis      Haiku 4.5            $1      80%
Implementation         Sonnet 4.6           $5      95%
Complex Reasoning      Opus 4.7             $15     99%
Code Review            Sonnet 4.6           $5      95%
Security Audit         Opus 4.7             $15     99%
Simple Tasks           Haiku 4.5            $1      75%
```

**実装 (Router Pattern)**:

```python
class ModelRouter:
    ROUTING_RULES = {
        "complexity:low": {
            "model": "claude-haiku-4-5-20251001",
            "max_tokens": 500,
            "cost_per_call": "$0.02"
        },
        "complexity:medium": {
            "model": "claude-sonnet-4-6-20250514",
            "max_tokens": 2000,
            "cost_per_call": "$0.10"
        },
        "complexity:high": {
            "model": "claude-opus-4-7-20250121",
            "max_tokens": 4000,
            "cost_per_call": "$0.30"
        }
    }
    
    def select_model(self, task_description: str) -> str:
        complexity = self.analyze_complexity(task_description)
        return self.ROUTING_RULES[f"complexity:{complexity}"]["model"]
    
    def analyze_complexity(self, task: str) -> str:
        """Determine task complexity without API call"""
        factors = {
            "num_files": count_files_mentioned(task),
            "has_architecture": "architecture" in task.lower(),
            "has_security": "security" in task.lower(),
            "num_languages": count_languages(task),
            "chain_of_thought_needed": len(task) > 1000
        }
        
        score = sum([
            factors["num_files"] * 0.1,
            factors["has_architecture"] * 0.3,
            factors["has_security"] * 0.4,
            factors["num_languages"] * 0.2,
            factors["chain_of_thought_needed"] * 0.5
        ])
        
        if score > 2.0:
            return "high"
        elif score > 1.0:
            return "medium"
        else:
            return "low"
```

## 📚 RAG Integration

### Problem: Full Context Inclusion

```
❌ 不効率な手法:
  Task: "Fix bug in auth.js"
  → System sends ENTIRE auth module (500 lines)
  → API processes 500 lines of code
  → Model reads 300 irrelevant lines
  → Cost: 500 tokens wasted on irrelevant code

✅ 効率的な手法 (RAG):
  Task: "Fix bug in auth.js"
  → System identifies 5 relevant functions (80 lines)
  → API processes 80 lines
  → Model reads 80 lines (all relevant)
  → Cost savings: 84%
```

### RAG実装パターン

```python
from anthropic import Anthropic
import embeddings  # Vector DB for similarity search

class RAGContext:
    def __init__(self, codebase_path):
        self.codebase = codebase_path
        self.embeddings = embeddings.load_embeddings(codebase_path)
    
    def retrieve_relevant_code(self, query: str, k: int = 5) -> str:
        """Retrieve only relevant code snippets"""
        relevant_chunks = self.embeddings.search(query, k=k)
        
        # Combine in logical order
        context = "RELEVANT CODE SECTIONS:\n\n"
        for i, chunk in enumerate(relevant_chunks, 1):
            context += f"[Snippet {i}] File: {chunk.file_path}\n"
            context += f"{chunk.code}\n\n"
        
        return context
    
    def create_message_with_rag(self, user_query: str):
        # Standard RAG flow
        rag_context = self.retrieve_relevant_code(user_query)
        
        client = Anthropic()
        response = client.messages.create(
            model="claude-sonnet-4-6-20250514",
            max_tokens=1000,
            system="You are a code expert. Use only the provided context.",
            messages=[
                {
                    "role": "user",
                    "content": f"{rag_context}\n\nQuestion: {user_query}"
                }
            ]
        )
        
        return response
```

## 🔄 Conversation History Management

### Problem: Quadratic Token Growth

```
Turn  Tokens (Bad)   Tokens (Good)
────  ────────────   ───────────
1     500            500
2     600            520
3     700            540
4     800            560
5     900            580
...
20    2400           700

Bad Strategy: Keep all history
  Total cost for 20 turns: 21,000 tokens

Good Strategy: Keep last 5-10 only
  Total cost for 20 turns: 9,000 tokens
  Savings: 57%
```

### 実装: Sliding Window History

```python
class ContextualMemory:
    def __init__(self, window_size: int = 10, max_size_tokens: int = 20000):
        self.window_size = window_size
        self.max_size_tokens = max_size_tokens
        self.history = []
    
    def add_turn(self, role: str, content: str):
        """Add new turn and prune if needed"""
        self.history.append({
            "role": role,
            "content": content
        })
        
        # Strategy 1: Keep only recent N turns
        if len(self.history) > self.window_size:
            self.history = self.history[-self.window_size:]
        
        # Strategy 2: Summarize old turns if token limit reached
        total_tokens = self._estimate_tokens()
        if total_tokens > self.max_size_tokens:
            self._compress_older_turns()
    
    def _compress_older_turns(self):
        """Summarize older turns into a single summary"""
        if len(self.history) > self.window_size:
            old_turns = self.history[:-self.window_size]
            summary = self._summarize_turns(old_turns)
            
            self.history = [
                {
                    "role": "system",
                    "content": f"PREVIOUS CONTEXT SUMMARY:\n{summary}"
                },
                *self.history[-self.window_size:]
            ]
    
    def get_history_for_api(self) -> list:
        """Return optimized history for API call"""
        return self.history
    
    def _estimate_tokens(self) -> int:
        """Rough estimate: 1 token per 4 characters"""
        total_chars = sum(len(m["content"]) for m in self.history)
        return total_chars // 4
```

## 📈 Cost Breakdown & Savings

### Before Optimization (Monthly)

```
Scenario: 50 AI tasks, 10K tokens per task average

Costs:
  Planning (5 tasks × 20K tokens):    100K tokens @ $1.50 = $150
  Implementation (30 tasks × 50K):    1500K tokens @ $5 = $7,500
  Review (15 tasks × 10K):            150K tokens @ $3 = $450
  
  Total: 1,750K tokens = $8,100/month
  Wasted (estimated 60%): $4,860/month
```

### After Optimization (Monthly)

```
Same scenario with optimization:

  Planning (Haiku): 50K tokens @ $0.80 = $40
  Implementation (Sonnet + caching): 400K tokens @ $2 = $800
  Review (Sonnet): 100K tokens @ $2 = $200
  
  Total: 550K tokens = $1,040/month
  Savings: 87% reduction
```

**Optimization Techniques Applied:**
1. Model routing (Haiku for planning) → 60% cost reduction
2. Prompt caching (1h TTL on system + context) → 75% cache hit rate = 50% savings
3. RAG implementation → 40% context reduction
4. History pruning → 30% conversation cost reduction
5. Batch API for async tasks → 50% discount

## 🛠️ Implementation Checklist

実装時の確認項目:

```yaml
phase_1_setup:
  - [ ] TTL を明示的に "1h" に設定
  - [ ] System prompt をキャッシュ対象に
  - [ ] Project context のサマリー作成
  - [ ] Model routing rules 定義
  
phase_2_integration:
  - [ ] RAG/検索機能導入
  - [ ] History sliding window 実装
  - [ ] Token tracking 導入
  - [ ] Cost monitoring ダッシュボード作成
  
phase_3_optimization:
  - [ ] Cache hit rate 測定 (Goal: >75%)
  - [ ] Model routing 効果測定
  - [ ] Token savings 検証 (Goal: 60-80%)
  - [ ] パフォーマンス劣化チェック
```

---

**最終更新**: 2026年04月21日  
**監査频率**: 月1回（キャッシュ効率、トークン使用量）  
**目標**: 60-80% コスト削減（品質維持）
