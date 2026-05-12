# SKILLS.md - Claude Skills 定義ガイド

**対象**: Harness Engineering 専用スキル定義、トリガー、実装パターン  
**作成日**: 2026-05-12  
**バージョン**: 1.0（Claude Skills 組織展開対応）

---

## 1. Skills の役割（CLAUDE.md との関係）

### Skills vs MCP vs Subagents

| 層 | 構成要素 | 役割 | 例 |
|---|---|---|---|
| **内部ロジック（Skills）** | Workflow + Knowledge | AI の「プレイブック」。複雑なタスクを構造化 | `code_review_audit`, `web_search_synthesis` |
| **外部接続（MCP）** | Protocol + API | AI の「神経系」。外部システムへのアクセス | GitHub API, Slack API, Notion API |
| **並列実行（Subagents）** | Independent Context | AI の「チームワーク」。複数視点による分析 | Researcher, Implementer, Reviewer, Debugger |

**設計原則**:
- **Skills**: DO THIS（手順を定義）
- **MCP**: ACCESS THAT（システムに接続）
- **Subagents**: PARALLEL WORK（複数タスクの並列化）

---

## 2. Harness Engineering のコアスキル（5+）

### Skill 1: Web Search Synthesis

```markdown
---
skill_id: web_search_synthesis
name: "Web Search & Synthesis"
description: "Investigate topics from 3+ independent sources with confidence scoring"
version: "1.0"
organization_scope: true  # Organization-level deployment
triggers:
  - "What's the latest..."
  - "Research shows..."
  - "Find examples of..."
  - "Investigate..."
target_subagent: Researcher
---

# Web Search Synthesis

## Purpose
Gather information from multiple sources and synthesize findings with confidence scores.
Avoids single-source bias by requiring 3+ independent sources.

## Input Validation
```python
def validate_input(query: str) -> dict:
    assert len(query) > 10, "Query too short"
    assert not any(c in query for c in ['<', '>', '"']), "XSS risk detected"
    return {"query": query, "valid": True}
```

## Step-by-Step Workflow

### 1. Query Decomposition
- Parse main query → identify 3+ distinct search angles
- Example:
  - Query: "Prompt caching best practices"
  - Angle 1: "Prompt caching implementation patterns"
  - Angle 2: "Prompt caching performance benchmarks"
  - Angle 3: "Prompt caching cost analysis"

### 2. Parallel Searches
- Execute 3 searches in parallel (Researcher can do this)
  - Google Search
  - GitHub Search (code examples)
  - Anthropic Blog (official sources)

### 3. Synthesis with Confidence
```python
@dataclass
class Finding:
    claim: str
    source_url: str
    source_type: str  # "official", "blog", "github", "forum"
    confidence: Literal["HIGH", "MEDIUM", "LOW"]
    evidence: str  # 1-2 sentence justification
```

### 4. Output Formatting
- Return findings in Researcher Response format
  ```json
  {
    "status": "SUCCESS",
    "findings": [
      {
        "claim": "Prompt caching reduces cost by 90%",
        "source": "https://platform.claude.com/docs/...",
        "confidence": "HIGH",
        "evidence": "Anthropic official documentation"
      }
    ]
  }
  ```

## Quality Checklist
- [ ] 3+ independent sources cited
- [ ] Confidence scores assigned to each finding
- [ ] No single-source bias
- [ ] URLs are valid and accessible
- [ ] Claims are factual (cross-reference where controversial)

## Confidence Thresholds
- **HIGH**: Official docs, published research, 2+ corroborating sources
- **MEDIUM**: Reputable blogs, Stack Overflow with 100+ upvotes
- **LOW**: Personal blogs, forums with low engagement
- **Exclude**: Unverified rumors, AI-generated summaries without sources

---
```

### Skill 2: Code Review Audit

```markdown
---
skill_id: code_review_audit
name: "Code Review & Security Audit"
description: "Multi-layer review: security, performance, maintainability"
version: "1.0"
organization_scope: true
triggers:
  - "Review this code"
  - "Audit for security"
  - "Check test coverage"
  - "Validate performance"
target_subagent: Reviewer
---

# Code Review & Security Audit

## Purpose
Comprehensive code review across security, performance, and maintainability dimensions.

## Review Dimensions

### 1. Security Audit
```python
security_checks = {
    "no_hardcoded_secrets": check_no_secrets,
    "no_sql_injection": check_sql_safety,
    "no_xss_vulnerabilities": check_xss_safety,
    "authentication_present": check_auth,
    "owasp_top_10": check_owasp,
}

