# Project Map

> Hierarchical summary of the current workspace context plus the target application structure from `AGENTS.md`.
> Load only the section relevant to your current task — not the whole file.
> Updated: May 14, 2026
> If this file conflicts with `AGENTS.md` or `.github/context/v1-constraints.md`, those files win.

---

## Top-Level Structure

```
nastart-agent-process/              ← current workspace: planning, context, lessons, and agent rules
├── AGENTS.md                       ← Agent rules file: load every session
├── .github/
│   ├── copilot-instructions.md     ← Coding standards (TDD, quality axes)
│   ├── context/
│   │   ├── v1-constraints.md       ← What v1 IS and IS NOT — load every coding session
│   │   ├── project-map.md          ← This file
│   │   └── phase-1-session.md      ← Context pack for Phase 1 build, aligned to AGENTS.md
│   ├── skills/                     ← Agent skill files
│   │   ├── test-driven-development/← Load for business logic changes
│   │   └── dotnet-skills/          ← Load relevant .NET skills before backend work
│   └── agents/                     ← Agent definitions
├── docs/
│   └── plans/
│       ├── 2026-05-14-versioned-business-flows-design.md
│       └── 2026-05-14-versioned-business-flows-implementation-plan.md
├── archive/
│   └── legacy-flows/              ← Retired draft flow files kept only for historical reference
├── lessons/                        ← Step-by-step build lessons L1–L9
├── business-flows/                 ← Versioned flow root: v1 active, v2-reference historical
├── personas/                       ← User persona research
```

**Missing expected files:**
- `AGENTS.md` references `CHECKPOINT.md` for full project state, but that file is not currently present in this workspace.
- Older OCR/scanning guidance references `docs/plans/2026-04-09-ocr-packaging-design.md`, but that file is also not currently present.

Until those files exist, use this project map, `business-flows/00-index.md`, the relevant v1 flow file under `business-flows/v1/`, and the lesson files.

---

## Target Application Structure (from `AGENTS.md`)

```
nastart/
├── backend/
│   ├── Nastart.Api/
│   ├── Nastart.Application/
│   ├── Nastart.Domain/
│   └── Nastart.Infrastructure/
├── ai-service/
├── frontend/
└── docker-compose.yml
```

---

## Context Loading Rules

- Always load `AGENTS.md` and `.github/context/v1-constraints.md` before coding.
- For flow-level behavior, load `business-flows/00-index.md` and then the relevant file under `business-flows/v1/`.
- Use `business-flows/v2-reference/` only as historical enterprise reference.
- Do not use `archive/legacy-flows/` for current implementation guidance.
- For .NET backend work, also load the relevant `.github/skills/dotnet-skills/` docs.
- For business logic changes, also load `.github/skills/test-driven-development/SKILL.md` and follow TDD.
- If older docs conflict with `AGENTS.md`, prefer `AGENTS.md` and `v1-constraints.md`.

---

## Lessons (`lessons/`)

Sequential build lessons. v1 solopreneur amendments have been applied to all.

| File | What it covers | v1 Amendment Status |
|---|---|---|
| `L1-clean-architecture-and-solution-scaffold.md` | Project scaffold, DI, 4-layer architecture | ✅ Clean — no changes needed |
| `L2-ef-core-postgresql-schema.md` | EF Core code-first, 9 v1 entities, migrations | ✅ Amended — v2 entities marked |
| `L3-vertical-slice-architecture-mediatr.md` | CQRS, MediatR, vertical slices | ✅ Amended — outlet slices marked v2-only |
| `L4-validation-and-error-handling.md` | ErrorOr<T>, FluentValidation, MediatR pipeline | ✅ Amended — UserId replaces OutletId |
| `L5-jwt-auth-and-role-authorization.md` | JWT, register/login, single-user auth | ✅ Amended — role-based policies removed |
| `L6-ingredient-management-and-price-history.md` | Ingredient CRUD, price history (7 slices) | ✅ Amended — fully user-scoped |
| `L7-cost-cascade-service-and-price-spike-alerts.md` | CostCascadeService, cascade formula, alerts | ✅ Amended — UserId, PackagingCost, TargetMargin |
| `L8-recipe-builder-and-costing-engine.md` | Recipe slices, sell price, unified DTO | ✅ Amended — no role-split DTOs |
| `L9-backend-observability-with-victoriametrics.md` | OpenTelemetry logs, metrics, traces, VictoriaMetrics products, alerts | ✅ v1-ready — backend-first, no product/schema scope changes |

