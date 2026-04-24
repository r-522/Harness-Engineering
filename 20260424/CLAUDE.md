# CLAUDE.md - AI エージェント実行規則

## Core Directives

### 1. Context は Sacred
- Context window を資源として扱う（token は金銭）
- 不要な履歴は自動的に `/compact` で圧縮
- 各 turn で used/available tokens を監視
- Loop や recursive calls では context を再利用しない

### 2. Tool Count は最大15個
```
許可ツール数: 8-15個（厳格）
危険域: 50+個（機能停止レベルの低下）
監査: 月1回、不要ツールを削除
```

**有効なツール（優先順）**:
1. Read（ファイル読込）
2. Edit（既存ファイル編集）
3. Write（新規ファイル作成）
4. Bash（シェルコマンド）
5. Agent（subagent delegation）
6. Monitor（長時間プロセス監視）
7. WebSearch（情報検索）
8. GitHub API（リポジトリ操作）
9. Bash（git操作）
10. TodoWrite（タスク管理）

**禁止ツール**:
- ExitPlanMode（予測不可）
- 過度な API 呼び出し（rate limit 回避）

### 3. Model Routing Rules
```python
if task.complexity == "trivial":
    model = "haiku"  # < 2s response
elif task.complexity == "moderate":
    model = "sonnet"  # 2-8s response
else:
    model = "opus"   # 8-30s response（research/deep thinking）
```

### 4. Effort Level Default
```
Default: "medium"（リソース効率重視）
Override conditions:
  - Research tasks: "high" or "max"
  - Simple queries: "low"
  - Complex refactoring: "medium"
```

### 5. Forbidden Patterns

#### ❌ Do NOT
```python
# 大量の tool calls を batch する
call_10_tools_in_parallel()

# Context を無視
print("長い履歴..." * 1000)

# Tool をランダムに選ぶ
tools = random.sample(all_tools, 50)

# user input を検証なしで使用
os.system(user_input)

# Secrets を commit する
git.commit("API_KEY=sk-xxx")

# Agent に全権委譲
delegate_everything_to_agent()
```

#### ✅ Do INSTEAD
```python
# 関連 tools のみ（3-5個）
read_file()
edit_file()
bash_command()

# Context を監視
if token_usage > 80%:
    compact_context()

# Tool は明示的に選択
use_bash_for_git()
use_read_for_inspection()

# User input は厳格に検証
if validate_command(user_input):
    execute(user_input)

# Secrets は環境変数
API_KEY = os.getenv("API_KEY")

# Agent は特定タスクのみ
delegate_only_specific_task()
```

## Code Style Rules

### Language-Agnostic
- 行長: 100 文字（markdown は除外）
- インデント: スペース 2個（Python は 4個）
- コメント: WHY のみ、WHAT は含まない
- 命名: snake_case（変数）、PascalCase（クラス）、CONSTANT_CASE（定数）

### Python
```python
# Good
def process_data(items):
    # Skip duplicates for performance
    return list(set(items))

# Bad
def ProcessData(Items):
    # This function processes items
    # It converts them to a set and back to list
    return list(set(Items))
```

### JavaScript/TypeScript
```typescript
// Good
const calculateMetrics = (data: DataPoint[]): Metrics => {
  // Group by timestamp for aggregation
  return groupBy(data, "timestamp");
};

// Bad
const Calculate = function(DATA) {
  // Calculate the metrics by grouping
  // the data by timestamp
  return groupBy(DATA, "timestamp");
};
```

## Git Conventions

### Commit Messages
```
<type>(<scope>): <subject>

<body>
<footer>

Examples:
fix(auth): prevent timing attacks
feat(agent): implement tool routing
docs(harness): add context management guide
```

### Allowed Commits
- ✅ New features
- ✅ Bug fixes
- ✅ Documentation
- ✅ Configuration updates
- ❌ WIP commits（branch に push しない）
- ❌ Revert without explanation

### Branch Strategy
```
claude/affectionate-faraday-OuwIm   (development)
  ↓
main branch (after review)
```

## Testing & Validation

### Pre-commit Requirements
```bash
✓ Type checking (mypy, TypeScript)
✓ Linting (eslint, pylint)
✓ Formatting (black, prettier)
✓ Security scanning (bandit, safety)
```

### Before Push
```bash
git log origin/main..HEAD  # Verify commits
git diff origin/main       # Check changes
npm run test               # Run test suite
npm run build              # Verify build
```

## Context Management

### Initialization
```
Session start:
- Load CLAUDE.md（this file）
- Load AGENT.md（skill definitions）
- Load DESIGN.md（architecture patterns）
- Initialize token counter
```

### Compression Triggers
```
Compression needed when:
- Token usage > 80% of window
- Response latency > 8s
- More than 20 prior turns
- Explicit user request: /compact
```

### Output Context
```
Keep in context:
✓ Current task description
✓ Recent failures/blockers
✓ Tool outputs（relevant only）
✓ Token usage metrics

Remove from context:
✗ Completed tasks
✗ Old tool outputs
✗ Verbose explanations
✗ Error traces（unless debugging）
```

## Security

### Input Validation
```python
# ✓ Validate before execution
if is_valid_path(user_path):
    read_file(user_path)

# ❌ Never assume input is safe
os.system(f"rm -rf {user_input}")
```

### Credential Handling
```python
# ✓ Use environment variables
api_key = os.getenv("API_KEY")

# ❌ Never hardcode secrets
API_KEY = "sk-proj-xxx"
```

### Tool Permissions
```
Automatically allowed:
- Read：プロジェクト内のみ
- Edit：プロジェクト内のみ
- Bash：git, npm, linting のみ
- WebSearch：public URLs のみ

Requires explicit permission:
- Write（新規ファイル）
- Bash（destructive commands）
- GitHub API（repository modifications）
```

## Performance Targets

| Metric | Target | Window |
|--------|--------|--------|
| Response latency | < 5s | avg over 10 turns |
| Token efficiency | > 70% | measured per session |
| Tool accuracy | > 95% | tool calls succeed |
| Code quality | 0 linting errors | pre-commit |

## When to Escalate

1. **Ambiguous requirements** → Ask user for clarification
2. **Out-of-scope changes** → Propose and wait for approval
3. **Security concerns** → Stop and report immediately
4. **Tool failures** → Investigate, don't retry blindly
5. **Destructive actions** → Always confirm first

---

**Version**: 1.0  
**Last updated**: 2026-04-24  
**Enforced**: Yes（all sessions）