def check_no_secrets(code: str) -> tuple[bool, list[str]]:
    """Detect hardcoded API keys, passwords, tokens."""
    patterns = [
        r"api_key\s*=\s*['\"].*['\"]",
        r"password\s*=\s*['\"].*['\"]",
        r"token\s*=\s*['\"].*['\"]",
    ]
    matches = []
    for pattern in patterns:
        if re.search(pattern, code):
            matches.append(pattern)
    return len(matches) == 0, matches
```

### 2. Performance Audit
```python
performance_checks = {
    "no_n_plus_1_queries": check_n_plus_1,
    "cache_strategy_valid": check_caching,
    "memory_efficient": check_memory_usage,
    "algorithm_complexity": check_complexity,
}

def check_caching(code: str) -> dict:
    """Validate Prompt Caching strategy."""
    required = [
        "immutable_prefix",
        "variable_suffix",
        "cache_hit_rate >= 0.60"
    ]
    # Check presence of caching logic
    pass
```

### 3. Maintainability Audit
```python
maintainability_checks = {
    "max_function_length": check_function_length,
    "cyclomatic_complexity": check_complexity,
    "test_coverage": check_test_coverage,
    "type_safety": check_type_hints,
    "code_duplication": check_dry_principle,
}

def check_test_coverage(code: str, coverage_report: str) -> bool:
    """Verify test coverage ≥ 80%."""
    return extract_coverage(coverage_report) >= 0.80
```

## Output Format
```json
{
  "status": "REVIEWED",
  "issues": [
    {
      "severity": "critical",
      "category": "security",
      "issue": "Hardcoded API key detected",
      "location": "src/api.py:42",
      "fix": "Move to environment variable"
    }
  ],
  "approval": false,
  "next_steps": ["Fix critical issues", "Re-review"]
}
```

## Quality Checklist
- [ ] Security: No hardcoded secrets
- [ ] Security: No injection vulnerabilities
- [ ] Performance: Cache strategy valid (if applicable)
- [ ] Maintainability: Function length ≤ 30 lines
- [ ] Maintainability: Cyclomatic complexity ≤ 10
- [ ] Tests: Coverage ≥ 80%
- [ ] Types: Strict mode enabled

---
```

### Skill 3: TDD Implementation

```markdown
---
skill_id: tdd_implementation
name: "Test-Driven Development (TDD)"
description: "Red → Green → Refactor workflow for code implementation"
version: "1.0"
organization_scope: true
triggers:
  - "Implement..."
  - "Add feature..."
  - "Fix bug..."
target_subagent: Implementer
---

# Test-Driven Development (TDD)

## Purpose
Ensure code quality and correctness by writing tests before implementation.

## TDD Workflow

### Phase 1: RED (Write Failing Tests)
```python
def test_user_authentication_fails_with_invalid_password():
    """Test must FAIL initially."""
    user = User(username="alice", password_hash=hash("correct"))
    
    # This should raise AuthenticationError
    with pytest.raises(AuthenticationError):
        user.authenticate(password="wrong")
```

### Phase 2: GREEN (Implement Minimum)
```python
class User:
    def authenticate(self, password: str) -> bool:
        """Implement minimum to pass test."""
        if self.password_hash == hash(password):
            return True
        raise AuthenticationError("Invalid password")
```

### Phase 3: REFACTOR (Improve Quality)
```python
class User:
    """User with secure authentication."""
    
    def authenticate(self, password: str) -> bool:
        """
        Authenticate user with password.
        
        Args:
            password: Plain-text password to verify
        
        Returns:
            True if authentication succeeds
        
        Raises:
            AuthenticationError: On invalid credentials
        """
        return self._verify_password(password)
    
    def _verify_password(self, password: str) -> bool:
        """Private helper for password verification."""
        return bcrypt.verify(password, self.password_hash)
```

## Implementation Constraints
- **Max Function Length**: 30 lines
- **Cyclomatic Complexity**: ≤ 10
- **Test Coverage**: ≥ 80%
- **Type Safety**: Strict mode (mypy, tsc)

## Validation Checklist
- [ ] All tests pass (pytest)
- [ ] Coverage ≥ 80% (pytest --cov)
- [ ] Linting passes (ruff, black)
- [ ] Type checking passes (mypy)
- [ ] No hardcoded values
- [ ] Docstrings present for public APIs

---
```

### Skill 4: MCP Integration & Security

