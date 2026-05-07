# SKILLS.md - Skillsフレームワーク詳細ガイド

**対象**: Claude Code Skills & Custom Skills  
**更新日**: 2026-05-07  
**参考**: https://code.claude.com/docs/en/skills, https://mikhail.io/2025/10/claude-code-skills/

---

## 1. Skills の基本概念

### 定義

**Skill** = 再利用可能な、ファイルシステムベースのリソース

```
Skill = {
  YAML Frontmatter (メタデータ)      ← Claude が「何をする？」を判定
  ↓
  Markdown Instructions (詳細)      ← Skill実行時にロード
  ↓
  Scripts & References (実装)       ← 実際の処理
}
```

### ライフサイクル

```
1. DISCOVERY  → CLaudeが.claude/skills/ をスキャン
    ↓
2. MATCHING   → ユーザーリクエスト ↔ Skillの frontmatter をマッチング
    ↓
3. LOADING    → マッチしたSkill の SKILL.md を完全ロード
    ↓
4. EXECUTION  → Markdown指示に従い、scripts/ を実行
    ↓
5. REPORTING  → 結果をユーザーに報告
```

---

## 2. ファイル構造

### ディレクトリレイアウト

```
.claude/skills/
├── orchestrate-parallel/
│   ├── SKILL.md                  ← メタデータ + 実行指示
│   ├── scripts/
│   │   ├── orchestrator.py       ← 並列実行ロジック
│   │   └── task_distributor.py   ← タスク分配
│   └── references/
│       └── subagent-spec.md      ← Subagent仕様書
│
├── context-optimize/
│   ├── SKILL.md
│   ├── scripts/
│   │   └── compaction.py
│   └── references/
│       └── token-budgeting.md
│
├── mcp-healthcheck/
│   ├── SKILL.md
│   ├── scripts/
│   │   ├── health_checker.py
│   │   └── diagnostics.sh
│   └── references/
│       └── mcp-server-list.yaml
│
└── [新規追加...]
```

### SKILL.md の構造

```markdown
---
name: skill-identifier
description: ユーザーリクエスト時に表示される説明
triggers:
  - user_request: "キーワード1"
  - user_request: "キーワード2"
  - complexity: "特定の複雑性パターン"
model: claude-opus-4-7  # オプション：別モデルを指定
timeout: 300  # 秒（オプション）
permissions:
  - Read
  - Bash
  - Write
---

# Skill実行フロー

## Step 1: 前提条件確認
[詳細指示...]

## Step 2: メイン処理
[詳細指示...]

## Step 3: 検証
[詳細指示...]

## 成功時の報告
[フォーマット指定...]
```

---

## 3. システム提供Skills（既存）

### 1. `/session-start-hook`

**目的**: セッション初期化

**frontmatter**:
```yaml
name: session-start-hook
description: 新しいセッション開始時の初期化処理
triggers:
  - event: session_start
timeout: 10
permissions:
  - Read
```

**用途**: 環境確認、デフォルト設定ロード

---

### 2. `/update-config`

**目的**: 設定ファイル更新

**使用例**:
```
/update-config add permission:Bash=git commands
/update-config set model=claude-opus-4-7
/update-config move permission Bash global→user
```

---

### 3. `/security-review`

**目的**: セキュリティレビュー実施

**実行フロー**:
1. 現在のコード変更をスキャン
2. OWASP Top 10 チェック
3. シークレット検出
4. レポート生成

---

### 4. `/simplify`

**目的**: コード最適化・リファクタリング提案

**チェック項目**:
- [ ] コード重複
- [ ] 未使用変数
- [ ] 複雑度削減機会
- [ ] パフォーマンス最適化

---

### 5. `/claude-api`

**目的**: Claude API統合サポート

**活用例**:
- Prompt caching 設定
- Model マイグレーション（4.5→4.7）
- Batch API 導入

---

### 6. `/review`

**目的**: PR レビュー実施

**フロー**:
1. PR変更差分を取得
2. 複数の観点でレビュー（機能、品質、セキュリティ）
3. コメント・提案を生成

---

## 4. カスタムSkills（プロジェクト定義）

### Skill: `orchestrate-parallel`

