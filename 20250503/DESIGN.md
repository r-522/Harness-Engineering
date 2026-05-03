# Design Principles & Architecture Guide

## Core Design Principles

Claude Code projects should follow established software engineering principles to maximize maintainability, scalability, and collaboration effectiveness. This document defines the philosophy and patterns that govern architectural decisions.

### SOLID Principles

#### **S — Single Responsibility Principle**
- Each function/module solves one problem cleanly
- If a function has multiple reasons to change, split it
- Example: A file parser should not also handle UI rendering

#### **O — Open/Closed Principle**
- Code is open for extension, closed for modification
- Use configuration files (JSON, YAML) for behavior changes, not code changes
- Add new features without rewriting existing logic

#### **L — Liskov Substitution Principle**
- Derived types must be substitutable for base types
- Contracts (inputs/outputs) must hold across implementations
- Avoid breaking interface changes

#### **I — Interface Segregation Principle**
- Classes/modules should not depend on interfaces they don't use
- Split large interfaces into smaller, focused contracts
- Example: "EmailSender" ≠ "EmailSender + Logger + Analytics"

#### **D — Dependency Inversion Principle**
- Depend on abstractions, not concrete implementations
- Pass dependencies explicitly (constructor injection, function parameters)
- Avoid hardcoded service locators

### DRY (Don't Repeat Yourself)

- **Threshold**: Avoid repetition after 3 occurrences
- **Before 3 occurrences**: Tolerate duplication; premature abstraction adds complexity
- **At 3+ occurrences**: Extract to shared function/module
- **Exception**: One-off scripts and glue code may tolerate repetition if abstraction is forced

### KISS (Keep It Simple, Stupid)

- Simplest solution that solves the problem wins
- No fancy patterns unless the problem demands them
- Add complexity only when concrete requirements justify it

### YAGNI (You Aren't Gonna Need It)

- Do not implement hypothetical features
- Do not add error handling for impossible scenarios
- Trust framework guarantees and internal code contracts
- Only validate at system boundaries (user input, external APIs)

---

## Directory Structure Template

### Web/Frontend Projects

```
project-root/
├── src/
│   ├── components/          # Reusable React/Vue/Angular components
│   ├── pages/              # Page-level components (Next.js, Remix)
│   ├── services/           # API clients, business logic, external integrations
│   ├── hooks/              # Custom React hooks (if applicable)
│   ├── stores/             # State management (Redux, Zustand, Pinia)
│   ├── utils/              # Helper functions, pure functions
│   ├── styles/             # Global CSS, theme definitions
│   ├── types/              # TypeScript interfaces, types (if TS project)
│   └── App.tsx             # Root component
├── tests/
│   ├── unit/               # Unit tests (functions, components)
│   ├── integration/        # Integration tests (component interactions)
│   └── e2e/                # End-to-end tests (Cypress, Playwright)
├── public/                 # Static assets (favicon, robots.txt)
├── docs/                   # Project documentation
├── .env.example            # Environment variable template (NO secrets)
├── package.json            # Dependencies, scripts
├── tsconfig.json           # TypeScript configuration (if TS)
├── jest.config.js          # Test configuration
└── README.md               # Getting started guide
```

### Backend/API Projects

```
project-root/
├── src/
│   ├── api/                # Route handlers, controllers
│   ├── services/           # Business logic, domain logic
│   ├── repositories/       # Data access layer (database, cache)
│   ├── models/             # Data models, schemas
│   ├── middleware/         # Auth, logging, error handling
│   ├── config/             # Configuration loaders
│   ├── utils/              # Helper functions
│   ├── types/              # TypeScript types, interfaces
│   └── main.ts             # Application entry point
├── tests/
│   ├── unit/               # Unit tests
│   ├── integration/        # API endpoint tests
│   └── fixtures/           # Mock data, test databases
├── migrations/             # Database schema versions (if applicable)
├── docs/                   # API documentation, architecture diagrams
├── .env.example            # Environment template
├── docker-compose.yml      # Local development services (DB, Redis, etc.)
├── Dockerfile              # Container image definition
└── README.md               # Setup & deployment guide
```

### Python Projects

```
project-root/
├── src/
│   └── project_name/
│       ├── __init__.py
│       ├── main.py
│       ├── services/       # Business logic
│       ├── models/         # Data structures
│       ├── utils/          # Utilities
│       ├── config.py       # Configuration
│       └── logging.py      # Logging setup
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py         # Pytest fixtures
├── docs/                   # Sphinx docs, guides
├── pyproject.toml          # Modern Python project config
├── requirements.txt        # Frozen dependencies (for reproducibility)
├── .env.example            # Environment variables
└── README.md
```

### Monorepo (Turborepo, Nx)

```
monorepo-root/
├── packages/
│   ├── shared/             # Shared utilities, types, components
│   ├── api/                # Backend service
│   ├── web/                # Frontend SPA/PWA
│   ├── cli/                # Command-line tool
│   └── sdk/                # Public SDK
├── apps/                   # Full applications
│   ├── dashboard/
│   └── admin-panel/
├── tools/                  # Build scripts, code generation
├── docs/                   # Monorepo documentation
├── turbo.json              # Turborepo config
└── package.json            # Root workspace config
```

---

## Naming Conventions

### Variables & Functions

| Context | Convention | Example |
|---------|-----------|---------|
| **JavaScript/TypeScript** | camelCase | `getUserById`, `isAuthenticatedUser` |
| **Python/Bash** | snake_case | `get_user_by_id`, `is_authenticated_user` |
| **Rust/C++** | snake_case (functions), UPPER_CASE (constants) | `fn get_user_by_id()`, `const MAX_RETRIES` |
| **Boolean variables** | Prefix: `is`, `has`, `can`, `should` | `isActive`, `hasPermission`, `canEdit` |

