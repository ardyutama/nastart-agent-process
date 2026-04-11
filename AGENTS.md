# Project: Priced Right — Recipe Costing for Solopreneurs

> Agent rules file. Load this at the start of every coding session.
> For full product context, read `CHECKPOINT.md`.
> For v1 build constraints (what NOT to build), read `.github/context/v1-constraints.md`.

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
│   ├── RecipeCost.API/           ← Program.cs, endpoints, middleware
│   ├── RecipeCost.Application/   ← MediatR handlers, commands, queries, DTOs
│   ├── RecipeCost.Domain/        ← Entities, interfaces, enums
│   └── RecipeCost.Infrastructure/← EF Core, repositories, external services
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
dotnet run --project src/RecipeCost.API

# EF Core migrations
dotnet ef migrations add <Name> --project src/RecipeCost.Infrastructure --startup-project src/RecipeCost.API
dotnet ef database update --project src/RecipeCost.Infrastructure --startup-project src/RecipeCost.API

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
- Use `ICostCascadeService.RecalculateForIngredient(ingredientId)` — never pass price as a parameter to cascade
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
| C-1 | `ICostCascadeService.RecalculateForIngredient(ingredientId)` — no price param |
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
| Phase 1 schema work | + `lessons/L2-ef-core-postgresql-schema.md` + `.github/context/phase-1-session.md` |
| Ingredient feature | + `lessons/L6-ingredient-management-and-price-history.md` |
| Recipe / cascade | + `lessons/L7-cost-cascade-service-and-price-spike-alerts.md` + `lessons/L8-recipe-builder-and-costing-engine.md` |
| OCR / scanning | + `docs/plans/2026-04-09-ocr-packaging-design.md` |
| Architecture decisions | + `business-flows/00-canonical-decisions.md` |
| Project overview | + `CHECKPOINT.md` |