**SKILL.md 完全版**:
```yaml
---
name: orchestrate-parallel
description: 複数の独立したタスクをSubagent並列実行（2-3個推奨）
triggers:
  - user_request: "並列で処理"
  - user_request: "同時に実行"
  - user_request: "複数の*を並行処理"
  - complexity: "independent_tasks_3plus"
model: claude-opus-4-7
timeout: 300
permissions:
  - Read
  - Bash
  - Agent  # Subagent起動
---

# 複数タスク並列実行 Skill

## メタデータ
- **実行時間**: 通常 30-120秒（タスクの複雑度による）
- **最大並列数**: 3（コンテキスト予算の限界）
- **失敗時動作**: 部分失敗は許容、全失敗のみエスカレーション

## フロー

### Step 1: タスク分解と依存関係分析
```python
# scripts/task_analyzer.py
tasks = parse_user_request(request)
dependencies = analyze_dependencies(tasks)

if not is_independent(dependencies):
    suggest_serial_execution()
    return

if len(tasks) > 3:
    split_into_batches(tasks, batch_size=3)
```

### Step 2: Subagent割り当て
```python
# scripts/subagent_allocator.py
assignments = {
    "researcher_task": ["Researcher_v1", "Researcher_v2"],
    "implementation_task": ["Implementer_v1"],
}

verify_context_budget(assignments)  # 予算確認
```

### Step 3: 並列起動（async実行）
```python
# scripts/orchestrator.py
results = await asyncio.gather(
    execute_subagent("researcher_api_docs_v1", task1),
    execute_subagent("researcher_api_docs_v2", task2),
    execute_subagent("implementer_core_v1", task3),
)
```

### Step 4: 結果統合
```python
# scripts/result_aggregator.py
final_report = {
    "task1_result": results[0],
    "task2_result": results[1],
    "task3_result": results[2],
    "success_rate": 3/3,
    "total_time": elapsed_time,
}
```

## 成功時の報告形式

```markdown
## 並列実行完了

### 実行結果
| タスク | Subagent | ステータス | 実行時間 |
|---|---|---|---|
| API分析 | researcher_api_docs_v1 | ✅ PASS | 42s |
| データベース設計 | researcher_db_schema_v1 | ✅ PASS | 38s |
| バックエンド実装 | implementer_core_v1 | ✅ PASS | 55s |

### 統計
- **並列度**: 3
- **総実行時間**: 55s（短縮: ~110s - 55s = 55秒節約）
- **コンテキスト使用率**: 32%

### 次のステップ
[統合テスト実行 or Reviewer起動等]
```

## トラブルシューティング

**Q: タスクが完了しない（タイムアウト）**
→ `timeout: 300` を増加、またはタスク分割

**Q: コンテキスト不足エラー**
→ 並列数を 3 → 2 に削減

**Q: 部分失敗（1/3が失敗）**
→ 失敗したタスク単体で `/orchestrate-parallel` 再実行
```

---

### Skill: `context-optimize`

**目的**: トークン消費を30%以下に最適化

**frontmatter**:
```yaml
name: context-optimize
description: セッションコンテキスト使用率を最適化（30%以下目標）
triggers:
  - user_request: "コンテキスト最適化"
  - system_event: "context_usage > 40%"
timeout: 60
permissions:
  - Read
  - Edit
  - Bash
```

**実行フロー**:

```markdown
## コンテキスト最適化フロー

### Step 1: 現在の使用率を把握
```bash
# scripts/context_analyzer.py
current_usage = measure_context_usage()
print(f"Current: {current_usage}% (target: ≤30%)")
```

### Step 2: 最適化アクション実施
```python
if usage > 60:
    # 激しく圧縮
    compress_conversation_history(ratio=0.7)
    delete_obsolete_references()
    
elif usage > 40:
    # 中程度の圧縮
    compress_old_messages(before_timestamp)
    move_to_discoveries_file()
```

### Step 3: 重要情報の永続化
```python
# .claude/DISCOVERIES.md に記録
with open(".claude/DISCOVERIES.md", "a") as f:
    f.write(important_findings)
```

### Step 4: 検証
```bash
# 最適化後の使用率確認
pytest tests/context_management.py
```

## 成功指標
- 使用率が 40% → 28% に低下（例）
- セッション継続可能（新Subagent起動可能）
```

---

### Skill: `mcp-healthcheck`

**目的**: MCP Server接続確認・診断

