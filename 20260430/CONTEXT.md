# CONTEXT.md - Context Window Management & Optimization

**Version**: 1.0 | **Last Updated**: 2026-04-30

---

## 1. Context Window Fundamentals

### Token Budget & Thresholds

```
Total Window: 200,000 tokens

Pre-Consumed (System Overhead):
├── System prompt & instructions: ~10,000 tokens
├── Tool definitions: ~5,000 tokens
├── CLAUDE.md loading: ~3,000 tokens
└── Session state: ~2,000 tokens
    → Total overhead: ~20,000 tokens

Safe Working Window: 120,000 tokens (60% rule)
├── Available for work: ~100,000 tokens
├── Buffer for final response: ~20,000 tokens
└── Hard limit before degradation: 120,000 tokens

Warning Thresholds:
├── Yellow (caution): 150,000 tokens (75% capacity)
├── Red (critical): 175,000 tokens (87% capacity)
└── Emergency: 195,000 tokens (97% capacity)
```

**Key Principle**: Quality degrades at 60% capacity. Attention mechanism gives earlier instructions less weight as context fills.

---

## 2. Per-Session Token Consumption Breakdown

### Typical Single-Turn Interaction

```
Session Start:            20,000 tokens (fixed overhead)

User Message:
  ├── Question text:      500 tokens
  ├── Code snippet:       2,000 tokens (if provided)
  └── Context links:      500 tokens
  → Subtotal:            3,000 tokens

Claude Response (planning):
  ├── Initial analysis:   1,000 tokens
  ├── Tool call output:   5,000 tokens (Web search results)
  └── Synthesis:          2,000 tokens
  → Subtotal:           8,000 tokens

Running Total: 20,000 + 3,000 + 8,000 = 31,000 tokens (15% capacity)
```

### Complex Multi-Step Task

```
Session Start:            20,000 tokens
User request:             2,000 tokens
Research (Web search):    8,000 tokens (5+ sources)
File reads (CLAUDE.md, config): 5,000 tokens
Code generation:          15,000 tokens (writing 5 files)
Validation (grep/parse):  3,000 tokens
Response summary:         2,000 tokens

Running Total: 55,000 tokens (27% capacity)
```

### Code Review / Large File Analysis

```
Session Start:            20,000 tokens
PR diff (500 lines):      12,000 tokens
Review comments:          5,000 tokens
Tool outputs:             8,000 tokens
Detailed response:        5,000 tokens

Running Total: 50,000 tokens (25% capacity)
```

---

## 3. Detection & Alerts

### How to Know You're Running Low

**Claude's indicators:**
- Message appears: "context window approaching limit"
- Responses become shorter, less detailed
- Stops exploring follow-up ideas ("I'll skip X for brevity")
- Tool output summaries instead of full results

**Manual check**:
```bash
# Estimate session size
echo "Session tokens ≈ (messages × 200) + (tool outputs × 300)"
# Example: 15 messages + 5 tool outputs ≈ (15 × 200) + (5 × 300) = 4,500 tokens
# Plus overhead = 24,500 tokens used
```

---

## 4. Context Preservation Strategies

### Strategy 1: Session Compression (Automatic)

Claude Code automatically compresses older messages when context fills:
1. **Detection**: System monitors token usage in real-time
2. **Compression**: Messages older than current + last 5 turns are summarized
3. **Preservation**: Compressed summary is inserted at top of context
4. **Result**: Freed ~30% of context, maintains continuity

**What survives compression**: Recent tool outputs, latest user requests, current task state

**What's lost**: Detailed reasoning from early messages (summaries kept instead)

---

### Strategy 2: Explicit Session Boundaries

When approaching capacity, explicitly start fresh:

```
Session 1 (completed):
├── Task: Analyze codebase structure
├── Output: architecture-summary.md
└── Context used: 95,000 tokens

Session 2 (new):
├── Load: architecture-summary.md from Session 1
├── Task: Implement refactoring based on findings
├── Context fresh: 20,000 tokens overhead
└── Result: No context bleed, fresh focus
```

**Best for**: Multi-day projects, architectural deep-dives, large codebases

---

### Strategy 3: Intermediate Documentation