```markdown
---
skill_id: mcp_secure_integration
name: "MCP Server Integration with Security Review"
description: "Safely connect to external systems via Model Context Protocol"
version: "1.0"
organization_scope: true
triggers:
  - "Connect to..."
  - "Integrate [GitHub/Slack/Notion]"
  - "Add MCP server"
target_subagent: Implementer (with Reviewer oversight)
---

# MCP Integration & Security

## Purpose
Securely integrate external systems (GitHub, Slack, etc.) via MCP servers.

## MCP Integration Checklist

### 1. Trustworthiness Verification
```python
def verify_mcp_trustworthiness(mcp_url: str, mcp_name: str) -> bool:
    """Verify MCP server is trustworthy."""
    checks = {
        "is_official": mcp_url.startswith("https://official-mcp"),
        "has_documentation": fetch_documentation(mcp_url) is not None,
        "github_stars": fetch_github_stars(mcp_url) > 100,
        "security_audit": fetch_security_audit_status(mcp_url),
    }
    return all(checks.values())
```

### 2. Permission Scope Definition
```yaml
# config/mcp-servers.yaml
mcp_servers:
  github:
    endpoint: "https://mcp.github.com"
    auth_method: "oauth"
    scopes:
      - "repo:read"       # Read-only, safe
      - "issues:write"    # Write to issues only
    
    # NOT granted (too permissive):
    # - "admin:repo_hook" # Could modify critical settings
    # - "user:email"      # Privacy concern
```

### 3. Rate Limiting & Timeout
```python
mcp_config = {
    "github": {
        "rate_limit": 100,  # requests/minute
        "timeout_sec": 10,
        "retry_strategy": "exponential_backoff",
        "max_retries": 3,
    }
}
```

### 4. Authentication Management
```python
# ✅ Good: Token in environment
os.environ["GITHUB_MCP_TOKEN"] = os.getenv("GITHUB_API_TOKEN")

# ❌ Bad: Token in code
github_token = "ghp_xxxxx"  # Never hardcode!

# ✅ Good: Use .local.md
# .local.md (gitignored):
# GITHUB_API_TOKEN=ghp_xxxxx
```

### 5. Error Handling
```python
@retry(stop=stop_after_attempt(3), wait=wait_exponential())
async def call_mcp_server(mcp_name: str, method: str, params: dict):
    """Call MCP server with retry logic."""
    try:
        result = await mcp_client.call(mcp_name, method, params)
        return {"success": True, "data": result}
    except RateLimitError:
        logger.warning(f"{mcp_name} rate limited; retrying...")
        raise  # Retry decorator handles
    except TimeoutError:
        logger.error(f"{mcp_name} timeout after {timeout_sec}s")
        return {"success": False, "error": "timeout"}
    except Exception as e:
        logger.error(f"{mcp_name} error: {e}")
        # Alert security team if suspicious
        return {"success": False, "error": str(e)}
```

## Security Review Gate
```
MCP Integration Request
    ↓
[Implementer] Submits integration proposal
    ↓
[Reviewer] Security checklist:
    ✓ Official MCP server?
    ✓ Documentation exists?
    ✓ Scopes minimal?
    ✓ Auth properly managed?
    ✓ Rate limiting configured?
    ↓
[Coordinator] Final approval
    ↓
Integration enabled in production
```

---
```

### Skill 5: Debugging & Root Cause Analysis

```markdown
---
skill_id: root_cause_debugging
name: "Debugging & Root Cause Analysis"
description: "Systematic debugging using hypothesis-driven error isolation"
version: "1.0"
organization_scope: true
triggers:
  - "Debug this error"
  - "Why did it fail?"
  - "Performance degraded"
  - "Timeout occurred"
target_subagent: Debugger
---

# Debugging & Root Cause Analysis

## Purpose
Systematically isolate and fix errors using hypothesis-driven analysis.

## Debugging Workflow

### Step 1: Understand the Error
```python
error_context = {
    "error_message": "KeyError: 'user_id'",
    "stacktrace": "[...full stacktrace...]",
    "environment": "production",
    "last_successful_run": "2026-05-12 10:00:00",
    "first_failure": "2026-05-12 11:30:00",
}
```

### Step 2: Generate Hypotheses
```python
hypotheses = [
    {
        "rank": 1,
        "hypothesis": "Database schema changed; user_id column missing",
        "probability": "HIGH",
        "test": "SELECT column_name FROM user_table SCHEMA"
    },
    {
        "rank": 2,
        "hypothesis": "Request parsing broke; JSON structure changed",
        "probability": "MEDIUM",
        "test": "Log incoming request JSON"
    },
    {
        "rank": 3,
        "hypothesis": "Cache invalidation issue; stale data returned",
        "probability": "MEDIUM",
        "test": "Check cache hit rate; flush if needed"
    },
]
```

### Step 3: Test Hypotheses
```python
def test_hypothesis(hypothesis: str, test_code: str) -> dict:
    """Execute test and return result."""
    try:
        result = eval(test_code)
        return {
            "hypothesis": hypothesis,
            "result": result,
            "status": "CONFIRMED" if result else "REFUTED"
        }
    except Exception as e:
        return {
            "hypothesis": hypothesis,
            "error": str(e),
            "status": "INCONCLUSIVE"
        }
