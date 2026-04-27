# PERFORMANCE.md - プロンプトキャッシング・コスト最適化ガイド

このドキュメントは、Claudeエージェント実行環境のパフォーマンス最適化とコスト削減戦略を詳細に規定します。

## 2026年の重大変更: TTL短縮への対応

### 背景
Anthropicは2026年初旬、プロンプトキャッシュのTTL（Time To Live）を **60分 → 5分** に短縮しました。

**影響**:
- 従来の実装で **30-60%のコスト増加**
- キャッシュ戦略の根本的見直しが必須
- 自動キャッシング依存度が低下

### 対応戦略

#### 戦略1: TTLを明示的に60分に設定
```json
{
  "messages": [...],
  "cache_control": {
    "type": "ephemeral",
    "ttl_seconds": 3600
  }
}
```

**実装例** (Python):
```python
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-opus-4.7",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": "[Large System Prompt - 10K tokens]",
                    "cache_control": {"type": "ephemeral"}
                },
                {
                    "type": "text",
                    "text": "[User Input]"
                }
            ]
        }
    ],
    system=[
        {
            "type": "text",
            "text": "[System Prompt]",
            "cache_control": {"type": "ephemeral"}
        }
    ],
    extra_headers={
        "anthropic-beta": "prompt-caching-2024-07-16"
    }
)
```

#### 戦略2: キャッシュ無効化の回避
**❌ ダメな例**:
```python
# 毎リクエスト新しいタイムスタンプが生成されるため、キャッシュミス確定
system_prompt = f"""
Current time: {datetime.now().isoformat()}
[Rest of prompt...]
"""
```

**✓ 良い例**:
```python
# タイムスタンプを動的部分に移動
system_prompt = """
[Large stable prompt...]
"""

# 動的部分（タイムスタンプ）はユーザーメッセージに含める
user_message = f"""
Current time: {datetime.now().isoformat()}
Task: [User request]
"""
```

#### 戦略3: キャッシュ対象の最適化
```yaml
キャッシュすべき:
  - システムプロンプト（安定、大容量）
  - 共有知識ベース（変更少）
  - 共通ライブラリ定義（再利用高）
  - API仕様ドキュメント（頻出参照）

キャッシュすべきでない:
  - ユーザー入力（毎回異なる）
  - 一時的なコンテキスト（再利用なし）
  - タイムスタンプ（常に変更）
```

---

## プロンプトキャッシング実装パターン

### パターン1: システムプロンプット大型キャッシュ

**適用**:
- 大規模システムプロンプト（5K-20K tokens）
- 変更頻度: 月1回以下
- 用途: 汎用コード生成、分析

**実装**:
```python
SYSTEM_PROMPT = """
[Large 10K+ token prompt with all rules and context]
"""

def call_api_with_cache(user_query: str) -> str:
    response = client.messages.create(
        model="claude-opus-4.7",
        max_tokens=2048,
        system=[
            {
                "type": "text",
                "text": SYSTEM_PROMPT,
                "cache_control": {"type": "ephemeral"}
            }
        ],
        messages=[
            {
                "role": "user",
                "content": user_query
            }
        ]
    )
    return response.content[0].text
```

**コスト試算**:
```
シナリオ: 1日1000リクエスト

従来（キャッシュなし）:
  システムプロンプト: 10K tokens × 1000 × $0.000003 = $30
  ユーザー入力: 100 tokens × 1000 × $0.000003 = $0.30
  合計: $30.30/日

キャッシュ活用（60分TTL）:
  キャッシュ書き込み: 1回 × 10K × $0.0000037.5 = $0.0375
  キャッシュ読み込み: 999回 × 10K × $0.0000003 = $2.997
  ユーザー入力: 1000 × 100 × $0.000003 = $0.30
  合計: $3.33/日
  削減率: 89%
```

### パターン2: マルチパート可変キャッシング

**適用**:
- 一部安定（キャッシュ）、一部可変（非キャッシュ）
- 用途: ドキュメント分析、コード審査