Break long workflows into documented checkpoints:

```
Step 1: Research (Session A)
  → Output: research-findings.md (3KB)
  
Step 2: Design (Session B)
  → Read research-findings.md (small file)
  → Output: design-decisions.md (2KB)
  
Step 3: Implement (Session C)
  → Read design-decisions.md
  → Output: implementation.md + code changes
```

**Benefit**: Each session stays lightweight; previous findings preserved

---

### Strategy 4: Code References Instead of Full Files

**Problem**: Reading a 2000-line file consumes 6000+ tokens

**Solution**: Reference by line number instead

```
❌ Bad:
"Please review file utils.js"
→ Entire 3000-line file loaded

✅ Good:
"Review the caching logic in utils.js:1-150"
→ Only 150 lines loaded (~500 tokens)
```

**How to do it**:
```bash
# Get file size
wc -l utils.js

# Read specific range
# Use Read tool with offset/limit parameters
# Bash: sed -n '1,150p' utils.js
```

---

## 5. Large File Handling

### Reading Large Files Efficiently

**File size matrix:**
| File Size | Tokens | Strategy |
|-----------|--------|----------|
| < 100 lines | ~300 | Read normally |
| 100-500 lines | ~1500 | Use offset/limit, read in chunks |
| 500-2000 lines | ~6000 | Reference specific functions, use Explore agent |
| > 2000 lines | ~20000+ | Delegate to Explore subagent, get summary |

**Example: Reading a 2000-line config**
```bash
# DON'T:
Read /path/to/huge-config.json

# DO:
# Step 1: Search for relevant section
grep -n "caching" huge-config.json

# Step 2: Read only that section (e.g., lines 150-200)
Read /path/to/huge-config.json (with offset=150, limit=50)
```

### Delegating to Explore Subagent

For large codebase exploration:

```
Main Session:
  "Use Explore agent to find all files importing utils.js"
  
Explore Agent (separate context):
  ├── File read overhead: fresh 20,000 tokens
  ├── grep -r "import.*utils.js" (finds 45 files)
  ├── Output: findings.txt (5KB)
  └── Return to main session

Main Session (resumed):
  ├── Load findings.txt (small, 100 tokens)
  ├── Act on results
  └── Total overhead: minimal
```

---

## 6. Managing Conversation History

### Conversation Lifecycle

```
Early Stage (Turns 1-5):
  Token usage: 20,000 - 50,000
  Behavior: Full context, detailed reasoning
  Action: Normal work

Mid Stage (Turns 6-15):
  Token usage: 50,000 - 100,000
  Behavior: Good detail, occasional summaries
  Action: Create intermediate checkpoints (save findings)

Late Stage (Turns 16+):
  Token usage: 100,000 - 150,000
  Behavior: Compressed older messages, summaries
  Action: Start fresh session if needed

Critical (>150,000 tokens):
  Behavior: Quality degradation, lost context
  Action: End session, summarize to file, start fresh
```

### When to Start a New Session

