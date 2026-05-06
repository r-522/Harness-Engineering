# CLAUDE.md - プロジェクト共有ルール

## 目的
このファイルは全 Claude Code セッションに **自動読み込み** されます。プロジェクト全体に適用される最小限の制約のみを記述します。

---

## I. コーディング規約

1. **命名規則**
   - JavaScript/TypeScript: camelCase（変数・関数）、PascalCase（クラス）
   - Python: snake_case（変数・関数）、PascalCase（クラス）
   - ファイル名: kebab-case（`.ts`, `.py`）

2. **フォーマット**
   - Auto-format hooks が `PostToolUse` で自動実行される（Prettier, Black, ESLint）
   - 手動フォーマットは不要（hook が処理）

3. **コメント** （参照: AGENT.md）
   - WHY が非自明な場合のみ1行コメント
   - docstring は不要（命名で表現）

---

## II. 禁止事項・動作制限

### 代表的な制約

| 制約 | 理由 |
|---|---|
| `git reset --hard` 未確認使用禁止 | 未コミット作業が消失する |
| `git push --force` 未確認使用禁止 | 他の開発者作業が上書きされる |
| 本番環境への直接デプロイ禁止 | ステージング検証が必須 |
| `.env`, `credentials.json` の commit 禁止 | シークレット漏洩リスク |
| Context window 70% 超過時の新規 Subagent spawn 禁止 | Performance degradation 回避 |

### 実装例外
- 既知の脆弱性対応時は即座に修正（セキュリティ > 変更管理）
- テスト失敗時は root cause を診断（bypass flag `--no-verify` 使用禁止）
- 確認なし destructive operation 禁止（削除、リセット、上書き）

---

## III. ファイル編集・テスト・ビルドポリシー

### 編集
- 既存ファイル: **Edit tool** 使用（差分のみ送信）
- 新規作成: **Write tool** 使用（新規ファイルのみ）
- 削除: confirm 後に実行（一度削除は復旧困難）

### テスト
- 全 commit 前に、`npm test`, `pytest` を実行
- ローカルテスト成功 = Git push 許可
- CI/CD 失敗時は revert または修正 commit

### ビルド
- Web UI 変更は必ず dev server で実装確認
- フロントエンド: ブラウザで動作検証 ✓ Type checking ✓ Test suite ✓
- バックエンド: API エンドポイント動作確認

---

## IV. Context Management（重要）

### Context Window 危機管理

1. **Monitor**: `context_usage` が 70% 超過時は段階的削減
2. **Prioritize**: 現在のタスク関連情報を優先保持
3. **Compress**: SessionEnd hook で過去ログ削除
4. **Separate**: Domain-specific guide は別ファイル参照

参照: `DESIGN.md`, `AGENT.md`（外部リンク形式）

---

## V. Progressive Disclosure

必要な情報は「自動読み込み」ではなく「参照指示」で提供：

```markdown
# CLAUDE.md（最小限）
# Agent behavior: See AGENT.md
# Design principles: See DESIGN.md
# Skills definition: See SKILLS.md
```

こうすることで CLAUDE.md は **15行以内** で完結。

---

## 備考

このファイルは **自動的に全セッション読み込み** されます。
ローカルオーバーライド: `.claude/settings.local.json` を参照。

最終更新: 2026-05-06
