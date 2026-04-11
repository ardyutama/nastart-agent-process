# Priced Right — Recipe Costing for Solopreneurs

## Problem Statement
How might we help a home baker or solo food entrepreneur know their true per-item cost — so they stop undercharging and can confidently price for profit?

---

## Recommended Direction
Build a single-user recipe costing app that calculates cost-per-item from real ingredient prices and outputs a recommended sell price based on the user's target margin. Strip all enterprise complexity — no multi-outlet, no multi-role, no invitation system. One user, one kitchen, many recipes.

The core loop: enter ingredients and prices → build a recipe → set your target margin → get your recommended sell price. When a receipt is scanned and an ingredient price changes, all recipe costs and sell prices update automatically.

Telegram becomes a personal cost alert channel: when a key ingredient price spikes, a message lands on your phone before you next quote a customer. This is ambient awareness, not team collaboration.

---

## Key Assumptions to Validate
- [ ] Home bakers will photograph supermarket receipts to update prices — test with 3 real bakers, compare manual vs. scan to see which they actually use
- [ ] "Recommended sell price" is more useful than raw cost alone — show both in early UI and ask which one they act on
- [ ] Target users already use Telegram personally — confirm before building the bot; if not, a simple in-app notification is enough

---

## MVP Scope

**In:**
- Single user account (email + password — no invitations, no roles)
- Ingredient library with price history (unit normalization: g, kg, ml, L, each)
- Recipe builder — add ingredients, quantities, portion count → live cost-per-portion
- Target margin input per recipe → recommended sell price output
- Manual price update with auto-cascade to all affected recipes
- Receipt photo scanning (OCR, any type — supermarket or supplier) → confidence-scored review queue → confirm → cascade
- `packaging_cost` flat add-on per recipe (e.g. $0.45/box) included in sell price calculation
- Telegram personal alert: ingredient price spike → "your brownie cost went up 12%, consider charging $X now"

**Out of MVP:**
- Multi-outlet, multi-user, invitation system
- Role-based access (Owner / Chef / Procurement / Viewer)
- Company / Outlet hierarchy
- PDF / CSV export
- Dashboard analytics and charts

---

## Not Doing (and Why)

- **4-role system** — there is only one user; role splitting adds zero value for a solopreneur and doubles the auth/API surface area
- **Company → Outlet → User hierarchy** — over-engineering for a single-kitchen user; add in v2 only if product expands to F&B teams
- **Invitation and team management** — no team to invite; this complexity exists purely to support the enterprise use case that was deprioritised
- **Telegram team notifications** — a solopreneur has no Chef or Procurement to notify; Telegram is reframed as a personal cost alert only
- **What-if scenario modeling** — powerful, but the core loop must work first; already on the future roadmap

---

## Open Questions

- ~~Does OCR accuracy hold for supermarket receipts?~~ **Resolved (April 9):** All receipt types go through the same confidence-scored review queue. LLM maps noisy strings to the ingredient library. Unmatched lines surface for manual resolution. See `docs/plans/2026-04-09-ocr-packaging-design.md`.
- ~~Should recommended sell price account for packaging costs?~~ **Resolved (April 9):** `packaging_cost` flat field per recipe (default $0). Sell price = (ingredient cost + packaging cost) / (1 − target margin). See `docs/plans/2026-04-09-ocr-packaging-design.md`.
- Is the .NET + Python + Vue dual-service stack the right call for a solopreneur product? Intentionally kept as a portfolio showcase — not a question to resolve, a choice to own.

---

> Refined: April 9, 2026 — via idea-refine ideation session
