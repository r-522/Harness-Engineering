# CLAUDE.md: Project-Specific Constraints & Operational Rules

## WHAT: Tech Stack & Project Map

**Language & Frameworks**: TypeScript/Node.js (primary), Python (data/ML), Bash (automation)  
**Package Manager**: npm (Node.js), pip (Python)  
**Build Tools**: TypeScript compiler, esbuild, Make  
**VCS**: Git (GitHub, branch-protected main/production)  
**Testing**: Jest (unit), Playwright (E2E), pytest (Python)  
**CI/CD**: GitHub Actions (automated tests, security scan)

**Key Directories**:
- `src/`: Source code (TypeScript/JavaScript)
- `scripts/`: Automation scripts (Bash/Python)
- `tests/`: Test suites (Jest, Playwright)
- `.claude/`: Configuration (CLAUDE.md, agents, skills, hooks)
- `docs/`: Guides and architecture (ADR format preferred)

## WHY: Project Purpose & Scope

This is an **AI Agent Orchestration Harness** for organization-scale multi-agent coordination. Goals:
1. Enable Subagent Teams patterns (shared task state, distributed execution)
2. Optimize Context via Skills (lazy-load reference material)
3. Automate workflows via Hooks (pre-commit, pre-push, CI integration)

**Non-goals**: General-purpose CRUD apps, monolithic designs, manual script execution.

## HOW: Development Rules

### Coding Standards
- **Naming**: camelCase (variables/functions), PascalCase (classes/types), SCREAMING_SNAKE_CASE (constants)
- **Line Length**: Max 100 chars (UNIX convention, readability)
- **Comments**: Only "WHY is non-obvious" (hidden constraints, subtle invariants, workarounds). Never describe WHAT or current task context.
- **Error Handling**: Validate at system boundaries (user input, external APIs). Trust internal code guarantees.
- **No Premature Abstraction**: 3 similar lines > unfinished helper. Write for current requirements only.

### Testing Expectations
- **Coverage**: ≥80% for Core logic (utilities, orchestration, MCP drivers)
- **E2E**: Test golden path and edge cases (Skills invocation, Subagent handoff, Hook execution)
- **Pre-Commit**: `npm run test && npm run type-check` must pass before staging
- **CI**: All tests must pass on main; feature branches use GitHub branch protection

### File Editing Constraints
- **Never Modify**: `.git/`, `.github/workflows/` (without DevOps review)
- **Avoid**: Creating new top-level directories without ADR (Architecture Decision Record)
- **Dependencies**: Frozen in package-lock.json; bump only with justification in commit message

### Workflow & Automation
- **Commit Message Format**: `<type>(<scope>): <description>\n\n<body>`  
  - Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`  
  - Example: `feat(orchestration): add Agent Teams task state sync`
- **Pre-Push Hook**: Linting + Type check must pass
- **Branch Protection**: Feature branches → PR → 1 approval + CI pass → squash merge

### Debugging & Logging
- **Logs**: Use structured logging (JSON format) in production code
- **Env Vars**: Dev secrets in `.env.local` (never commit); production via GitHub Secrets
- **Debug Mode**: Enable via `DEBUG=* npm run dev` (process.env.DEBUG check)

### Security Boundary
- **Input Validation**: All user-provided data, API responses
- **Output Encoding**: XSS-safe rendering (React JSX, template escaping)
- **Secrets Scanning**: GitHub secrets detection enabled; fail if detected

---

## Skills & Subagent Invocation Triggers

Reference `AGENT.md` for detailed skill definitions. Skill bodies load only when invoked.

**Pre-loaded Instructions**: This CLAUDE.md only (under 70 lines). For deep context:
- Agent workflows → See `AGENT.md`
- Design patterns → See `DESIGN.md`
- Hook details → See `HOOKS.md`
- Stripe integration details → See `docs/stripe-guide.md` (invoke when needed)

---

## When to Escalate

1. **Architecture Decision**: Multi-day refactor or new dependency → ADR + team discussion
2. **Security Finding**: Potential vulnerability → security-review skill + manual audit
3. **Blocked by External**: API timeout, rate limit, deployment issue → log detail + notify team
4. **Ambiguous Requirements**: PR reviewer feedback conflict → ask clarifying questions before implementing

---

## Version & Sync

- **Last Updated**: 2026-05-02
- **Claude Code Version**: Opus 4.6+ (with thinking)
- **Effective From**: 2026-05-02 (Harness-Engineering active development)

