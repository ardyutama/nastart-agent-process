# Project: Priced Right — Recipe Costing for Solopreneurs

> Agent rules file. Load this at the start of every coding session.
> For full product context, read `CHECKPOINT.md` if it exists; otherwise use `.github/context/project-map.md` and `business-flows/00-index.md`.
> For v1 build constraints (what NOT to build), read `.github/context/v1-constraints.md`.

---

## Agent Operating Rules

- Always load this file and `.github/context/v1-constraints.md` before coding.
- Load only the task-relevant lesson, context, or architecture files from the table below; avoid flooding the agent with unrelated docs.
- Before editing source code, read the file being changed, its related tests, one similar implementation pattern, and the involved contracts/interfaces.
- If loaded docs conflict, the v1 constraints and locked architecture decisions in this file win. Surface the conflict instead of guessing.
- For flow-level product behavior, load `business-flows/v1/` by default. Use `business-flows/v2-reference/` only as historical enterprise reference.
- If requirements are incomplete and no existing code establishes the behavior, stop and ask before inventing product rules.
- For any new or changed behavior, use TDD: write the failing test first, then implement the smallest change that makes it pass.
- Keep changes small and verifiable: implement, test, verify, then summarize exactly what changed.
- Do not add dependencies, schema changes, auth claims, roles, multi-tenant concepts, or cascade interface changes without explicit approval.

---

## What This Project Is

A **single-user recipe costing app** for home bakers and solo food entrepreneurs.
Core loop: add ingredients + prices → build recipes → set target margin → get recommended sell price.
When a receipt is scanned, ingredient costs cascade automatically through all affected recipes.

