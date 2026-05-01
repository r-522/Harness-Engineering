# DESIGN.md - 設計原則・アーキテクチャパターン・ディレクトリ構成

## 適用する設計原則

### SOLID 原則（5つの設計原則）

| 原則 | 説明 | 実装例 |
|---|---|---|
| **S**ingle Responsibility | クラス/関数は 1 つの責務のみ | `UserValidator`, `EmailService`, `DataRepository` |
| **O**pen/Closed | 拡張に開かれ、修正に閉じている | Interface 使用、Strategy Pattern |
| **L**iskov Substitution | サブクラスは親クラスの仕様を守る | `Repository` interface の継承実装 |
| **I**nterface Segregation | 不要なメソッドを強制しない | 小粒な interface 設計 |
| **D**ependency Inversion | 具体的実装ではなく抽象に依存 | DI Container / Constructor Injection |

**実装ガイドライン**:

```typescript
// ❌ 悪い例: SRP 違反（複数責務）
class User {
  validate() { /* 入力検証 */ }
  save() { /* DB保存 */ }
  sendEmail() { /* メール送信 */ }
}

// ✅ 良い例: 責務の分離
class UserValidator { /* 入力検証のみ */ }
class UserRepository { /* DB操作のみ */ }
class EmailService { /* メール送信のみ */ }
```

### DRY 原則（Don't Repeat Yourself）

**規則**: 同じロジックが 3 回以上現れたら、共通化の候補

**適用**:

1. **関数化**: 重複したコード → 共有関数に抽出
2. **モジュール化**: 複数ファイルで使用 → 独立モジュール化
3. **ユーティリティ**: 汎用的なロジック → `utils/` に集約

**例**:

```typescript
// ❌ 悪い例: ログイン・登録・パスワードリセットで同じメール検証
function validateLoginEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

function validateSignupEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// ✅ 良い例: 共通関数に抽出
function isValidEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}
```

### KISS 原則（Keep It Simple, Stupid）

**規則**: 過剰な抽象化・設計パターンは避ける

**判断基準**:

- 複雑度が 10 を超えるなら関数分割を検討
- 3 段階以上のネストは簡潔化を検討
- 導入するライブラリは「必要性」を明確に

### YAGNI 原則（You Aren't Gonna Need It）

**規則**: 今必要でない機能は実装しない

**適用シーン**:

- 「将来的に必要かもしれない」汎用オプション
- 複数環境サポート（現在は 1 環境のみ）
- プラグインシステム（実際のプラグインがない場合）

---

## アーキテクチャパターン

### 推奨アーキテクチャ: 層状アーキテクチャ（Layered Architecture）

**構成**:

```
┌─────────────────────────────────┐
│  Presentation Layer             │ (API/CLI)
│  ├─ routes/
│  └─ handlers/
├─────────────────────────────────┤
│  Application Layer              │ (ビジネスロジック)
│  ├─ services/
│  └─ validators/
├─────────────────────────────────┤
│  Data Access Layer              │ (DB・外部API)
│  ├─ repositories/
│  └─ models/
├─────────────────────────────────┤
│  Utility / Cross-cutting        │ (共通機能)
│  ├─ utils/
│  ├─ middleware/
│  └─ errors/
└─────────────────────────────────┘
```

**特徴**:
- 各層は上位層にのみ依存（下位層に依存しない）
- テストが容易（各層を独立にテスト）
- チームスケーリングに適している

### イベント駆動パターン（EventEmitter / Observer）

**用途**: 非同期処理、モジュール間通信、ロギング

**実装例**:

```typescript
import { EventEmitter } from 'events';

const userEvents = new EventEmitter();

// ユーザー登録完了時にイベント発行
userEvents.emit('user:created', { userId: '123', email: 'user@example.com' });

// 複数のリスナーが反応
userEvents.on('user:created', (user) => {
  emailService.sendWelcomeEmail(user.email);
});

userEvents.on('user:created', (user) => {
  analytics.track('user_signup', { userId: user.userId });
});
```

### リポジトリパターン（Data Access Abstraction）

**目的**: DB操作の抽象化、テスト容易性向上

**実装構成**:

```typescript
// 1. Interface 定義
interface IUserRepository {
  findById(id: string): Promise<User | null>;
  save(user: User): Promise<void>;
  delete(id: string): Promise<void>;
}

// 2. 実装
class UserRepository implements IUserRepository {
  async findById(id: string): Promise<User | null> {
    return db.query('SELECT * FROM users WHERE id = ?', [id]);
  }
  // ...
}

// 3. 注入
class UserService {
  constructor(private repo: IUserRepository) {}
  
  async getUser(id: string) {
    return this.repo.findById(id);
  }
}
```

---

## ディレクトリ構成ガイド

### 推奨ディレクトリ構成

