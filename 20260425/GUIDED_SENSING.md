# GUIDED_SENSING.md - Guide & Sensor 制御メカニズム

このファイルは、2026年4月のハーネスエンジニアリング最新トレンドである **Guide（事前制御）** と **Sensor（事後制御）** の詳細な実装方法を記述します。

**最終更新**: 2026-04-25  
**アーキテクチャ**: Guided & Sensored Agents  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

---

## 概要：Guide & Sensor の役割

```
従来のエージェント設計:
  User → Generator → Evaluator → Actioner → Result
  （品質は Evaluator の検証に依存）

Guided & Sensored 設計:
  User → Guide → Generator → Evaluator → Sensor → Actioner → Sensor → Result
         (事前制御)                              (事後制御)
  （品質は事前+事後の双方向制御で実現）
```

### Guide（ガイド：事前制御）
- **タイミング**: Generator 実行 **前**
- **役割**: 潜在的リスクを予測し、不適切な経路を **事前に防止**
- **方式**: Feedforward Control（フィードフォワード）
- **例**: 「このタスクではハードコード値は絶対に避けてください」

### Sensor（センサー：事後制御）
- **タイミング**: Actioner 実行 **後**
- **役割**: 予期しない結果・副作用を **検出** し自己補正
- **方式**: Feedback Control（フィードバック）
- **例**: 「実行後のログで過度な API 呼び出しを検知 → 自動補正」

---

## Part 1: Guide（事前制御）の実装

### Guide の思考プロセス

```
タスク受信
    ↓
[1] 行動予測
    - Generator がどの経路を取る可能性が高いか
    - その経路に潜在リスクはないか
    ↓
[2] 危険行動パターンの列挙
    - セキュリティリスク（権限逸脱、データ漏洩等）
    - パフォーマンスリスク（無限ループ、過度リソース等）
    - ビジネスリスク（規制違反、許容不可な副作用）
    ↓
[3] 制約条件の生成
    - DO: 推奨される安全な手法
    - DON'T: 避けるべき危険な手法
    ↓
[4] Generator への指示追加
    - プロンプトに制約条件を埋め込み
    - 優先度を明示
```

### Guide のプロンプト テンプレート

```
あなたはGuideエージェント（事前制御）です。

次のタスクをGeneratorが実行する際の
「潜在的リスク」を予測し、
「制約条件」を Generator に提示してください。

=== TASK ===
[ユーザーのリクエスト]

=== YOUR ANALYSIS ===

1. このタスクで Generator が取る可能性が高い行動:
   - アプローチA: [説明]
   - アプローチB: [説明]
   - アプローチC: [説明]

2. 各アプローチのリスク分析:
   
   アプローチA:
   ✓ 利点: [2点]
   ✗ リスク: [CRITICAL|HIGH|MEDIUM]
      - リスク詳細: [記載]
      - 影響: [記載]
   
   [B, C も同様]

3. 最も安全なアプローチ:
   アプローチ: [推奨]
   理由: [明確に]

4. Generator への制約条件指示:

   ✓ DO:
     - [安全な方法1]
     - [安全な方法2]
     - [安全な方法3]
   
   ✗ DON'T:
     - [避けるべき手法1] → 理由: [権限逸脱のリスク]
     - [避けるべき手法2] → 理由: [無限ループの可能性]
     - [避けるべき手法3] → 理由: [データ漏洩リスク]

=== OUTPUT TO GENERATOR ===

「以下のタスクを実行する際、絶対に以下の指示を守ってください:

タスク: [タスク説明]

MUST DO:
- [制約1]
- [制約2]
- [制約3]

MUST NOT:
- [禁止事項1]
- [禁止事項2]
- [禁止事項3]

推奨アプローチ: [安全な経路]
」
```

### Guide 実装例 1: 新規ユーザー作成スクリプト

