# Business Flows — Index

> Recipe Cost Management App  
> All flows validated by Lead Architect Agent — final round: April 2, 2026

---

## Validation Summary

| Flow | File | Lead Architect Verdict | Rounds |
|---|---|---|---|
| 00 — Canonical Decisions | [00-canonical-decisions.md](00-canonical-decisions.md) | ✅ Authority document | — |
| 01 — Auth, Company Setup, Telegram Linking | [01-auth-company-telegram-linking.md](01-auth-company-telegram-linking.md) | ✅ ACCEPTED | Round 1 |
| 02 — Ingredient Management & Price History | [02-ingredient-price-management.md](02-ingredient-price-management.md) | ✅ ACCEPTED | Round 2 (3 defects + 1 late defect fixed) |
| 03 — Recipe Builder & Costing Engine | [03-recipe-builder-costing-engine.md](03-recipe-builder-costing-engine.md) | ✅ ACCEPTED (canonical reference) | Round 1 |
| 04 — Invoice Scanning, Review Queue & Price Commit | [04-invoice-scanning-review-commit.md](04-invoice-scanning-review-commit.md) | ✅ ACCEPTED | Round 2 (4 defects fixed) |
| 05 — Telegram Bot | [05-telegram-bot-flows.md](05-telegram-bot-flows.md) | ✅ ACCEPTED | Round 2 (6 defects fixed) |

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

```
[01 Auth]
  └─ Creates: Company, Outlet, User, OutletUser, TelegramLink (pending)
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

| Mechanism | Defined In | Used By |
|---|---|---|
| `ICostCascadeService.RecalculateForIngredient(ingredientId)` | Flow 03 | Flows 02, 04 |
| Cost formula: `SUM((price/unit_size)*qty*(1/yield))/portions` | Flow 03 | Flows 02, 04, 05 |
| Price lookup: `ORDER BY committed_at DESC LIMIT 1` | Flow 02 & 03 | Flows 04, 05 |
| TelegramLink recipient query | Flow 02 (corrected) | Flows 03, 04, 05 |
| `/api/telegram-links/{id}/deactivate` | Flow 01 | Flows 02, 05 |
| Role-split alert payloads | Flow 03 | Flow 05 |

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
