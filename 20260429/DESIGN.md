# DESIGN.md - 設計原則とアーキテクチャガイド

## 設計原則

### SOLID 原則の実装指針

#### S: Single Responsibility Principle（単一責任原則）

**原則**: 各クラス・関数は 1 つの理由でのみ変更される。

```typescript
// ❌ 悪い例：複数の責任を混在
class UserManager {
  fetchUser(id: string) { /* API 呼び出し */ }
  validateEmail(email: string) { /* 検証 */ }
  saveToDatabase(user: User) { /* DB 保存 */ }
  sendEmailNotification(user: User) { /* メール送信 */ }
}

// ✅ 良い例：責任を分離
class UserRepository {
  fetchUser(id: string) { /* API 呼び出し */ }
  save(user: User) { /* DB 保存 */ }
}

class UserValidator {
  validateEmail(email: string) { /* 検証 */ }
}

class NotificationService {
  sendEmailNotification(user: User) { /* メール送信 */ }
}
```

#### O: Open/Closed Principle（開放閉鎖原則）

**原則**: 拡張には開放、修正には閉鎖。新機能追加時に既存コードを変更しない。

```typescript
// ❌ 悪い例：新規ロジック追加で既存コード修正
if (type === "email") { sendEmail(); }
else if (type === "sms") { sendSms(); }  // 新規追加時に修正

// ✅ 良い例：interface で拡張
interface NotificationChannel {
  send(message: string): Promise<void>;
}

class EmailChannel implements NotificationChannel {
  send(message: string) { /* email */ }
}

class SmsChannel implements NotificationChannel {
  send(message: string) { /* sms */ }
}

// 新規チャネル追加時に既存コード不変
```

#### L: Liskov Substitution Principle（リスコフの置換原則）

**原則**: 派生クラスは基底クラスの代わりに使用できる（型安全性）。

```typescript
// ❌ 違反例：子クラスが親インターフェースを破る
interface PaymentProcessor {
  process(amount: number): boolean;
}

class CreditCardProcessor implements PaymentProcessor {
  process(amount: number) {
    if (amount > 1000) throw new Error("上限超過");  // ❌ 実装側で例外
    return true;
  }
}

// ✅ 改善例：インターフェース契約を守る
class CreditCardProcessor implements PaymentProcessor {
  process(amount: number): boolean {
    return amount <= 1000 && /* 処理実行 */;
  }
}
```

#### I: Interface Segregation Principle（インターフェース分離原則）

**原則**: 不要なメソッドに依存させない。大きなインターフェースは分割。

```typescript
// ❌ 悪い例：不要なメソッドまで実装強制
interface Worker {
  work(): void;
  eat(): void;  // 機械には不要
}

class Robot implements Worker {
  work() { /* 実行 */ }
  eat() { throw new Error("Not applicable"); }  // ❌ ダック・タイピング違反
}

// ✅ 良い例：インターフェース分割
interface Workable {
  work(): void;
}

interface Eatable {
  eat(): void;
}

class Robot implements Workable {
  work() { /* 実行 */ }
}

class Human implements Workable, Eatable {
  work() { /* 実行 */ }
  eat() { /* 実行 */ }
}
```

#### D: Dependency Inversion Principle（依存性逆転原則）

**原則**: 高水準モジュールは低水準モジュールに依存しない。両者とも抽象に依存。

```typescript
// ❌ 悪い例：具体実装に依存
class UserService {
  private db = new MySQLDatabase();  // 直接依存
  getUser(id: string) { return this.db.query(...); }
}

// ✅ 良い例：インターフェースに依存
interface Database {
  query(sql: string): any[];
}

class UserService {
  constructor(private db: Database) {}  // DI
  getUser(id: string) { return this.db.query(...); }
}

// 実装を切り替え可能
const service = new UserService(new MySQLDatabase());
// または
const service = new UserService(new PostgresDatabase());
```

### DRY 原則（Don't Repeat Yourself）

