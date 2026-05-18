# DESIGN.md - 設計原則とアーキテクチャパターン

**対象**: ソフトウェア設計・アーキテクチャ判定  
**最終更新**: 2026-05-13

---

## 1. 適用する設計原則

### SOLID 原則

| 原則 | 実装例 | チェック |
|------|------|--------|
| **S** (Single Responsibility) | 1ファイル = 1責任。Logger, Parser, Validator を分離 | コード行数 > 200 行なら分割検討 |
| **O** (Open-Closed) | 新機能追加は extension で、既存コード修正なし。Strategy パターン活用 | 新Skill追加時、既存Skill修正不要か確認 |
| **L** (Liskov Substitution) | 派生Agentは親Agentと互換性あり | Subagent入力・出力スキーマは継承可能か確認 |
| **I** (Interface Segregation) | 不要なメソッドに依存させない。細粒度インターフェース | MCP Server は必要な操作のみ公開 |
| **D** (Dependency Inversion) | 具体実装ではなく抽象（インターフェース）に依存 | `from abc import ABC` で Skill / Agent の基底クラス定義 |

### DRY (Don't Repeat Yourself)

- **適用範囲**: 3回以上同じロジック出現 → 関数・クラス化
- **例外**: マジックナンバーの小さな値（True/False）, 1〜2行の設定値

### YAGNI (You Aren't Gonna Need It)

- **原則**: 使っていない機能・ファイルは削除。将来の「かもしれない」で追加しない
- **例外**: 契約インターフェース（互換性必須）の public メソッドは残す

---

## 2. アーキテクチャパターン

### パターン1: Orchestrator Pattern（推奨）

**用途**: マルチAgent、複数の独立タスク

```
┌─────────────────┐
│ Main Orchestrator│
│ (Coordinator)   │
└────────┬────────┘
         │
    ┌────┴────┬────────┬────────┐
    ▼         ▼        ▼        ▼
  Agent-A  Agent-B  Agent-C  Agent-D
  (研究)    (実装)   (テスト)  (検証)
```

**特徴**:
- Main が各 Subagent に指示、結果を統合
- 各 Subagent は独立した Context（100k tokens）
- 並列実行で全体実行時間短縮

**コード例**: `src/orchestrators/main.py`

```python
from src.agents import researcher, developer, tester

class MainOrchestrator:
    async def execute(self, task):
        # PLAN: 分割
        research_task, impl_task = self.decompose(task)
        
        # ACT: 並列実行
        research = await researcher.run(research_task)
        implementation = await developer.run(impl_task)
        
        # VERIFY: 統合
        return self.integrate(research, implementation)
```

### パターン2: Specialist Routing Pattern

**用途**: ドメイン別タスク自動振り分け

```
User Input
    ↓
┌─────────────────────┐
│  Router/Dispatcher  │ ← 入力を分析し、相応Specialist へ
└─────────────────────┘
    ↓
   ┌─────────────┬──────────────┬──────────┐
   ▼             ▼              ▼          ▼
Backend-Spec Frontend-Spec DevOps-Spec ML-Spec
(Python)      (TypeScript)  (Terraform) (PyTorch)
```

**特徴**:
- ドメイン知識に特化した Specialist
- 1 Specialist = 1 ファイルスコープ（src/backend, src/frontend 等）
- Router が入力キーワード・ファイルパスで振り分け

### パターン3: Pipeline Pattern

**用途**: 線形フロー（前処理 → 処理 → 後処理）

```
Input → [Stage 1] → [Stage 2] → [Stage 3] → Output
           ↓          ↓          ↓
        Validate   Transform  Validate
```

**特徴**:
- 各 Stage は独立・交換可能
- 中間結果をキャッシュ可能（リトライ時効率向上）

**例**: データパイプライン
```
Raw Data → Parse → Validate → Enrich → Load → Analytics
```

---

## 3. ディレクトリ構成ガイド