**Automatic trigger** (you don't choose):
- System auto-compresses at 80%+ capacity
- New session needed after ~50 back-and-forth turns

**Manual trigger** (you choose):
- Task completed, start different task → fresh session
- Lost track of context → create checkpoint, restart
- Switching between projects → separate sessions

**How to start fresh**:
```bash
# In Claude Code web app:
/clear        # Clear conversation history
# OR just open a new tab

# In CLI:
claude-code --new-session
```

---

## 7. Multi-Session Context Handoff

### Pattern: Checkpoint Files

**Session 1**: Research phase
```markdown
# Session 1: Architecture Analysis (2026-04-30 14:00)

## Findings
- Main bottleneck: Database queries in user_loader.py:42-56
- Impact: ~500ms per request
- Solution candidates: Caching, indexing, async

## Recommendations
1. Implement Redis cache (est. 2 days)
2. Add database index (est. 0.5 days)
3. Async queries with asyncio (est. 1 day)

## Next Steps
→ Session 2: Detailed implementation plan
```

**Session 2**: Design phase
```markdown
# Session 2: Implementation Design (2026-04-30 15:00)

**Input**: architecture-findings.md (from Session 1)

## Decision: Cache Strategy
- Use Redis (lower complexity than async rewrite)
- TTL: 5 minutes
- Cache key pattern: user_{user_id}_{version}

## Implementation Plan
1. Install redis-py
2. Create cache decorator
3. Integrate in user_loader.py
4. Test with load simulation
```

**Session 3**: Implementation
```markdown
# Session 3: Code Implementation (2026-04-30 16:00)

**Input**: implementation-design.md (from Session 2)

## Changes Made
- cache.py (new file)
- user_loader.py (modified lines 42-56)
- tests/test_cache.py (new)

## Testing
- ✓ Local unit tests pass
- ✓ 5min cache TTL verified
- → Next: Load test with Apache Bench
```

---

## 8. Token Counting Checklists

### Before Starting a Long Task

```
✓ Estimate tokens:
  - Current conversation size: ____
  - New files to read: ____
  - Expected responses: ____
  - Total projected: ____ / 200,000

✓ If total > 100,000:
  - Break into sessions
  - Create checkpoint documents
  - Use Explore subagent for large reads

✓ Confirm capacity:
  - Safe margin: 20,000+ tokens remaining
  - Proceed if OK
```

### For Complex Code Reviews

```
✓ Scope estimation:
  - Files affected: ____
  - Total lines: ____
  - Estimated tokens: (lines ÷ 6) ≈ ____

✓ Strategy:
  - [ ] Small PR (< 200 lines) → Full read + detailed review
  - [ ] Medium PR (200-800 lines) → Reference key sections, delegate edge cases
  - [ ] Large PR (> 800 lines) → Explore subagent, focus on 3-5 files

✓ Capacity check:
  - Session tokens before: ____
  - PR read tokens: ____
  - Review response: ~2,000 tokens
  - Total: ____ / 200,000
```

---

## 9. Optimization Tips (Advanced)

### Tip 1: Leverage Built-in Summaries

Instead of re-reading long outputs, ask for a summary:

```
❌ Too verbose:
"Generate a full test suite for utils.js"
→ Response: 3000+ tokens of code

✅ Concise:
"Outline test coverage needed for utils.js (no code)"
→ Response: ~500 tokens (checklist format)
→ Then: "Generate tests for case #1, #2, #3"
```

### Tip 2: Use Streaming for Long Responses

In Claude Code CLI:
```bash
claude-code --stream  # Tokens still count, but output appears in real-time
```

### Tip 3: Cache Markdown Templates

For repetitive outputs (README, architecture docs), reuse templates:

```
Instead of:
"Generate a README for this project"
→ Full generation: 2000+ tokens

Better:
"Use the README template from docs/ and fill in: name=X, description=Y"
→ Template reading: 300 tokens, filling: 500 tokens
→ Savings: ~1200 tokens
```

### Tip 4: Defer Non-Critical Output

```
❌ Full response:
"Analyze code, find bugs, suggest refactors, and write fixes"
→ Single response: ~5000 tokens

✅ Staged:
1. "Find bugs" → ~1500 tokens
2. "Based on bugs found, suggest refactors" → ~1000 tokens
3. "Write fixes for top 3" → ~2000 tokens
→ Can spread across sessions, easier to iterate
```

---

## 10. Emergency Recovery

### If You Hit Context Limit

```
Symptom: "Context window exceeded" error

Action:
1. STOP immediately (don't send more messages)
2. Create checkpoint:
   - Save current thinking to file
   - Note: what was done, what's next
3. Start new session
4. Read checkpoint file
5. Continue from where you left off
```

### If Compressed Context Is Confusing

Symptom: Claude forgets early setup, makes contradictory decisions

Solution:
```
In new session:
1. Load saved checkpoint
2. Explicitly state: "Previous context was compressed. Our goal is X. Here's what we've done so far: [checkpoint content]"
3. Claude will now have full explicit context
```

---

## References

- [Claude Code Docs - Context Management](https://code.claude.com/docs/en/best-practices) → Context limits and strategies
- [Token Counting Guide](https://platform.claude.com/docs/en/resources/tokens) → Anthropic official token counting
- [Reddit r/ClaudeAI - Context Optimization](https://reddit.com/r/ClaudeAI) → Community practices
