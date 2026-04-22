# CLAUDE.md - プロジェクト規約と動作制限

このファイルはAIエージェント（Claude）が従うべきプロジェクト固有のルールを定義します。

## プロジェクト情報

**プロジェクト名**: Harness Engineering Configuration Hub 2026  
**構成対象**: エンタープライズレベルのマルチエージェント・オーケストレーションシステム  
**チーム規模**: 5-50名（エージェント開発チーム）  
**環境**: 本番環境（セキュリティ・ガバナンス重視）  

## 最重要原則

### 1. コード化ファースト
- **スキル、プロンプト、MCP設定は全てコード化する**
  - 隠蔽されたテンプレートや口頭での「約束」は禁止
  - 全てのエージェント設定はGitで管理され、PR・レビュー対象
  - 不備なく再現可能であることが必須

### 2. リポジトリが唯一の情報源
- スタイルガイド、命名規則、アーキテクチャ決定はこのリポジトリに記録
- 外部ドキュメント（Notion, Google Docs等）は補助資料のみ
- 紛争時はリポジトリ内のファイルが最優先

### 3. 段階的拡張（段階的Scaling原則）
- **最初は小さく始める**: 1-3 エージェントの検証システムから開始
- 十分な監視と自動テストが整ったら初めてスケール
- 「急いで全部構築」は避ける

## コーディング規約

### ファイル構成

```
.
├── agents/                    # エージェント定義
│   ├── orchestrator/         # オーケストレータエージェント
│   ├── reasoning/            # 推論エージェント
│   ├── retrieval/            # 検索・検索エージェント
│   ├── action/               # アクションエージェント
│   └── validator/            # 検証エージェント
├── skills/                    # スキル定義（Skill用）
│   ├── code_analysis.md
│   ├── security_check.md
│   └── ...
├── mcp/                       # Model Context Protocol設定
│   ├── github.mcp.json
│   ├── database.mcp.json
│   └── ...
├── prompts/                   # プロンプトテンプレート
│   ├── system_prompts/
│   ├── task_prompts/
│   └── ...
├── orchestration/            # オーケストレーション設定
│   ├── plan_do_evaluate.yaml
│   ├── adaptive_routing.yaml
│   └── ...
└── monitoring/               # 監視・ガバナンス
    ├── health_checks.yaml
    └── audit_logs.yaml
```

### 命名規則

#### エージェント名
- **形式**: `{role}_{specialization}_{version}`
- **例**:
  - `orchestrator_main_v1`
  - `reasoning_code_analysis_v1`
  - `validator_security_v2`

#### スキル名
- **形式**: `{domain}_{action}_{clarity}`
- **例**:
  - `github_pr_review`
  - `security_vulnerability_scan`
  - `database_query_optimize`

#### MCP設定ファイル
- **形式**: `{system_name}.mcp.json`
- **例**: `github.mcp.json`, `slack.mcp.json`

### プロンプト設計のルール

#### 1. 明示性（Explicitness）
```
✗ 悪い例
"エージェントらしく振る舞って"

✓ 良い例
"以下の順序で実行せよ: 1) 入力を解析、2) 関連ルールを確認、3) 実行、4) 結果を報告"
```

#### 2. コンテキスト配置
- クエリ・タスク指示は**コンテキストの後**に配置
- モデルの注意機構がクエリ周辺に集中する性質を利用

#### 3. XML タグの廃止
- 2026年最新のClaude Opus 4.7 は自然な構造化を理解
- 必要な場合だけマークダウンのセクション記法を使用

#### 4. Sub-Agent の文脈制御
```yaml
dispatcher:
  task: "ユーザーのリクエストを処理"
  agents:
    - id: "security_check"
      prompt: "コード内のセキュリティ脆弱性を検査"
      # DispatcherはSub-Agentの最終結果のみ参照
    - id: "performance_audit"
      prompt: "パフォーマンスボトルネックを分析"
  synthesizer: "結果を統合し、優先順位をつけた報告を生成"
```

## ガードレール（Guardrails）

### 1. 権限制限
- エージェントは本来の役割範囲内のツールのみ使用可能
- データアクセスは最小権限原則（Least Privilege）
  - Orchestrator エージェント: 読み取りのみ（決定権はなし）
  - Action エージェント: 割り当てられたリソースのみ操作可能
  - Validator エージェント: 全体を監査するが書き込み権なし

### 2. 出力検証
```yaml
output_validation:
  - rule: "sensitive_data_leak"
    pattern: "password|token|secret|api_key"
    action: "block_and_alert"
  - rule: "unsafe_command"
    pattern: "rm -rf|DROP TABLE|kill -9"
    action: "escalate_to_human"
```