**実装**:
```python
REFERENCE_DOCS = """
[100+ pages of API reference - 50K tokens]
This section is STABLE and cached
"""

CODE_ANALYSIS_RULES = """
[Code style guide, patterns - 5K tokens]
This is also STABLE
"""

def analyze_user_code(user_code: str, current_time: str) -> str:
    response = client.messages.create(
        model="claude-opus-4.7",
        max_tokens=4096,
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": REFERENCE_DOCS,
                        "cache_control": {"type": "ephemeral"}
                    },
                    {
                        "type": "text",
                        "text": CODE_ANALYSIS_RULES,
                        "cache_control": {"type": "ephemeral"}
                    },
                    {
                        "type": "text",
                        "text": f"""
Current timestamp: {current_time}

Analyze this code:
{user_code}
"""
                    }
                ]
            }
        ]
    )
    return response.content[0].text
```

### パターン3: セッション永続キャッシング

**適用**:
- マルチターン会話
- 用途: チャットボット、対話型システム

**実装**:
```python
class ConversationManager:
    def __init__(self):
        self.session_context = None
        self.cache_ttl = 3600
    
    def initialize_session(self, system_prompt: str):
        self.session_context = {
            "system": system_prompt,
            "messages": [],
            "cached_at": time.time()
        }
    
    def add_message(self, role: str, content: str):
        message = {
            "role": role,
            "content": content,
            "timestamp": time.time()
        }
        self.session_context["messages"].append(message)
    
    def get_response(self) -> str:
        messages = []
        
        # System prompt always cached
        messages.append({
            "role": "user",
            "content": [
                {
                    "type": "text",
                    "text": self.session_context["system"],
                    "cache_control": {"type": "ephemeral"}
                }
            ]
        })
        
        # Conversation history
        for msg in self.session_context["messages"]:
            messages.append({
                "role": msg["role"],
                "content": msg["content"]
            })
        
        response = client.messages.create(
            model="claude-opus-4.7",
            max_tokens=2048,
            messages=messages
        )
        
        return response.content[0].text
```

---

## コンテキスト効率性の最適化

### コンテキスト使用率の監視

```yaml
目標配分:
  システムプロンプト: 10-15% (2K-3K tokens)
  入力タスク: 5-10% (1K-2K tokens)
  会話履歴: 20-30% (4K-6K tokens)
  ワーキングスペース: 30-40% (6K-8K tokens)
  予約領域: 10-15% (2K-3K tokens)

警告閾値:
  80%: 新しいAgent委譲検討
  90%: 会話圧縮実行
  95%: 古い履歴削除
```

### 圧縮アルゴリズム

#### 1. 会話サマリー圧縮
```python
def compress_conversation(messages: list) -> str:
    """
    古いメッセージを高圧縮率のサマリーに変換
    """
    if len(messages) > 20:
        old_messages = messages[:-10]  # Keep last 10
        
        # APIに圧縮要求
        summary = client.messages.create(
            model="claude-haiku-4-5-20251001",  # Cheap model
            max_tokens=200,
            messages=[
                {
                    "role": "user",
                    "content": f"""
Compress this conversation into a concise bullet-point summary:

{format_messages(old_messages)}

Summary (max 200 tokens):
"""
                }
            ]
        ).content[0].text
        
        return f"[CONVERSATION SUMMARY]\n{summary}\n\n"
    
    return format_messages(messages)
```

#### 2. セマンティック重複排除
```python
def deduplicate_context(messages: list) -> list:
    """
    意味的に重複する情報を削除
    """
    embeddings = []
    for msg in messages:
        emb = get_embedding(msg["content"])
        embeddings.append(emb)
    
    # Cosine similarity > 0.95 の重複削除
    unique_messages = []
    for i, msg in enumerate(messages):
        is_duplicate = False
        for j, unique_msg in enumerate(unique_messages):
            if cosine_similarity(embeddings[i], embeddings[j]) > 0.95:
                is_duplicate = True
                break
        
        if not is_duplicate:
            unique_messages.append(msg)
    
    return unique_messages
```

---

## レイテンシ最適化

### 並列処理による削減

**パターン1: 複数ツール並列実行**
```python
import asyncio

async def parallel_analysis(code: str):
    # 3つの分析を並列実行
    results = await asyncio.gather(
        analyze_security(code),
        analyze_performance(code),
        analyze_style(code)
    )
    
    return {
        "security": results[0],
        "performance": results[1],
        "style": results[2]
    }

# 実行時間: max(各分析時間) ≈ 2秒
# vs 逐次実行: 6秒
# 削減: 67%
```

**パターン2: ストリーミング応答**
```python
def stream_large_response(prompt: str):
    """
    トークンが生成される度に出力（遅延感を軽減）
    """
    with client.messages.stream(
        model="claude-opus-4.7",
        max_tokens=4096,
        messages=[
            {"role": "user", "content": prompt}
        ]
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
```

