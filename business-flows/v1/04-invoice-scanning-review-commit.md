# Flow 04 - Invoice Scanning, Review Queue & Price Commit

> **Track:** `business-flows/v1/`
> **Authority:** Active implementation guidance for the current single-user v1 build. Read this file with `AGENTS.md`, `.github/context/v1-constraints.md`, `business-flows/00-canonical-decisions.md`, and the v1 ingredient flow.
>
> **Service owner:** Nastart.Api + Python FastAPI (OCR/LLM)
> **Scope note:** The repo currently does not contain the OCR packaging design doc referenced by `AGENTS.md`. This flow therefore locks the stable v1 rules already established in the repo and avoids inventing missing packaging-specific details.

---

## Active Rules For This Flow

- Invoice scanning is a single-user flow; there is no outlet or role gate
- OCR output must be reviewed before any price commit updates ingredient costs
- Confirmed invoice prices append `IngredientPriceHistory` rows with `Source='InvoiceScan'`
- Price commits happen inside a transaction; cascade runs outside the transaction
- Cascade failures never roll back committed invoice prices
- Any price-spike alert triggered by invoice prices goes only to the single linked user

---

## Flow 1: Upload Invoice Image For OCR

**Actor:** Authenticated User
**Entry point:** Invoice scanning page in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Selects or captures invoice image | Vue.js | image file | upload request | - | Client-side file validation |
| 2 | Nastart.Api | Authorizes upload and validates request | Nastart.Api | JWT, file metadata | User.id | Unauthenticated -> 401 | Reject |
| 3 | Nastart.Api | Stores file in S3-compatible storage | S3-compatible storage | image binary | storage key | Storage failure -> abort before DB write | 500 |
| 4 | Nastart.Api | Creates invoice-processing record | PostgreSQL | User.id, storage key, status='PROCESSING' | invoice processing id | - | DB error -> attempt cleanup and fail |
| 5 | Nastart.Api | Calls Python FastAPI OCR workflow asynchronously | Python FastAPI | storage key, processing id | accepted OCR job | OCR service unavailable -> mark processing failure | Retry/log |
| 6 | Vue.js | Shows processing state | Vue.js | processing id | waiting UI | - | Retry option if processing fails |

---

## Flow 2: Persist OCR Output Into Review Queue

**Trigger:** Python FastAPI finishes OCR and extraction.

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | Python FastAPI | Extracts line items from invoice image | Python FastAPI | OCR text, parsed fields | extracted line items | Low-confidence extraction -> mark for review | Partial extraction allowed |
| 2 | Python FastAPI | Attempts ingredient matching | Python FastAPI or Nastart.Api-assisted lookup | extracted descriptions | candidate ingredient ids | No match -> keep unmatched | - |
| 3 | Python FastAPI | Sends structured extraction result to Nastart.Api | Nastart.Api | processing id, line items, confidence metadata, raw OCR text | persistence request | Invalid payload -> reject | 400 |
| 4 | Nastart.Api | Creates invoice line item records and review queue items | PostgreSQL | extracted items, confidence, matched ingredient references | review queue rows | - | DB error -> 500 |
| 5 | Nastart.Api | Updates processing record to `REVIEW` | PostgreSQL | processing id | updated status | - | DB error -> 500 |
| 6 | Vue.js | Shows review queue | Vue.js | refreshed status | review UI | - | Empty/error state if no items persisted |

**Review principle:** OCR extraction is never silently committed to ingredient prices. Human review is required first.

---

## Flow 3: Review, Edit, Approve, Or Reject Items

**Actor:** Authenticated User
**Entry point:** Invoice review queue screen

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens review queue | Vue.js | processing id | queue items | - | - |
| 2 | User | Reviews extracted item | Vue.js | line item, confidence, matched ingredient | item decision | - | - |
| 3 | User | Optionally edits price, quantity, unit, or ingredient match | Vue.js | corrected values | draft corrections | - | Validation error |
| 4 | Nastart.Api | Saves review edits | PostgreSQL | review item id, corrected values | updated review item | - | DB error -> 500 |
| 5 | User | Approves or rejects the item | Vue.js | review item id, decision | decision request | Approve requires ingredient mapping | Show correction prompt |
| 6 | Nastart.Api | Persists final review status | PostgreSQL | review item id, approved/rejected | updated review item | - | DB error -> 500 |

**Approval rule:** Only approved items can produce `IngredientPriceHistory` rows.

---

## Flow 4: Commit Approved Prices & Trigger Cascade

**Actor:** Authenticated User
**Entry point:** Review screen -> confirm approved items

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Confirms approved items | Vue.js | processing id | commit request | Pending unresolved items can block commit if business rule requires full review | Show pending count |
| 2 | Nastart.Api | Verifies review state server-side | PostgreSQL | processing id, User.id | approved item set | Invalid state -> 422 | Reject |
| 3 | Nastart.Api | Begins transaction | PostgreSQL | - | transaction scope | - | Transaction failure -> 500 |
| 4 | Nastart.Api | Inserts `IngredientPriceHistory` rows for approved items | PostgreSQL | ingredientId, price, unitSize snapshot, source='InvoiceScan', effectiveDate, committedAt, invoice reference | price history rows | DB error -> rollback transaction | 500 |
| 5 | Nastart.Api | Marks invoice-processing record as committed | PostgreSQL | processing id | committed status | - | DB error -> rollback transaction |
| 6 | Nastart.Api | Commits transaction | PostgreSQL | - | committed data | - | - |
| 7 | Nastart.Api | Outside the transaction, deduplicates ingredient ids and calls `ICostCascadeService.RecalculateForIngredient(ingredientId)` | Application service | ingredient ids | cascade results | Per-recipe failure -> log `CascadeErrorLog` and continue | Do not rollback committed price rows |
| 8 | Nastart.Api | Optionally triggers price spike alerts for affected ingredients | Nastart.Api -> Python FastAPI | ingredient price change payload | async alert requests | No confirmed link or threshold not crossed -> skip | Log delivery failures |
| 9 | Nastart.Api | Returns commit summary | Nastart.Api | committed count, affected recipes | success response | - | - |
| 10 | Vue.js | Shows commit results | Vue.js | commit summary | updated UI | - | - |

---

## Flow 5: Audit Trail

The invoice flow must preserve a trace from uploaded image to committed ingredient price.

Required evidence chain:
- uploaded file storage key
- raw OCR text or equivalent source capture
- review edits and approval decision
- resulting `IngredientPriceHistory` rows with `Source='InvoiceScan'`

**Audit rule:** committed ingredient prices remain append-only even when the original extraction needed manual correction.

---

## Stable v1 Guarantees In This Flow

- No company, outlet, or team routing
- No supplier comparison or supplier-scoped approval logic
- No direct price overwrite on `Ingredient`
- No cascade inside the price-commit transaction
- No rollback of committed invoice prices because a downstream recipe update failed

---

## Open Detail Pending Missing OCR Design Doc

These details should be finalized when the missing OCR packaging design doc is restored or replaced:
- exact review queue field set
- confidence scoring model and thresholds
- storage metadata shape for uploaded invoice assets
- any packaging-specific extraction fields beyond ingredient price commit