```yaml
guide_example_user_creation:
  task: "新規ユーザー作成スクリプトを生成"
  
  analysis:
    generator_likely_approaches:
      - "DB 直接クエリでユーザー作成"
      - "API エンドポイント経由でユーザー作成"
      - "Admin API を使用"
    
    risk_analysis:
      approach_db_direct:
        risks:
          - risk: "hardcoded_password"
            severity: "CRITICAL"
            impact: "パスワード露出 → セキュリティ侵害"
          - risk: "sql_injection"
            severity: "CRITICAL"
            impact: "DB 侵害"
          - risk: "excessive_privilege"
            severity: "HIGH"
            impact: "権限逸脱"
      
      approach_api:
        risks:
          - risk: "rate_limiting"
            severity: "MEDIUM"
            impact: "API 制限で処理失敗の可能性"
        mitigations:
          - "retry logic with backoff"
          - "batch processing with delays"
      
      approach_admin_api:
        risks:
          - risk: "credential_exposure"
            severity: "CRITICAL"
            impact: "Admin 権限が漏洩"
    
    recommendation: "API エンドポイント + 環境変数認証 が最適"
  
  generator_constraints:
    must_do:
      - "環境変数から認証情報を読み込む（ハードコード禁止）"
      - "API rate limit を考慮（max_retries=3, backoff=exponential）"
      - "エラーハンドリング: 各 API 呼び出しで try-catch"
      - "ロギング: ユーザー作成成功/失敗を記録"
      - "入力検証: ユーザー名、メールアドレスの形式確認"
    
    must_not:
      - "❌ パスワードをスクリプトにハードコード"
      - "❌ DB に直接接続（SQLインジェクション リスク）"
      - "❌ Admin トークンを可視化（ログに出力）"
      - "❌ 無限ループでの再試行（max_retries を設定）"
      - "❌ ユーザー入力の無検証使用（SQL injection 対策）"
```

### Guide 実装例 2: デフォルト設定更新

```yaml
guide_example_default_setting:
  task: "システムデフォルト設定を更新するスクリプト"
  
  generator_constraints:
    must_do:
      - "変更対象を明確に限定（すべて変更禁止）"
      - "ドライランで事前テスト：変更前後の差分を表示"
      - "バックアップ作成：変更前設定を保存"
      - "ロールバック計画：エラー時の復帰方法を記述"
      - "変更内容をログに記録（監査対応）"
    
    must_not:
      - "❌ 全ホスト対象の一括変更"
      - "❌ バックアップなしの変更"
      - "❌ 無限再試行（最大3回まで）"
      - "❌ 本番環境の直接変更（ステージング先）"
      - "❌ 権限昇格なしの実行（sudo 不可の場合を示唆）"
```

---

## Part 2: Sensor（事後制御）の実装

### Sensor の思考プロセス

```
Actioner が実行
    ↓
[1] 実行結果の観測
    - 実行ログを収集
    - システム状態を記録
    - 期待値との比較
    ↓
[2] 異常検知
    - 統計的異常（動作量が多い/少ない等）
    - ルールベース異常（禁止事項が実行された等）
    - セマンティック異常（意図と乖離）
    ↓
[3] 自動補正可能性の判定
    - 復帰不可な操作か？ → エスカレーション
    - 軽微な問題か？ → 自動補正試行
    - 警告のみか？ → 記録 + 報告
    ↓
[4] 補正実行またはエスカレーション
    - 自動補正: キャッシュリセット、接続リトライ等
    - エスカレーション: 人間レビューへ
```

### Sensor のプロンプト テンプレート

```
あなたはSensorエージェント（事後制御）です。

次のアクションが実行されました。
実行結果を観測し、異常を検知して自動補正を試みてください。

=== EXECUTION LOG ===
[実行ログ詳細]

=== EXPECTED BEHAVIOR ===
[予期された挙動]

=== ACTUAL BEHAVIOR ===
[実際の挙動]

=== YOUR ANALYSIS ===

1. 実行結果の分類:
   ✓ Success / ⚠️ Warning / ✗ Error

2. 異常検知:
   
   異常1: [異常説明]
   - 検知方法: [統計的 or ルールベース]
   - 重要度: [CRITICAL|HIGH|MEDIUM|LOW]
   - 影響範囲: [説明]
   
   異常2: [異常説明]
   - [詳細]
   
   [その他]

3. 自動補正の可能性:
   
   異常1 → 補正方法: [対応内容]
            成功率: [確度 0.0-1.0]
            実行: [YES/NO]
   
   異常2 → 補正方法: [対応内容]
            成功率: [確度]
            実行: [YES/NO]
   
   補正不可 → エスカレーション理由: [記載]

4. 実行後の状態:
   - システム健全性: [OK|DEGRADED|FAILURE]
   - セキュリティ: [OK|COMPROMISED]
   - データ整合性: [OK|INCONSISTENT]

=== OUTPUT ===

異常検知結果:
[詳細レポート]

補正実行状況:
[実行内容 + 成功/失敗]

最終結果:
[元のアクション結果 + 補正による改善]

エスカレーション必要: [YES/NO]
エスカレーション理由: [YES の場合のみ記載]
```

### Sensor 実装例 1: API 呼び出し監視

