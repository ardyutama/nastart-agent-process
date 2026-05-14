# Business Flows - v1

> **Track:** Active implementation track for the current single-user v1 build.
> **Authority order:** Start with `AGENTS.md`, then `.github/context/v1-constraints.md`, then the v1 flow files in this directory.
>
> **Rewrite status (May 14, 2026):** Auth, ingredient, recipe, invoice, and Telegram v1 flow files now exist in this directory. The invoice flow is intentionally guarded because the OCR packaging design doc referenced by `AGENTS.md` is missing from the repo.

---

## Purpose

This directory is the canonical home for current v1 flow specifications.

The v1 flow rewrites in this track must:
- use single-user terminology only
- avoid `Company`, `Outlet`, `Invitation`, `Role`, and other enterprise-only concepts
- match the routes, JWT claims, and response rules in `.github/context/v1-constraints.md`

Current files:
- `01-auth-telegram-linking.md`
- `02-ingredient-price-management.md`
- `03-recipe-builder-costing-engine.md`
- `04-invoice-scanning-review-commit.md`
- `05-telegram-bot-flows.md`

Use `AGENTS.md`, `.github/context/v1-constraints.md`, and the relevant lesson files alongside these flow files whenever a flow leaves implementation details open.