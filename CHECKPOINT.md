# Project Checkpoint — Recipe Cost Management App
> Saved: April 9, 2026  
> Purpose: Full conversation state export. Use this file to resume the project in a new conversation with complete context.
> 
> **Update (April 17, 2026):** This file is now a historical snapshot. For the current v1 implementation scope, read `AGENTS.md`, `.github/context/v1-constraints.md`, `docs/decisions/ADR-001-single-user-v1-scope.md`, and `docs/progress/00-current-status.md` first.

---

## How to resume this project

Paste this into a new conversation:

> "I am resuming a project from a saved checkpoint. The file is at `nastart-agent-process/CHECKPOINT.md`. Please read it fully before anything else — then cross-check it against `AGENTS.md`, `.github/context/v1-constraints.md`, `docs/decisions/ADR-001-single-user-v1-scope.md`, and `docs/progress/00-current-status.md` before making implementation decisions. Ask me nothing until you have confirmed you have read and understood the full checkpoint and the current v1 constraints."

---

## 1. What This Project Is

A **recipe costing app for solopreneurs** — home bakers, small food entrepreneurs, and solo F&B operators — with three core pillars:
1. **Recipe costing engine** — calculates cost-per-portion from real ingredient prices and outputs a recommended sell price based on the user's target margin
2. **Receipt scanning** — photograph a supermarket receipt, OCR extracts prices, ingredient costs cascade automatically across all affected recipes
3. **Telegram personal alerts** — when an ingredient price spikes, a message lands on your phone before you next quote a customer (ambient awareness, not team collaboration)

**Primary target user:** A solo home baker or food entrepreneur currently managing costs in Excel or by gut feel, who undercharges because they don't know their true cost.

**Dual purpose:**
- A **real product** for solopreneurs — single user, one kitchen, many recipes, no team hierarchy
- A **portfolio project** targeting full-stack developer jobs and freelance clients

> **Architecture note (from April 9 ideation):** v1 is single-user only. The 4-role system (Owner / Chef / Procurement / Viewer) and Company → Outlet → User hierarchy are parked for v2 if the product expands to F&B teams. See `docs/ideas/recipe-cost-solopreneur.md` for full rationale.

---

## 2. Tech Stack (Locked)

