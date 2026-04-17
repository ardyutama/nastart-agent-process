# Business Flows — Index

> Recipe Cost Management App  
> All flows validated by Lead Architect Agent — final round: April 2, 2026
>
> **Update (April 17, 2026):** These files still capture important architecture history, but the current v1 implementation authority is `AGENTS.md`, `.github/context/v1-constraints.md`, `docs/decisions/ADR-001-single-user-v1-scope.md`, and `docs/progress/00-current-status.md`. Flow 01 is historical enterprise reference, and Flow 05 is historical except for the `/link` mechanics in Flow 1.

---

## v1 Applicability Snapshot

| Flow | Current v1 use |
|---|---|
| 00 — Canonical Decisions | Use with the v1 applicability note in `00-canonical-decisions.md` |
| 01 — Auth, Company Setup, Telegram Linking | Historical enterprise reference only |
| 02 — Ingredient Management & Price History | Current with user-scoped ownership from the v1 constraints |
| 03 — Recipe Builder & Costing Engine | Current with user-scoped recipes and derived sell price rules |
| 04 — Invoice Scanning, Review Queue & Price Commit | Current with the single-user review and commit flow |
| 05 — Telegram Bot | Historical reference; only Flow 1 (`/link`) remains actionable for v1 |

---

## Validation Summary

| Flow | File | Lead Architect Verdict | Rounds |
|---|---|---|---|
| 00 — Canonical Decisions | [00-canonical-decisions.md](00-canonical-decisions.md) | ✅ Authority document | — |
| 01 — Auth, Company Setup, Telegram Linking | [01-auth-company-telegram-linking.md](01-auth-company-telegram-linking.md) | Historical reference | Round 1 |
| 02 — Ingredient Management & Price History | [02-ingredient-price-management.md](02-ingredient-price-management.md) | ✅ ACCEPTED | Round 2 (3 defects + 1 late defect fixed) |
| 03 — Recipe Builder & Costing Engine | [03-recipe-builder-costing-engine.md](03-recipe-builder-costing-engine.md) | ✅ ACCEPTED (canonical reference) | Round 1 |
| 04 — Invoice Scanning, Review Queue & Price Commit | [04-invoice-scanning-review-commit.md](04-invoice-scanning-review-commit.md) | ✅ ACCEPTED | Round 2 (4 defects fixed) |
| 05 — Telegram Bot | [05-telegram-bot-flows.md](05-telegram-bot-flows.md) | Historical reference; Flow 1 still relevant to v1 | Round 2 (6 defects fixed) |

---

## How to Read These Flows

Each flow file contains step-by-step tables with:
- **Actor** — who or what performs the step
- **Component** — `.NET 10 API` / `Python FastAPI` / `Vue.js` / `PostgreSQL` / `AWS S3` / `Telegram Bot`
- **Data IN / Data OUT** — inputs and outputs at each step
- **Decision point** — branch conditions
- **Error case** — what happens when the step fails

---

## Cross-Flow Data Connections

> **Historical map note:** The diagram below still reflects the enterprise validation pass from April 2, 2026. For the current v1 build, replace company or outlet ownership with direct `UserId` ownership and replace team alert routing with direct single-user Telegram delivery.

```
[01 Auth]
  └─ Creates: User, TelegramLink (pending)
       └─ TelegramLink.status='confirmed' unlocks → [05 Telegram Bot] flows

[02 Ingredients]
  └─ Creates: Ingredient, IngredientPriceHistory (source='Manual')
       └─ ICostCascadeService triggered → [03 Recipe Costing] auto-cascade

[04 Invoice Scanning]
  └─ Creates: Invoice, InvoiceLineItem, ReviewQueueItem
       └─ After commit: Creates IngredientPriceHistory (source='InvoiceScan')
            └─ ICostCascadeService triggered → [03 Recipe Costing] auto-cascade
                 └─ Threshold breach → alert dispatched → [05 Telegram Bot] Flow 4

[03 Recipe Costing]
  └─ Reads: IngredientPriceHistory (MAX committed_at) for live costs
  └─ Writes: Recipe.cost_per_portion after every cascade
  └─ Threshold breach → alert dispatched → [05 Telegram Bot] Flow 4

[05 Telegram Bot]
  └─ Reads only — calls .NET 10 API for all data
  └─ Receives outbound alerts from [02], [03], [04]
  └─ /link flow writes to: TelegramLink (status='confirmed')
```

---

## Shared Mechanisms

> **Historical note:** The table below preserves some team-oriented mechanisms for reference. When it conflicts with the v1 constraints, the v1 constraints win.

| Mechanism | Defined In | Used By |
|---|---|---|
| `ICostCascadeService.RecalculateForIngredient(ingredientId)` | Flow 03 | Flows 02, 04 |
| Cost formula: `SUM((price/unit_size)*qty*(1/yield))/portions` | Flow 03 | Flows 02, 04, 05 |
| Price lookup: `ORDER BY committed_at DESC LIMIT 1` | Flow 02 & 03 | Flows 04, 05 |
| TelegramLink recipient query | Historical enterprise reference | v1 simplifies this to the single linked user |
| `/api/telegram-links/{id}/deactivate` | Flow 01 | Flows 02, 05 |
| Role-split alert payloads | Historical v2-only reference | Not part of the current v1 build |

---

## Defects Found and Resolved

| ID | Flow | Description | Resolution |
|---|---|---|---|
| 2-A | 02 | Cascade service wrong name + unsafe NewPrice param | Fixed: `ICostCascadeService.RecalculateForIngredient(ingredientId)` |
| 2-B | 02 | Formula missing `/unit_size` | Fixed: added `/ unit_size` to formula |
| 2-C | 02 | `TelegramLink.IsActive` bool vs canonical status enum | Fixed: uses `status='confirmed'`; added 403 handling |
| 2-D | 02 | `TelegramLink.OutletId` does not exist | Fixed: filter through `User.OutletId` via existing join |
| 4-A | 04 | Transaction boundary engulfed cascade (critical) | Fixed: cascade runs OUTSIDE transaction |
| 4-B | 04 | Formula missing `/unit_size` | Fixed: `(unitPrice / unitSize)` in formula |
| 4-C | 04 | `createdAt` instead of `committed_at` | Fixed: canonical column name throughout |
| 4-D | 04 | Undefined role "Cost Controller" | Fixed: replaced with "Procurement" |
| 5-A | 05 | `User.telegram_user_id` column does not exist | Fixed: all writes go to TelegramLink only |
| 5-B | 05 | TelegramLink schema wrong (plaintext code, bool flag) | Fixed: canonical schema with codeHash + status enum |
| 5-C | 05 | Wrong formula + `unit_conversion_factor` + `yield_portions` | Fixed: canonical formula + `unit_size` + `portion_count` |
| 5-D | 05 | `recorded_at` vs canonical `committed_at` | Fixed: `committed_at` throughout |
| 5-E | 05 | `Outlet.food_cost_alert_threshold` does not exist | Fixed: `Recipe.cost_threshold_percentage` per-recipe |
| 5-F | 05 | Chef push alert missing data masking (received food_cost_pct) | Fixed: role-split payloads — Chef gets `cost_per_portion` only |