```yaml
sensor_example_api_calls:
  executed_action: "ユーザー作成 API 呼び出し"
  
  monitoring_rules:
    - metric_name: "api_call_count"
      expected: "< 10"  # 単一ユーザー作成は 1-5 呼び出し
      actual: 250
      status: "ANOMALY_DETECTED"
      severity: "HIGH"
      root_cause: "ループ内で何度も API 呼び出し"
      
      correction_strategy:
        option_1:
          method: "API 呼び出しの去重（キャッシング）"
          success_rate: 0.8
          action: "実行"
        
        option_2:
          method: "バッチ API を使用"
          success_rate: 0.9
          action: "試行（option_1 失敗時）"
    
    - metric_name: "response_time"
      expected: "< 5s"
      actual: "65s"
      status: "WARNING"
      severity: "MEDIUM"
      root_cause: "rate limit による遅延"
      
      correction_strategy:
        method: "exponential backoff を適用"
        success_rate: 0.7
        action: "実行"
    
    - metric_name: "error_rate"
      expected: "0%"
      actual: "0%"
      status: "OK"
      severity: "-"
```

### Sensor 実装例 2: システム リソース監視

```yaml
sensor_example_resources:
  executed_action: "バッチ ユーザー作成（500ユーザー）"
  
  monitoring_rules:
    - metric_name: "memory_usage"
      baseline: "500MB"
      expected_increase: "100MB"  # 0.2MB per user
      actual: "2GB"
      status: "ANOMALY_DETECTED"
      severity: "CRITICAL"
      
      correction_strategy:
        action: "メモリリーク検知"
        step_1: "GC（ガベージコレクション）手動実行"
        step_2: "失敗時 → エスカレーション（人間対応）"
        success_rate: 0.6
    
    - metric_name: "disk_io_operations"
      expected: "< 1000 ops"
      actual: "50000 ops"
      status: "ANOMALY_DETECTED"
      severity: "HIGH"
      
      correction_strategy:
        method: "バッチサイズ削減（500 → 100 per batch）"
        success_rate: 0.85
        action: "実行"
    
    - metric_name: "database_connections"
      expected: "1"
      actual: "50"
      status: "ANOMALY_DETECTED"
      severity: "MEDIUM"
      
      correction_strategy:
        method: "接続プールを有効化"
        success_rate: 0.9
        action: "実行"
```

### Sensor 実装例 3: セキュリティ監視

```yaml
sensor_example_security:
  executed_action: "設定ファイル作成（credentials 含む）"
  
  monitoring_rules:
    - metric_name: "file_permissions"
      expected: "600 (owner read/write only)"
      actual: "644 (world readable)"
      status: "SECURITY_ALERT"
      severity: "CRITICAL"
      
      correction_strategy:
        step_1: "ファイルパーミッション自動修正（chmod 600）"
        step_2: "被害確認（ファイルアクセスログ）"
        step_3: "失敗時 → エスカレーション"
        success_rate: 0.95
    
    - metric_name: "credential_in_logs"
      check: "API_KEY, PASSWORD, TOKEN キーワード"
      expected: "0 occurrences"
      actual: "3 occurrences"
      status: "SECURITY_ALERT"
      severity: "CRITICAL"
      
      correction_strategy:
        step_1: "ログから機密情報を削除（マスキング）"
        step_2: "削除対象箇所を記録"
        step_3: "セキュリティチームに通知"
        action: "実行（自動）"
    
    - metric_name: "privilege_escalation"
      check: "sudo 実行、権限変更"
      expected: "none"
      actual: "1 unauthorized sudo"
      status: "SECURITY_ALERT"
      severity: "CRITICAL"
      
      correction_strategy:
        action: "即座にエスカレーション（自動補正不可）"
        escalation_type: "security_incident"
```

---

## Guide & Sensor の統合ワークフロー

### End-to-End Flow

```
USER REQUEST
    ↓
1. ORCHESTRATOR (タスク分解)
    ↓
2. GUIDE (リスク予測 + 制約条件生成)
    └─ 出力: Generator へのプロンプト追加
    ↓
3. GENERATOR (提案生成 with 制約)
    └─ 出力: 3-5個の提案（安全な範囲内）
    ↓
4. EVALUATOR (品質検証)
    ├─ 入力: Generator の提案
    └─ 出力: GO / CAUTION / NO-GO
    ↓
5. HUMAN DECISION (承認)
    ↓
6. SENSOR (プリチェック)
    └─ 出力: 実行予定内容の事前検証
    ↓
7. ACTIONER (実行)
    └─ 出力: 実行ログ
    ↓
8. SENSOR (ポストチェック + 自動補正)
    ├─ 異常検知
    ├─ 自動補正試行
    └─ 出力: 補正済み結果 or エスカレーション
    ↓
9. RESULT
```

### Guide & Sensor の相互作用

