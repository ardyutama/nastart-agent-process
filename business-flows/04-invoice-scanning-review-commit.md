# Flow 04 — Invoice Scanning, Review Queue & Price Commit

> **Status:** ACCEPTED (Round 2, after defect corrections) — Lead Architect validated
> **Service owner:** .NET 10 Web API + Python FastAPI (OCR/LLM)
> **Entities:** Invoice, InvoiceLineItem, ReviewQueueItem, IngredientPriceHistory, Supplier
> **Storage:** S3-compatible object storage (MiniStack locally, Cloudflare R2 on VPS, AWS S3 future — swapped via `AWS_ENDPOINT_URL`. Never stored pre-signed URLs, TTL 15min)

---

## Flow 1: Invoice Upload & OCR Processing

**Actors:** Owner or Procurement ONLY (enforced at both Vue.js UI and .NET 10 API)
**Entry point:** Invoice scanning page → "Upload Invoice"

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Selects/photographs invoice image | Vue.js | image file (jpg/png/pdf) | - | Role check: only Owner/Procurement see upload button | - |
| 2 | .NET 10 API | Validates role | .NET 10 | JWT (role) | - | Chef/Viewer → 403 | 403 Forbidden |
| 3 | .NET 10 API | Uploads image to S3-compatible storage | S3-compatible (env-controlled) | image binary | s3_key (stored), pre-signed URL not stored | S3 failure → abort; no DB record created | 500 |
| 4 | .NET 10 API | Creates Invoice record | PostgreSQL | outlet_id, supplier_id (optional), s3_key, status='PROCESSING', uploaded_by=User.id, uploadedAt=NOW() | Invoice.id | - | DB error → attempt S3 delete + 500 |
| 5 | .NET 10 API | Returns 202 Accepted to Vue.js | .NET 10 | Invoice.id | 202 + Invoice.id | - | - |
| 6 | Vue.js | Displays "Processing…" state | Vue.js | Invoice.id | Polling / SSE listener | - | - |
| 7 | .NET 10 API | Calls Python FastAPI OCR endpoint (async) | Python FastAPI | s3_key, invoice_id, outlet_id | HTTP 202 from FastAPI | FastAPI unreachable → retry 3x, set status='FAILED' | Notify user |
| 8 | Python FastAPI | Fetches image from S3-compatible storage, runs OCR (Tesseract or AWS Textract) | S3-compatible + OCR | s3_key | raw_ocr_text, line_items[] | OCR fails → status='FAILED' | Notify via callback |
| 9 | Python FastAPI | Runs LLM extraction on OCR text | LLM | raw_ocr_text | structured line_items[{description, quantity, unit, unit_price}] | Low LLM score → confidence_level='LOW' | Partial results still returned |
| 10 | Python FastAPI | Attempts ingredient name matching | PostgreSQL (via .NET 10 API) | description[], outlet_id | [{description, matched_ingredient_id, confidence_score}] | null match → confidence='LOW' regardless of other scores | - |
| 11 | Python FastAPI | Computes confidence_level per line item | Python FastAPI | ocr_score, llm_score, match_result | confidence_level: 'HIGH' (all ≥ 0.85 + match found) or 'LOW' (any signal fails) | - | - |
| 12 | Python FastAPI | Calls .NET 10 API to persist line items | .NET 10 API | invoice_id, line_items[], confidence_levels[], raw_ocr_text per line | InvoiceLineItem[], ReviewQueueItem[] | - | - |
| 13 | .NET 10 API | Creates InvoiceLineItem + ReviewQueueItem records; updates Invoice.status='REVIEW' | PostgreSQL | invoice_id, line_items[], raw_ocr_text | Records created | - | - |
| 14 | Vue.js | Review queue appears (SSE/polling detects REVIEW status) | Vue.js | Invoice.status='REVIEW' | Review queue screen | - | - |

**InvoiceLineItem schema (immutable after creation):**
```
id                  UUID
invoice_id          UUID        FK → Invoice.id
raw_ocr_text        text        Original machine read — IMMUTABLE
description         text        LLM-extracted description
quantity            decimal
unit                text
unit_price          decimal
confidence_level    enum        'HIGH' | 'LOW'
matched_ingredient_id UUID      FK → Ingredient.id — null if unmatched
```

---

## Flow 2: Review Queue — Approve, Edit, Reject

**Actors:** Owner or Procurement
**Entry point:** Invoice review queue page

### Sub-flow A: Approve high-confidence item

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sees HIGH confidence item with matched ingredient | Vue.js | ReviewQueueItem | Pre-filled row | - | - |
| 2 | User | Clicks Approve | Vue.js | ReviewQueueItem.id | - | matched_ingredient_id must be non-null to approve | Show "Map ingredient first" |
| 3 | .NET 10 API | Sets ReviewQueueItem.status='APPROVED' | PostgreSQL | ReviewQueueItem.id | Updated status | - | - |

