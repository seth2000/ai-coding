# AI Coding Workflows: Pitfalls, Philosophies, Tooling & Architecture
Practical lessons for building real software with AI coding agents, drawn from the r/ClaudeAI discussion [*"Half a year of AI coding got me nowhere"*](https://www.reddit.com/r/ClaudeAI/comments/1y1my5l/half_a_year_of_ai_coding_got_me_nowhere/) (Nov 2026) and combined with patterns that hold up in production.

> **Last verified:** 26 Nov 2026
> **Status:** Stable guidance. The tooling examples age faster than the practices.
> Tool commands and behaviour were checked against public documentation in November 2026. AI tooling changes fast, so check the linked docs before relying on any detail.

## Contents
1. [Common pitfalls](#1-common-pitfalls)
2. [How experienced builders avoid them](#2-how-experienced-builders-avoid-them)
3. [Tools and commands that help](#3-tools-and-commands-that-help)
4. [Two philosophies: Tech Lead vs Product Manager](#4-two-philosophies-tech-lead-vs-product-manager)
5. [A lean CLAUDE.md with progressive disclosure and ADRs](#5-a-lean-claudemd-with-progressive-disclosure-and-adrs)
6. [Worked example: a PostgreSQL + .NET sample project](#6-worked-example-a-postgresql--net-sample-project)
7. [References](#7-references)
8. [Related reading](#8-related-reading)

---

## 1. Common pitfalls
| Pitfall | What happens |
|---|---|
| **Big-bang design**: asking the AI to design and generate the whole app on day one | Nobody, including you, understands the system. Later bugs mean debugging a large, unfamiliar codebase: one step forward, two steps back. |
| **Letting the AI drive the vision**: vague prompts like *"continue development"* | The model has no long-term product view. It guesses, over-engineers, and produces brittle, inconsistent code. |
| **Endless bug-fix loops**: re-prompting the same failed fix | The AI stacks patches instead of finding the root cause, adds more buggy code, and burns through usage limits. |
| **Spaghetti and duplicate code**: new features bolted on without refactoring | Agents rarely see the whole repo, so they duplicate logic, introduce competing styles, and leave dead code behind. |
| **Trusting AI-written tests**: "everything is done" and the tests are green | Tests may be shallow, may not reflect real behaviour, or may have been edited to pass. Basic features can still be broken. |
| **Context bloat and session drift**: one very long chat, bloated instruction files, unused MCP servers | As context fills up, the model forgets rules, contradicts itself, and breaks working features. |

## 2. How experienced builders avoid them
- **Build thin vertical slices.** Finish and manually check one small feature end-to-end before starting the next.
- **Use a two-strike rule.** If the AI fails to fix a bug twice, stop. Revert to the last clean commit, find the cause yourself, then give the agent a short, targeted instruction.
- **Separate writing from checking.**
  - Ask for a **failing test that reproduces the bug** before any code change, and/or
  - Use a **separate reviewer session** (fresh context) to check the diff against the acceptance criteria.
- **Keep short, human-owned documentation.** A concise spec plus Architecture Decision Records (ADRs) gives every new session the same source of truth without bloating the prompt.
- **Start fresh sessions often.** One task per session; don't let a chat run forever.

## 3. Tools and commands that help

### 3.1 Claude Code commands
| Command | What it does | When to use |
|---|---|---|
| `/simplify` | Bundled skill. Reviews your current changes for code reuse, quality and efficiency in parallel, then applies clean-ups. | After a feature works, before committing. Several community members say they use it even more than code review. |
| `/code-review` (alias `/review`) | Checks the current diff for correctness bugs. `--fix` applies findings; pass a PR number to review a pull request. | Before every commit or PR. |
| `/security-review` | Checks the diff for security issues. | Anything touching auth, user input, or secrets. |
| `/context`, `/compact`, `/clear` | See what's filling the context window, summarise it, or start fresh. | When responses start drifting. |

**Frontend tip:** generate a visual mock-up first and hand it to the agent along with a UI/UX skill, so it builds against a concrete target instead of guessing.

### 3.2 Guardrails that actually enforce rules

> `CLAUDE.md` / `AGENTS.md` are **guidance, not enforcement**. Anything that must *never* happen belongs in a linter, a CI check, or a hook.
- **Linters and analyzers.** Turn the AI's repeat mistakes into rules so bad code can't pass.
  - .NET: Roslyn analyzers, `dotnet format --verify-no-changes`
  - Kotlin: Detekt
  - JavaScript/TypeScript: ESLint (or oxlint)
  - SQL: `sqlfluff` for style; a migration linter such as `squawk` for risky PostgreSQL migrations
- **Hooks.** In Claude Code, a `PreToolUse` hook can block an action outright (for example, editing an already-applied migration).
- **CI as the final gate.** Build, tests, format check and lint must pass before merge, whoever wrote the code.

### 3.3 Agent harnesses, IDEs and MCP servers
| Tool | What it is | Why it helps |
|---|---|---|
| **[pstack](https://github.com/michael-denyer/pstack-claude)** | An engineering harness: a skill layer for Claude Code (also ported to Codex and other agents) that pushes disciplined practices such as TDD, verification and review. | Requires concrete evidence before a task is marked "done", without filling the context window with long workflow prompts. |
| **[bb](https://getbb.app/)** | An open-source "agentic IDE" with a plugin architecture. Runs multiple coding agents (Claude Code, Codex, Cursor and others) in one workspace. | You shape the editor around your workflow with plugins (or by prompting an agent to change it) instead of adapting to a fixed tool. |
| **[qt-commander](https://github.com/tiefeiyu/qt-commander)** | An MCP server that lets an agent inspect and drive a running Qt (QWidget/QML) desktop app, like Playwright for Qt. | A good example of a *targeted* MCP server: it gives the agent eyes on real UI state instead of guessing. |
A setup mentioned in the thread pairs the two: **pstack** as the engineering harness, running inside **bb** as the customisable IDE.

> **MCP hygiene:** every connected MCP server adds tool descriptions to the context. Keep only the ones you actively use for the current project.

### 3.4 Planning and specification tools
| Tool | Approach | Notes |
|---|---|---|
| **[Spec Kit](https://github.github.com/spec-kit/)** (GitHub) | Set project rules once with `constitution`, then per feature: `specify → plan → tasks → implement`. | Agent-agnostic (Copilot, Claude, Codex, Gemini and more), so specs survive switching tools or hitting usage limits. |
| **[OpenSpec](https://github.com/Fission-AI/OpenSpec)** | Lightweight `propose → apply → archive` flow, producing proposal, specs, design and tasks as Markdown. | Designed for existing codebases as well as new ones. |
| **ADRs + lean CLAUDE.md / AGENTS.md** | Keep the root file short; move detail into ADRs that load only when needed. | See section 5. |

### 3.5 Models and plans (community observations)
- **Stronger models (e.g. Claude Opus)** are often reported to need fewer turns on hard problems because they get stuck in fix loops less often.
- **Higher-tier plans (e.g. Claude Max)** are what the "hands-off" camp relies on, since self-refactoring loops consume a lot of usage.
- **Using a second tool** (Claude Code plus Codex or Gemini) is common, either to keep working when one hits a limit or to have a second model review plans and diffs with fresh eyes.

## 4. Two philosophies: Tech Lead vs Product Manager
Both camps try to prevent drift and spaghetti code. They differ on **who owns the code** and **where the guardrails live**.

### 4.1 Human as Tech Lead (strict architecture)
The AI is a fast junior developer. You own the architecture and quality.
- **Principle:** never merge code you don't understand.
- **Workflow:** write specs, ADRs and interface boundaries first. Give the AI small, isolated tasks inside them. Build in thin slices.
- **Verification:** read diffs, write tests before implementation, enforce linters and CI.
- **Best for:** production systems, complex backends, regulated domains, long-lived codebases, anything with database migrations.
- **Trade-off:** slower at the start; needs strong engineering judgement to spot drift early.

### 4.2 Human as Product Manager ("ignore the code")
Code is disposable. You care about behaviour and automated proof.
- **Principle:** don't read the code. Define outcomes and make the AI audit and refactor itself.
- **Workflow:** high-level goals with strict acceptance criteria, plus regular whole-repo refactor passes (find duplicates, merge abstractions, delete dead code).
- **Verification:** end-to-end browser/API tests, runtime checks, and fresh-context reviewer agents testing the live app against user stories. One commenter grew roughly 7,000 automated QA tests over 10 months this way.
- **Best for:** prototypes, internal tools, solo MVPs, greenfield apps where speed matters most.
- **Trade-off:** much higher usage cost, and a risk of dead ends when a deep design flaw needs a rewrite that the agent keeps patching around.

### 4.3 Side by side
| Dimension | Tech Lead | Product Manager |
|---|---|---|
| **Human role** | Architect / tech lead | Product manager / QA lead |
| **Code ownership** | Human understands every module and boundary | Code is a disposable black box |
| **Main guardrail** | ADRs, linters, static analysis, diff review | E2E tests, fresh-context reviewer agents |
| **Tech debt** | Prevented up front | Cleaned up periodically |
| **Usage cost** | Lower (small, focused context) | High (many self-correction loops) |

### 4.4 Where both camps agree
- **Never trust an agent's own "done"** without an independent check.
- **Never let one session run forever** once quality starts slipping.
- **Choose per project, not per person.** Write the chosen mode at the top of the root instruction file so every session knows the rules of the game.

## 5. A lean CLAUDE.md with progressive disclosure and ADRs
**Progressive disclosure** means the root file stays short and only *points* to detailed docs, which the agent reads when it touches the relevant area.

**Rule of thumb:** aim for a root file under ~100 lines; if it grows past ~150, move detail into ADRs. (There's no official hard limit, but shorter, more specific instructions are followed more reliably.)

> **CLAUDE.md vs AGENTS.md:** `CLAUDE.md` is Claude Code's file; Codex and many other agents read `AGENTS.md`. Claude Code reads `AGENTS.md` only when no `CLAUDE.md` exists. For a repo used with several agents, keep the real content in `AGENTS.md` and make `CLAUDE.md` a single line: `@AGENTS.md`.

### 5.1 How scoped files load
- The root `CLAUDE.md` loads at the start of every session.
- In Claude Code, a `CLAUDE.md` inside a subfolder loads **on demand**, the first time Claude reads a file in that folder.
- Claude Code also supports path-specific rules in `.claude/rules/`, useful when rules follow file types (e.g. `*.sql`) rather than folders.
- Use `/context` and `/memory` to check what actually loaded.

### 5.2 Keeping it from drifting
- **Treat every repeat mistake as a missing guardrail.** Don't just correct it in chat. Add one line to the relevant ADR or scoped file; if it must *never* happen, add a lint rule, CI check, or hook instead.
- **Prune regularly.** When the root file passes ~150 lines, move detail into an ADR and leave a one-line pointer.

## 6. Worked example: a PostgreSQL + .NET sample project
A fictional order-tracking API ("**ShopLite**") using .NET, PostgreSQL, Dapper for reads, and Testcontainers for integration tests. Everything here is illustrative.

### 6.1 Repository layout

```text
shoplite/
├── AGENTS.md                         # Lean root index (aim < 100 lines)
├── CLAUDE.md                         # One line: @AGENTS.md
├── docker-compose.yml                # Local PostgreSQL
├── docs/
│   └── adr/
│       ├── 0000-template.md
│       ├── 0001-layer-boundaries.md
│       ├── 0002-postgres-data-access.md
│       └── 0003-test-strategy.md
├── db/
│   └── migrations/
│       ├── V001__create_orders.sql
│       └── V002__add_orders_customer_index.sql
├── src/
│   ├── ShopLite.Domain/
│   ├── ShopLite.Application/
│   ├── ShopLite.Infrastructure/
│   │   └── AGENTS.md                 # Data-access rules, loaded when relevant
│   └── ShopLite.Api/
└── tests/
    ├── AGENTS.md                     # Test conventions
    ├── ShopLite.UnitTests/
    └── ShopLite.IntegrationTests/
```

> If you use Claude Code only, name the scoped files `CLAUDE.md` instead so they load on demand.

### 6.2 Root `AGENTS.md`

````markdown
# ShopLite: Agent Guide
**Mode:** Tech Lead. Small slices; humans review every diff.
## Build & Verify
- Start DB:          `docker compose up -d postgres`
- Restore:           `dotnet restore`
- Build:             `dotnet build --no-restore`
- Unit tests:        `dotnet test tests/ShopLite.UnitTests`
- Integration tests: `dotnet test tests/ShopLite.IntegrationTests` (needs Docker)
- Format check:      `dotnet format --verify-no-changes`
## Non-Negotiable Rules
1. Dependencies point inward: Api -> Application -> Domain. Domain has no data-access packages.
2. Never edit an applied migration in `db/migrations/`. Add a new, higher-numbered file.
3. No unbounded queries. Every list endpoint is paginated with a capped page size.
4. Done = build passes + relevant tests run. Never change an existing assertion to make a test pass; stop and explain.
## Read Before Acting
- SQL, repositories, migrations -> `docs/adr/0002-postgres-data-access.md`
- New tests or fixtures         -> `docs/adr/0003-test-strategy.md`
- Structural changes            -> draft a new ADR from `docs/adr/0000-template.md` first
````

### 6.3 AI-friendly ADR
A normal ADR explains *why*. An AI-friendly ADR adds an **Agent Rules** section saying exactly what to do and avoid.

````markdown
# ADR-0002: PostgreSQL Data Access
- **Status:** Accepted
- **Scope:** `src/ShopLite.Infrastructure/**`, `db/migrations/**`
## Context
Tables use `snake_case`; C# models use `PascalCase`. Early prototypes used per-query aliases,
`SELECT *`, and OFFSET paging on the `orders` table, which caused mapping bugs and slow pages
once the table grew.
## Decision
1. Set `Dapper.DefaultTypeMap.MatchNamesWithUnderscores = true` once at startup.
2. Use keyset (cursor) pagination for growing tables such as `orders`.
   OFFSET is allowed only for small, bounded lists. Max page size: 100.
3. Every keyset query has a matching composite index.
4. Schema changes ship as forward-only SQL migrations.
## Agent Rules
- DO parameterise every value (`@param`); never build SQL with string concatenation.
- DO list columns explicitly; no `SELECT *`.
- DO keep SQL in `*Queries.cs` classes, not inline in services.
- DO create indexes on large existing tables with `CREATE INDEX CONCURRENTLY`
  (in its own migration, outside a transaction).
- DO NOT edit or delete an applied migration.
- DO NOT add a column as `NOT NULL` without a default on a populated table in one step.
````

### 6.4 Migration and query that follow the ADR
`db/migrations/V001__create_orders.sql`

```sql
CREATE TABLE orders (
    id           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id  bigint        NOT NULL,
    status       text          NOT NULL CHECK (status IN ('pending', 'paid', 'shipped', 'cancelled')),
    total_amount numeric(12,2) NOT NULL CHECK (total_amount >= 0),
    created_at   timestamptz   NOT NULL DEFAULT now()
);
```
`db/migrations/V002__add_orders_customer_index.sql`

```sql
-- Supports keyset paging: newest orders first, per customer.
-- Run outside a transaction (required for CONCURRENTLY).
CREATE INDEX CONCURRENTLY IF NOT EXISTS ix_orders_customer_created_id
    ON orders (customer_id, created_at DESC, id DESC);
```
`src/ShopLite.Infrastructure/Orders/OrderQueries.cs`

```csharp
internal static class OrderQueries
{
    // Keyset pagination: pass the last row's (created_at, id) from the previous page.
    // The row comparison matches the index order, so PostgreSQL can seek instead of scan.
    public const string ListByCustomer = """
        SELECT id, customer_id, status, total_amount, created_at
        FROM orders
        WHERE customer_id = @CustomerId
          AND (@AfterCreatedAt IS NULL
               OR (created_at, id) < (@AfterCreatedAt, @AfterId))
        ORDER BY created_at DESC, id DESC
        LIMIT @PageSize;
        """;
}
```
`src/ShopLite.Infrastructure/Orders/OrderReadRepository.cs`

```csharp
public sealed class OrderReadRepository(NpgsqlDataSource dataSource) : IOrderReadRepository
{
    private const int MaxPageSize = 100;
    public async Task<IReadOnlyList<OrderDto>> ListByCustomerAsync(
        long customerId, OrderCursor? after, int pageSize, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        var rows = await conn.QueryAsync<OrderDto>(new CommandDefinition(
            OrderQueries.ListByCustomer,
            new
            {
                CustomerId     = customerId,
                AfterCreatedAt = after?.CreatedAt,
                AfterId        = after?.Id ?? 0L,
                PageSize       = Math.Clamp(pageSize, 1, MaxPageSize)
            },
            cancellationToken: ct));
        return rows.AsList();
    }
}
```

### 6.5 Scoped test rules: `tests/AGENTS.md`

````markdown
# Test Conventions
- Naming: `Method_State_ExpectedResult`.
- Unit tests may stub repository interfaces. Real SQL behaviour is tested only in
  `ShopLite.IntegrationTests` against PostgreSQL via Testcontainers (same major version as production).
- Every bug fix starts with a failing test that reproduces it.
- Paging tests must cover: first page, next page via cursor, last page, and ties on `created_at`.
````

### 6.6 Matching guardrails in CI
| Rule in the docs | Enforced by |
|---|---|
| Format and style | `dotnet format --verify-no-changes` |
| Risky migrations (missing `CONCURRENTLY`, unsafe `NOT NULL`) | Migration linter, e.g. `squawk db/migrations/*.sql` |
| Applied migrations never edited | CI step failing if any existing file in `db/migrations/` is modified |
| Real SQL works | Integration tests against a PostgreSQL container |

## 7. References
- Reddit thread: [Half a year of AI coding got me nowhere](https://www.reddit.com/r/ClaudeAI/comments/1y1my5l/half_a_year_of_ai_coding_got_me_nowhere/)
- Claude Code: [Commands](https://code.claude.com/docs/en/commands) · [Memory, CLAUDE.md and AGENTS.md](https://code.claude.com/docs/en/memory) · [Debug your configuration](https://code.claude.com/docs/en/debug-your-config)
- [pstack-claude](https://github.com/michael-denyer/pstack-claude) · [bb](https://getbb.app/) ([GitHub](https://github.com/get-bb/bb)) · [qt-commander](https://github.com/tiefeiyu/qt-commander)
- [Spec Kit](https://github.github.com/spec-kit/) · [OpenSpec](https://github.com/Fission-AI/OpenSpec)
- [Dapper `DefaultTypeMap`](https://github.com/DapperLib/Dapper/blob/main/Dapper/DefaultTypeMap.cs)

---

## 8. Related reading
- [DeepSeek V4.1 Flash vs Claude Opus 5.5](deepseek-v4.1-flash-vs-opus-5.5.md) — which model to pair with this workflow, and when to escalate.
- [Verification-First AI Coding](verification-first-ai-coding.md) — how to make "done" checkable instead of trusted.
- [Context Engineering for Coding Agents](context-engineering-for-coding-agents.md) — the budget behind section 5's lean root file.
- [Agent Guardrails and Security](agent-guardrails-and-security.md) — what to enforce when a linter is not enough.
- [Asking an Agent for a Code Review](review-requests-for-coding-agents.md) — how to get a second opinion that is worth reading.
- [Multi-Agent Orchestration](multi-agent-orchestration.md) — the same thin slices, run in parallel, and what breaks at merge.
- [All notes](README.md)
