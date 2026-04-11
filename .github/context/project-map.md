# Project Map

> Hierarchical summary of all project files.
> Load only the section relevant to your current task — not the whole file.
> Updated: April 10, 2026

---

## Top-Level Structure

```
nastart/                            ← product code repo (source code lives here)

nastart-evaluation/                 ← THIS repo — planning, lessons, docs
├── AGENTS.md                       ← Agent rules file: load every session
├── CHECKPOINT.md                   ← Full project state: resume from here
├── .github/
│   ├── copilot-instructions.md     ← Coding standards (TDD, quality axes)
│   ├── context/
│   │   ├── v1-constraints.md       ← What v1 IS and IS NOT — load every coding session
│   │   ├── project-map.md          ← This file
│   │   └── phase-1-session.md      ← Context pack for Phase 1 build
│   ├── skills/                     ← Agent skill files (idea-refine, brainstorming, etc.)
│   └── agents/                     ← Agent definitions
├── docs/
│   ├── ideas/
│   │   └── recipe-cost-solopreneur.md  ← Ideation one-pager (April 9)
│   └── plans/
│       └── 2026-04-09-ocr-packaging-design.md  ← OCR + packaging_cost design
├── lessons/                        ← Step-by-step build lessons L1–L8
├── business-flows/                 ← Validated business flows and canonical decisions
├── personas/                       ← User persona research
└── flows/                          ← Earlier draft flow files (superseded by business-flows/)
```

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

**How to use lessons:** Follow lessons sequentially. Every `⚠️ v2-only` callout in a lesson body = skip that block. The amendment block at the top of each lesson summarizes all changes.

---

## Business Flows (`business-flows/`)

Validated by Lead Architect Agent — 13 defects found and resolved.
**Note:** These flows were designed for the enterprise version. v1-incompatible sections (role-based routing, outlet scoping, invitation flows) are preserved for v2 reference but should not be coded in Phase 1–3.

| File | Contents | v1 applicability |
|---|---|---|
| `00-index.md` | Flow index + defect resolution log | Reference only |
| `00-canonical-decisions.md` | All 14 canonical decisions with full detail | See v1-constraints.md for which apply |
| `01-auth-company-telegram-linking.md` | Company registration, invitations, `/link XXXX` | **Strip**: company/invitation flows; **Keep**: TelegramLink `/link` flow |
| `02-ingredient-price-management.md` | Ingredient CRUD, price update, cascade algorithm | ✅ Mostly applicable — ignore outlet-scoping |
| `03-recipe-builder-costing-engine.md` | Recipe creation, cascade, threshold alerts | ✅ Applicable — ignore outlet scope, role visibility, CostThresholdPct |
| `04-invoice-scanning-review-commit.md` | OCR upload, review queue, price commit | ✅ Applicable — updated by `docs/plans/2026-04-09-ocr-packaging-design.md` |
| `05-telegram-bot-flows.md` | Account linking, alerts, `/cost` command | ✅ Applicable — reframed as personal alerts (no role-based routing) |

---

## Docs (`docs/`)

| File | Contents |
|---|---|
| `docs/ideas/recipe-cost-solopreneur.md` | Ideation one-pager: problem statement, MVP scope, Not Doing list, resolved open questions |
| `docs/plans/2026-04-09-ocr-packaging-design.md` | Approved design: confidence-scored OCR review queue + `packaging_cost` flat field |

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
  └─ points to → CHECKPOINT.md                       (full project state)

CHECKPOINT.md
  └─ references → business-flows/                    (validated flows)
  └─ references → docs/ideas/recipe-cost-solopreneur.md (refined direction)
  └─ references → docs/plans/                        (approved designs)

Lessons
  L1 → L2 → L3 → L4 → L5 → L6 → L7 → L8
  Each lesson builds on the previous.
  L2 establishes entities used in L3–L8.
  L5 establishes auth used in L6–L8.
  L7 establishes cascade service used in L8.
```

---

## Current Build Status (April 10, 2026)

| Phase | Status |
|---|---|
| Phase 1 — Foundation | ❌ Not started. Context pack: `.github/context/phase-1-session.md` |
| Phase 2 — Recipe Costing Engine | ❌ Not started |
| Phase 3 — AI Invoice Scanning | ❌ Not started |
| Phase 4 — Telegram Bot | ❌ Not started |
| Phase 5 — Portfolio Polish & Deploy | ❌ Not started |