```

### Step 4: Isolate Root Cause
```
Hypothesis 1 (Database): CONFIRMED
  └─ Root Cause: Schema migration incomplete
     └─ User table missing user_id column
     └─ Migration log shows: failed at step 3/5

→ ROOT CAUSE: Incomplete database migration
```

### Step 5: Propose Fix
```python
fix_proposal = {
    "root_cause": "Incomplete database migration",
    "fix": "Run pending migrations: alembic upgrade head",
    "validation": "pytest tests/integration/test_user_db.py",
    "rollback_plan": "alembic downgrade -1",
    "estimated_impact": "30 min maintenance window"
}
```

## Error Classification
```python
error_severity = {
    "CRITICAL": "Production outage, immediate fix required",
    "HIGH": "Feature broken, fix within hours",
    "MEDIUM": "Degraded performance, fix within day",
    "LOW": "Edge case, fix within sprint",
}
```

## Debugging Checklist
- [ ] Error reproduced with minimal steps
- [ ] 3+ hypotheses generated
- [ ] Each hypothesis tested
- [ ] Root cause isolated with evidence
- [ ] Fix proposed and validated
- [ ] Rollback plan documented

---
```

---

## 3. Skill 作成テンプレート

```markdown
---
skill_id: unique_identifier
name: "Human-Readable Name"
description: "One-line description of what this skill does"
version: "1.0"
organization_scope: true  # Organization-wide deployment?
triggers:
  - "keyword1"
  - "phrase that triggers..."
  - "pattern to match"
target_subagent: [Researcher, Implementer, Reviewer, Debugger]
dependencies:
  - "other_skill_id"  # If this skill depends on another
---

# Skill: [Name]

## Purpose
[Why this skill exists, what problem it solves]

## Input Validation
[Validate inputs for safety - XSS, injection, etc.]

## Step-by-Step Workflow

### Step 1: [Description]
[Code or process]

### Step 2: [Description]
[Code or process]

### Step 3: [Output]
[Output format, examples]

## Quality Checklist
- [ ] Check 1
- [ ] Check 2
- [ ] Check 3

---
```

---

## 4. Skill デプロイメント（Organization-level）

### 4.1 ファイル構成

```
.claude/skills/
├── web_search_synthesis.md
├── code_review_audit.md
├── tdd_implementation.md
├── mcp_secure_integration.md
├── root_cause_debugging.md
└── README.md (Skill index)
```

### 4.2 組織レベル デプロイ

```bash
# CLI (Claude Code web app)
/skill-deploy web_search_synthesis
/skill-deploy code_review_audit
/skill-deploy tdd_implementation

# Organization Admin Dashboard で一括デプロイ
# https://code.claude.com/org/harness-engineering/skills
# → すべてのメンバーに自動配布
```

### 4.3 バージョン管理

```yaml
# .claude/skills/MANIFEST.yaml
skills:
  web_search_synthesis:
    version: "1.0"
    last_updated: "2026-05-12"
    author: "@team-lead"
    status: "ACTIVE"
  
  code_review_audit:
    version: "1.1"
    last_updated: "2026-05-10"
    changelog: "Added OWASP Top 10 checks"
    author: "@security-team"
    status: "ACTIVE"
```

---

## 5. チェックリスト（Skills 導入）

- [ ] 5+ コアスキルを `.claude/skills/` に定義
- [ ] 各スキルが target_subagent を明記
- [ ] トリガーキーワードが一意かつ検出可能
- [ ] Quality Checklist を各スキルに記載
- [ ] テンプレート (SKILLS.md) が社内 wiki に掲載
- [ ] Organization-level デプロイが有効
- [ ] 月1回の skill audit & update スケジュールを設定
- [ ] Skill usage analytics をダッシュボード化

---

**参考**:
- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Extending Claude's capabilities with skills and MCP](https://claude.com/blog/extending-claude-capabilities-with-skills-mcp-servers)
