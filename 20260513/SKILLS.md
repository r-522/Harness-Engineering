# SKILLS.md - Claude Code Skills 定義 & 実装ガイド

**対象**: カスタムスキル開発・統合  
**最終更新**: 2026-05-13

---

## 1. Claude Skills とは

**Skill**: Claude が特定のタスクを効率的に実行するための「内部プレイブック」。AI の知識と手順セットを統一フォーマットで定義する。

### Skills の特徴

| 特性 | 値 |
|------|------|
| **トークン消費** | 30〜50 tokens（起動時のみ） |
| **配置** | `.claude/skills/` または Claude.ai ダッシュボード |
| **認証** | 不要（内部ルール） |
| **実行範囲** | Claude.ai, Claude Code, Claude API |

### Skills vs MCP vs Plugins

| 比較軸 | **Skills** | **MCP** | **Plugins** |
|------|----------|--------|----------|
| 説明方法 | Markdown | Protocol | REST API |
| 外部アクセス | ❌ | ✅ | ✅ |
| トークンコスト | 低 | 中 | 高 |
| セットアップ | 簡単 | 中程度 | 複雑 |

**原則**: 「説明・手順」なら Skill / 「アクセス」なら MCP

---

## 2. Core Skills テンプレート

### 基本構造

```markdown
# {Skill Name}

**Purpose**: {目的の1〜2行説明}

**Trigger Conditions**:
- {条件1: キーワード、ファイルタイプ等}
- {条件2}

**How to Use**:
1. {ステップ1}
2. {ステップ2}
3. {ステップ3}

**Output Format**:
```json
{
  "status": "success|failure",
  "result": {...}
}
```

**Examples**: 
- {例1}
- {例2}
```

### 実装例1: `plan_decompose` Skill

**ファイル**: `.claude/skills/plan_decompose.md`

```markdown
# Plan Decompose

**Purpose**: 複雑なタスクを独立した Subagent に分割し、Context 予算を最適化する。

**Trigger Conditions**:
- ユーザーが複雑なタスク（10ステップ以上）を指示
- 見積もり Context > 400k tokens
- キーワード: 「統合」「複数機能」「並列」

**How to Use**:

1. **タスク分析**
   - 全ステップを列挙
   - 依存関係を図式化（DAG形式）
   
2. **Subagent 分割案**
   - 各 Subagent の責務を明確化
   - Context 割り当て（150k, 100k, 80k 等）
   
3. **並列度判定**
   - 独立タスク数 → 並列可能度
   - デフォルト: 最大 3 並列
   
4. **実装提案**
   - Agent 名: `{role}_{objective}`
   - ファイルパス: `src/agents/{agent_name}.py`

**Output Format**:
\`\`\`json
{
  "status": "success",
  "decomposition": {
    "tasks": [
      {"id": "T1", "title": "Research", "agent": "researcher_context", "context_tokens": 150000},
      {"id": "T2", "title": "Implementation", "agent": "developer_impl", "context_tokens": 100000}
    ],
    "dependencies": [
      {"from": "T1", "to": "T2"}
    ],
    "parallel_groups": [
      ["T1", "T2"]
    ],
    "total_estimated_tokens": 250000,
    "safety_margin": true
  }
}
\`\`\`

**Examples**:

1. **データパイプライン最適化**
   - Task: 「既存パイプラインの性能を3倍改善」
   - Decomposition:
     - Researcher: CloudWatch ログ分析
     - Optimizer: アルゴリズム改善
     - Tester: ベンチマーク検証

2. **Web アプリケーション開発**
   - Task: 「React + FastAPI で TODO アプリ構築」
   - Decomposition:
     - Frontend Dev: React コンポーネント
     - Backend Dev: FastAPI エンドポイント
     - Tester: E2E テスト & セキュリティレビュー
```

### 実装例2: `context_estimate` Skill

**ファイル**: `.claude/skills/context_estimate.md`