### 3. リトライとタイムアウト
```yaml
resilience:
  max_retries: 3
  retry_backoff: "exponential"  # 2s, 4s, 8s
  timeout_seconds: 300
  circuit_breaker: "enabled"
```

## 監視とObservability

### 必須ログ項目
```json
{
  "timestamp": "ISO-8601",
  "agent_id": "orchestrator_main_v1",
  "task_id": "unique_id",
  "action": "plan|execute|evaluate",
  "input_tokens": 1500,
  "output_tokens": 800,
  "cost_usd": 0.042,
  "status": "success|warning|error",
  "human_review_required": false,
  "error_message": "if_any"
}
```

### 監視指標
- **レイテンシ**: エージェント間の応答時間
- **エラー率**: タスク失敗の割合
- **コスト**: 走行コスト（月次・年次集計）
- **Human Escalation Rate**: エスカレーション頻度

## Human-on-the-Loop ポリシー

### エスカレーション条件
以下の場合は自動的に人間レビューにエスカレーション：

1. **セキュリティ影響**: 権限変更、データ削除予定
2. **コスト閾値超過**: 単一タスク > $100
3. **エラーの反復**: 同じエラーが3回以上発生
4. **合意不在**: エージェント間で合意形成失敗
5. **意思決定の重要性**: ビジネス影響度が高い決定

### 人間レビューの最大時間
- 緊急タスク: 30分以内（又は自動リトライ）
- 通常タスク: 2時間以内

## セキュリティ要件

### 認証・認可
```yaml
security:
  auth_mechanism: "OAuth2 + Service Account"
  mcp_encryption: "TLS 1.3 required"
  agent_isolation: "separate_contexts"
  secret_management: "HashiCorp Vault or AWS Secrets Manager"
```

### 監査ログ
- 全エージェント操作のログ記録（不変ログ）
- 外部エージェントの変更は2つの署名必須（2FA principle）
- ログ保持期間: 最低12ヶ月

## テスト要件

### ユニットテスト
- 各エージェントスキルのテストカバレッジ: 最低80%
- プロンプトテンプレート: 複数の入力パターンで検証

### インテグレーションテスト
```yaml
test_scenarios:
  - name: "normal_path"
    agents: [orchestrator, reasoning, action, validator]
    expected_output: "validated_result"
  - name: "error_path"
    agents: [orchestrator, reasoning, validator]
    expected_output: "escalation_to_human"
  - name: "timeout_path"
    timeout_seconds: 10
    expected_output: "graceful_failure_with_partial_result"
```

### パフォーマンステスト
- レイテンシ: P95 < 60秒（通常タスク）
- スループット: >= 10タスク/分（並列オーケストレーション）

## デプロイメント規約

### 本番へのプロモーション
1. **ステージング環境で最低24時間の検証**
   - 本番相当の負荷テスト
   - セキュリティスキャン
   
2. **PR・レビュー必須**
   - 最低2名のレビュアー（異なる専門分野）
   - 全テスト合格
   - セキュリティレビュー合格

3. **段階的ロールアウト**
   ```yaml
   canary: 5%
   staged_rollout: 25% -> 50% -> 100%
   rollback_plan: "enabled"
   ```

## バージョン管理

### セマンティックバージョニング
- **MAJOR**: アーキテクチャ変更、互換性破棄
- **MINOR**: 新スキル追加、オーケストレーション拡張
- **PATCH**: バグ修正、最適化

### タグ規約
```
v{MAJOR}.{MINOR}.{PATCH}-{environment}
例: v1.0.0-prod, v1.1.0-staging
```

## チェックリスト：新しいスキル追加時

- [ ] スキル実装（agents/skills ディレクトリ）
- [ ] テストカバレッジ（最低80%）
- [ ] プロンプト定義が明示的か確認
- [ ] ガードレール設定（権限、出力検証）
- [ ] ドキュメント更新（使用方法、パラメータ）
- [ ] セキュリティレビュー（秘密情報リーク検査）
- [ ] パフォーマンステスト（レイテンシ、コスト）
- [ ] PR提出 & 最低2名のレビュー待機

## FAQ

**Q: Sub-Agent はいつ使うべき？**  
A: コンテキストウィンドウが逼迫（>80%）、または結果の中間検査が必要な場合。

**Q: XML タグは廃止？**  
A: 構造化が必要な場合のみマークダウンセクションを使用。自然言語で十分な場合はタグ廃止。

**Q: エージェント間通信はどのプロトコル？**  
A: MCP（Model Context Protocol）が標準。ファイアウォール外ではHTTP + TLS 1.3。

**Q: 監視データはどこに保管？**  
A: CloudWatch（AWS）またはDatadog。最低12ヶ月の保持期間。

---

**最終更新**: 2026-04-22  
**レビュー対象**: 全ファイル変更時のPRで確認必須