```
project-root/
│
├── .claude/                    # Claude Code 設定（ベストプラクティス）
│   ├── CLAUDE.md               # コーディング規約・テストポリシー
│   ├── AGENT.md                # エージェント振る舞い
│   ├── DESIGN.md               # このファイル
│   ├── MCP.md                  # MCP 統合
│   ├── SUBAGENT.md             # Orchestration パターン
│   └── SKILLS.md               # Skills 定義
│
├── src/                        # ソースコード（本体）
│   ├── agents/                 # Subagent 定義
│   │   ├── __init__.py
│   │   ├── researcher_context.py    # 研究系 Agent
│   │   ├── developer_impl.py        # 実装系 Agent
│   │   └── tester_validation.py     # テスト系 Agent
│   │
│   ├── skills/                 # カスタムスキル
│   │   ├── plan_decompose.py
│   │   ├── context_estimate.py
│   │   └── safe_delete_check.py
│   │
│   ├── mcp/                    # MCP Server 接続
│   │   ├── __init__.py
│   │   ├── servers.yaml        # MCP Server 定義
│   │   ├── github_connector.py
│   │   └── aws_connector.py
│   │
│   ├── core/                   # 共通ライブラリ
│   │   ├── logger.py           # ロギング統一
│   │   ├── config.py           # 設定管理
│   │   ├── utils.py            # Utility 関数
│   │   └── exceptions.py       # カスタム例外
│   │
│   └── orchestrators/          # Main Orchestrator
│       ├── __init__.py
│       └── main.py
│
├── tests/                      # テストコード
│   ├── unit/                   # 単体テスト（テスト込みで読み込まない）
│   │   ├── test_agents.py
│   │   ├── test_skills.py
│   │   └── test_mcp.py
│   │
│   ├── integration/            # 統合テスト（複数Agent）
│   │   ├── test_orchestration.py
│   │   └── test_subagent_parallel.py
│   │
│   └── fixtures/               # テスト用データ・モック
│       ├── mock_agents.py
│       └── sample_data.json
│
├── config/                     # 設定ファイル
│   ├── settings.yaml           # アプリケーション全般設定
│   ├── agents.yaml             # Agent 定義・並列度・タイムアウト
│   └── skills.yaml             # Skill トリガー条件
│
├── docs/                       # ドキュメント（コード参照不可）
│   ├── ARCHITECTURE.md         # アーキテクチャ全体図
│   ├── ORCHESTRATION.md        # マルチAgent実行フロー
│   ├── PLAYBOOK.md             # トラブルシューティング
│   └── API.md                  # 外部API仕様
│
├── .env.local                  # ローカル開発用（.gitignore）
├── .env.example                # テンプレート（参考用）
├── .gitignore                  # Git除外設定
├── pyproject.toml              # Python 依存関係
├── requirements.txt            # Python パッケージ
├── package.json                # Node.js 依存関係（TypeScript 使用時）
└── Makefile                    # ローカル開発・テストコマンド
```

---

## 4. 命名規則の詳細

### Python（snake_case）

```python
# ✅ 正しい
def calculate_token_count(text):
    max_context_tokens = 400000
    return len(text.split())

class DataPipeline:
    pass

# ❌ 間違い
def calculateTokenCount(text):  # camelCase は使わない
    MaxContextTokens = 400000    # 定数なら UPPER_SNAKE_CASE
```

### TypeScript（camelCase）

```typescript
// ✅ 正しい
function calculateTokenCount(text: string): number {
    const maxContextTokens = 400000;
    return text.split(' ').length;
}

class DataPipeline {
    constructor() {}
}

// ❌ 間違い
function calculate_token_count(text: string) {}  // snake_case は使わない
const MAX_CONTEXT_TOKENS = 400000;               // 定数も camelCase
```

### ファイル名

```
✅ 正しい:
  - src/agents/researcher_context.py
  - src/skills/plan-decompose.ts
  - src/orchestrators/main-orchestrator.py

❌ 間違い:
  - src/agents/ResearcherContext.py     # PascalCase
  - src/skills/planDecompose.ts         # camelCase
  - src/orchestrators/MainOrchestrator  # 拡張子なし
```

### Subagent / Skill

```
命名ルール: {role}_{objective} (Subagent)
          {verb}_{noun}          (Skill)

✅ 正しい:
  - researcher_context
  - developer_backend
  - tester_e2e
  - plan_decompose
  - safe_delete_check

❌ 間違い:
  - agent1, agent2           # 番号不可
  - ResearcherContextAgent   # PascalCase
  - research                 # 短すぎる、目的不明確
```

---

## 5. Coupling と Cohesion

### 低 Coupling（疎結合）を目指す

```python
# ❌ 高 Coupling: ParserがDatabaseに直接依存
class Parser:
    def parse(self, data):
        db = Database()
        db.save(parsed_data)

# ✅ 低 Coupling: ParserはCallback経由で通知
class Parser:
    def __init__(self, on_parsed_callback):
        self.on_parsed = on_parsed_callback
    
    def parse(self, data):
        parsed = self._parse(data)
        self.on_parsed(parsed)  # 誰が処理するかは Parser 知らない
```

### 高 Cohesion（高凝聚）を目指す

```python
# ❌ 低 Cohesion: 関係ない機能が混在
class User:
    def validate_email(self): ...      # ユーザー管理
    def send_email(self): ...          # メール配信
    def log_to_db(self): ...           # ロギング
    def compress_image(self): ...      # 画像処理

# ✅ 高 Cohesion: 関連機能のみ
class User:
    def validate_email(self): ...      # ユーザー管理のみ
    def get_name(self): ...
    def update_profile(self): ...
```

---

## 6. リファクタリング ルール

**判定**: 以下のいずれかに該当したら実施

- [ ] **関数が 30行以上** → 機能分割
- [ ] **同じロジックが 3回以上** → 共通関数化
- [ ] **クラスメソッドが 10個以上** → クラス分割
- [ ] **循環複雑度 > 10** → 条件分岐を関数に抽出

**注意**: 「今後のために」という理由でリファクタリングしない。必要になったら実施（YAGNI）

---

## 参考

- SOLID Principles: https://en.wikipedia.org/wiki/SOLID
- Design Patterns: https://refactoring.guru/design-patterns
- Clean Code by Robert C. Martin