```
Guide が事前に警告:
  「無限ループに注意」

    ↓

Generator が制約下で提案:
  「max_retries=3 でループ制御」

    ↓

実行後 Sensor が観測:
  「actual_retries=3 ✓」
  「正常に完了」

異常なし → 完了報告
```

---

## Guide & Sensor の設計パターン

### パターン A: Proactive Guide + Reactive Sensor

```
Guide: 事前に 10 個のリスクを列挙 + 制約
Sensor: 実行後に 10 個のリスク監視 → 検知

利点: 包括的
欠点: オーバーヘッド大
推奨: 高リスクタスク
```

### パターン B: Focused Guide + Selective Sensor

```
Guide: 最大 3 個のトップリスクに焦点
Sensor: その 3 個に特化した監視

利点: 効率的
欠点: 見落とし可能性
推奨: 標準タスク
```

### パターン C: Implicit Guide + Explicit Sensor

```
Guide: 暗黙的（Generator の学習に頼る）
Sensor: 明示的で詳細な異常検知

利点: バランス型
欠点: 中程度の複雑性
推奨: 中程度リスクタスク
```

---

## トラブルシューティング

### Issue: Guide が過度に保守的

**症状**: Generator の提案肢が極めて限定的

**原因**: Guide が制約を強すぎた

**対策**:
```yaml
fix:
  1_review_guide_output:
    action: "Guide が生成した制約を確認"
    check: "制約が実装を事実上不可能にしていないか"
  
  2_adjust_risk_tolerance:
    action: "リスク閾値を段階的に引き上げ"
    example: "CRITICAL のみ制限 → HIGH も制限 → MEDIUM も制限"
  
  3_retrain_guide:
    action: "Guide のプロンプト改善"
    focus: "リスク軽減措置の提示 → 禁止ではなく"
```

### Issue: Sensor が異常を検知できない

**症状**: 明らかなエラーを見逃している

**原因**: 監視ルールが不十分

**対策**:
```yaml
fix:
  1_expand_monitoring_rules:
    action: "カバー対象の異常を拡張"
    example: "API呼び出し数 → API呼び出し数 + エラー率 + レイテンシ"
  
  2_add_baseline_comparison:
    action: "期待値ベースラインを設定"
    
  3_test_sensor:
    action: "意図的にエラーを注入 → 検知確認"
```

### Issue: 自動補正が失敗して、かえって状態が悪化

**症状**: Sensor の補正試行 > エスカレーション

**原因**: 補正可能判定が誤

**対策**:
```yaml
fix:
  1_safety_threshold:
    action: "補正試行を保守的に → 失敗 > 即エスカレーション"
  
  2_rollback_preparation:
    action: "各補正試行前にロールバック計画を記述"
  
  3_human_approval:
    action: "自動補正前に人間に確認を求める（重大リスク時）"
```

---

## ベストプラクティス

### 1. Guide の明確性

```
✓ 良い例:
「DB 直接クエリを避け、API エンドポイントを経由してください。
 理由: SQL インジェクション リスク削減」

✗ 悪い例:
「セキュアに実装してください」
```

### 2. Sensor の網羅性

```
✓ 監視対象:
- 実行時間（レイテンシ異常）
- リソース使用（メモリリーク）
- エラー頻度（無限ループ）
- セキュリティ（権限逸脱）
- ビジネス指標（意図した結果か）

✗ 不十分:
- エラーログのみ監視
```

### 3. 自動補正の制限

```
✓ 自動補正可能:
- キャッシュリセット
- 接続リトライ
- リソース解放
- ログマスキング

✗ 自動補正不可（エスカレーション）:
- 権限変更
- データ削除
- セキュリティ設定変更
```

---

## チェックリスト：Guide/Sensor 導入

- [ ] Guide エージェント実装完了
- [ ] Guide のリスク分析テンプレート作成
- [ ] Sensor エージェント実装完了
- [ ] Sensor の異常検知ルール定義（最低5個）
- [ ] 自動補正vs人間エスカレーション基準を明文化
- [ ] End-to-End ワークフローテスト実施
- [ ] ドキュメント（このファイル）をチーム内で共有
- [ ] 最初の10タスクで運用パターン確認

---

## 参考資料

- [Addy Osmani - Agent Harness Engineering](https://addyosmani.com/blog/agent-harness-engineering/)
- [Anthropic Engineering - Scaling Managed Agents](https://www.anthropic.com/engineering/managed-agents)
- [Medium - Agent Harness Engineering (Adnan Masood, Apr 2026)](https://medium.com/@adnanmasood/agent-harness-engineering-the-rise-of-the-ai-control-plane-938ead884b1d)

---

**最終更新**: 2026-04-25  
**対応Claude モデル**: Opus 4.7, Sonnet 4.6, Haiku 4.5