**実装指針**:
- 同じコードが 3 回以上出現 → 関数・ユーティリティ化
- 設定値の重複 → 環境変数・定数ファイルで一元管理
- テストコード内の重複 → setup/teardown で共通化

```typescript
// ❌ コード重複
const validate1 = (email: string) => /^.+@.+\..+$/.test(email);
const validate2 = (email: string) => /^.+@.+\..+$/.test(email);

// ✅ DRY
const isValidEmail = (email: string) => /^.+@.+\..+$/.test(email);
```

### KISS 原則（Keep It Simple, Stupid）

**実装指針**:
- 処理フロー：最短経路を選択（over-engineering を避ける）
- 関数長：20 行以下が目安
- 分岐深度：最大 3 層まで（4 層以上は関数抽出検討）

```typescript
// ❌ 複雑化の例：不要なラッパー層
class UserFactory {
  static create(...args) { return new User(...args); }
}

// ✅ シンプル版：直接 new で十分
const user = new User(...args);
```

---

## アーキテクチャパターン

### レイヤーアーキテクチャ（3層構成）

```
┌─────────────────────────────────┐
│   Presentation Layer (API/UI)   │
│  ├─ HTTP handlers / React pages │
│  └─ Request/Response formatting │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│   Application Layer (Logic)     │
│  ├─ Business rules              │
│  ├─ Validation                  │
│  └─ Service orchestration       │
└──────────────┬──────────────────┘
               ↓
┌─────────────────────────────────┐
│   Data Layer (Persistence)      │
│  ├─ Database queries (ORM)      │
│  ├─ Cache management            │
│  └─ External API integration    │
└─────────────────────────────────┘
```

**適用場面**: Web API, REST サービス  
**メリット**: 疎結合、テスト容易性、責任分離  
**デメリット**: 層を跨ぐとコストが増える（最小化する）

### MVC パターン（フロントエンド向け）

```
User Input
    ↓
┌─ Controller ─┐
│ ├─ Route     │
│ └─ Handler   │ → Model (State)
│              │    ↓
│              ├─→ View (Render)
│              │    ↓
│              └─ DOM Update
```

**React での実装例**:
```typescript
// Model: useState + useReducer
const [state, dispatch] = useReducer(reducer, initialState);

// View: JSX component
return <UserList users={state.users} />;

// Controller: event handler
const onDelete = (id) => dispatch({ type: "DELETE", id });
```

### Service Locator パターン（外部依存管理）

**使用場面**: API 呼び出し、外部サービス統合

```typescript
// ✅ 推奨：DI で管理
class OrderService {
  constructor(
    private paymentService: PaymentService,
    private emailService: EmailService
  ) {}
}

// テスト時は Mock を注入
const mockPayment = { process: jest.fn() };
const service = new OrderService(mockPayment, mockEmail);
```

---

## ディレクトリ構成ガイド

### Node.js/TypeScript プロジェクト

```
project-root/
├── src/
│   ├── api/
│   │   ├── routes/              # Express routes
│   │   ├── controllers/         # Request handlers
│   │   └── middlewares/         # Auth, validation, etc
│   ├── services/
│   │   ├── user-service.ts      # Business logic
│   │   ├── payment-service.ts
│   │   └── notification-service.ts
│   ├── repositories/            # Data access (DB)
│   │   ├── user-repository.ts
│   │   └── order-repository.ts
│   ├── models/
│   │   ├── user.model.ts        # Data structures (Typeorm, Prisma)
│   │   └── order.model.ts
│   ├── utils/
│   │   ├── validators.ts        # 共通検証
│   │   ├── formatters.ts        # 共通フォーマット
│   │   └── logger.ts            # ログ
│   ├── config/
│   │   ├── database.ts          # DB 設定
│   │   └── env.ts               # 環境変数
│   ├── types/
│   │   └── index.ts             # 型定義
│   └── index.ts                 # Entry point
├── test/
│   ├── unit/                    # ユニットテスト
│   ├── integration/             # 統合テスト
│   └── fixtures/                # テストデータ
├── .env.example                 # 環境変数テンプレート
├── package.json
├── tsconfig.json
└── .claude/
    ├── CLAUDE.md                # プロジェクト規約
    ├── AGENT.md                 # エージェント定義
    └── ...
```