### Files & Directories

| Type | Convention | Example |
|---|---|---|
| **Components** | PascalCase | `UserCard.tsx`, `LoginForm.jsx` |
| **Utilities** | camelCase or snake_case | `stringUtils.ts`, `date_helpers.py` |
| **Configuration** | kebab-case or lowercase | `babel.config.js`, `.env.local` |
| **Database migrations** | Timestamp + snake_case | `20250503_001_create_users_table.sql` |

### Git Branches

| Type | Convention | Example |
|---|---|---|
| **Feature** | `feature/description` | `feature/user-authentication` |
| **Bug fix** | `bugfix/issue-title` | `bugfix/session-timeout-bug` |
| **Hotfix** | `hotfix/critical-issue` | `hotfix/payment-api-error` |
| **Refactor** | `refactor/area` | `refactor/auth-service` |

---

## Code Quality Standards

### Commenting

- **Default**: No comments (let clear naming speak for itself)
- **When to comment**: 
  - Hidden constraints or non-obvious invariants
  - Workarounds for specific bugs (reference the bug/issue)
  - Complex algorithm explanations (1-2 lines max)
  - **NEVER**: Comment what the code does (that's the job of naming)
  - **NEVER**: Document business logic in comments; move to docstring if truly necessary

### Documentation

- **README.md**: Getting started, dev setup, running tests
- **API docs**: Auto-generated from code annotations (Swagger, Sphinx, etc.) or explicit docs file
- **Architecture doc**: High-level design, diagrams, decision logs (separate file if >500 lines)
- **TypeScript/Python docstrings**: Only for public APIs; 1-2 lines max

### Test Strategy

- **Unit tests**: Single function/component in isolation (~80% coverage)
- **Integration tests**: Component interactions, API endpoints (~10% coverage)
- **E2E tests**: Critical user journeys (~10% coverage)
- **Coverage target**: 60-70% is realistic; >90% often indicates over-testing

#### Test File Organization

```
tests/
├── unit/
│   ├── services/
│   │   └── user.service.test.ts    # Tests for src/services/user.service.ts
│   └── utils/
│       └── string.utils.test.ts
├── integration/
│   ├── api.test.ts                 # Full API endpoint tests
│   └── database.test.ts
└── fixtures/
    ├── user.fixture.ts             # Mock data
    └── database.fixture.ts         # Test database setup
```

### Build & Deployment

- **Local development**: `npm run dev`, `cargo run`, `python manage.py runserver`
- **Type checking**: Required before commit (TypeScript, Python mypy)
- **Linting**: Run before commit (ESLint, Pylint, Clippy)
- **Testing**: All tests pass before merge to main
- **Build**: Optimize/minify for production

---

## Error Handling Strategy

### Validation Layers

| Layer | Responsibility | Example |
|---|---|---|
| **User Input** | Validate all external input | Form field validation, API request body schema |
| **API Boundary** | Validate data from external services | HTTP response validation, third-party API contracts |
| **Internal Code** | Trust contracts; minimal validation | No need to validate function parameters from internal code |
| **System Boundary** | Catch edge cases (file I/O, network) | Handle file not found, connection timeout |

### Exception Handling

- **Catch specific exceptions**, not generic `Exception`
- **Propagate** vs **Handle**: Propagate if caller should decide; handle if you can recover
- **Log with context**: Include user ID, request ID, timestamp, full stack trace
- **No silent failures**: Always log or report errors

### Error Response Format

```javascript
// Good: Structured error response
{
  error: {
    code: "VALIDATION_ERROR",
    message: "Email is required",
    details: [
      { field: "email", reason: "required" }
    ]
  }
}

// Avoid: Unstructured error message
{
  error: "Something went wrong"
}
```

---

## Performance Considerations

### Frontend

- **Code splitting**: Lazy-load routes and heavy components
- **Image optimization**: Use WebP, srcset, responsive images
- **Bundle size**: Monitor with `webpack-bundle-analyzer`
- **Rendering**: Use React.memo, useMemo, useCallback sparingly (only on proven bottlenecks)

### Backend

- **Database queries**: Use indexes, avoid N+1 queries
- **Caching**: Redis for hot data, HTTP caching headers
- **Rate limiting**: Protect APIs from abuse
- **Async processing**: Queue heavy jobs (message queues, background workers)

### General

- **Measure first**: Use profilers before optimizing
- **80/20 rule**: 20% of code causes 80% of slowness
- **Trade-off awareness**: Sometimes clarity > performance (until proven bottleneck)

---

## Security Baseline

### Code-Level

- **No hardcoded secrets** (.env files, environment variables only)
- **Input validation** at all user-facing APIs
- **SQL injection prevention** (parameterized queries, ORMs)
- **CSRF protection** (SameSite cookies, CSRF tokens)
- **XSS prevention** (HTML escaping, Content Security Policy)

### Infrastructure

- **TLS/HTTPS** for all network communication
- **Authentication**: Implement proper session/token management
- **Authorization**: Role-based access control (RBAC) for sensitive operations
- **Secrets management**: Use environment variables, secret managers (not hardcoded)

---

## References

- [SOLID Principles](https://en.wikipedia.org/wiki/SOLID)
- [Clean Code by Robert C. Martin](https://www.oreilly.com/library/view/clean-code-a/9780136083238/)
- [Design Patterns: Elements of Reusable Object-Oriented Software](https://refactoring.guru/design-patterns/book)

---

**Last Updated**: 2025-05-03  
**Audience**: Medium to advanced developers working with Claude Code
