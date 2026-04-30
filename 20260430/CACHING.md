# CACHING.md - Prompt Caching Strategy & Optimization

**Version**: 1.0 | **Last Updated**: 2026-04-30

---

## 1. Prompt Caching Fundamentals

### What is Prompt Caching?

Prompt caching reduces costs by ~70% for repeated inputs. Instead of processing the same context repeatedly, Claude caches the embedding and reuses it.

**Cost structure:**
```
Cache Write (1st request):       1.25x base input tokens (5-min TTL)
                                 2.0x base input tokens (1-hour TTL)

Cache Read (subsequent requests): 0.1x base input tokens

Example (Claude 3.5 Sonnet):
- Input token: $0.003 per 1K
- Cache write (5-min): $0.00375 per 1K
- Cache read: $0.0003 per 1K (90% savings!)
```

**When to use:**
- Analyzing large documents repeatedly
- Running the same analysis on multiple files
- System prompts + repeated user queries
- API responses you'll reference multiple times

---

## 2. Cache Control Placement

### YAML/JSON Frontmatter Caching

**Pattern**: Place `cache_control` at the start of structured data

```json
{
  "cache_control": { "type": "ephemeral" },
  "system_prompt": "You are a code reviewer...",
  "codebase_structure": { ... },
  "guidelines": [ ... ]
}
```

**Implementation in Claude API:**

```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
  model="claude-3-5-sonnet-20241022",
  max_tokens=2048,
  system=[
    {
      "type": "text",
      "text": "You are a code reviewer...",
      "cache_control": {"type": "ephemeral"}  # ← Cache this system prompt
    }
  ],
  messages=[
    {
      "role": "user",
      "content": [
        {
          "type": "text",
          "text": "[Large codebase context]",
          "cache_control": {"type": "ephemeral"}  # ← Cache context too
        },
        {
          "type": "text",
          "text": "Review function X"
        }
      ]
    }
  ]
)
```

### TTL Options

**5-minute cache (default, cheaper to write):**
```json
{ "cache_control": { "type": "ephemeral" } }
```

**1-hour cache (costs more to write, better for long tasks):**
```json
{ "cache_control": { "type": "ephemeral", "keep_alive": true } }
```

**Decision matrix:**
| Scenario | TTL | Reasoning |
|----------|-----|-----------|
| Multiple quick reviews of same code | 5-min | Cost optimized |
| Batch processing (30+ requests) | 1-hour | Stable cache, amortized cost |
| Long-running task (hours) | 1-hour | Prevents cache eviction |
| One-off analysis | None (no cache) | Cache overhead not worth it |

---

## 3. Cache Breakpoints & Movement

### How Cache Breakpoints Work

```
Input structure:
┌─────────────────────────────────────┐
│ System Prompt (w/ cache_control)    │ ← CACHEABLE
├─────────────────────────────────────┤
│ Large Context Block (w/ cache)      │ ← CACHEABLE
├─────────────────────────────────────┤
│ User Question 1                     │ ← BREAKPOINT (not cacheable)
├─────────────────────────────────────┤
│ Claude Response 1                   │
├─────────────────────────────────────┤
│ User Question 2                     │ ← NEW BREAKPOINT (moves forward)
└─────────────────────────────────────┘
```

**Key rule**: Cache extends from start of message up to the **last cacheable block**. Anything after is fresh.

### Cache Movement During Conversation

**First request:**
```
Tokens: [System (500) + Context (10,000) + Q1 (200)]
Cache: System + Context (10,500 tokens cached)
Cost: 10,500 × 1.25x = 13,125 tokens
```

**Second request (same system + context, new Q2):**
```
Tokens: [System (500) + Context (10,000) + Q1 (200) + Response1 (800) + Q2 (200)]
Cache hit: System + Context (reuse)
Cache write: Response 1 (newly cacheable)
Cost: (10,500 cached) + (800 × 1.25x) = 11,500 tokens
Savings: ~35% vs. reprocessing everything
```

### Optimal Cache Placement Strategy

```python
# Architecture: System + Document Pool + Dynamic Query

message = [
  {
    "role": "user",
    "content": [
      # Part 1: Static instructions (cache once)
      {
        "type": "text",
        "text": """Review the following code for bugs, security issues, and performance.
        Output format: [severity] [issue] [line number]
        Severity levels: CRITICAL, HIGH, MEDIUM, LOW""",
        "cache_control": {"type": "ephemeral"}
      },
      
      # Part 2: Reference documents (cache once)
      {
        "type": "text",
        "text": load_company_coding_standards(),  # ~5KB
        "cache_control": {"type": "ephemeral"}
      },
      
      # Part 3: This request's code (NOT cached - new each time)
      {
        "type": "text",
        "text": f"Code to review:\n{user_code}"
        # No cache_control = fresh tokens
      }
    ]
  }
]
```

**Result**: 
- Write cost: ~6,000 tokens (system + standards)
- Read cost: ~600 tokens (reuse + new code)
- Savings: ~80% per review after first