| Layer | Technology |
|---|---|
| Frontend | Vue.js (Vite + TypeScript) |
| Main Backend API | .NET 10 Web API (C#) |
| AI / OCR / Bot Service | Python FastAPI |
| Database | PostgreSQL 18 |
| File Storage | MiniStack (local dev) → Cloudflare R2 (VPS) → AWS S3 (future) — all S3-compatible, swap via `AWS_ENDPOINT_URL` |
| Bot Framework | python-telegram-bot |
| Deployment | Docker Compose → VPS |
| Auth | JWT tokens with `userId` + `email` claims only in v1 |

**Why dual-service backend:** .NET 10 handles all business logic, auth, and data. Python FastAPI handles all AI/OCR/LLM/Telegram concerns as an isolated microservice. This separation is intentional and locked.

---

## 3. Roles

> **v1 decision (April 9):** v1 is single-user — there are no roles. The user is simultaneously Owner, Chef, and Procurement. Role logic is not built in Phase 1–3.
> The 4-role system below is preserved for v2 reference only, if the product expands to multi-user F&B teams.

### v2 reference — canonical roles (do not build in v1)

| Role | What they see |
|---|---|
| Owner | cost_per_portion + food_cost_pct + gross_margin + supplier prices + all alerts |
| Chef | cost_per_portion only — NO margins, NO food_cost_pct |
| Procurement | cost_per_portion + supplier prices — NO food_cost_pct |
| Viewer | cost_per_portion only — read-only |

Role stripping is **server-side only**. Vue.js never receives `food_cost_pct` or `gross_margin` for non-Owner roles.

---

## 4. Build Plan — 5 Phases

### Phase 1 — Foundation (Weeks 1–3)
- PostgreSQL schema: `User`, `Ingredient`, `IngredientPriceHistory`, `Unit`, `Category`, `TelegramLink`
- .NET 10 API: JWT auth, authenticated-user access, ingredient CRUD foundations
- Vue.js: login screen, ingredient management page
- **Checkpoint**: Register, verify, log in, then create and price an ingredient in a user-scoped library

### Phase 2 — Recipe Costing Engine (Weeks 4–7)
- .NET 10: `Recipe`, `RecipeItem`, cost-per-portion calculation with unit normalization
- Single-user scope: recipes and ingredients belong directly to the authenticated user
- Vue.js: recipe builder UI with live cost display, derived sell price, and packaging or target margin inputs
- **Checkpoint**: Build a 5-ingredient recipe, change one ingredient price, watch cost-per-portion update live

### Phase 3 — AI Invoice Scanning (Weeks 8–11)
- Python FastAPI: image upload → OCR (AWS Textract or Tesseract) → LLM line-item extraction
- Review queue: low-confidence items flagged for manual confirmation before any price commits
- Auto-cascade: confirmed price ripples through all affected recipe costs
- Vue.js: invoice upload screen, review queue UI, price change alert badges
- **Checkpoint**: Upload a photographed invoice, approve OCR output, verify recipe costs updated

### Phase 4 — Telegram Bot (Weeks 12–15)
- `python-telegram-bot` calling Python FastAPI
- Commands: `/cost [dish]`, `/ingredients [name]`, `/summary`, `/alerts`, `/link XXXX`
- Personal responses for the single linked user (no role filtering in v1)
- LLM natural language fallback for free-text queries
- **Checkpoint**: Send a natural language Telegram message, receive the correct personal cost summary

### Phase 5 — Portfolio Polish & Deploy (Weeks 16–20)
- Docker Compose wiring all 4 services
- Deploy to VPS (Docker Compose)
- Vue.js: responsive polish, loading/empty/error states
- GitHub: Mermaid architecture diagram, detailed README, setup guide
- Loom demo video covering all 3 product pillars
- Portfolio website case study page
- **Checkpoint**: Public URL live + GitHub public + demo video linked on portfolio

---

## 5. Repo Structure

```
nastart/
├── backend/                    ← .NET 10
│   ├── Nastart.API/
│   ├── Nastart.Application/
│   └── Nastart.Infrastructure/
├── ai-service/                 ← Python FastAPI
│   ├── ocr/
│   ├── llm/
│   └── telegram/
├── frontend/                   ← Vue.js + Vite + TypeScript
│   └── src/
│       ├── pages/
│       ├── components/
│       └── api/
├── docker-compose.yml
└── README.md
```

---

## 6. Canonical Design Decisions (14 — all locked, do not deviate)

> **v1 applicability note (April 17, 2026):** For the current single-user build, C-1 through C-7, C-13, and C-14 remain active. C-8 through C-12 are preserved below as historical v2 reference and are superseded for v1 by `.github/context/v1-constraints.md` and `docs/decisions/ADR-001-single-user-v1-scope.md`.

### C-1 — Cascade Service Interface
`ICostCascadeService.RecalculateForIngredient(ingredientId)` — no price parameter. Service fetches current price internally.

### C-2 — Cost Formula
```
cost_per_portion = SUM(
  (current_price / unit_size) * quantity * (1 / yield_percentage)
) / portion_count
```

### C-3 — Current Price Lookup
```sql
SELECT price FROM IngredientPriceHistory
WHERE ingredient_id = @id
ORDER BY committed_at DESC LIMIT 1
```
No `current_price` column on `Ingredient`. Current price is always derived.

### C-4 — IngredientPriceHistory Timestamps
- `committed_at` — system timestamp on INSERT, non-editable, used for ordering
- `effective_date` — user-editable, business effective date (defaults to invoice date)

### C-5 — Cascade Failure Isolation
Per-recipe cascade errors NEVER roll back `IngredientPriceHistory`. Log to `CascadeErrorLog{ingredientId, recipeId, error, timestamp}` and skip. Cascade runs OUTSIDE the price commit transaction.

### C-6 — TelegramLink Entity
```
TelegramLink {
  id               UUID
  userId           UUID          FK → User.id
  codeHash         varchar(64)   SHA-256 hash (NEVER plaintext)
  expiresAt        timestamptz   15-minute TTL
  status           enum          'pending' | 'confirmed' | 'unlinked'
  telegramUserId   bigint        Telegram user_id
  telegramUsername varchar(255)  nullable
  linkedAt         timestamptz   nullable
}
```
**User entity has NO `telegram_user_id` column.**

### C-7 — Telegram Deactivation
Both `400 Bad Request` (invalid chat_id) AND `403 Forbidden` (bot blocked) → `PATCH /api/telegram-links/{id}/deactivate` → `status = 'unlinked'`. Nothing written to User.

### C-8 — Canonical Roles
Exactly: `Owner`, `Chef`, `Procurement`, `Viewer` — no others.

### C-9 — Alert Threshold Location
`Recipe.cost_threshold_percentage` — per recipe, NOT per outlet. No `food_cost_alert_threshold` on `Outlet`.

### C-10 — Role Visibility (All Responses + Alerts)
| Role | cost_per_portion | food_cost_pct | gross_margin |
|---|---|---|---|
| Owner | ✅ | ✅ | ✅ |
| Chef | ✅ | ❌ | ❌ |
| Procurement | ✅ | ❌ | ❌ |
| Viewer | ✅ | ❌ | ❌ |

Push alert payloads must be built separately per role — absent fields are not null, they are not present.

### C-11 — Price Spike Alert Recipients
Owner + Procurement

### C-12 — Food Cost Threshold Alert Recipients
Owner + Chef — with role-split payloads:
- Owner: `{ recipe_name, food_cost_pct, gross_margin, cost_per_portion }`
- Chef: `{ recipe_name, cost_per_portion }` — food_cost_pct absent

### C-13 — IngredientPriceHistory.source Enum
Exactly `'Manual'` or `'InvoiceScan'` (case-sensitive).

### C-14 — TelegramLink Chat ID Column
`telegramUserId` — not `chatId`, not `TelegramChatId`. Used everywhere.

---

## 7. Key Design Decisions (from interviews)

- **Recipe scope**: In v1, each recipe belongs to ONE user directly. Outlet scoping is parked for v2.
- **Telegram linking**: User generates a code in the web app → sends `/link XXXX` to the bot
- **Invoice scanning access**: The single authenticated user owns the full OCR review and commit flow in v1
- **Ingredient price history**: Full history with timestamps — NEVER overwrite, always append
- **Portfolio "done" definition**: Live deployed app + clean GitHub repo + Loom demo video
- **Real users in v1**: No — dummy data and self-testing only
- **Timeline**: 3–6 months, 5 phases to prevent scope creep
- **The developer's biggest risk**: Scope creep / never finishing — phases are designed to be independently shippable

---

## 8. Business Flow Files (all validated)

> **Scope note (April 17, 2026):** These flows remain valuable design history, but not every file is a current v1 implementation guide. Flow 01 is historical enterprise onboarding reference, and Flow 05 is historical except for the `/link` mechanics in Flow 1. Use `AGENTS.md`, `.github/context/v1-constraints.md`, and `docs/progress/00-current-status.md` when there is a conflict.

All flows are saved in `nastart-agent-process/business-flows/`. Full step-by-step tables with actors, components, data in/out, decision points, and error cases.

| File | Contents |
|---|---|
| `00-index.md` | Index + cross-flow data map + full defect resolution log |
| `00-canonical-decisions.md` | All 14 canonical decisions with full detail |
| `01-auth-company-telegram-linking.md` | Company registration, user invitation, `/link XXXX` |
| `02-ingredient-price-management.md` | Ingredient CRUD, manual price update, cascade algorithm, spike alerts |
| `03-recipe-builder-costing-engine.md` | Recipe creation, auto-cascade, threshold alerts, versioning |
| `04-invoice-scanning-review-commit.md` | OCR upload, review queue (approve/edit/reject), price commit, audit trail |
| `05-telegram-bot-flows.md` | Historical Telegram reference; only the `/link` mechanics in Flow 1 remain actionable for v1 |

These flows were validated by a Lead Architect Agent in a 3-round review. 13 defects were found and resolved before final acceptance.

---

## 9. Persona Files

All personas saved in `nastart-agent-process/personas/`. Each file contains 10 user stories, 5 use cases, pain points, concerns, and a "day in the life" narrative.

| File | Persona |
|---|---|
| `01-fnb-owner.md` | F&B Owner / Restaurateur |
| `02-head-chef.md` | Head Chef / Kitchen Manager |
| `03-procurement.md` | Procurement / Purchasing Staff |
| `04-cost-controller.md` | Accountant / F&B Cost Controller |
| `05-home-baker.md` | Casual Home Baker / Small Food Entrepreneur |

### Cross-persona shared pain points (all 5 agreed on these)
1. OCR must flag low-confidence items for manual review — silent errors are worse than no automation
2. Telegram access must be role-based — different personas see different data
3. Ingredient costs must auto-update when an invoice is scanned, rippling through all affected recipes

### Cross-persona shared feature gaps (raised by multiple personas)
- Yield/waste factor support at ingredient-recipe level
- Unit normalization (suppliers invoice in kg, box, carcass — must normalize)
- Packaging and overhead cost add-ons (critical for home baker persona)
- Export CSV/PDF for accountants
- Audit trail with immutable records

---

## 10. Future Features / Roadmap (discussed but not in v1)

These were raised during persona and planning sessions. Not in scope for the 5-phase build, but validated as valuable for future versions:

### Near-term (post-MVP additions)
- **CSV/PDF export** — food cost summary for accountants and management packs (raised by Cost Controller + Home Baker personas)
- **What-if scenario modeling** — temporarily adjust an ingredient price and see the ripple effect across all recipes before committing to a supplier contract (raised by Cost Controller)
- **Dashboard with charts and analytics** — visual food cost % trends over time (raised during portfolio planning)
- **User authentication improvements** — password reset, session management, remember me

### Medium-term
- **Packaging and overhead cost add-ons** — add packaging cost (per unit) and labour rate (per hour) as add-ons to recipe cost, not just raw ingredients (critical gap from home baker persona)
- **Async background cascade processing** — move auto-cascade to a background job for outlets with many recipes (advisory note from Lead Architect: synchronous cascade acceptable for MVP, but revisit for Phase 4+)
- **Supplier management** — full supplier CRUD, preferred supplier flags, delivery schedule tracking
- **POS integration** — connect to POS sales data to automatically calculate actual vs. theoretical food cost variance (raised by Head Chef as ideal future state)
- **Inventory tracking** — actual stock counts to calculate waste and shrinkage vs. recipe-theoretical usage

### Long-term / v2
- **Multi-currency support** — for international or multi-country outlet groups
- **Franchise / corporate group reporting** — consolidated food cost reporting across many outlets under one company (raised by Cost Controller multi-outlet use case)
- **Mobile app** — native iOS/Android for the home baker and loading-bay invoice scanning use case
- **Accounting system integration** — Xero, QuickBooks push for COGS entries
- **Recipe scaling for events** — bulk scaling a recipe to any number of covers with automatic cost recalculation (mentioned by Head Chef catering use case)

---

## 11. Current Status

**As of April 17, 2026:**

| Deliverable | Status |
|---|---|
| Product concept | ✅ Defined — refined to solopreneur focus (April 9) |
| Persona research (5 personas) | ✅ Complete — files in `personas/` |
| Tech stack | ✅ Locked (.NET 10, Vue.js, Python FastAPI, PostgreSQL) |
| Build plan (5 phases) | ✅ Defined |
| Canonical design decisions (14) | ✅ Locked (role/hierarchy decisions parked for v2) |
| Business flows (5 flows, multi-agent validated) | ✅ Complete — files in `business-flows/` |
| Portfolio interview (goals, done definition) | ✅ Complete |
| Ideation refinement | ✅ Complete — see `docs/ideas/recipe-cost-solopreneur.md` |
| Lesson series (`L1`–`L8`) | ✅ Drafted — v1 amendments applied across `L2`–`L8` |
| Documentation sync | ✅ Updated — see `docs/progress/00-current-status.md` |
| Application code in this repo | ❌ Not present — this repo stores process and documentation artifacts |

**Next action:** use `docs/progress/00-current-status.md` as the working lesson ledger, then continue cleaning the lesson bodies that still preserve enterprise or authoring residue (`L3`, `L5`, `L6`, `L7`, `L8`).

---

## 12. File Map of This Project

```
nastart-agent-process/
├── CHECKPOINT.md                         ← THIS FILE — full conversation state
├── docs/
│   ├── decisions/
│   │   └── ADR-001-single-user-v1-scope.md ← Current v1 scope decision
│   └── ideas/
│       └── recipe-cost-solopreneur.md    ← Ideation one-pager (April 9)
│   └── progress/
│       └── 00-current-status.md           ← Lesson and documentation status ledger
├── personas/
│   ├── 00-index.md                       ← Persona index + cross-persona themes
│   ├── 01-fnb-owner.md
│   ├── 02-head-chef.md
│   ├── 03-procurement.md
│   ├── 04-cost-controller.md
│   └── 05-home-baker.md
└── business-flows/
    ├── 00-index.md                       ← Flow index + defect log
    ├── 00-canonical-decisions.md        ← 14 locked canonical decisions
    ├── 01-auth-company-telegram-linking.md
    ├── 02-ingredient-price-management.md
    ├── 03-recipe-builder-costing-engine.md
    ├── 04-invoice-scanning-review-commit.md
    └── 05-telegram-bot-flows.md
```
