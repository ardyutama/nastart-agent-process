# Business Flows — Index

> Canonical flow root for the recipe cost management app.
> The repository now uses a versioned structure under `business-flows/`.
>
> **Authority order:** `AGENTS.md` → `.github/context/v1-constraints.md` → `business-flows/v1/` → supporting lessons/context docs → `business-flows/v2-reference/`.

---

## Track Routing

| Track | Purpose | Current status | Use for implementation? |
|---|---|---|---|
| Root docs | Routing plus shared canonical decisions | Active | Yes, for discovery and shared rules |
| `v1/` | Single-user v1 flow specifications | Structure created; detailed rewrites pending | Yes |
| `v2-reference/` | Historical enterprise flow documents | Preserved for reference | No |

---

## Current Rewrite Status

- The flat numbered enterprise flow files were moved into `business-flows/v2-reference/`.
- `business-flows/v1/` now contains the active single-user rewrite set.
- The invoice flow is written as a guarded v1 draft because the OCR packaging design doc referenced by `AGENTS.md` is not present in the repo.

---

## Root Documents

| File | Purpose |
|---|---|
| [00-index.md](00-index.md) | This router page |
| [00-canonical-decisions.md](00-canonical-decisions.md) | Shared canonical decisions with explicit v1 applicability notes |

---

## v1 Track

**Default track for all current implementation work.**

Start here:
- [v1/README.md](v1/README.md)

Current v1 rewrite set:
- [v1/01-auth-telegram-linking.md](v1/01-auth-telegram-linking.md)
- [v1/02-ingredient-price-management.md](v1/02-ingredient-price-management.md)
- [v1/03-recipe-builder-costing-engine.md](v1/03-recipe-builder-costing-engine.md)
- [v1/04-invoice-scanning-review-commit.md](v1/04-invoice-scanning-review-commit.md)
- [v1/05-telegram-bot-flows.md](v1/05-telegram-bot-flows.md)

v1 rules:
- single-user terminology only
- no `Company`, `Outlet`, `Invitation`, or `Role`
- routes, JWT claims, and response shapes must match `.github/context/v1-constraints.md`

---

## v2 Reference Track

**Historical enterprise reference only.** These files preserve earlier architecture and validation history but are not current implementation authority.

| Flow | File |
|---|---|
| 01 — Auth, Company Setup, Telegram Linking | [v2-reference/01-auth-company-telegram-linking.md](v2-reference/01-auth-company-telegram-linking.md) |
| 02 — Ingredient Management & Price History | [v2-reference/02-ingredient-price-management.md](v2-reference/02-ingredient-price-management.md) |
| 03 — Recipe Builder & Costing Engine | [v2-reference/03-recipe-builder-costing-engine.md](v2-reference/03-recipe-builder-costing-engine.md) |
| 04 — Invoice Scanning, Review Queue & Price Commit | [v2-reference/04-invoice-scanning-review-commit.md](v2-reference/04-invoice-scanning-review-commit.md) |
| 05 — Telegram Bot | [v2-reference/05-telegram-bot-flows.md](v2-reference/05-telegram-bot-flows.md) |

Shared mechanics that still matter across tracks remain documented in [00-canonical-decisions.md](00-canonical-decisions.md).