---

## 4. Cache Hit Rate Diagnosis

### How to Check If Cache Is Working

**Sign 1: Cost reduction**
```
Expected:
- First request: ~13,000 tokens
- 2nd request: ~1,000 tokens (reuse) + ~500 tokens (new code)
- If you see similar cost both times: NO CACHE HIT
```

**Sign 2: Usage API logs**
```bash
# Check Anthropic API usage dashboard
# Look for "cache_creation_input_tokens" and "cache_read_input_tokens"

# If both are 0: Cache not being used
# If cache_read > cache_write: Good hit rate!
```

### Diagnostic Checklist

```
□ cache_control field present?
  → Check: `cache_control` in system or content blocks

□ Placement before breaking point?
  → Cache must be at START of message
  → NOT in the middle or end

□ TTL not expired?
  → Ephemeral: 5 minutes max
  → Keep-alive: 1 hour max
  → If > TTL since write: cache evicted

□ Same model version?
  → Cache is model-specific
  → claude-3-5-sonnet vs claude-opus = different caches

□ API call (not CLI)?
  → Claude Code CLI doesn't expose cache control
  → Use Anthropic SDK directly for fine-grained control

□ Input unchanged?
  → Even 1 token change invalidates cache
  → Moving spaces, changing punctuation = new cache entry
```

---

## 5. Batch Processing Patterns

### Pattern 1: Review Multiple Files with Shared Context

**Setup:**
```
Initial cache:
- System prompt: "Code review guidelines" (cached)
- Company coding standards (cached)
- Architecture rules (cached)

For each file:
- New code to review (fresh tokens)
- Get review output (new)
```

**Cost benefit:**
```
Without cache:
- File 1: 5,000 tokens (system + standards + code + review)
- File 2: 5,000 tokens
- File 3: 5,000 tokens
- Total: 15,000 tokens

With cache:
- File 1: 6,000 tokens (cache write: system + standards × 1.25)
- File 2: 1,500 tokens (cache read: system + standards × 0.1, + new code + review)
- File 3: 1,500 tokens
- Total: 9,000 tokens
- Savings: 40%
```

### Pattern 2: Analysis Batch with 1-Hour TTL

**When**: Processing 100+ documents with same analysis template

```python
# Use 1-hour cache to keep context alive
cache_type = "ephemeral"  # with keep_alive for 1 hour
if num_documents > 20:
    cache_type = "ephemeral"  # "keep_alive": true implied

for doc_id in document_ids:
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        system=[
            {
                "type": "text",
                "text": analysis_template,
                "cache_control": {"type": cache_type}
            }
        ],
        messages=[
            {
                "role": "user",
                "content": f"Analyze: {documents[doc_id]}"
            }
        ]
    )
```

### Pattern 3: Multi-Stage Processing

**Stage 1: Cache setup** (one-time cost)
```
System prompt + company guidelines + API docs
Cost: 10,000 tokens
Cached: ✓
```

**Stage 2: Process batch** (reuse cache)
```
For 50 API calls:
- First: 1,500 tokens (new) = 1,500 total
- Rest 49: 150 tokens each (cache read 90% cheaper)
- Total: 1,500 + (49 × 150) = 8,850 tokens
- Savings vs. fresh: 50,000 - 8,850 = 82% reduction
```

---

## 6. Cost Optimization Formulas

### Calculating Break-Even Point

```
Cache write cost (5-min TTL):  context_size × 1.25x
Cache read cost per request:    context_size × 0.1x
Fresh processing cost:          context_size × 1.0x

Break-even (when cache pays for itself):
  cache_write + (N × cache_read) < (N × fresh)
  
  For context_size = 10,000 tokens:
  12,500 + (N × 1,000) < (N × 10,000)
  12,500 < N × 9,000
  N > 1.4
  
  → Cache pays for itself after 2 requests
```

### ROI for 1-Hour Cache

```
1-hour cache write (2x cost):  context_size × 2.0x
1-hour cache read:             context_size × 0.1x

More expensive to write, but better for batch jobs:
  20,000 + (N × 1,000) < (N × 10,000)
  20,000 < N × 9,000
  N > 2.2
  
  → Break-even at 3 requests
  → For 50+ requests, 1-hour cache is 5-10% cheaper overall
```

---

## 7. Cache Management in Production

### Cache Invalidation

**Automatic** (when to refresh):
```
Cache expires:
- 5-min ephemeral: After 5 minutes of creation
- 1-hour ephemeral: After 1 hour of creation

Manual (force refresh):
- Send request with cache_control removed
- Change any token in cached block (even whitespace)
- Use different model/version
```

### Monitoring Cache Performance

**Metrics to track:**
```
Weekly:
- Cache hit rate: cache_read / (cache_read + fresh)
- Avg cost per request: (total_tokens × rate) / num_requests
- Context size: KB of cached content

Target:
- Hit rate: > 70% (if using cache)
- Cost: 20-30% of non-cached baseline
```

