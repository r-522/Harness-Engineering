# GOVERNANCE.md - セキュリティ・監査・コンプライアンス

**バージョン**: 1.0 (2026年4月 Claude Managed Agents準拠)

## 1. セキュリティ設計の原則

### 1.1 Zero-Trust Architecture

```
前提: ネットワーク内も外も同等にセキュリティ脅威が存在

実装:
- すべてのツール呼び出しを認証・授権
- 最小権限原則 (Principle of Least Privilege)
- すべての通信を暗号化
- マイクロセグメンテーション
```

### 1.2 防御層設計（多層防御）

```
Layer 1: Input Validation
         ↓ (Prevent injection attacks)
Layer 2: Authentication & Authorization
         ↓ (Verify identity & permissions)
Layer 3: Rate Limiting & Throttling
         ↓ (Prevent DoS)
Layer 4: Data Encryption
         ↓ (At-rest & in-transit)
Layer 5: Audit & Monitoring
         ↓ (Detect anomalies)
Layer 6: Incident Response
         ↓ (Contain & remediate)
```

## 2. LLM固有のセキュリティ脅威

### 2.1 幻覚（Hallucination）対策

```python
class HallucinationMitigation:
    """
    LLMの幻覚リスク低減策
    """
    
    MITIGATION_STRATEGIES = {
        "source_grounding": {
            "description": "すべての主張に情報源を要求",
            "implementation": "Cite sources in every response",
            "validation": "Automated source verification"
        },
        "confidence_scoring": {
            "description": "信頼度レベルを明示",
            "implementation": "Confidence level in [0-100]",
            "threshold": "Only act on > 80% confidence"
        },
        "fact_checking": {
            "description": "重要な事実を検証エージェントで確認",
            "implementation": "Validation Agent cross-check",
            "critical_fields": ["financial_data", "legal_info", "safety_critical"]
        },
        "retrieval_grounding": {
            "description": "回答の前に情報を検索",
            "implementation": "Always retrieve before response",
            "coverage": "≥ 80% of factual claims"
        }
    }
```

### 2.2 プロンプトインジェクション対策

```python
class PromptInjectionDefense:
    """
    ユーザー入力による攻撃防止
    """
    
    def sanitize_user_input(self, user_text):
        """
        レベル1: 構造の分離
        - ユーザー入力は明示的に区切る
        - システムプロンプトと混在させない
        """
        
        structured_prompt = f"""
[SYSTEM INSTRUCTIONS]
{self.system_prompt}

[USER INPUT]
{self.safely_format(user_text)}

[ASSISTANT RESPONSE]
"""
        return structured_prompt
    
    def detect_injection_attempts(self, user_input):
        """
        レベル2: パターン検出
        """
        
        suspicious_patterns = [
            r"(ignore|disregard|forget).*instructions",
            r"(execute|run|do).*code",
            r"(tell me|show me).*system.*prompt",
            r"(pretend|imagine).*you are",
            r"(override|bypass).*security"
        ]
        
        for pattern in suspicious_patterns:
            if re.search(pattern, user_input, re.IGNORECASE):
                return {
                    "is_suspicious": True,
                    "pattern_matched": pattern,
                    "action": "log_and_reject"
                }
        
        return {"is_suspicious": False}
```

## 3. アクセス制御とロールベース権限

### 3.1 Role-Based Access Control (RBAC)

```
ロール定義:

┌────────────────────────────────────┐
│ ADMIN                              │
│ - All actions                      │
│ - System configuration             │
│ - Audit log access                 │
└────────────────────────────────────┘
        ↑
┌────────────────────────────────────┐
│ SECURITY_REVIEWER                  │
│ - View audit logs                  │
│ - Approve high-risk actions        │
│ - Security incident response       │
└────────────────────────────────────┘
        ↑
┌────────────────────────────────────┐
│ OPERATOR                           │
│ - Standard tool execution          │
│ - View own audit records           │
│ - Report errors                    │
└────────────────────────────────────┘
        ↑
┌────────────────────────────────────┐
│ READ_ONLY                          │
│ - View reports                     │
│ - Query knowledge base             │
│ - No write/execute permissions     │
└────────────────────────────────────┘
```

### 3.2 権限の最小化原則

