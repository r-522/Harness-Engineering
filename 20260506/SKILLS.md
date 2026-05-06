# SKILLS.md - Skill 実装ガイド・カタログ

## I. Skill の本質

### 定義
**Skill** = `.claude/skills/SKILL.md` Markdown ファイル として定義される、**再利用可能な専門能力**。
Claude が自動検出し、ユーザーリクエスト or コンテキストマッチで**自動実行**される。

### 特徴
- **ファイルベース**: Programmatic API 不要（設定駆動）
- **自動検出**: `.claude/skills/` ディレクトリを走査
- **Context-aware**: ユーザー要求とマッチで自動起動
- **再利用可能**: 複数プロジェクト間で共有可能

### Skill vs Tool vs Subagent

| 概念 | スコープ | 実行形態 | 用途 |
|---|---|---|---|
| **Skill** | 単一責務 | 自動（context match） | 専門能力の定義 |
| **Tool** | API or CLI | 明示的（Claude call） | Bash, GitHub API等 |
| **Subagent** | 複雑タスク | 明示的（Agent spawn） | 並行処理、context isolation |

---

## II. Skill ファイル構造テンプレート

```markdown
# SKILL: [skill-name]

## メタデータ
- **Trigger**: [自動実行条件]
- **Category**: [Code Review | Testing | Documentation | Security | Performance]
- **Context**: [必要な context 情報]
- **Output**: [出力形式]

## 説明
[何をするスキルか、いつ使うのか]

## 前提条件
- [環境要件]
- [ツール依存]
- [ファイル構成]

## 実装ステップ
1. [ステップ 1]
2. [ステップ 2]
3. ...

## ツール使用
- [使用 tool 1]
- [使用 tool 2]

## エラーハンドリング
- [エラーケース 1]: [対応]
- [エラーケース 2]: [対応]

## 出力例
```
[出力サンプル]
```

## パフォーマンス
- **Typical runtime**: [秒数]
- **Context usage**: [トークン数 or %]
- **Success rate**: [%]
```

---

## III. 標準 Skill カタログ

### 1. code-review.md

```markdown
# SKILL: code-review

## メタデータ
- **Trigger**: "Review this PR", "Code review needed", "Check code quality"
- **Category**: Code Review
- **Output**: Markdown report with style/security/performance issues

## 説明
PR diff に対して自動的に style, security, performance レビューを実施。
複数チェッカーを並行実行し、統合レポートを生成。

## 実装ステップ
1. PR diff を取得 (github__pull_request_read method: get_diff)
2. ESLint/Black 実行 (Bash)
3. SAST scan (Bash: npm audit / pip check)
4. Performance analysis
5. 統合レポート生成

## 出力例
```
# Code Review Report: PR #123

## Style Issues
- Line 45: Missing semicolon (ESLint)
- Indentation: 2 spaces expected, 4 found

## Security Issues
- Line 78: SQL injection risk detected (SAST)
- Hardcoded API key in config

## Performance
- No optimization opportunities detected

## Recommendations
1. Add error handling for API calls
2. Cache expensive computations
```

## 典型的な実行時間: 15-30 秒
## Context usage: 5-10%
```

### 2. test-coverage.md

```markdown
# SKILL: test-coverage

## メタデータ
- **Trigger**: "Check test coverage", "Coverage report", "Test analysis"
- **Category**: Testing
- **Output**: Coverage report + gap analysis

## 説明
テストカバレッジを計測し、未テスト領域を特定。
テスト戦略の改善提案を生成。

## 実装ステップ
1. Jest/pytest コマンド実行 (Bash)
2. Coverage report パース
3. Gap analysis（未カバー行抽出）
4. Recommendation 生成

## 出力例
```
# Test Coverage Report

## Overall: 73.5%
- Statements: 75%
- Branches: 68%
- Functions: 80%
- Lines: 73%

## Low Coverage Areas
- src/api/auth.ts: 45% (needs integration tests)
- src/utils/crypto.js: 52% (missing edge cases)

## Recommendations
1. Add integration tests for auth flows
2. Increase edge case coverage for crypto utils
3. Target: 85% within 2 sprints
```

## 典型的な実行時間: 20-40 秒
## Context usage: 8-12%
```

### 3. security-audit.md

```markdown
# SKILL: security-audit

## メタデータ
- **Trigger**: "Security review", "Audit for vulnerabilities", "Check dependencies"
- **Category**: Security
- **Output**: Security report + remediation plan

## 説明
依存関係のセキュリティスキャン、コード脆弱性検査、secrets 検出を実施。

## 実装ステップ
1. Dependency scan (npm audit, pip check)
2. SAST (Static Analysis)
3. Secret scanning (github__run_secret_scanning)
4. CVE lookup
5. Remediation plan 生成

## 出力例
```
# Security Audit Report

## Critical Issues: 2
1. axios@0.21.1 - CVE-2021-41773 (RCE)
   Fix: Upgrade to axios@1.4.0

2. lodash@4.17.20 - Deep prototype pollution
   Fix: Upgrade to lodash@4.17.21

## High Issues: 3
...

## Secrets Detected: 0 ✓

## Remediation Timeline
- Critical: Fix within 7 days
- High: Fix within 14 days
```

## 典型的な実行時間: 30-60 秒
## Context usage: 10-15%
```

### 4. doc-generator.md