```markdown
# Context Estimate

**Purpose**: 大規模ファイル読み込み前に Token 消費量を見積もり、安全性を判定する。

**Trigger Conditions**:
- 大規模ファイル（> 10MB）の読み込み
- 複数ファイルの同時読み込み
- キーワード: 「Context」「メモリ」「トークン」

**How to Use**:

1. **ファイル分析**
   - ファイルサイズを特定
   - 行数・複雑度を見積もり
   
2. **Token 計算**
   - 平均: 1トークン ≈ 4文字（English）
   - Python コード: 1行 ≈ 50〜100トークン
   - 公式: Tokens ≈ (Chars / 4) × 1.2 (マージン)
   
3. **安全性判定**
   - 合計 > 400k tokens → ❌ 危険
   - 合計 < 400k tokens → ✅ 安全

**Output Format**:
\`\`\`json
{
  "status": "success",
  "estimate": {
    "file_size_bytes": 1048576,
    "estimated_tokens": 262144,
    "confidence": 0.85,
    "safety_assessment": "SAFE",
    "recommendation": "Single agent can handle"
  }
}
\`\`\`

**Examples**:

1. **単一 Python ファイル（200行）**
   - 見積もり: 10k tokens
   - 判定: ✅ SAFE

2. **モデル重み（100MB）**
   - 見積もり: 52.4M tokens
   - 判定: ❌ DANGEROUS (Subagent分割必須)
```

### 実装例3: `safe_delete_check` Skill

**ファイル**: `.claude/skills/safe_delete_check.md`

```markdown
# Safe Delete Check

**Purpose**: ファイル削除前に、バックアップ可能性・依存関係を確認する。

**Trigger Conditions**:
- ユーザーがファイル削除指示（`rm`, `delete` 命令）
- コマンド: `git rm`, MCP 削除ツール
- キーワード: 「削除」「remove」「clean up」

**How to Use**:

1. **依存関係確認**
   \`\`\`bash
   grep -r "import {filename}" src/
   grep -r "require({filename})" src/
   \`\`\`
   
2. **Git 履歴確認**
   \`\`\`bash
   git log --oneline {filename} | head -5
   \`\`\`
   
3. **バックアップ可能性**
   - ファイル最終更新: {timestamp}
   - Git で復旧可能: YES/NO
   - テストで保護: YES/NO
   
4. **ユーザー確認**
   - 削除対象を明示
   - 代替案を提示（リネーム、アーカイブ）

**Output Format**:
\`\`\`json
{
  "status": "requires_confirmation",
  "deletion_target": "src/legacy/old-parser.py",
  "dependency_check": {
    "internal_imports": [
      "src/handlers/handler.py:12"
    ],
    "external_usage": 2,
    "test_coverage": false
  },
  "backup_status": {
    "git_recoverable": true,
    "last_commit": "abc1234",
    "last_modified": "2026-05-10T10:30:00Z"
  },
  "recommendation": "✅ Safe to delete (no active dependents). Backupable via git."
}
\`\`\`

**Examples**:

1. **不要な設定ファイル**
   - File: `.old-config.yaml`
   - Dependencies: 0
   - Judgment: ✅ SAFE to delete

2. **使用中のモジュール**
   - File: `src/core/auth.py`
   - Dependencies: 5 files, 12 imports
   - Judgment: ❌ NOT SAFE (依存関係あり)
```

---

## 3. Domain Skills（条件付きロード）

### Researcher Skills

```markdown
# Researcher Skill: Literature Analysis

**Purpose**: Web トレンドや GitHub リポジトリを分析し、技術選定・ベストプラクティスを提示。

**Trigger**: 「ベストプラクティスは？」「どのライブラリを使うべき？」

**Procedure**:
1. Web 検索（過去24時間のトレンド）
2. GitHub Star数・ Fork数で評価
3. Reddit / Twitter での言及数を集計
4. 総合スコアで TOP 3 を提示

**Output**: 技術選定レポート + 使用例
```

### Optimizer Skills

```markdown
# Optimizer Skill: Cost & Performance Analysis

**Purpose**: コード・インフラのパフォーマンス・コスト改善を提案。

**Trigger**: 「パフォーマンスを改善したい」「AWS コストを削減」

**Procedure**:
1. 計算複雑度分析（Big O 記法）
2. メモリプロファイリング結果確認
3. AWS CloudWatch メトリクス取得（MCP）
4. 改善提案を ROI 付きで提示

**Output**: 改善案 + 実装コード + 期待効果
```

### Reviewer Skills

