# CLAUDE.md - プロジェクト規約・制限事項

**対象**: 高性能AIエージェント環境（2026年4月版）

## コーディング規約

### 1. Python/TypeScript における Agent SDK の使用

```python
# Claude Agent SDK を使用（2026年4月推奨）
from anthropic import Anthropic

# Managed Agents を活用する場合
client = Anthropic(api_key="your-key")
response = client.agents.create(
    name="orchestrator",
    instructions="You are a specialized orchestrator agent...",
    tools=[...]
)
```

### 2. ツール定義の標準化

すべてのツール定義は以下の形式に統一：

```json
{
  "name": "tool_name",
  "description": "Clear, concise description (max 200 chars)",
  "input_schema": {
    "type": "object",
    "properties": {},
    "required": []
  }
}
```

### 3. エラーハンドリングとリトライ

```python
# リトライ戦略（Exponential Backoff）
max_retries = 3
retry_delay = 1  # 1秒から開始

for attempt in range(max_retries):
    try:
        result = execute_tool(...)
        break
    except Exception as e:
        if attempt < max_retries - 1:
            time.sleep(retry_delay * (2 ** attempt))
        else:
            raise
```

## 運用制限事項

### ハーネス境界（Non-Negotiable）

1. **モデル選択**
   - 推論: Claude Opus 4.7（大規模タスク）/ Claude Sonnet 4.6（標準）
   - 実行: Claude Haiku 4.5（軽量・頻繁なツール呼び出し）
   - マルチエージェント: Managed Agents SDK必須

2. **トークン制限**
   - 単一セッション: 最大200,000トークン
   - コンテキストウィンドウ圧縮: 超過時は自動有効化
   - キャッシング: 4Kトークン以上のシーケンスで有効

3. **セキュリティ境界**
   - すべてのツール呼び出しは監査ログに記録（必須）
   - エージェント間通信は認証付きAPI経由のみ
   - 機密データはメモリシステムで暗号化

4. **自律性制限**
   - Human-in-the-loop チェックポイント:
     - 金銭取引: 100ドル以上は必須確認
     - データ削除: すべて事前確認必須
     - 外部API呼び出し: 初回は必須確認

### データパイプライン要件

```python
# 必須: パイプライン品質検証
def validate_data_pipeline():
    checks = [
        "data_freshness",      # リアルタイム性（< 5分）
        "schema_validation",   # スキーマ一致確認
        "null_handling",       # NULL値処理定義
        "error_recovery"       # エラー時の回復戦略
    ]
    return all(check_passed(c) for c in checks)

assert validate_data_pipeline(), "Pipeline validation failed"
```

## API 呼び出し仕様

### Managed Agents API呼び出し例

```python
# Beta header required (2026-04-01)
client = Anthropic(
    default_headers={"anthropic-beta": "managed-agents-2026-04-01"}
)

# エージェント作成
agent = client.agents.create(
    model="claude-opus-4-7",
    instructions="You are...",
    tools=[{...}],
    tool_choice="auto"
)

# エージェント実行
response = client.agents.messages.create(
    agent_id=agent.id,
    messages=[{"role": "user", "content": "..."}],
    max_tokens=4096
)
```

## ログとモニタリング

### 必須ログ項目

```json
{
  "timestamp": "ISO-8601",
  "agent_id": "...",
  "tool_name": "...",
  "input": "...",
  "output": "...",
  "execution_time_ms": 0,
  "status": "success|error|timeout",
  "error_message": "...",
  "user_id": "...",
  "session_id": "..."
}
```

## パフォーマンス基準

| メトリクス | 目標値 | 測定方法 |
|----------|--------|--------|
| エージェント応答時間 | < 5秒（P95） | Cloudwatch/Datadog |
| ツール実行成功率 | > 99.5% | ログ分析 |
| コンテキスト圧縮率 | 70%以上 | キャッシュヒット率 |
| 監査ログ完全性 | 100% | ランダムサンプリング検証 |

## 禁止事項（MUST NOT）

1. ❌ モデルのダウングレード（Opus4.7未満）をマルチエージェント環境で使用
2. ❌ Human-in-the-loopチェックポイントのバイパス
3. ❌ 監査ログの削除または改ざん
4. ❌ 未検証のデータパイプラインで本番実行
5. ❌ エージェント間通信の暗号化なし
6. ❌ プロンプトインジェクション対策なしでのユーザー入力処理

## 推奨事項（SHOULD）

1. ✅ セッションごとの明示的な目標定義
2. ✅ 各ツール呼び出しの実行時間記録
3. ✅ 月1回のセキュリティ監査
4. ✅ 四半期ごとのパフォーマンス評価
5. ✅ エージェント間の責任分離文書化

---

**最終更新**: 2026年4月23日