```markdown
# SKILL: doc-generator

## メタデータ
- **Trigger**: "Generate API docs", "Document this", "Create README"
- **Category**: Documentation
- **Output**: Markdown documentation files

## 説明
コードから自動的に API ドキュメント、README を生成。
型情報・docstring から詳細記述を抽出。

## 実装ステップ
1. ソースコード解析（型・function signature 抽出）
2. docstring/comment パース
3. 例示コード抽出
4. Markdown 生成
5. ファイル出力（docs/ ディレクトリ）

## 出力例
```markdown
# API Documentation - auth.ts

## Functions

### `authenticate(username: string, password: string): Promise<User>`
Authenticate user with username/password.

**Parameters:**
- `username` (string): User email or username
- `password` (string): Plain text password

**Returns:** User object with authentication token

**Throws:** 
- `InvalidCredentialsError` if credentials are incorrect
- `UserNotFoundError` if user does not exist

**Example:**
```typescript
const user = await authenticate('user@example.com', 'password');
console.log(user.token);
```
```

## 典型的な実行時間: 10-20 秒
## Context usage: 5-8%
```

### 5. performance-profiler.md

```markdown
# SKILL: performance-profiler

## メタデータ
- **Trigger**: "Profile performance", "Check bottlenecks", "Optimize code"
- **Category**: Performance
- **Output**: Performance report + optimization suggestions

## 説明
実行時性能の詳細分析。CPU/Memory/Network ボトルネック特定。

## 実装ステップ
1. Profiler 実行（Node: clinic.js, Python: py-spy）
2. ホットスポット特定
3. Memory leak 検査
4. I/O 遅延分析
5. 最適化提案生成

## 出力例
```
# Performance Profile Report

## CPU Hotspots
1. `processLargeArray()` - 45% CPU
   Location: src/processors/bulk.js:234
   Suggestion: Use worker threads for parallelization

2. `validateSchema()` - 28% CPU
   Suggestion: Cache validation schema, use `ajv` compiler

## Memory
- Peak: 245MB
- Leak detected: No
- Top consumers: React components (82MB), Redux store (45MB)

## Network
- Longest request: 2.3s (API to /data endpoint)
- Suggestion: Implement caching, pagination

## Optimization Priority
1. High: Parallelize array processing (est. 30% speedup)
2. Medium: Cache validation schema (est. 20% CPU reduction)
```

## 典型的な実行時間: 40-120 秒
## Context usage: 12-18%
```

---

## IV. Skill 使用ルール

### 自動実行の条件

```
Skill match = (Trigger pattern in user request) OR (Context pattern detected)
   ├─ Explicit: "Review this PR" → code-review.md
   ├─ Explicit: "Check coverage" → test-coverage.md
   ├─ Implicit: PR created → code-review.md auto-trigger（オプション）
   └─ Implicit: Test failure → test-coverage.md auto-trigger（オプション）
```

### Context の優先順位

1. **Explicit trigger** （ユーザー明示指示）→ 最優先
2. **Implicit match** （コンテキスト自動検出）→ 確認後実行
3. **Fallback** （Skill 不一致）→ Main Agent logic

### 複数 Skill の並行実行

```
ユーザー: "Full code quality check"
    ↓
┌─ code-review.md
├─ test-coverage.md  ← Parallel execution
└─ security-audit.md
    ↓
Results aggregation → Final report
```

---

## V. Skill 開発ガイド

### チェックリスト

- [ ] SKILL.md ファイル作成（`.claude/skills/` に配置）
- [ ] トリガー条件を明確に定義
- [ ] 前提条件・依存 tool を記述
- [ ] エラーハンドリング実装
- [ ] 出力形式（Markdown）を統一
- [ ] 典型実行時間・context usage 測定
- [ ] チームレビュー（設計段階）
- [ ] テスト実装（単体テスト）

### 命名規則

```
skill-[機能名].md
例：
  - skill-code-review.md
  - skill-test-coverage.md
  - skill-security-audit.md
  - skill-performance-profiler.md
```

### Skill 間の依存関係

避けるべき：
```
code-review.md → test-coverage.md → security-audit.md （直列依存）
```

推奨：
```
code-review.md
test-coverage.md        （独立、並行実行可）
security-audit.md
```

---

## VI. Skill 共有・再利用

### チーム内共有

```bash
# .claude/skills/ に配置（Git コミット）
.claude/skills/
├── code-review.md
├── test-coverage.md
└── [共有 skills...]

# チーム全員が自動アクセス可能
```

### マルチプロジェクト共有

```bash
# Shared skill library リポジトリ
git clone https://github.com/org/shared-skills-repo.git

# プロジェクト .claude/settings.json で登録
{
  "skills": {
    "paths": [
      "./.claude/skills/",
      "../shared-skills-repo/skills/"
    ]
  }
}
```

### GitHub Marketplace（今後の展開）

- Community Skill 公開・発見
- Version management
- Auto-update

---

## VII. パフォーマンス・Context 最適化

### Skill の Context コスト最小化

| 最適化手法 | コスト削減 |
|---|---|
| Tool result compress | 20-30% |
| Output summarize | 10-15% |
| Caching intermediate results | 30-40% |
| Lazy evaluation（tool 呼び出し遅延） | 15-25% |

### 実装例（code-review.md の最適化）

```markdown
## ステップ 1: Diff サイズ制限
ファイル数 > 50 or Diff行数 > 1000 の場合は segment

## ステップ 2: Tool 呼び出し最小化
- ESLint: --format=json でパース効率化
- npm audit: 重大度フィルタで結果絞る

## ステップ 3: 結果圧縮
- Issue summary（詳細ではなく要点）
- 同種エラーはグループ化
```

---

最終更新: 2026-05-06
参照: AGENT.md, DESIGN.md, HOOKS.md