```python
class PermissionGate:
    """
    各ツール呼び出しに対する権限チェック
    """
    
    TOOL_PERMISSIONS = {
        "read_file": {
            "required_roles": ["READ_ONLY", "OPERATOR", "SECURITY_REVIEWER", "ADMIN"],
            "per_path_acl": True,
            "audit": "always"
        },
        "write_file": {
            "required_roles": ["OPERATOR", "SECURITY_REVIEWER", "ADMIN"],
            "per_path_acl": True,
            "audit": "always"
        },
        "delete_file": {
            "required_roles": ["ADMIN"],
            "per_path_acl": True,
            "audit": "always",
            "requires_approval": "human"
        },
        "execute_code": {
            "required_roles": ["OPERATOR", "ADMIN"],
            "requires_sandbox": True,
            "audit": "complete_code_dump",
            "requires_approval": "conditional"
        },
        "call_external_api": {
            "required_roles": ["OPERATOR", "SECURITY_REVIEWER", "ADMIN"],
            "per_endpoint_acl": True,
            "audit": "request_response_log",
            "requires_approval": "first_time"
        }
    }
    
    def check_permission(self, user_role, tool_name, resource):
        if tool_name not in self.TOOL_PERMISSIONS:
            raise SecurityError(f"Unknown tool: {tool_name}")
        
        tool_spec = self.TOOL_PERMISSIONS[tool_name]
        
        # ロールチェック
        if user_role not in tool_spec["required_roles"]:
            self.log_unauthorized_access(user_role, tool_name)
            raise PermissionError(f"Role {user_role} cannot use {tool_name}")
        
        # リソースレベルのACLチェック
        if tool_spec.get("per_path_acl"):
            if not self.check_acl(user_role, resource):
                raise PermissionError(f"Access denied to {resource}")
        
        # 承認要件チェック
        if tool_spec.get("requires_approval"):
            if not self.get_approval(user_role, tool_name, resource):
                raise ApprovalError(f"Approval required for {tool_name}")
        
        return True
```

## 4. 監査ログとコンプライアンス

### 4.1 必須ログ項目（IMMUTABLE RECORD）

```python
class AuditLog:
    """
    すべての監査ログは改ざん防止機構を実装
    """
    
    AUDIT_RECORD_SCHEMA = {
        # 基本情報
        "timestamp": "ISO-8601 UTC",
        "transaction_id": "UUID v4",
        "session_id": "UUID v4",
        
        # 実行者情報
        "user_id": "string",
        "user_role": "ADMIN|SECURITY_REVIEWER|OPERATOR|READ_ONLY",
        "authenticated": "boolean",
        "authentication_method": "api_key|oauth|mfa",
        
        # 操作情報
        "action": "tool_call|data_access|permission_change|system_change",
        "tool_name": "string",
        "resource": "string (fully qualified path)",
        "input": "encrypted_json",
        "output": "encrypted_json",
        "output_size_tokens": "integer",
        
        # 結果情報
        "status": "success|error|timeout",
        "error_code": "string or null",
        "error_message": "encrypted_string",
        "execution_time_ms": "integer",
        
        # セキュリティ情報
        "approval_status": "approved|auto_approved|denied",
        "approval_by": "user_id or 'system'",
        "risk_level": "low|medium|high",
        "security_flags": ["flag1", "flag2"],
        
        # 整合性確保
        "hash": "SHA-256(previous_record_hash + this_record)",
        "signature": "RSA-4096 signature of this record"
    }
    
    def write_immutable_record(self, record):
        """
        append-onlyデータストアに記録
        """
        # 1. 前のレコードのハッシュを含める（チェーン）
        record["hash"] = self.compute_hash(record)
        
        # 2. デジタル署名（改ざん防止）
        record["signature"] = self.sign_record(record)
        
        # 3. データベースに追加（更新・削除は不可）
        self.append_only_store.write(record)
        
        # 4. バックアップに即座にレプリケート
        self.backup_store.write(record)
```

### 4.2 監査ログの検証

```python
class AuditLogVerification:
    """
    監査ログの整合性と改ざん検出
    """
    
    def verify_log_integrity(self, log_range):
        """
        すべてのレコードが:
        1. チェーンで繋がっている
        2. デジタル署名が有効
        3. 削除されていない
        """
        
        for i, record in enumerate(log_range):
            # 署名検証
            if not self.verify_signature(record):
                raise IntegrityError(f"Record {i}: Invalid signature")
            
            # ハッシュチェーン検証
            if i > 0:
                prev_record = log_range[i-1]
                expected_hash = self.compute_hash(prev_record)
                if record.get("prev_hash") != expected_hash:
                    raise IntegrityError(f"Record {i}: Hash chain broken")
        
        return True
```

## 5. データ保護とプライバシー

### 5.1 データ分類スキーム

```
分類レベル:

┌─────────────────────────────────┐
│ PUBLIC                          │
│ No special protection           │
└─────────────────────────────────┘
        ↓ 制限強化
┌─────────────────────────────────┐
│ INTERNAL                        │
│ Encryption at-rest (AES-256)   │
│ Audit all access               │
└─────────────────────────────────┘
        ↓ 制限強化
┌─────────────────────────────────┐
│ CONFIDENTIAL                    │
│ AES-256 at-rest + in-transit   │
│ Access logging + approvals     │
│ Data masking in logs           │
└─────────────────────────────────┘
        ↓ 制限強化
┌─────────────────────────────────┐
│ RESTRICTED (PII/PHI/PCI)       │
│ AES-256 + HSM key storage      │
│ Multi-factor approval required │
│ Complete audit trail            │
│ Encryption in memory           │
│ Auto-deletion policies         │
└─────────────────────────────────┘
```

