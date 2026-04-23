# CONTEXT.md - コンテキスト最適化戦略

**対象**: Claude Managed Agents環境でのコンテキストウィンドウ最適化

## 1. コンテキストウィンドウの構成

### 1.1 総容量と配分

```
Claude Opus 4.7 Standard: 200,000トークン
├── System Prompt Area (Reserved): 5,000トークン
│   └─ Base instructions, guidelines, safety rules
├── Knowledge Base Area (Cached): 80,000トークン
│   └─ Embeddings, retrieval results, domain knowledge
├── Session History Area (Managed): 50,000トークン
│   └─ Recent conversation, context chain
└── Response Generation Area (Dynamic): 65,000トークン
    └─ Real-time task processing, tool results
```

### 1.2 トークン使用ルール

```python
class ContextBudgetManager:
    """
    段階的なコンテキスト圧縮戦略
    """
    
    THRESHOLDS = {
        "optimal": (0, 0.70),          # 0-140Kトークン: 最適
        "caution": (0.70, 0.80),       # 140-160Kトークン: 注意
        "compress": (0.80, 0.95),      # 160-190Kトークン: 圧縮開始
        "critical": (0.95, 1.00)       # 190-200Kトークン: 危機的
    }
    
    def get_compression_level(self, usage_ratio):
        for level, (low, high) in self.THRESHOLDS.items():
            if low <= usage_ratio < high:
                return level
```

## 2. コンテキストエンジニアリングの3層

### 2.1 Layer 1: Instruction Context

```markdown
**目的**: エージェントの基本的な行動指針を定義

**内容**:
- エージェントの役割定義
- 主要な責務と境界
- 思考プロセスの手順
- 禁止事項とガードレール
- 出力フォーマット要件

**サイズ**: ~1,000トークン（キャッシュ推奨）

**最適化**:
- 冗長な説明を削除
- 具体例は最小限（類推可能な場合）
- 構造化フォーマット（JSON/マークダウン）を使用
```

### 2.2 Layer 2: Knowledge Context

```markdown
**目的**: エージェントが参照する専門知識を提供

**内容タイプ**:
1. Domain Knowledge
   - 業界ガイドライン
   - 技術仕様書
   - ベストプラクティス集

2. Reference Data
   - 製品カタログ
   - API仕様
   - データベーススキーマ

3. Learned Patterns
   - 過去の成功例
   - エラーパターン
   - 最適化トリック

**サイズ**: ~70,000トークン（プロンプトキャッシング対象）

**最適化**:
- 関連度ベースの優先度付け
- クエリ駆動型の検索結果
- 定期的な鮮度更新（週1回推奨）
```

### 2.3 Layer 3: Session Context

```markdown
**目的**: 進行中の会話と状態を追跡

**内容**:
- ユーザー入力
- エージェント応答
- ツール実行結果
- 状態変数
- 決定履歴

**サイズ**: ~40,000トークン（動的、無期限保持なし）

**最適化**:
- 直近20-30メッセージのみ保持
- 要約による旧メッセージの圧縮
- 30分以上古い情報は削除
```

## 3. プロンプトキャッシング戦略

### 3.1 キャッシング対象の特定

```python
CACHE_ELIGIBLE = {
    "system_prompt": {
        "size": "~1-2K",
        "frequency": "constant",
        "benefit": "99% hit rate",
        "ttl": "indefinite",
        "strategy": "static_cache"
    },
    "knowledge_base": {
        "size": "~70-80K",
        "frequency": "frequent",
        "benefit": "60-70% hit rate",
        "ttl": "7 days",
        "strategy": "semantic_cache"
    },
    "reference_docs": {
        "size": "~10-20K",
        "frequency": "occasional",
        "benefit": "30-40% hit rate",
        "ttl": "30 days",
        "strategy": "static_cache"
    }
}

CACHE_INELIGIBLE = {
    "session_messages": {
        "reason": "unique per conversation",
        "strategy": "selective_summary"
    },
    "user_input": {
        "reason": "variable per request",
        "strategy": "compress_if_long"
    },
    "tool_results": {
        "reason": "time-sensitive",
        "strategy": "aggregate_recent"
    }
}
```

### 3.2 キャッシュ実装パターン

```python
class PrompCachingManager:
    """
    2層キャッシング戦略:
    1. API Level (Anthropic): トークン効率化
    2. Application Level: 応答時間最適化
    """
    
    def build_context_with_caching(self, task):
        # Layer 1: System prompt (always cached)
        system = self.get_system_prompt()
        
        # Layer 2: Knowledge base (semantic cache)
        knowledge = self.retrieve_relevant_knowledge(
            query=task,
            max_tokens=80000,
            use_cache=True
        )
        
        # Layer 3: Session context (selective)
        session = self.summarize_recent_history(
            max_messages=30,
            max_tokens=40000
        )
        
        return {
            "system": system,      # キャッシュキー: "system_v1"
            "knowledge": knowledge, # キャッシュキー: hash(knowledge)
            "session": session      # キャッシュキー: なし（常に新規）
        }
```

## 4. コンテキスト圧縮技法

### 4.1 動的圧縮トリガー

```
ラウンド1: 70% 超過 → Warning をログ出力
    ↓
ラウンド2: 80% 超過 → 古い session メッセージを要約
    ↓
ラウンド3: 90% 超過 → Knowledge base をセマンティック抽出
    ↓
ラウンド4: 95% 超過 → Reference data を削除
    ↓
ラウンド5: 100% 超過 → エラー + Human escalation
```