```
project-root/
│
├── src/
│   ├── api/                    # API エンドポイント（Express Router等）
│   │   ├── routes/
│   │   ├── handlers/
│   │   └── middleware/
│   │
│   ├── services/               # ビジネスロジック
│   │   ├── user-service.ts
│   │   ├── auth-service.ts
│   │   └── email-service.ts
│   │
│   ├── repositories/           # データアクセス層
│   │   ├── user-repository.ts
│   │   ├── interfaces/         # Repository interface
│   │   └── implementations/    # DB実装
│   │
│   ├── models/                 # データ型・スキーマ
│   │   ├── user.ts
│   │   ├── auth.ts
│   │   └── types.ts            # 共有型定義
│   │
│   ├── validators/             # 入力検証（Zod等）
│   │   ├── user-validator.ts
│   │   └── auth-validator.ts
│   │
│   ├── utils/                  # ユーティリティ関数
│   │   ├── logger.ts
│   │   ├── error-handler.ts
│   │   └── helpers.ts
│   │
│   ├── errors/                 # カスタムエラークラス
│   │   ├── application-error.ts
│   │   ├── validation-error.ts
│   │   └── not-found-error.ts
│   │
│   ├── config/                 # 設定
│   │   ├── database.ts
│   │   ├── env.ts
│   │   └── constants.ts
│   │
│   └── index.ts                # エントリーポイント
│
├── tests/
│   ├── unit/                   # ユニットテスト
│   │   ├── services/
│   │   ├── repositories/
│   │   ├── validators/
│   │   └── utils/
│   │
│   ├── integration/            # 統合テスト
│   │   ├── api/
│   │   └── repositories/
│   │
│   └── fixtures/               # テストデータ・モック
│       └── mock-data.ts
│
├── docs/                       # ドキュメント
│   ├── API.md
│   ├── ARCHITECTURE.md
│   └── DEPLOYMENT.md
│
├── .github/
│   └── workflows/              # GitHub Actions
│       ├── test.yml
│       ├── lint.yml
│       └── build.yml
│
├── .claude/                    # Claude Code 設定
│   ├── CLAUDE.md
│   ├── AGENT.md
│   ├── DESIGN.md
│   ├── HOOKS.md
│   └── settings.json
│
├── .env.example                # 環境変数テンプレート
├── tsconfig.json               # TypeScript設定
├── jest.config.js              # Jest設定
├── package.json
├── pnpm-lock.yaml              # 依存関係ロック
└── README.md
```

### ディレクトリ命名規則

| パターン | ルール | 例 |
|---|---|---|
| **複数ファイルの集約** | kebab-case（複数形） | `services/`, `repositories/`, `handlers/` |
| **単一ファイル** | kebab-case（単数形） | `user-service.ts`, `email-validator.ts` |
| **機能グループ** | kebab-case | `api/routes/`, `tests/unit/services/` |
| **ユーティリティ** | `utils/`, `helpers/`, `lib/` | `utils/logger.ts` |

### インポートパス管理（Path Alias）

**tsconfig.json の設定**:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@api/*": ["src/api/*"],
      "@services/*": ["src/services/*"],
      "@repositories/*": ["src/repositories/*"],
      "@models/*": ["src/models/*"],
      "@utils/*": ["src/utils/*"],
      "@errors/*": ["src/errors/*"],
      "@config/*": ["src/config/*"],
      "@tests/*": ["tests/*"]
    }
  }
}
```

**使用例**:

```typescript
// ❌ 相対パスは避ける
import { UserService } from '../../services/user-service';

// ✅ Path Alias を使用
import { UserService } from '@services/user-service';
```

### テストファイルの配置

**ルール**: テストは対応するソースファイルと同じ階層構造

```
src/
├── services/
│   └── user-service.ts

tests/
├── unit/
│   └── services/
│       └── user-service.test.ts    # ← 同じ階層構造
```

---

## アーキテクチャ決定記録（ADR）

### ADR-001: TypeScript Strict Mode の採用

**背景**: ランタイムエラーの早期発見

**決定**: `tsconfig.json` で `strict: true` を必須

**影響**: 
- 開発効率: +5% （型定義の手間）
- バグ減少: -20% （ランタイムエラー）

### ADR-002: Zod によるランタイムバリデーション

**背景**: TypeScript の型システムはコンパイル時のみで、ランタイムバリデーションが必要

**決定**: API 入力・DB取得後に Zod スキーマでバリデーション

**実装例**:

```typescript
import { z } from 'zod';

const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
  name: z.string(),
});

export function createUser(data: unknown) {
  const validData = createUserSchema.parse(data);
  // validData は 100% 安全
}
```

### ADR-003: イベント駆動による結合度低減

**背景**: サービス間の強い結合はスケーリングの障害

**決定**: ドメインイベント発行 → リスナーが反応する設計

**利点**:
- 新機能追加時、既存コード変更不要
- テスト容易性向上
- マイクロサービス化への準備

---

## レビューチェックリスト

実装完了後、以下の設計原則が守られているか確認：

- [ ] 各関数は 1 つの責務のみか（SRP）
- [ ] 同じロジックが 3 回以上ないか（DRY）
- [ ] 複雑度は 10 以下か（KISS）
- [ ] 必要な機能のみ実装されているか（YAGNI）
- [ ] テストが書きやすい構成か
- [ ] ディレクトリ構成が階層構造を守っているか
- [ ] Path Alias が正しく使用されているか
- [ ] エラー処理は統一されているか

---

## 参考文献

- [SOLID Principles - Robert C. Martin](https://en.wikipedia.org/wiki/SOLID)
- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Layered Architecture Pattern](https://martinfowler.com/api-guide/patterns/)
- [TypeScript Design Patterns](https://www.typescriptlang.org/docs/handbook/2/narrowing.html)