### 5.2 ログマスキング戦略

```python
class DataMasking:
    """
    監査ログに機密情報を含めない
    """
    
    MASKING_PATTERNS = {
        "credit_card": r"\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}",
        "ssn": r"\d{3}-\d{2}-\d{4}",
        "email": r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
        "phone": r"(\+?1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}",
        "api_key": r"[A-Za-z0-9_\-]{32,}",
        "bearer_token": r"Bearer [A-Za-z0-9._\-~+/]+"
    }
    
    def mask_sensitive_data(self, text):
        """
        機密データを [REDACTED_TYPE] に置換
        """
        for pattern_name, pattern in self.MASKING_PATTERNS.items():
            text = re.sub(
                pattern,
                f"[REDACTED_{pattern_name.upper()}]",
                text
            )
        return text
```

## 6. インシデント対応

### 6.1 セキュリティイベント分類

```
重大度分類:

CRITICAL
  ├─ Unauthorized access to RESTRICTED data
  ├─ Audit log tampering detected
  ├─ Multiple failed authentication attempts
  └─ System compromise indicators

HIGH
  ├─ CONFIDENTIAL data access without approval
  ├─ Unusual API usage pattern
  ├─ Security policy violation
  └─ Tool abuse attempt

MEDIUM
  ├─ Failed approval process
  ├─ Rate limiting triggered
  ├─ Malformed input detected
  └─ Cache anomaly

LOW
  ├─ Permission denied attempt
  ├─ Log rotation event
  ├─ Configuration change
  └─ Informational alerts
```

### 6.2 対応フロー

```python
class IncidentResponse:
    """
    セキュリティインシデント対応プロセス
    """
    
    def handle_security_event(self, event):
        severity = self.classify_severity(event)
        
        if severity == "CRITICAL":
            # 即座に対応
            self.pause_all_operations()  # 全処理停止
            self.trigger_incident_response_team()  # チーム招集
            self.snapshot_system_state()  # 状態保存
            self.initiate_forensics()  # フォレンジクス開始
            self.notify_stakeholders("immediate")
            
        elif severity == "HIGH":
            self.isolate_affected_resources()
            self.enable_enhanced_monitoring()
            self.schedule_response("within_1_hour")
            self.notify_stakeholders("urgent")
            
        elif severity == "MEDIUM":
            self.enable_detailed_logging()
            self.schedule_investigation("within_24_hours")
            self.notify_stakeholders("routine")
            
        else:
            self.log_for_review()
            self.schedule_review("weekly")
```

## 7. コンプライアンス フレームワーク

### 7.1 適用規制

```
対象規制:
- GDPR (General Data Protection Regulation)
  └─ EU顧客データ → 必須準拠
  
- CCPA (California Consumer Privacy Act)
  └─ CA州顧客データ → 必須準拠
  
- SOC 2 Type II
  └─ エンタープライズ顧客向け → 監査対象
  
- ISO 27001
  └─ 情報セキュリティ管理 → 目標
```

### 7.2 定期監査スケジュール

```
┌────────────────────────────────┐
│ Daily Checks                   │
├────────────────────────────────┤
│ • Audit log integrity          │
│ • Failed auth attempts (> 10)  │
│ • Abnormal API usage           │
└────────────────────────────────┘
        ↓
┌────────────────────────────────┐
│ Weekly Review                  │
├────────────────────────────────┤
│ • Access pattern analysis      │
│ • Permission changes           │
│ • Tool usage statistics        │
└────────────────────────────────┘
        ↓
┌────────────────────────────────┐
│ Monthly Audit                  │
├────────────────────────────────┤
│ • Compliance checklist         │
│ • Risk assessment              │
│ • Remediation status           │
└────────────────────────────────┘
        ↓
┌────────────────────────────────┐
│ Quarterly Review               │
├────────────────────────────────┤
│ • Policy updates               │
│ • Threat landscape analysis    │
│ • Security training            │
└────────────────────────────────┘
        ↓
┌────────────────────────────────┐
│ Annual Assessment              │
├────────────────────────────────┤
│ • External audit               │
│ • Penetration testing          │
│ • Architecture review          │
└────────────────────────────────┘
```

## 8. セキュリティチェックリスト

### デプロイ前チェックリスト

- [ ] すべての機密データが暗号化されているか
- [ ] 監査ログが append-only で実装されているか
- [ ] 権限チェックがすべてのツール呼び出しで実装されているか
- [ ] Human-in-the-loop チェックポイントが設定されているか
- [ ] レート制限が実装されているか
- [ ] エラーメッセージに機密情報が含まれていないか
- [ ] デジタル署名とハッシュチェーン検証が実装されているか
- [ ] セキュリティ監査ログが外部システムにレプリケートされているか

---

**最終更新**: 2026年4月23日  
**次回レビュー**: 2026年7月23日