### 4.2 要約アルゴリズム

```python
class ContextCompressor:
    """
    セッション履歴を圧縮しながら重要情報を保持
    """
    
    def summarize_messages(self, messages, target_tokens=10000):
        """
        戦略:
        1. 最新 N メッセージ: 原文保持（contextual)
        2. それ以前: Key-Value 形式で要約
        """
        
        recent_messages = messages[-10:]  # 最新10メッセージ
        older_messages = messages[:-10]
        
        summary = {
            "recent": recent_messages,  # 原文
            "older": {
                "decisions": self.extract_decisions(older_messages),
                "outcomes": self.extract_outcomes(older_messages),
                "errors": self.extract_errors(older_messages),
                "count": len(older_messages)
            }
        }
        
        return summary
    
    def extract_key_info(self, messages):
        """
        重要情報フィルタリング:
        - ユーザーの意図変更
        - 重要な決定ポイント
        - エラーと解決方法
        - 状態変更
        """
        
        patterns = {
            "intent_change": r"(change|instead|rather|actually)",
            "decision": r"(decided|chose|selected|approved)",
            "error": r"(error|failed|problem|issue)",
            "result": r"(success|completed|done|result)"
        }
        
        return self.filter_by_patterns(messages, patterns)
```

## 5. 検索結果の統合

### 5.1 Retrieval-Augmented Generation (RAG) 設定

```python
class RAGContextManager:
    """
    検索結果をコンテキストに効率的に統合
    """
    
    def integrate_retrieval_results(self, query, top_k=5):
        # 1. セマンティック検索実行
        results = self.semantic_search(
            query=query,
            top_k=top_k,
            min_similarity=0.7
        )
        
        # 2. トークン予算内での最適化
        optimized = []
        token_budget = 30000  # Knowledge contextの一部
        
        for result in results:
            if self.estimate_tokens(result) < token_budget:
                optimized.append(result)
                token_budget -= self.estimate_tokens(result)
            else:
                # トークンオーバーなら、重要度の高い部分のみ抽出
                excerpt = self.extract_key_parts(
                    result,
                    max_tokens=token_budget
                )
                optimized.append(excerpt)
                break
        
        return optimized
    
    def estimate_tokens(self, text):
        """ざっくりとした推定: 1トークン ≈ 4文字"""
        return len(text) // 4
```

## 6. マルチエージェント環境でのコンテキスト共有

### 6.1 共有コンテキストプール

```
┌─────────────────────────────────────┐
│   Shared Context Pool (30K tokens)  │
├─────────────────────────────────────┤
│ Orchestrator State                  │
├─────────────────────────────────────┤
│ Task Definition & Progress          │
├─────────────────────────────────────┤
│ Inter-Agent Communication Buffer    │
├─────────────────────────────────────┤
│ Common Knowledge References         │
└─────────────────────────────────────┘
    ↓        ↓        ↓
┌──────┬──────────┬───────┐
│ Reas │Retrieval │Action │
│ oning│  Agent   │ Agent │
└──────┴──────────┴───────┘
    ↑        ↑        ↑
    └────────┴────────┘
    Validation Agent (Read-Only)
```

### 6.2 コンテキスト衝突の回避

```python
class SharedContextManager:
    """
    複数エージェント間のコンテキスト同期
    """
    
    def broadcast_state_update(self, agent_id, state_delta):
        """
        1. 変更をロックで保護
        2. 他エージェント関連部分のみ更新
        3. バージョン番号でコンフリクト検出
        """
        
        with self.context_lock:
            current_version = self.shared_context["version"]
            
            # マージ可能性チェック
            if self.is_mergeable(state_delta, current_version):
                self.merge_state(state_delta)
            else:
                # コンフリクト → Orchestratorに報告
                self.escalate_conflict(agent_id, state_delta)
```

## 7. パフォーマンスメトリクス

### 7.1 モニタリング対象

```python
CONTEXT_METRICS = {
    "cache_hit_rate": {
        "target": "> 70%",
        "measurement": "successful_cache_hits / total_requests"
    },
    "average_context_size": {
        "target": "< 100K tokens",
        "measurement": "mean(tokens_per_request)"
    },
    "compression_ratio": {
        "target": "> 0.5 (50% compression)",
        "measurement": "original_size / compressed_size"
    },
    "response_latency": {
        "target": "P95 < 5s",
        "measurement": "time_to_first_token"
    },
    "token_efficiency": {
        "target": "> 0.8",
        "measurement": "useful_output_tokens / input_tokens"
    }
}
```

### 7.2 ダッシュボード例

```
┌──────────────────────────────────────┐
│ Context Performance Dashboard        │
├──────────────────────────────────────┤
│ Cache Hit Rate:      72% ✓           │
│ Avg Context Size:    85K tokens ✓    │
│ Compression Ratio:   0.58 ✓          │
│ Response Latency:    2.3s ✓          │
│ Token Efficiency:    0.84 ✓          │
├──────────────────────────────────────┤
│ Recent Trend: ↑ +5% hit rate         │
│ Action: None (within SLA)            │
└──────────────────────────────────────┘
```

---

**最終更新**: 2026年4月23日
