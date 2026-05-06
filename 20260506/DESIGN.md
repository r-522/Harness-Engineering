# DESIGN.md - 設計原則・アーキテクチャガイド

## I. 適用する設計原則

### A. SOLID 原則

| 原則 | 適用 | 例 |
|---|---|---|
| **S**ingle Responsibility | 1 クラス/関数 = 1 責務 | Skill: code-review ≠ doc-generator |
| **O**pen/Closed | 拡張に開く、修正に閉じる | Hook ライブラリは追加可能、既存修正不可 |
| **L**iskov Substitution | Subtype を交換可能に | Subagent インターフェース統一 |
| **I**nterface Segregation | 不要な依存を避ける | 小さな特化 tool より大き汎用 tool 避ける |
| **D**ependency Inversion | 抽象に依存 | Settings / Skill catalog の抽象化 |

### B. DRY 原則（Don't Repeat Yourself）

**例外**:
- 3 行以下のコード重複は許容（無駄な抽象化回避）
- 小規模スクリプトは独立ファイル（後の保守を優先）

### C. KISS 原則（Keep It Simple, Stupid）

**実装規則**:
- 機能を最小限に（追加要件を予想しない）
- エラーハンドリングは境界（外部 API, User input）のみ
- 内部ロジックのバリデーション禁止

### D. AOP 原則（Aspect-Oriented Programming）

**Hook による横断的関心事の実装**:

```
Concerns:
├─ Logging (SessionStart/SessionEnd hook)
├─ Formatting (PostToolUse hook)
├─ Permission check (PreToolUse hook)
└─ Error handling (StopFailure hook)
```

各関心事は **`settings.json` hook** で独立・再利用可能に。

---

## II. アーキテクチャパターン

### パターン 1: Main Agent + Specialized Subagents

```
┌─ Main Agent ──────────────────────────────┐
│  ・Task coordination                       │
│  ・Context management                     │
│  ・User interaction                       │
└──────────┬──────────────────────────────┘
           │
      ┌────┼────┐
      ▼    ▼    ▼
    ┌──┐┌──┐┌──┐
    │S1││S2││S3│  (Subagents: Isolated contexts)
    └──┘└──┘└──┘
     │    │    │
     └────┼────┘
          ▼
    ┌──────────────┐
    │Result Merge  │
    │& Output      │
    └──────────────┘
```

**特徴**:
- Subagent 並行実行（処理時間削減）
- 各 Subagent context isolation（エラー局所化）
- Main Agent が統合・出力

### パターン 2: Skills-First Execution

```
User Request
    ▼
┌─ Skill Catalog ────────────────┐
│ [code-review]                  │
│ [test-coverage]                │
│ [security-audit]               │
│ [doc-generator]                │
│ [performance-profiler]         │
└─────────────────────────────┬──┘
                              │
                    Match skill?
                    ├─ Yes: Execute
                    └─ No: Fallback to Main Agent logic
```

**特徴**:
- Skill は `.claude/skills/SKILL.md` で自動検出
- Context match で自動実行
- Main Agent logic 不要（設定駆動）

### パターン 3: Hooks-Based Automation

```
Claude Code Lifecycle:
│
├─ SessionStart ──→ [Init hooks] ──→ Setup environment
│
├─ UserPromptSubmit ──→ [Pre hooks] ──→ Validate input
│
├─ PreToolUse ──→ [Tool-specific hooks] ──→ Permission check
│
├─ Tool execution ──→ [PostToolUse] ──→ Auto-format
│
├─ Stop ──→ [Summary hooks] ──→ Generate report
│
└─ SessionEnd ──→ [Cleanup hooks] ──→ Delete temp files
```

**特徴**:
- Automation は **設定駆動**（コード不要）
- Hook は shell command / HTTP / LLM prompt 可
- Async execution 対応

---

## III. ディレクトリ構成ガイド

### 標準構成

```
project-root/
├── .claude/
│   ├── CLAUDE.md              # Shared rules (auto-loaded)
│   ├── settings.json          # Team-shared config
│   ├── settings.local.json    # Personal overrides (untracked)
│   ├── skills/
│   │   ├── code-review.md
│   │   ├── test-coverage.md
│   │   ├── security-audit.md
│   │   ├── doc-generator.md
│   │   └── performance-profiler.md
│   └── hooks/                 # Hook script library
│       ├── format-on-write.sh
│       ├── git-status-check.sh
│       └── cleanup-on-end.sh
│
├── src/                       # Source code
│   ├── api/
│   ├── utils/
│   └── types/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
│
├── docs/
│   ├── API.md
│   ├── ARCHITECTURE.md
│   └── DEPLOYMENT.md
│
├── CLAUDE.md                  # Project reference (not auto-loaded)
├── AGENT.md
├── DESIGN.md
├── SKILLS.md
├── HOOKS.md
└── README.md
```