```markdown
# Reviewer Skill: Security & Code Quality

**Purpose**: セキュリティ脆弱性・コード品質問題を検出。

**Trigger**: PR 作成・API 接続・認証コード

**Procedure**:
1. OWASP Top 10 チェック
2. 入力検証・SQL インジェクション確認
3. 認証情報の埋め込み検査（secret scanning）
4. コード複雑度（cyclomatic complexity）測定

**Output**: セキュリティレポート + 修正案
```

---

## 4. Meta Skills（内部用）

### `introspect_context`

```markdown
# Introspect Context

**Purpose**: 現在の Context 残量・Token 消費をリアルタイム追跡。

**Trigger**: 自動実行（定期的）/ ユーザーが「Context は？」と質問

**Output**:
\`\`\`json
{
  "total_tokens": 600000,
  "consumed_tokens": 320000,
  "remaining_tokens": 280000,
  "remaining_percent": 46,
  "safe_status": "SAFE",
  "subagent_headroom": true
}
\`\`\`
```

### `abort_graceful`

```markdown
# Abort Graceful

**Purpose**: Subagent 失敗時に安全に処理を中止・ロールバック。

**Trigger**: Subagent エラー / Timeout / ユーザーが Ctrl+C

**Procedure**:
1. 現在のファイル変更をロールバック（git checkout）
2. Subagent プロセスを安全に終了（signal SIGTERM）
3. ユーザーにエスカレーション報告
4. セッション状態をリセット

**Output**:
\`\`\`json
{
  "status": "aborted",
  "reason": "Agent timeout",
  "rollback": {
    "files_reverted": 3,
    "timestamp": "2026-05-13T10:30:00Z"
  }
}
\`\`\`
```

---

## 5. Skills の有効化・登録

### Claude Code で Skills を使用開始

**ステップ1**: Skills ファイルを `.claude/skills/` に配置

```bash
mkdir -p .claude/skills
cp plan_decompose.md context_estimate.md safe_delete_check.md .claude/skills/
```

**ステップ2**: `.claude/AGENT.md` で Skills を参照

```markdown
## 適用 Skills

### Core Skills（常時ロード）
- [plan_decompose](.claude/skills/plan_decompose.md)
- [context_estimate](.claude/skills/context_estimate.md)
- [safe_delete_check](.claude/skills/safe_delete_check.md)

### Domain Skills（条件付きロード）
- researcher_literature
- optimizer_cost
- reviewer_security
```

**ステップ3**: Claude Code セッション開始時に自動ロード

```bash
# Claude Code が起動時に .claude/skills/*.md を読み込む
claude code start --project /path/to/project
# → Core Skills が自動有効化
```

---

## 6. Skills の実装チェックリスト

新しい Skill を追加する際：

```markdown
- [ ] **命名規則**: kebab-case (例: `plan-decompose.md`)
- [ ] **Purpose**: 1〜2行で目的を明記
- [ ] **Trigger Conditions**: 3〜5個の明確な条件列挙
- [ ] **How to Use**: ステップバイステップで手順化（3〜7ステップ）
- [ ] **Output Format**: JSON スキーマで出力形式を定義
- [ ] **Examples**: 最低 2〜3個の実装例を提示
- [ ] **Brevity**: 全体 80〜150行（簡潔さを優先）
- [ ] **Cross-references**: AGENT.md / CLAUDE.md で参照
- [ ] **Testing**: 実際に Skill を試行し、動作確認
```

---

## 7. Skills の活用パターン

### パターン1: Plan → Skill → Subagent

```
User: 「大規模データパイプライン最適化」
  ↓
[plan_decompose Skill] → Subagent 分割案 생성
  ↓
Agent-Researcher + Agent-Optimizer (並列)
  ↓
[context_estimate Skill] → Token 安全性確認
  ↓
実装 & テスト
```

### パターン2: Skill が Subagent 結果を検証

```
Subagent-Developer: 実装完了
  ↓
[reviewer_security Skill] → セキュリティチェック
  ↓
（脆弱性あり）
  ↓
Subagent-Developer: 修正 → Skill で再検証
  ↓
✅ PASS
```

---

## 参考

- Extend Claude with Skills: https://code.claude.com/docs/en/skills
- The Complete Guide to Building Skills: https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf
- Claude Skills vs MCP: https://intuitionlabs.ai/articles/claude-skills-vs-mcp