**frontmatter**:
```yaml
name: mcp-healthcheck
description: MCP Server接続状態と応答性能を診断
triggers:
  - user_request: "MCP確認"
  - system_event: "mcp_startup"
  - scheduled: "every 30 minutes"
timeout: 30
permissions:
  - Bash
  - Read
```

**チェック項目**:
```bash
# scripts/mcp_health_checker.sh
#!/bin/bash

echo "🔍 MCP Health Check"
echo "===================="

# 1. TCP接続確認
echo -n "AWS MCP Server: "
timeout 2 bash -c </dev/tcp/mcp-aws.local/8080 && echo "✅ OK" || echo "❌ FAIL"

# 2. 認証確認
echo -n "Auth Token: "
curl -s -H "Authorization: Bearer $MCP_AUTH_TOKEN" \
     http://mcp-aws.local/health && echo "✅ VALID" || echo "❌ INVALID"

# 3. レスポンス時間
echo -n "Latency: "
time curl -s http://mcp-aws.local/health > /dev/null
```

---

## 5. Skill作成テンプレート

### 最小構成（すぐに使える）

```markdown
---
name: my-skill
description: [何をするのか、20文字以内]
triggers:
  - user_request: "キーワード"
timeout: 120
permissions:
  - Read
  - Bash
---

# [Skill名]

## ステップ1: 前提確認
[必要な条件をチェック]

## ステップ2: メイン処理
[具体的な手順]

## ステップ3: 検証
[結果が正しいか確認]

## 成功時の報告
[ユーザーへの出力フォーマット]
```

### 拡張構成（スクリプト + リファレンス）

```
my-skill/
├── SKILL.md
├── scripts/
│   ├── main.py
│   ├── utils.py
│   └── config.yaml
└── references/
    ├── api-docs.md
    └── example-output.json
```

---

## 6. Skill呼び出しパターン

### パターン1: ユーザーが明示的に呼び出し
```
ユーザー: "並列で3つのAPIを分析して"
    ↓
Claude: "orchestrate-parallel スキルを使用します"
    ↓
[Skill実行]
```

### パターン2: システムイベントで自動トリガー
```
コンテキスト使用率が 45% を超える
    ↓
システム: "context-optimize スキルを自動起動"
    ↓
[Skill実行]
```

### パターン3: 複合実行（チェーン）
```
タスク1: orchestrate-parallel (3つのSubagent並列起動)
    ↓ 結果統合後
タスク2: security-review (セキュリティチェック)
    ↓ レポート生成後
タスク3: context-optimize (最適化)
```

---

## 7. ベストプラクティス

### DO ✅

- [ ] 1つの Skill = 1つの明確な責任
- [ ] frontmatter に複数の `triggers` を指定（ユーザー表現の多様性対応）
- [ ] `timeout` を現実的な値に設定
- [ ] scripts/ はテストカバレッジ ≥80%
- [ ] references/ に外部仕様書を配置

### DON'T ❌

- [ ] Skill内で別のSkillを呼び出さない（ネスト禁止）
- [ ] frontmatter を複雑にしない（最大3つのトリガー推奨）
- [ ] スクリプトを SKILL.md に埋め込まない（分離必須）
- [ ] プレースホルダ（TODO等）を残さない

---

## 8. トラブルシューティング

### Q: Skillが自動トリガーされない

**原因**: `triggers` のキーワードマッチング失敗

**解決**:
```yaml
triggers:
  - user_request: "並列"      # ← より一般的にする
  - user_request: "複数タスク"
  - complexity: "independent_tasks_3plus"
```

### Q: Skillの実行時間が長い

**対応**:
1. `timeout` 値を増加
2. スクリプトのプロファイリング実施
3. 非同期処理を導入（`asyncio` 等）

### Q: Skillが権限不足で失敗

**確認**:
```yaml
permissions:
  - Read       # ファイル読み込み
  - Edit       # ファイル編集
  - Write      # 新規作成
  - Bash       # コマンド実行
  - Agent      # Subagent起動
```

---

**最終チェック**:
- [ ] すべてのSkillテンプレートは完成形か？
- [ ] スクリプトはダミー実装でなく実行可能か？
- [ ] トラブルシューティングは実践的か？

✅ **本SKILLS.md はSkills構築の完全ガイド**