**How to use lessons:** Follow lessons sequentially. Every `⚠️ v2-only` callout in a lesson body = skip that block. The amendment block at the top of each lesson summarizes all changes. If an older lesson snippet still shows `RecipeCost.*` or `Nastart.API`, translate it to `Nastart.*` and `Nastart.Api`.

---

## Business Flows (`business-flows/`)

The business-flow docs now use a versioned structure.

| Path | Contents | Status |
|---|---|---|
| `00-index.md` | Router page for flow discovery and authority order | Active |
| `00-canonical-decisions.md` | Shared canonical decisions with v1 applicability notes | Active |
| `v1/README.md` | Entry point for the active v1 flow set | Active |
| `v1/01-auth-telegram-linking.md` | Single-user auth, verification, login, Telegram linking | Active v1 guidance |
| `v1/02-ingredient-price-management.md` | User-scoped ingredient CRUD, append-only price history, cascade trigger | Active v1 guidance |
| `v1/03-recipe-builder-costing-engine.md` | User-scoped recipes, derived sell price, unified response DTOs | Active v1 guidance |
| `v1/04-invoice-scanning-review-commit.md` | Single-user invoice review and `InvoiceScan` price commits; guarded where missing OCR design details remain | Active v1 guidance |
| `v1/05-telegram-bot-flows.md` | Single-user Telegram commands and alerts | Active v1 guidance |
| `v2-reference/` | Historical enterprise flow set preserved for reference | Historical only |

**Discovery rule:** Use the `v1/` files for current implementation work. Do not implement from `v2-reference/` unless you are explicitly working on future enterprise scope.

---

## Docs (`docs/`)

| File | Contents |
|---|---|
| `docs/plans/2026-05-14-versioned-business-flows-design.md` | Approved design for versioning the business-flow docs into active v1 and historical v2-reference tracks |
| `docs/plans/2026-05-14-versioned-business-flows-implementation-plan.md` | Ordered migration plan for the versioned business-flow rewrite |

**Missing OCR doc:** `AGENTS.md` still references `docs/plans/2026-04-09-ocr-packaging-design.md`, but that file is not present in this workspace.

---

## Personas (`personas/`)

5 persona files with user stories, use cases, pain points. Primary v1 persona is `05-home-baker.md`. Others are v2 reference.

| File | Persona | v1 relevance |
|---|---|---|
| `01-fnb-owner.md` | F&B Owner / Restaurateur | v2 |
| `02-head-chef.md` | Head Chef / Kitchen Manager | v2 |
| `03-procurement.md` | Procurement / Purchasing Staff | v2 |
| `04-cost-controller.md` | Accountant / F&B Cost Controller | v2 |
| `05-home-baker.md` | Casual Home Baker / Solo Entrepreneur | **v1 primary persona** |

---

## Key Relationships Between Files

```
AGENTS.md
  └─ points to → .github/context/v1-constraints.md  (load every session)
  └─ points to → CHECKPOINT.md                       (expected full project state file)
  └─ points to → business-flows/00-index.md         (flow routing)
  └─ points to → business-flows/v1/                 (active flow specifications by task)
  └─ requires → .github/skills/dotnet-skills/       (.NET backend work)
  └─ requires → .github/skills/test-driven-development/SKILL.md (business logic work)

CHECKPOINT.md
  └─ if present, references → business-flows/        (validated flows)
  └─ if present, references → docs/plans/            (approved designs)

business-flows/00-index.md
  └─ routes to → business-flows/v1/README.md
  └─ routes to → business-flows/v1/01-auth-telegram-linking.md
  └─ routes to → business-flows/v1/02-ingredient-price-management.md
  └─ routes to → business-flows/v1/03-recipe-builder-costing-engine.md
  └─ routes to → business-flows/v1/04-invoice-scanning-review-commit.md
  └─ routes to → business-flows/v1/05-telegram-bot-flows.md
  └─ preserves → business-flows/v2-reference/        (historical enterprise reference)

Lessons
  L1 → L2 → L3 → L4 → L5 → L6 → L7 → L8 → L9
  Each lesson builds on the previous.
  L2 establishes entities used in L3–L8.
  L5 establishes auth used in L6–L8.
  L7 establishes cascade service used in L8.
  L9 makes the backend operationally observable without changing v1 business rules.
```

---

## Current Build Status (May 14, 2026)

| Phase | Status |
|---|---|
| Phase 1 — Foundation | ❌ Not started. Context pack: `.github/context/phase-1-session.md` |
| Phase 2 — Recipe Costing Engine | ❌ Not started |
| Phase 3 — AI Invoice Scanning | ❌ Not started |
| Phase 4 — Telegram Bot | ❌ Not started |
| Phase 5 — Portfolio Polish & Deploy | ❌ Not started |