**Not** a multi-tenant SaaS. **Not** multi-user in v1. One user, one kitchen, many recipes.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend API | .NET 10 Web API (C#) — Clean Architecture (Domain / Application / Infrastructure / API) |
| AI / OCR / Bot | Python FastAPI — OCR, LLM name-matching, Telegram alerts |
| Frontend | Vue.js 3 + Vite + TypeScript |
| Database | PostgreSQL 18 |
| ORM | EF Core (code-first, Fluent API, migrations) |
| Auth | JWT (userId + email claims only — no role, no outletId) |
| File Storage | MiniStack (local dev) → Cloudflare R2 (VPS) → AWS S3 (future) — all S3-compatible, swap via `AWS_ENDPOINT_URL` env var |
| Deployment | Docker Compose → VPS |

---

## Repo Structure (target)

```
nastart/
├── backend/
│   ├── Nastart.Api/           ← Program.cs, endpoints, middleware
│   ├── Nastart.Application/   ← MediatR handlers, commands, queries, DTOs
│   ├── Nastart.Domain/        ← Entities, interfaces, enums
│   └── Nastart.Infrastructure/← EF Core, repositories, external services
├── ai-service/                   ← Python FastAPI: OCR, LLM, Telegram
├── frontend/                     ← Vue.js 3 + Vite + TypeScript
└── docker-compose.yml
```

---

## Commands

```bash
# .NET backend
dotnet build
dotnet test
dotnet run --project src/Nastart.Api

# EF Core migrations
dotnet ef migrations add <Name> --project src/Nastart.Infrastructure --startup-project src/Nastart.Api
dotnet ef database update --project src/Nastart.Infrastructure --startup-project src/Nastart.Api

# Python AI service
pip install -r requirements.txt
uvicorn main:app --reload

# Frontend
npm install
npm run dev
npm run build
npm test
```

---

## Code Conventions

### .NET (C#)
- **Vertical Slice Architecture** — one folder per feature under `Application/Features/`
- **MediatR CQRS** — every operation is a Command or Query with a Handler
- **ErrorOr<T>** — never throw exceptions for business logic; return `Error.NotFound`, `Error.Conflict`, etc.
- **FluentValidation** — every Command and Query gets a `Validator` class; wired via MediatR pipeline behavior
- **Minimal API** — no controllers; endpoints defined in `*Endpoints.cs` files, mapped in `Program.cs`
- **EF Core Fluent API** — no data annotations; all config in `*Configuration.cs` files
- **No `[Authorize(Roles = ...)]`** — v1 uses `RequireAuthorization()` only (single authenticated user)
- **API project name** — use `Nastart.Api` for the project folder, project file, and namespaces; do not use the all-caps `API` variant

### .NET 10 / .NET Skills Rules
- Before any .NET backend task, load the relevant local .NET skill docs under `.github/skills/dotnet-skills/`.
- Always include `.github/skills/dotnet-skills/dotnet-best-practices/SKILL.md` for .NET implementation or review work.
- For ASP.NET Core Minimal API work, load the relevant `dotnet-aspnet` skill before implementing endpoint-specific behavior.
- For EF Core query or persistence work, load `.github/skills/dotnet-skills/dotnet-data/skills/optimizing-ef-core-queries/SKILL.md` and keep PostgreSQL/Npgsql behavior in mind.
- For test execution, load `.github/skills/dotnet-skills/dotnet-test/skills/run-tests/SKILL.md`; detect VSTest vs Microsoft.Testing.Platform before choosing `dotnet test` syntax.
- For new MSTest-based .NET 10 test projects, prefer `MSTest.Sdk` v4+ with Microsoft.Testing.Platform instead of the older `Microsoft.NET.Sdk` + `MSTest.*` + `Microsoft.NET.Test.Sdk` stack unless a tool explicitly requires VSTest compatibility.
- If the repo opts into Microsoft.Testing.Platform in `global.json`, use native .NET 10 syntax (`dotnet test --project ...` / `--solution ...`) and do not mix VSTest and MTP test projects in the same solution.
- For SDK/runtime migrations, load the applicable `dotnet-upgrade` skill and follow .NET 10 breaking-change guidance.
- Target .NET 10 (`net10.0`) and C# 14-compatible code. Prefer current .NET 10 approaches when they do not conflict with the project architecture.
- Use `WebApplication.CreateBuilder`, ASP.NET Core Minimal APIs, endpoint groups, typed contracts, dependency injection, async I/O, and cancellation tokens for request/database work.
- Use EF Core 10 code-first Fluent API configuration, async LINQ operators, `AsNoTracking()` for read-only queries, parameterized SQL APIs, and projections that avoid N+1 queries.
- Do not invent framework APIs or package references. Verify project files and existing imports first, and ask before adding a new NuGet package.

### Business Logic TDD Rules
- Load `.github/skills/test-driven-development/SKILL.md` before implementing, fixing, or changing business logic.
- Business logic belongs in `Nastart.Domain` and `Nastart.Application` first; keep `Nastart.Api` endpoints thin and focused on HTTP mapping, auth context, request validation, and result translation.
- Before adding or changing business logic, write or update the lowest-level test that captures the expected behavior. Run that targeted test and confirm it fails for the right reason before implementing.
- Implement the smallest domain/application change needed to pass the failing test. Refactor only after the test is green.
- Prefer unit tests for pure domain rules, value calculations, validators, handlers, and services. Use integration tests when behavior depends on EF Core queries, database constraints, transactions, or API pipeline behavior.
- For any test that touches `DbContext`, EF Core query translation, migrations, relational constraints, or Npgsql/PostgreSQL behavior, prefer PostgreSQL Testcontainers over EF Core InMemory or SQLite so the test exercises the real provider used in production.
- Reserve no-database tests for pure calculations, validators, mapping, and other logic that does not depend on relational behavior. If a test needs SQL semantics, treat it as an integration test and run it against the containerized PostgreSQL instance.
- Containerized test runs require a local container runtime (Docker Desktop, Rancher Desktop, or Podman with a Docker-compatible socket). If it is unavailable, report that environment blocker explicitly instead of substituting EF Core InMemory.
- For bugs, use the Prove-It pattern: reproduce the bug with a failing test, then fix the implementation, then prove the test passes.
- Do not remove, weaken, skip, or rewrite failing tests just to make the suite pass. If a test is wrong, explain why and update it to assert the correct business rule.
- If no suitable test project or test framework exists, stop before adding dependencies and ask whether to create the missing test infrastructure.
- A business logic task is not complete until the targeted test passes and the relevant full test command has been run or a blocker is clearly reported.

### Python (FastAPI)
- Pydantic v2 for all request/response models
- `async def` route handlers throughout
- Services are plain classes injected via FastAPI `Depends()`

### Vue.js
- Composition API with `<script setup>` — no Options API
- TypeScript strict mode
- Pinia for state management
- Axios for API calls; typed response models in `src/api/`

---

## Critical Boundaries

### Always
- Run tests before committing
- Validate all user input at API boundary
- Use `ICostCascadeService.RecalculateForIngredientAsync(ingredientId, cancellationToken)` — never pass price as a parameter to cascade
- Derive current price from `IngredientPriceHistory ORDER BY committed_at DESC LIMIT 1` — never store `current_price` on `Ingredient`
- Sell price = `(CostPerPortion + PackagingCost) / (1 - TargetMargin)` — derived at read time, never stored

### Ask First
- Database schema changes beyond what's in the lesson plan
- New NuGet / npm / pip dependencies
- Any change to the cascade service interface (C-1)

### Never
- Commit secrets, connection strings, or JWT signing keys
- Remove or skip failing tests
- Accept `CostPerPortion` from the client — it is server-authoritative only
- Store sell price in the database
- Add `outletId`, `role`, or `companyId` to JWT claims in v1
- Build Role enum, Company, Outlet, OutletUser, or Invitation entities in v1 code

---

## Architecture Decisions (locked — do not deviate)

| Decision | Rule |
|---|---|
| C-1 | `ICostCascadeService.RecalculateForIngredientAsync(ingredientId, cancellationToken)` — no price param |
| C-2 | Cost formula: `SUM((price/unitSize) * quantity * (1/yield)) / portionCount` |
| C-3 | Current price: `ORDER BY committed_at DESC LIMIT 1` on `IngredientPriceHistory` |
| C-4 | Two timestamps: `committed_at` (system, ordering) + `effective_date` (user, business) |
| C-5 | Cascade failures → `CascadeErrorLog`, never roll back `IngredientPriceHistory` |
| C-6 | `TelegramLink.codeHash` = SHA-256 hash — never store plaintext link code |
| C-13 | `IngredientPriceHistory.source` is exactly `'Manual'` or `'InvoiceScan'` (case-sensitive) |

---

## What to Load Per Task

| Task | Load these files |
|---|---|
| Any coding session | This file + `.github/context/v1-constraints.md` |
| Flow routing / overview | + `business-flows/00-index.md` + `business-flows/v1/README.md` |
| Auth / Telegram linking | + `business-flows/v1/01-auth-telegram-linking.md` + `lessons/L5-jwt-auth-and-role-authorization.md` |
| .NET backend work | + relevant `.github/skills/dotnet-skills/` docs before implementing |
| Business logic changes | + `.github/skills/test-driven-development/SKILL.md` before implementation |
| Phase 1 schema work | + `lessons/L2-ef-core-postgresql-schema.md` + `.github/context/phase-1-session.md` |
| Ingredient feature | + `business-flows/v1/02-ingredient-price-management.md` + `lessons/L6-ingredient-management-and-price-history.md` |
| Recipe / cascade | + `business-flows/v1/03-recipe-builder-costing-engine.md` + `lessons/L7-cost-cascade-service-and-price-spike-alerts.md` + `lessons/L8-recipe-builder-and-costing-engine.md` |
| OCR / scanning | + `business-flows/v1/04-invoice-scanning-review-commit.md` (the OCR packaging design doc referenced in older notes is currently missing from the repo) |
| Telegram bot | + `business-flows/v1/05-telegram-bot-flows.md` |
| Architecture decisions | + `business-flows/00-canonical-decisions.md` |
| Project overview | + `CHECKPOINT.md` if present, otherwise `.github/context/project-map.md` |