### Sub-flow B: Edit then approve low-confidence item

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sees LOW confidence item (highlighted) | Vue.js | ReviewQueueItem | Editable row | - | - |
| 2 | User | Edits price, quantity, or maps to correct ingredient | Vue.js | corrected values, matched_ingredient_id | - | - | - |
| 3 | .NET 10 API | Saves edits to ReviewQueueItem | PostgreSQL | ReviewQueueItem.id, corrected values, wasEdited=true, editedByUserId=User.id, editedAt=NOW() | Updated ReviewQueueItem | - | - |
| 4 | User | Clicks Approve | Vue.js | ReviewQueueItem.id | - | matched_ingredient_id must be non-null | Show "Map required" |
| 5 | .NET 10 API | Sets status='APPROVED' | PostgreSQL | ReviewQueueItem.id | Approved | - | - |

### Sub-flow C: Reject item

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Clicks Reject on any item | Vue.js | ReviewQueueItem.id | Confirmation modal | - | - |
| 2 | User | Confirms rejection | Vue.js | - | - | Cancel → no change | - |
| 3 | .NET 10 API | Sets ReviewQueueItem.status='REJECTED' | PostgreSQL | ReviewQueueItem.id | Rejected (no IngredientPriceHistory created) | - | - |

---

## Flow 3: Price Commit & Auto-Cascade

**Entry point:** Review queue → all items reviewed → "Confirm All" button
**Trigger:** User confirms all non-rejected ReviewQueueItems

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Clicks "Confirm All Approved Items" | Vue.js | Invoice.id | - | Any items still PENDING → button disabled | Show count of pending |
| 2 | Vue.js | Calls commit endpoint | .NET 10 API | Invoice.id | - | - | - |
| 3 | .NET 10 API | Server-side PENDING check (independent of UI) | PostgreSQL | Invoice.id | PENDING count | Any PENDING → 422 Unprocessable | Return error |
| 4 | .NET 10 API | **BEGINS ATOMIC TRANSACTION** | PostgreSQL | — | — | — | — |
| 5 | .NET 10 API | For each APPROVED ReviewQueueItem: INSERT IngredientPriceHistory | PostgreSQL | ingredient_id, price, source='InvoiceScan', committed_at=NOW(), effective_date=Invoice.invoiceDate??NOW(), invoiceLineItemId | IngredientPriceHistory.id | — | Transaction rollback on DB error |
| 6 | .NET 10 API | Sets Invoice.status='COMMITTED' | PostgreSQL | Invoice.id | — | — | Transaction rollback on DB error |
| 7 | .NET 10 API | **COMMITS TRANSACTION** | PostgreSQL | — | Committed | — | — |
| 8 | .NET 10 API | (OUTSIDE transaction) For each unique ingredient_id: calls ICostCascadeService.RecalculateForIngredient(ingredientId) | .NET 10 | ingredient_id[] (deduplicated) | affectedRecipeCount, new costs | Per-recipe error → log to CascadeErrorLog{ingredientId, recipeId, error, timestamp}, skip, NEVER rollback | Continue to next |
| 9 | .NET 10 API | (OUTSIDE transaction) Checks price spike threshold per ingredient | .NET 10 | new_price, previous_price, Ingredient.price_spike_threshold_pct | change_pct | change_pct > threshold → Fire price spike alert (Owner+Procurement) | - |
| 10 | .NET 10 API | Returns 200 + summary to Vue.js | .NET 10 | {committed_count, affected_recipes, spike_alerts_sent} | 200 OK | - | - |
| 11 | Vue.js | Shows confirmation screen with recipe cost changes | Vue.js | summary | Results dashboard | - | - |

**Telegram alerts are OUTSIDE the DB transaction boundary.** A failed Telegram delivery can never roll back a committed price.

---

## Flow 4: Audit Trail

Complete chain from image to price history — satisfies Cost Controller / Procurement audit requirements.

```
S3-compatible storage (invoice image)
  └── Invoice.s3_key  [Invoice record]
        └── Invoice.id → InvoiceLineItem.invoice_id  [raw_ocr_text immutable]
              └── InvoiceLineItem.id → ReviewQueueItem.invoiceLineItemId  [wasEdited, editedByUserId]
                    └── InvoiceLineItem.id → IngredientPriceHistory.invoiceLineItemId  [committed_at, price]
```

| Element | Storage | Mutable? | Purpose |
|---|---|---|---|
| Invoice image | S3-compatible storage (MiniStack → R2 → AWS S3) | No | Original evidence |
| `InvoiceLineItem.raw_ocr_text` | PostgreSQL | No (set at OCR time) | Original machine read |
| `ReviewQueueItem.wasEdited`, `editedByUserId`, `editedAt` | PostgreSQL | No (set on save) | Who corrected it |
| `IngredientPriceHistory` | PostgreSQL | INSERT only, never UPDATE | Final committed price |
| S3 pre-signed URL | Not stored | N/A | Generated on demand with 15-min TTL |

**Auditor query:** Given an `IngredientPriceHistory` record, retrieve its justifying invoice image S3 key:
```sql
SELECT inv.s3_key, inv.uploadedAt, u.name AS uploaded_by, s.name AS supplier
FROM IngredientPriceHistory iph
JOIN InvoiceLineItem ili ON ili.id = iph.invoiceLineItemId
JOIN Invoice inv ON inv.id = ili.invoice_id
JOIN User u ON u.id = inv.uploaded_by
LEFT JOIN Supplier s ON s.id = inv.supplier_id
WHERE iph.id = @priceHistoryId
```