### キャッシング効果の実測値

```
シナリオ: コード審査API

リクエスト1（キャッシュミス）:
  システムプロンプト: 10K tokens
  ユーザーコード: 500 tokens
  合計入力: 10.5K tokens
  レイテンシ: 2.5秒
  コスト: $0.03

リクエスト2-100（キャッシュヒット）:
  キャッシュ読み込み: 10K tokens × $0.0000003 = $0.003
  ユーザーコード: 500 tokens × $0.000003 = $0.0015
  合計コスト: $0.0045/回
  レイテンシ: 1.2秒 (52%削減)

100リクエスト合計:
  キャッシュなし: $3.00
  キャッシュあり: $0.03 + $0.45 = $0.48
  削減: 84%
```

---

## コスト最適化の総合戦略

### モデル選択最適化

```yaml
タスク複雑度別推奨:
  
  高複雑度（設計、戦略決定）:
    - Model: Opus 4.7
    - 入力: $15/1M tokens
    - 出力: $45/1M tokens
    - 用途: 複雑な設計, 複数言語対応
  
  中複雑度（コード生成、レビュー）:
    - Model: Sonnet 4.6
    - 入力: $3/1M tokens
    - 出力: $15/1M tokens
    - 用途: 大多数の実運用タスク
  
  低複雑度（質問応答、フォーマット）:
    - Model: Haiku 4.5
    - 入力: $0.80/1M tokens
    - 出力: $4/1M tokens
    - 用途: 簡単な処理, 大量リクエスト
```

**コスト削減試算**:
```
月1M リクエスト、平均500トークン入力の場合

全てOpus使用:
  入力: 1M × 500 × $0.000015 = $7,500
  
適切に使い分け（70% Sonnet, 20% Haiku, 10% Opus）:
  Sonnet: 700K × 500 × $0.000003 = $1,050
  Haiku: 200K × 500 × $0.0000008 = $80
  Opus: 100K × 500 × $0.000015 = $750
  合計: $1,880
  
削減: 75%
```

### バッチ処理の活用

```python
# 低優先度な大量タスクはBatch APIで処理
# レイテンシ: 5分単位のバッチ処理だが、コストは50%削減

batch_requests = []
for task in large_task_queue:
    batch_requests.append({
        "custom_id": f"task-{task.id}",
        "params": {
            "model": "claude-opus-4.7",
            "max_tokens": 1024,
            "messages": [...]
        }
    })

# Batch APIで投入
batch_response = client.batches.create(
    requests=batch_requests
)

# 結果を後で取得
results = client.batches.retrieve(batch_response.id)
```

---

## パフォーマンス監視・アラート

### メトリクス定義

```yaml
Primary SLIs (Service Level Indicators):
  - Cache Hit Rate: 目標 > 60%
  - P99 Latency: 目標 < 5秒
  - Error Rate: 目標 < 0.5%

Cost Metrics:
  - Cost per Request: 監視・トレンド分析
  - Cache Savings: 月額ベースで集計
  - Model Distribution: Opus/Sonnet/Haiku の比率
```

### アラート閾値

```yaml
Severity: Critical
  - Cache Hit Rate < 30% (1時間継続)
  - P99 Latency > 10秒 (5分継続)
  - Error Rate > 2% (5分継続)
  
Severity: Warning
  - Cache Hit Rate < 50% (15分継続)
  - Cost per Request 上昇 > 20% (1時間比較)
  - Opus使用率 > 80% (チューニング検討)
```

---

## チェックリスト: 導入時の確認項目

- [ ] システムプロンプト（変更頻度: 月1回以下）をキャッシュ対象に
- [ ] TTLを明示的に60分に設定（`ttl_seconds=3600`）
- [ ] タイムスタンプ、動的コンテンツはキャッシュ対象外に移動
- [ ] 並列処理可能なタスクは `asyncio` で非ブロッキング化
- [ ] 月1Mリクエスト以上の場合、Batch APIを検討
- [ ] モデル選択基準を明示（Opus/Sonnet/Haiku の使い分け）
- [ ] キャッシュヒット率・コストを日次監視
- [ ] 90日ごとに最適化戦略を見直し

---

**バージョン**: 1.0  
**最終更新**: 2026-04-27