### Python プロジェクト

```
project-root/
├── src/
│   ├── app/
│   │   ├── __init__.py
│   │   ├── main.py              # Entry point
│   │   ├── api/
│   │   │   ├── routes/          # Flask/FastAPI routes
│   │   │   └── schemas.py       # Pydantic models
│   │   ├── services/
│   │   │   └── user_service.py
│   │   ├── repositories/
│   │   │   └── user_repository.py
│   │   ├── utils/
│   │   │   ├── validators.py
│   │   │   └── logger.py
│   │   └── config.py            # 環境設定
│   └── __init__.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py              # pytest fixtures
├── requirements.txt
├── .env.example
└── .claude/
    └── ...
```

---

## モジュール化ガイドライン

### 関数サイズの目安

| 関数行数 | 判定 | 対処 |
|---|---|---|
| 0-20 行 | ✅ 良好 | そのまま |
| 20-50 行 | ⚠️ 要検討 | 複雑度高い場合は分割 |
| 50+ 行 | ❌ 要分割 | 即座に関数抽出 |

### クラス（モジュール）サイズの目安

| メソッド数 | 判定 | 対処 |
|---|---|---|
| 3-7 個 | ✅ 最適 | 単一責任で明確 |
| 8-15 個 | ⚠️ 要検討 | 責任が複数か確認、分割検討 |
| 15+ 個 | ❌ 要分割 | 複数の責任を持つ。domain ごとに分割 |

---

## パフォーマンス最適化パターン

### キャッシング戦略

```typescript
// ❌ キャッシュなし：毎回 DB 照会
const getUser = (id: string) => db.query(...);

// ✅ メモリキャッシュ（小規模）
const userCache = new Map();
const getUser = (id: string) => {
  if (userCache.has(id)) return userCache.get(id);
  const user = db.query(...);
  userCache.set(id, user);
  return user;
};

// ✅ Redis キャッシュ（大規模）
const getUser = async (id: string) => {
  const cached = await redis.get(`user:${id}`);
  if (cached) return JSON.parse(cached);
  const user = await db.query(...);
  await redis.set(`user:${id}`, JSON.stringify(user), "EX", 3600);
  return user;
};
```

### 遅延読み込み（Lazy Loading）

```typescript
// ❌ 不要な関連データまで読込
const users = await db.query(`
  SELECT * FROM users 
  JOIN posts ON users.id = posts.user_id
  JOIN comments ON posts.id = comments.post_id
`);

// ✅ 必要な時点で読込
const users = await db.query("SELECT * FROM users");
// 詳細が必要な場合のみ：
const userWithPosts = await db.query(
  "SELECT * FROM users JOIN posts WHERE users.id = ?"
);
```

---

## テスト戦略

### テスト ピラミッド

```
        △
       /│\
      / │ \
     /  │  \ E2E Tests (10%)
    /   │   \
   / Unit Tests (70%) \
  / Integration Tests (20%) \
 /─────────────────────────────\
```

### テストパターン

**ユニットテスト**: 単一関数の入出力検証

```typescript
describe("isValidEmail", () => {
  it("returns true for valid email", () => {
    expect(isValidEmail("user@example.com")).toBe(true);
  });
  it("returns false for invalid email", () => {
    expect(isValidEmail("invalid")).toBe(false);
  });
});
```

**統合テスト**: サービス間の連携検証

```typescript
describe("UserService", () => {
  it("creates user and sends welcome email", async () => {
    const user = await userService.create({ email: "test@example.com" });
    expect(emailService.send).toHaveBeenCalledWith(user.email);
  });
});
```

---

**最終更新**: 2026-04-29  
**参照すべきファイル**: CLAUDE.md（禁止事項）  
**次のステップ**: CONTEXT.md でトークン管理を学習