### Cache Warming (Proactive Caching)

```python
# Pre-compute and cache once daily
def warm_cache():
    # Load static data that will be reused
    standards = load_coding_standards()
    api_docs = load_api_documentation()
    
    client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=100,  # Minimal output, just trigger cache
        system=[
            {
                "type": "text",
                "text": standards,
                "cache_control": {"type": "ephemeral"}
            },
            {
                "type": "text",
                "text": api_docs,
                "cache_control": {"type": "ephemeral"}
            }
        ],
        messages=[
            {"role": "user", "content": "OK"}  # Dummy message
        ]
    )
    # Cache is now warm; subsequent requests reuse it
```

---

## 8. Common Pitfalls & Solutions

### Pitfall 1: Cache not working (no savings)

**Problem**: Sending same query 10 times, but no cost reduction

**Solution**:
1. Verify cache_control is in the message:
   ```python
   # ✓ Correct
   "system": [{"type": "text", "text": "...", "cache_control": {...}}]
   
   # ✗ Wrong (cache_control on wrong field)
   "messages": [{"role": "user", "cache_control": {...}}]
   ```

2. Ensure cache placement is before non-cached content:
   ```python
   # ✓ Correct order
   [System (cached), Context (cached), Query (fresh)]
   
   # ✗ Wrong order (cache after breaks it)
   [System (cached), Query (fresh), Context (cached)]
   ```

### Pitfall 2: Cache expires mid-batch

**Problem**: Processing 100 files, cache expires after 5 minutes halfway through

**Solution**: Use 1-hour TTL for batches
```python
# For batch > 10 requests over > 3 minutes, use keep_alive
if num_requests > 10 or estimated_time_minutes > 3:
    # Note: keep_alive not explicitly set in ephemeral v1
    # Instead, refresh cache every 4 minutes
    if time.time() - cache_start > 240:
        # Refresh cache by re-submitting with cache_control
```

### Pitfall 3: Different versions of same content

**Problem**: "We're caching the same code review template, why no savings?"

**Solution**: Template must be byte-for-byte identical
```python
# ✗ Different spacing = different cache
template_a = "Review for bugs:\n\n- Check null safety"
template_b = "Review for bugs:\n- Check null safety"

# ✓ Use constants
REVIEW_TEMPLATE = """Review for bugs:

- Check null safety"""

# Reuse everywhere: REVIEW_TEMPLATE
```

---

## 9. Advanced: Combining Cache with Batch API

**Scenario**: Process 10,000 documents with 90% cost reduction

```python
from anthropic import Anthropic

# Step 1: Cache the prompt once (in main session)
client_single = Anthropic()
response = client_single.messages.create(
    model="claude-3-5-sonnet-20241022",
    max_tokens=50,
    system=[
        {
            "type": "text",
            "text": REVIEW_TEMPLATE,
            "cache_control": {"type": "ephemeral"}
        }
    ],
    messages=[{"role": "user", "content": "Warmup"}]
)
# Cache ID is created (transparent to us)

# Step 2: Process documents with Batch API + cache
batch_requests = []
for doc_id, content in documents.items():
    batch_requests.append({
        "custom_id": f"doc-{doc_id}",
        "params": {
            "model": "claude-3-5-sonnet-20241022",
            "max_tokens": 500,
            "system": [
                {
                    "type": "text",
                    "text": REVIEW_TEMPLATE,
                    "cache_control": {"type": "ephemeral"}
                }
            ],
            "messages": [{"role": "user", "content": content}]
        }
    })

batch = client_single.beta.messages.batches.create(requests=batch_requests)
# Batch API is already 50% off
# Plus cache read is 90% of fresh tokens
# Combined: ~95% savings for subsequent documents!
```

---

## 10. Best Practices Checklist

**DO:**
- ✅ Cache system prompts (large, reused)
- ✅ Cache reference documents (company standards, API docs)
- ✅ Use 5-min cache for most workflows
- ✅ Use 1-hour cache for batch jobs (10+ items)
- ✅ Monitor cache hit rate weekly
- ✅ Combine cache with Batch API for maximum savings

**DON'T:**
- ❌ Cache dynamic content (user queries, IDs)
- ❌ Cache if only processing 1-2 documents
- ❌ Mix cached + non-cached in same section (breaks cache)
- ❌ Assume cache is available in Claude Code CLI (use SDK)
- ❌ Cache very short prompts (< 500 tokens) — overhead isn't worth it
- ❌ Rely on cache for security (always validate outputs)

---

## References

- [Prompt Caching - Official Docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) — 2026 update with workspace isolation
- [Batch API - Pricing](https://platform.claude.com/docs/en/about-claude/pricing) — 50% off + caching combo
- [Claude API Prompt Caching: Cost Optimization Guide](https://jangwook.net/en/blog/en/claude-api-prompt-caching-cost-optimization-guide/) — Detailed patterns
- [Anthropic API Pricing 2026](https://www.finout.io/blog/anthropic-api-pricing) — Current rates