### 継承階層（設定読み込み順）

1. **Global** (`~/.claude/CLAUDE.md`, `~/.claude/settings.json`)
   - 全プロジェクト共通ルール
   - 個人用ツール設定

2. **Project** (`.claude/CLAUDE.md`, `.claude/settings.json`)
   - プロジェクト固有ルール
   - チーム共有設定
   - **Git でコミット**

3. **Local Override** (`.claude/settings.local.json`)
   - 個人用追加設定
   - ローカル環境変数
   - **`.gitignore` に追加**

---

## IV. Context Window 最適化戦略

### 危機管理フロー

```
Context Usage Monitor
│
├─ < 50%: Normal operation
│
├─ 50-70%: Yellow alert
│   ├─ 過去ログ削除
│   ├─ Tool result compress
│   └─ 新規 Subagent 制限
│
└─ > 70%: Red alert
    ├─ SessionEnd hook 実行
    ├─ Cache clear
    └─ 新タスク延期 or 新セッション開始
```

### 最小化テクニック

| テクニック | 削減率 | 実装 |
|---|---|---|
| **Prompt caching** | 30-40% | Built-in（API level） |
| **Skill external link** | 20% | See AGENT.md for details |
| **Hook externalize** | 15% | .claude/hooks/ ディレクトリ |
| **Log compression** | 25% | SessionEnd hook |
| **Binary encoding** | 10-20% | For large outputs |

---

## V. セキュリティ設計

### Threat Model

| 脅威 | 対策 |
|---|---|
| **Secret Exposure** | Pre-commit hook: .env check / Secret scanning |
| **Code Injection** | Input validation at boundary / Type checking |
| **Unauthorized Deploy** | Permission matrix in settings.json |
| **Data Loss** | Git backup / Confirm destructive ops |
| **Supply Chain** | Dependency scan / Subagent isolation |

### 実装マトリックス

```
Layer 1: User Input
  └─ Validate + Sanitize (Bash trap handler)

Layer 2: Tool Execution
  └─ Permission check (PreToolUse hook)

Layer 3: Data Processing
  └─ Type safety + Linting

Layer 4: Output
  └─ Secret scanning (PostToolUse hook)

Layer 5: Audit
  └─ Session log (SessionEnd hook)
```

---

## VI. スケーリング戦略

### チームサイズ別戦略

| サイズ | 設定方針 | 推奨ツール |
|---|---|---|
| **1 人** | `.claude/settings.local.json` のみ | Personal skills |
| **2-5 人** | `.claude/settings.json` + AGENT.md | Team skills + shared hooks |
| **6-20 人** | 上記 + permissions matrix | Subagent orchestration |
| **20+ 人** | 上記 + 組織レベル hook | Enterprise policies |

### マイクロサービス対応

複数プロジェクト間での Skill 共有：

```bash
# Shared skill library
git clone shared-skills-repo .claude/skills/

# Project override
.claude/settings.json → skills_path: ["./.claude/skills/", "./shared-skills/"]
```

---

## VII. テスト戦略

### Test Pyramid

```
        △
       / \        E2E (UI/API integration)
      / E \       (少数、遅い、費用大)
     /─────\
    /   I   \    Integration (Subagent collaboration)
   /─────────\   (中程度)
  /     U     \  Unit (Single function)
 /─────────────\ (多数、高速、安価)
```

### Test Type と実装

| Test Type | 対象 | 実装 |
|---|---|---|
| **Unit** | Skill/Subagent ロジック | pytest / Jest |
| **Integration** | Skill 間連携 / Subagent 並行 | Integration tests |
| **E2E** | Main Agent workflow | API/UI tests |
| **Security** | Secret exposure / injection | Dependency scan + static analysis |
| **Performance** | Context window usage / Subagent latency | Profiling hooks |

---

最終更新: 2026-05-06
参照: AGENT.md, SKILLS.md, HOOKS.md, CLAUDE.md
