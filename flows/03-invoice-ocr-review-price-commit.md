# Business Process Flow: Invoice Scanning, OCR, Review Queue & Price Commit

**Document type**: Implementation-ready business process specification  
**Produced for**: Lead Architect review — must be consistent with Ingredient Management and Recipe Costing flows  
**Status**: Draft v2 — Revised per Lead Architect review  
**Date**: 2026-04-02  

---

## Design Constraints Active in This Document

| Constraint | Value |
|---|---|
| Backend | .NET 10 Web API |
| AI Service | Python FastAPI |
| Frontend | Vue.js (Vite + TypeScript) |
| Database | PostgreSQL |
| File Storage | S3-compatible (MiniStack locally → Cloudflare R2 on VPS → AWS S3 future, via `AWS_ENDPOINT_URL`) |
| Auth | JWT tokens |
| Authorised upload roles | Owner, Procurement only |
| Price history write mode | Append-only — always INSERT new `IngredientPriceHistory`; never UPDATE |
| Review queue | Mandatory — no OCR result commits without human approval of each line item |
| Confidence flagging | LOW_CONFIDENCE items visually flagged; different user interaction path |
| Auto-cascade trigger | Shared mechanism with Ingredient Management and Recipe Costing flows |

---

## Canonical Entity Names

`Company` · `Outlet` · `User` · `Role` · `TelegramLink` · `Ingredient` · `IngredientPriceHistory` · `Unit` · `Category` · `Recipe` · `RecipeItem` · `Invoice` · `InvoiceLineItem` · `ReviewQueueItem` · `Supplier`

---

## Flow 1 — Invoice Upload & OCR Processing

**Scope**: From the moment an Owner or Procurement user photographs or selects an invoice image, through OCR and LLM processing in the Python FastAPI AI Service, to the moment extracted line items appear in the Vue.js Review Queue.

---

### Step 1.1

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement (`User`) |
| **Action** | User taps "Upload Invoice" in the Vue.js frontend. Selects or photographs an invoice image on their device. |
| **System component** | Vue.js |
| **Data IN** | Local image file (JPEG / PNG / PDF); selected `Outlet`; optional `Supplier` selection |
| **Data OUT** | File object held in component state; `outletId` and optional `supplierId` ready for submission |
| **Decision point** | **YES** — Is the authenticated `User.Role` Owner or Procurement? If NO → UI blocks upload button; "Insufficient permissions" shown. |
| **Error case** | File picker cancelled — no action taken. Role check failure navigates to access-denied screen. |

---

### Step 1.2

| Field | Detail |
|---|---|
| **Actor** | Vue.js |
| **Action** | Client-side validation: check MIME type (image/jpeg, image/png, application/pdf); check file size ≤ configured maximum (e.g., 10 MB). If valid, submit `multipart/form-data` POST to `.NET 10 Web API` at `POST /api/v1/invoices/upload`. |
| **System component** | Vue.js |
| **Data IN** | Image file; `outletId`; optional `supplierId`; optional `invoiceDate`; optional `invoiceReferenceNumber`; `Authorization: Bearer {jwt}` header |
| **Data OUT** | HTTP POST to `.NET 10 Web API` |
| **Decision point** | **YES** — File fails MIME or size check? If YES → display inline validation error; halt submission. |
| **Error case** | Network timeout — display retry prompt with exponential backoff hint. |

---

### Step 1.3

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Validate JWT. Resolve `User` from token claims. Confirm `User.Role` is Owner or Procurement. Confirm `User` belongs to the `Company` that owns the target `Outlet`. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | JWT (Bearer); `outletId` |
| **Data OUT** | Resolved `User` principal with `Role`; authorised or rejected |
| **Decision point** | **YES** — Is role Owner or Procurement AND does the `User` belong to the correct `Company`? If NO → return `403 Forbidden`. |
| **Error case** | Expired JWT → `401 Unauthorized`. DB unreachable → `503 Service Unavailable`. |

---

### Step 1.4

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Generate a unique, non-guessable S3 object key: `invoices/{userId}/{uuid}.{ext}`. Upload raw image bytes to S3-compatible storage using server-side boto3/AWS SDK (endpoint controlled by `AWS_ENDPOINT_URL`). |
| **System component** | .NET 10 Web API + S3-compatible storage |
| **Data IN** | Raw image bytes; generated S3 key |
| **Data OUT** | S3 object confirmed stored; `s3Key` (string); final S3 object URL |
| **Decision point** | **YES** — Did S3 upload succeed? If NO → return `502 Bad Gateway`; do NOT create `Invoice` record; surface error to user. |
| **Error case** | S3 access denied (IAM misconfiguration) → `502`. S3 service outage → retry once, then fail gracefully with user-visible message. File never stored client-side or forwarded onward without S3 confirmation. |

---

### Step 1.5

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | INSERT new `Invoice` record in PostgreSQL within a database transaction. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | `outletId`; `uploadedByUserId` (from JWT); `supplierId` (nullable); `s3Key`; `invoiceDate` (nullable; defaults to `uploadedAt`); `invoiceReferenceNumber` (nullable); `uploadedAt = NOW()` |
| **Data OUT** | New `Invoice.id` (UUID); `Invoice.status = 'PENDING_OCR'`; `Invoice.uploadedAt` |
| **Decision point** | **YES** — DB INSERT succeeds? If NO → rollback transaction; attempt to delete S3 object (best-effort); return `500 Internal Server Error`. |
| **Error case** | Duplicate `invoiceReferenceNumber` for same `Supplier` within the same `Outlet` → return `409 Conflict` with message "Invoice reference already exists". |

---

### Step 1.6

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Asynchronously dispatch OCR job to Python FastAPI AI Service via internal HTTP POST to `POST /internal/ocr/process`. Return `202 Accepted` to the Vue.js frontend immediately (do not make the user wait for OCR). |
| **System component** | .NET 10 Web API → Python FastAPI |
| **Data IN** | `invoiceId`; `s3Key`; `outletId`; `companyId` |
| **Data OUT** | `202 Accepted` to Vue.js with body `{ invoiceId, status: "PENDING_OCR", message: "Invoice received. OCR processing started." }` |
| **Decision point** | **YES** — Is Python FastAPI AI Service reachable? If NO → set `Invoice.status = 'OCR_FAILED'`; queue job for retry; notify user that processing is delayed. |
| **Error case** | AI Service returns non-2xx immediately → set `Invoice.status = 'OCR_FAILED'`; alert via Telegram if configured. |

---

### Step 1.7

| Field | Detail |
|---|---|
| **Actor** | Python FastAPI AI Service |
| **Action** | Receive OCR job. Generate a time-limited pre-signed URL (or use server-side credentials to download). Fetch image bytes from S3-compatible storage. |
| **System component** | Python FastAPI + S3-compatible storage |
| **Data IN** | `s3Key`; `invoiceId` |
| **Data OUT** | Raw image bytes in memory |
| **Decision point** | **YES** — S3 download successful? If NO → POST failure callback to `.NET 10 Web API` (`POST /internal/ocr/failed`); set `Invoice.status = 'OCR_FAILED'`. |
| **Error case** | S3 key not found (stale job, manual deletion) → failure callback. Image corrupt/unreadable → failure callback with reason. |

---

### Step 1.8

| Field | Detail |
|---|---|
| **Actor** | Python FastAPI AI Service |
| **Action** | Pass image bytes to OCR engine (AWS Textract preferred; Tesseract fallback). Extract raw text, per-word bounding boxes, and per-token confidence scores (0.0–1.0). |
| **System component** | Python FastAPI (OCR engine) |
| **Data IN** | Raw image bytes |
| **Data OUT** | OCR result set: `{ raw_text: string, tokens: [{ text, confidence_score, bounding_box }] }` |
| **Decision point** | **YES** — Did OCR return any usable text (at least one token with confidence_score > 0.3)? If NO → POST failure callback; set `Invoice.status = 'OCR_FAILED'`. |
| **Error case** | Image too blurry or low-resolution → all confidence scores near zero → failure callback with reason `"LOW_IMAGE_QUALITY"`. Textract API quota exceeded → retry with backoff. |

---

### Step 1.9

| Field | Detail |
|---|---|
| **Actor** | Python FastAPI AI Service |
| **Action** | Pass OCR `raw_text` to LLM (structured extraction prompt) instructing it to identify line items. For each detected line, extract: `ingredient_name`, `quantity`, `unit_raw_text`, `unit_price`, `total_price`, `supplier_item_code` (if present). Return a JSON array. |
| **System component** | Python FastAPI (LLM) |
| **Data IN** | `raw_text` (full OCR output); `invoiceId` |
| **Data OUT** | `extracted_items: [{ ingredient_name, quantity, unit_raw_text, unit_price, total_price, supplier_item_code, llm_confidence_score }]` |
| **Decision point** | **YES** — LLM returns valid parseable JSON? If NO → treat all items as `confidence_level = 'LOW'`; use regex fallback parser; never discard the OCR attempt. |
| **Error case** | LLM timeout → fall back to regex parser, tag all items `LOW_CONFIDENCE`. LLM token limit exceeded (very large invoice) → chunk the text and merge results. |

---

### Step 1.10

| Field | Detail |
|---|---|
| **Actor** | Python FastAPI AI Service |
| **Action** | For each extracted item: (1) Normalise `unit_raw_text` against known `Unit` records (read from DB via internal API call to `.NET 10 Web API` at `GET /internal/units`). (2) Perform fuzzy name-match against `Ingredient` records for the `outletId` (via `GET /internal/ingredients?outletId=...&search=...`). (3) Assign `matched_ingredient_id` if best match score ≥ threshold (e.g., 0.80); else leave `null`. (4) Compute final `confidence_level`: `HIGH` if (OCR token confidence ≥ 0.85) AND (LLM confidence ≥ 0.80) AND (`matched_ingredient_id` is not null); otherwise `LOW`. |
| **System component** | Python FastAPI + .NET 10 Web API (internal read) |
| **Data IN** | `extracted_items[]`; `outletId` |
| **Data OUT** | Enriched items: each has `matched_ingredient_id` (UUID or null); `matched_unit_id` (UUID or null); `confidence_level` (`'HIGH'` or `'LOW'`); `ocr_confidence_score`; `llm_confidence_score` |
| **Decision point** | **YES** — Is `matched_ingredient_id` null? If YES → `confidence_level` forced to `LOW` regardless of OCR/LLM scores, because a manual Ingredient mapping is required. |
| **Error case** | `.NET 10 Web API` internal endpoint unreachable → tag all items `LOW_CONFIDENCE`; proceed rather than abort. |

---

### Step 1.11

| Field | Detail |
|---|---|
| **Actor** | Python FastAPI AI Service |
| **Action** | POST all enriched extracted items back to `.NET 10 Web API` at `POST /internal/ocr/results`. |
| **System component** | Python FastAPI → .NET 10 Web API |
| **Data IN** | `invoiceId`; `items: [{ raw_ocr_text, ingredient_name, matched_ingredient_id, matched_unit_id, quantity, unit_raw_text, unit_price, total_price, supplier_item_code, confidence_level, ocr_confidence_score, llm_confidence_score }]` |
| **Data OUT** | `200 OK` acknowledgement from .NET 10 Web API |
| **Decision point** | **YES** — Did `.NET 10 Web API` acknowledge with 2xx? If NO → retry up to 3 times with exponential backoff. After 3 failures → set `Invoice.status = 'OCR_FAILED'` via direct DB write (emergency fallback). |
| **Error case** | Network partition between services → retry logic handles transient failure. Persistent failure → alert via Telegram to Owner. |

---

### Step 1.12

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Receive OCR results. Within a single database transaction: (1) For each item, INSERT one `InvoiceLineItem` record. (2) For each `InvoiceLineItem`, INSERT one linked `ReviewQueueItem` record with `status = 'PENDING'`. (3) UPDATE `Invoice.status = 'PENDING_REVIEW'`. Commit transaction. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | `invoiceId`; array of enriched extracted items |
| **Data OUT** | N × `InvoiceLineItem` records (each with `status = 'PENDING'`); N × `ReviewQueueItem` records (each with `status = 'PENDING'`, `confidence_level` from OCR step); `Invoice.status = 'PENDING_REVIEW'` |
| **Decision point** | **YES** — Any DB constraint violated (e.g., invalid `matched_ingredient_id` FK)? If YES → set that item's `matched_ingredient_id = null`; force `confidence_level = 'LOW'`; continue inserting rather than rolling back entire batch. |
| **Error case** | DB transaction timeout → full rollback; leave `Invoice.status = 'OCR_FAILED'`; surface error. |

---

### Step 1.13

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Emit a real-time notification to the Vue.js frontend. Mechanism: Server-Sent Events (SSE) or polling endpoint `GET /api/v1/invoices/{invoiceId}/status`. Push: review queue badge count updated for the `Outlet`. |
| **System component** | .NET 10 Web API → Vue.js |
| **Data IN** | `invoiceId`; `outletId`; `reviewQueueItemCount` |
| **Data OUT** | Vue.js displays "OCR complete — N items awaiting review" notification; Review Queue badge count updated |
| **Decision point** | NO |
| **Error case** | SSE connection dropped → non-fatal; user sees updated count on next manual page refresh or poll interval. |

---

**End state of Flow 1:**  
`Invoice.status = 'PENDING_REVIEW'` · N × `InvoiceLineItem` (status `'PENDING'`) · N × `ReviewQueueItem` (status `'PENDING'`, each with `confidence_level`)  
Original image immutably stored at `s3Key`. No prices committed yet.

---

---

## Flow 2 — Review Queue: Approve, Edit, and Reject Line Items

**Scope**: A User (Owner or Procurement) works through the Review Queue. Three sub-flows cover all terminal states for a `ReviewQueueItem`.

---

### Step 2.1 — Load Review Queue (Shared Entry Point)

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement (`User`) |
| **Action** | Navigate to Review Queue screen. Vue.js fetches all `ReviewQueueItem` records for the `Outlet` with `status = 'PENDING'`, sorted: `LOW_CONFIDENCE` first, then `HIGH`. |
| **System component** | Vue.js + .NET 10 Web API + PostgreSQL |
| **Data IN** | JWT; `outletId`; optional `invoiceId` filter |
| **Data OUT** | `ReviewQueueItem[]` each showing: `confidence_level` (badge colour: AMBER for LOW, GREEN for HIGH); `ingredient_name` (OCR extracted); matched `Ingredient.name` (if found); `quantity`; `Unit.name`; `unit_price`; `total_price`; `Supplier.name`; `Invoice.invoiceReferenceNumber` |
| **Decision point** | **YES** — Is `User.Role` authorised (Owner or Procurement)? If NO → `403`; do not expose queue. |
| **Error case** | Empty queue → show "No items pending review" empty state. |

---

### Sub-Flow 2A — Approve a HIGH_CONFIDENCE Item

---

#### Step 2A.1

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement |
| **Action** | Inspect a `ReviewQueueItem` with `confidence_level = 'HIGH'`. Review: matched `Ingredient.name`, `quantity`, `Unit.name`, `unit_price`. Click **"Approve"**. |
| **System component** | Vue.js |
| **Data IN** | `ReviewQueueItem` display data |
| **Data OUT** | `PATCH /api/v1/review-queue/{reviewQueueItemId}/approve` |
| **Decision point** | NO — HIGH_CONFIDENCE items do not require editing before approval. |
| **Error case** | User may still choose to edit before approving (see Sub-Flow 2B) — Approve is a one-click fast path only when the data is correct. |

---

#### Step 2A.2

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Validate JWT + role. UPDATE `ReviewQueueItem`: `status = 'APPROVED'`, `approvedByUserId = {userId}`, `approvedAt = NOW()`, `wasEdited = false`. UPDATE linked `InvoiceLineItem.status = 'APPROVED'`. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | `reviewQueueItemId`; JWT |
| **Data OUT** | `ReviewQueueItem.status = 'APPROVED'`; `InvoiceLineItem.status = 'APPROVED'` · No `IngredientPriceHistory` record created yet (price commit is deferred to Flow 3) |
| **Decision point** | **YES** — Does `ReviewQueueItem.matched_ingredient_id` exist (non-null)? If NO → reject the approval attempt; return `400 "Ingredient not mapped — edit required before approving"`. |
| **Error case** | DB error → `500`; Vue.js retains item in PENDING state. |

---

**DB outcome — Sub-Flow 2A:**  
`ReviewQueueItem.status = 'APPROVED'` · `wasEdited = false` · `approvedByUserId` set · `approvedAt` set  
`InvoiceLineItem.status = 'APPROVED'`  
No `IngredientPriceHistory` record created.

---

### Sub-Flow 2B — Edit a LOW_CONFIDENCE Item, then Approve

---

#### Step 2B.1

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement |
| **Action** | Select a `ReviewQueueItem` with `confidence_level = 'LOW'` (AMBER badge). Vue.js renders the item in an inline edit form. Displays both the original OCR raw text and the current (possibly wrong) parsed values side-by-side for reference. |
| **System component** | Vue.js |
| **Data IN** | `ReviewQueueItem` with `confidence_level = 'LOW'`; `raw_ocr_text` displayed as read-only reference |
| **Data OUT** | Edit form opened with fields: `Ingredient` (searchable dropdown over `Ingredient` records for `outletId`); `quantity` (numeric); `Unit` (dropdown over `Unit` records); `unit_price` (decimal); `total_price` (auto-calculated, editable override) |
| **Decision point** | NO |
| **Error case** | None — form is always available for LOW_CONFIDENCE items. |

---

#### Step 2B.2

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement |
| **Action** | Correct one or more fields. Common corrections: re-map `Ingredient` (select from dropdown), fix `unit_price` (OCR misread), correct `Unit` (e.g., OCR read "KGS" → correct to "KG"). Click **"Save Changes"**. |
| **System component** | Vue.js |
| **Data IN** | User-edited values for: `matched_ingredient_id`, `quantity`, `matched_unit_id`, `unit_price`, `total_price` |
| **Data OUT** | `PATCH /api/v1/review-queue/{reviewQueueItemId}` with corrected fields |
| **Decision point** | **YES** — Client-side validation: `unit_price > 0`; `quantity > 0`; `matched_ingredient_id` must be a valid UUID from the dropdown. If any fail → inline validation error; do not submit. |
| **Error case** | Network failure → retry prompt; edited values retained in form state. |

---

#### Step 2B.3

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Validate JWT + role. UPDATE `InvoiceLineItem` with corrected values. UPDATE `ReviewQueueItem`: set corrected values, `wasEdited = true`, `editedByUserId = {userId}`, `editedAt = NOW()`. Status remains `'PENDING'` — still requires explicit Approval. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | `reviewQueueItemId`; corrected `matched_ingredient_id`, `quantity`, `matched_unit_id`, `unit_price`, `total_price` |
| **Data OUT** | `InvoiceLineItem` updated with corrected values; `ReviewQueueItem.wasEdited = true`; `ReviewQueueItem.editedByUserId` set; `ReviewQueueItem.editedAt` set; `ReviewQueueItem.status` remains `'PENDING'` |
| **Decision point** | **YES** — Is `matched_ingredient_id` a valid FK to an existing `Ingredient` record belonging to this `Outlet`? If NO → `400 Bad Request`. |
| **Error case** | DB FK violation → `400`; surface "Ingredient not found" error in Vue.js. |

---

#### Step 2B.4

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement |
| **Action** | Review the now-corrected item. Click **"Approve"**. |
| **System component** | Vue.js |
| **Data IN** | Corrected `ReviewQueueItem` data |
| **Data OUT** | `PATCH /api/v1/review-queue/{reviewQueueItemId}/approve` |
| **Decision point** | NO |
| **Error case** | None. |

---

#### Step 2B.5

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | UPDATE `ReviewQueueItem`: `status = 'APPROVED'`, `approvedByUserId = {userId}`, `approvedAt = NOW()` (preserves `wasEdited = true` and `editedByUserId` from Step 2B.3). UPDATE linked `InvoiceLineItem.status = 'APPROVED'`. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | `reviewQueueItemId`; JWT |
| **Data OUT** | `ReviewQueueItem.status = 'APPROVED'`; `InvoiceLineItem.status = 'APPROVED'` (with corrected field values) |
| **Decision point** | **YES** — Does `matched_ingredient_id` exist (non-null, valid)? If NO → `400`; approval blocked until ingredient is mapped. |
| **Error case** | DB error → `500`. |

---

**DB outcome — Sub-Flow 2B:**  
`ReviewQueueItem.status = 'APPROVED'` · `wasEdited = true` · `editedByUserId` set · `editedAt` set · `approvedByUserId` set · `approvedAt` set  
`InvoiceLineItem` updated with corrected field values · `InvoiceLineItem.status = 'APPROVED'`  
No `IngredientPriceHistory` record created.

---

### Sub-Flow 2C — Reject a Line Item

---

#### Step 2C.1

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement |
| **Action** | Click **"Reject"** on a `ReviewQueueItem` (e.g., OCR artefact, duplicate line item, item from a different invoice, completely unreadable entry). Optionally enter a free-text `rejectionReason`. Click **"Confirm Reject"** in confirmation modal. |
| **System component** | Vue.js |
| **Data IN** | `reviewQueueItemId`; optional `rejectionReason` (string, max 500 chars) |
| **Data OUT** | `PATCH /api/v1/review-queue/{reviewQueueItemId}/reject` with `{ rejectionReason }` |
| **Decision point** | **YES** — Confirmation modal: "Are you sure? Rejected items will not update ingredient prices." User must confirm before PATCH is sent. |
| **Error case** | Network failure → retry prompt; item remains `PENDING`. |

---

#### Step 2C.2

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Validate JWT + role. UPDATE `ReviewQueueItem`: `status = 'REJECTED'`, `rejectedByUserId = {userId}`, `rejectedAt = NOW()`, `rejectionReason = {value or null}`. UPDATE linked `InvoiceLineItem.status = 'REJECTED'`. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | `reviewQueueItemId`; `rejectionReason`; JWT |
| **Data OUT** | `ReviewQueueItem.status = 'REJECTED'`; `InvoiceLineItem.status = 'REJECTED'` |
| **Decision point** | NO — Any item can be rejected regardless of confidence level or edit state. |
| **Error case** | DB error → `500`. |

---

**DB outcome — Sub-Flow 2C:**  
`ReviewQueueItem.status = 'REJECTED'` · `rejectedByUserId` set · `rejectedAt` set · `rejectionReason` stored (nullable)  
`InvoiceLineItem.status = 'REJECTED'`  
No `IngredientPriceHistory` record created — rejected items never commit prices.

---

### Review Queue — Complete Status Matrix

| Path | `ReviewQueueItem.status` | `wasEdited` | `InvoiceLineItem.status` | `IngredientPriceHistory` created? |
|---|---|---|---|---|
| 2A: Approve HIGH | `APPROVED` | `false` | `APPROVED` | No (deferred to Flow 3) |
| 2B: Edit + Approve LOW | `APPROVED` | `true` | `APPROVED` (corrected values) | No (deferred to Flow 3) |
| 2C: Reject (any) | `REJECTED` | unchanged | `REJECTED` | Never |

---

---

## Flow 3 — Price Commit & Auto-Cascade

**Scope**: From the "Confirm all approved items" action, through price commit, through the auto-cascade in the Recipe Costing Engine, to updated recipe costs and threshold alerts.

> **CRITICAL NOTE**: The **auto-cascade** in Step 3.6 calls `ICostCascadeService.RecalculateForIngredient(ingredientId)` — the **exact same service interface** invoked by the Ingredient Management flow when a price is manually edited and by the Recipe Costing Engine when a recalculation is triggered. The canonical cost formula is: `SUM((current_price / unit_size) * quantity * (1 / yield_percentage)) / portion_count`. **The cascade executes OUTSIDE the atomic DB transaction**, which covers only the `IngredientPriceHistory` INSERT(s) and `Invoice.status` UPDATE (Steps 3.3–3.4).

---

### Step 3.1

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement |
| **Action** | After reviewing all items for an `Invoice`, the "Commit Prices" button becomes active (Vue.js enables it only when zero `ReviewQueueItem` records for the `Invoice` have `status = 'PENDING'`). User clicks **"Commit Prices"**. |
| **System component** | Vue.js |
| **Data IN** | `invoiceId`; user intent to commit |
| **Data OUT** | `POST /api/v1/invoices/{invoiceId}/commit` |
| **Decision point** | **YES** — Are ALL `ReviewQueueItem` records for this `Invoice` in a terminal state (`APPROVED` or `REJECTED`)? If ANY remain `PENDING` → button is disabled; tooltip: "All items must be approved or rejected before committing." |
| **Error case** | Network failure → retry prompt. |

---

### Step 3.2

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Validate JWT + role. Server-side check: query DB for any `ReviewQueueItem` linked to this `Invoice` with `status = 'PENDING'`. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | `invoiceId`; JWT |
| **Data OUT** | Proceed to Step 3.3 if zero PENDING items; else error |
| **Decision point** | **YES** — Any `ReviewQueueItem` still `PENDING`? If YES → return `400 Bad Request { error: "Unreviewed items remain. All ReviewQueueItems must reach a terminal state before commit." }`. This is a server-side guard independent of the client-side button state. |
| **Error case** | DB query failure → `503`. |

---

### Step 3.3

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | **[BEGIN ATOMIC TRANSACTION — scope: Steps 3.3 + 3.4 only.]** For each `InvoiceLineItem` with `status = 'APPROVED'` linked to this `Invoice`: INSERT a new `IngredientPriceHistory` record. **Do not UPDATE any existing record. Always INSERT.** |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | Each approved `InvoiceLineItem`: `ingredientId` (from `matched_ingredient_id`), `supplierId`, `unit_price`, `matched_unit_id`, `quantity`, `invoiceLineItemId`, `Invoice.invoiceDate` |
| **Data OUT** | New `IngredientPriceHistory` record per approved line item: `{ id (UUID), ingredientId, supplierId, unitPrice, unitId, effective_date = Invoice.invoiceDate ?? NOW(), source = 'InvoiceScan', invoiceLineItemId (FK — preserves audit link), recordedByUserId = {userId from JWT}, committed_at = NOW() }` |
| **Decision point** | **YES** — Does `ingredientId` resolve to a valid `Ingredient` record in this `Outlet`? If NO → rollback entire transaction; return `400`; this should not occur if Review Queue approval gate (Step 2A.2) was enforced correctly. |
| **Error case** | DB constraint violation → full rollback. Duplicate prevention: `IngredientPriceHistory` has no unique constraint on `(ingredientId, effectiveDate)` — multiple deliveries on the same day are valid; do not deduplicate. |

---

### Step 3.4

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | UPDATE `Invoice.status = 'COMMITTED'`, `Invoice.committedAt = NOW()`, `Invoice.committedByUserId = {userId}`. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | `invoiceId`; `userId` |
| **Data OUT** | `Invoice.status = 'COMMITTED'`; `Invoice.committedAt`; `Invoice.committedByUserId` |
| **Decision point** | NO |
| **Error case** | DB error → rollback entire transaction (rolls back Step 3.3 `IngredientPriceHistory` inserts as well). |

---

### Step 3.5 — COMMIT ATOMIC TRANSACTION

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Commit the atomic database transaction. **Transaction scope: Steps 3.3–3.4 only** — `IngredientPriceHistory` INSERT(s) and `Invoice.status = 'COMMITTED'` UPDATE. After this commit, both records are durable and the price history is permanent regardless of what happens next. The auto-cascade (Step 3.6) executes **outside** this transaction boundary. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | Open transaction containing Step 3.3 INSERTs and Step 3.4 UPDATE |
| **Data OUT** | Transaction committed; `IngredientPriceHistory` records durable; `Invoice.status = 'COMMITTED'`; list of committed `ingredientId` values passed to Step 3.6 |
| **Decision point** | **YES** — Did the transaction commit successfully? If NO (DB error, timeout, deadlock) → full rollback of Steps 3.3–3.4; `Invoice.status` reverts to `'PENDING_REVIEW'`; no `IngredientPriceHistory` records are created; return `500 Internal Server Error`. |
| **Error case** | DB transaction timeout → rollback. Deadlock → retry once; on second failure → rollback + `500`. |

---

### Step 3.6 — AUTO-CASCADE: Recalculate Affected Recipe Costs (Outside Transaction)

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API (`ICostCascadeService`) |
| **Action** | **After the transaction in Step 3.5 has committed**, for each `ingredientId` in the committed list: call `ICostCascadeService.RecalculateForIngredient(ingredientId)`. No price is passed as a parameter — the service resolves the current price internally via: `SELECT price FROM IngredientPriceHistory WHERE ingredient_id = @id ORDER BY committed_at DESC LIMIT 1`. The service then fetches `Ingredient.unit_size` and all `RecipeItem` records referencing this ingredient. For each affected `Recipe`, cost is recalculated using the canonical formula: `costPerPortion = SUM((current_price / unit_size) * quantity * (1 / yield_percentage)) / portion_count`. UPDATE `Recipe.totalCost`, `Recipe.costPerPortion`, `Recipe.lastRecalculatedAt = NOW()`. **Error isolation (per C-5)**: each recipe's recalculation runs in an isolated try/catch. Any per-recipe error is caught, written to `CascadeErrorLog { ingredientId, recipeId, error, timestamp }`, and skipped. Per-recipe cascade failures **NEVER** roll back the `IngredientPriceHistory` records committed in Step 3.5 — the price commit is already durable. |
| **System component** | .NET 10 Web API (`ICostCascadeService`) + PostgreSQL (`CascadeErrorLog` table) |
| **Data IN** | List of committed `ingredientId` values (from Step 3.5); no price passed — service reads `committed_at`-ordered price history internally |
| **Data OUT** | Updated `Recipe.costPerPortion` and `Recipe.totalCost` for all successfully recalculated recipes; any per-recipe failures written to `CascadeErrorLog { ingredientId, recipeId, error, timestamp }` |
| **Decision point** | **YES** — Does a per-recipe recalculation fail? If YES → catch error, INSERT into `CascadeErrorLog`, skip that recipe, continue to next. The price commit in Step 3.5 is **NOT** undone. |
| **Error case** | All affected recipes fail → all logged to `CascadeErrorLog`; price commit remains durable. `Recipe.portionCount = 0` or null → skip `costPerPortion`, log warning. No `IngredientPriceHistory` for an ingredient → treat as `current_price = 0`; flag `Recipe.hasMissingPrices = true`. |

---

### Step 3.7 — THRESHOLD CHECK: Price Spike Detection

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | For each `Ingredient` that received a new `IngredientPriceHistory` in Step 3.3: (1) Fetch the **previous** `IngredientPriceHistory` record (the one before the newly inserted record, same `ingredientId`, ORDER BY `committed_at DESC LIMIT 1 OFFSET 1`). (2) Calculate: `priceChangePercent = ((newUnitPrice - previousUnitPrice) / previousUnitPrice) × 100`. (3) If `|priceChangePercent| > alertThreshold` (default 10%, configurable per `Company` or `Ingredient`): create an alert payload for Telegram notification. |
| **System component** | .NET 10 Web API + PostgreSQL |
| **Data IN** | New `IngredientPriceHistory.unitPrice`; previous `IngredientPriceHistory.unitPrice`; `alertThreshold` |
| **Data OUT** | `alerts: [{ ingredientId, ingredientName, oldPrice, newPrice, priceChangePercent, affectedRecipes: [{ recipeId, recipeName, oldCostPerPortion, newCostPerPortion }] }]` |
| **Decision point** | **YES** — Is this the first-ever `IngredientPriceHistory` record for this `Ingredient` (no previous record exists)? If YES → no comparison possible; no alert raised; this is expected behaviour for new ingredients. |
| **Error case** | None fatal — threshold check failure must not roll back the price commit. Log error; continue. |

---

### Step 3.8 — TELEGRAM ALERT: Notify Subscribers

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API → Python FastAPI (Telegram Bot) |
| **Action** | For each alert payload from Step 3.7: POST to Python FastAPI at `POST /internal/telegram/notify` with the alert payload. Python FastAPI queries `TelegramLink` records for the `Company` to find subscribed `User` Telegram chat IDs with roles `Owner` or `Procurement` (canonical alert recipients — C-11). Sends formatted Telegram message: ingredient name, old price, new price, % change, list of affected recipe names with revised cost-per-portion. |
| **System component** | Python FastAPI + Telegram Bot API + PostgreSQL (`TelegramLink` lookup) |
| **Data IN** | `{ companyId, ingredientName, oldPrice, newPrice, priceChangePercent, affectedRecipes[] }` |
| **Data OUT** | Telegram message delivered to all subscribed users with eligible roles for this `Company` |
| **Decision point** | **YES** — Are there any `TelegramLink` records for this `Company` with an active Telegram chat ID? If NO → skip silently; alert is not fatal. |
| **Error case** | Telegram API rate limit → queue message for retry; do not block the price commit transaction. User has blocked the bot → log error; remove or flag `TelegramLink` record. |

---

### Step 3.9 — RESPOND TO CLIENT

| Field | Detail |
|---|---|
| **Actor** | .NET 10 Web API |
| **Action** | Return `200 OK` to Vue.js with a summary payload. The DB transaction was already committed in Step 3.5. The cascade in Step 3.6 has run (any per-recipe failures were logged to `CascadeErrorLog`, not raised). Threshold checks (Step 3.7) and Telegram alerts (Step 3.8) have been dispatched. |
| **System component** | .NET 10 Web API |
| **Data IN** | Commit result metadata: `committedLineItems`, `updatedIngredients`, `affectedRecipes` (cascade successes), `alertsRaised` |
| **Data OUT** | `HTTP 200 { committedLineItems: N, updatedIngredients: N, affectedRecipes: N, alertsRaised: N }` |
| **Decision point** | NO — the DB transaction (Step 3.5) was committed or failed independently of this response. A response failure here does not roll back any data. |
| **Error case** | If Step 3.5 failed and returned `500`, this step is not reached — the `500` is returned immediately from Step 3.5. |

---

### Step 3.10

| Field | Detail |
|---|---|
| **Actor** | Vue.js |
| **Action** | Receive `200 OK`. Display success banner: "Prices committed — N recipes updated." Refresh the currently viewed recipe cost displays if any are open. Update `Invoice` status badge to "COMMITTED". Navigate user to invoice summary or dashboard. |
| **System component** | Vue.js |
| **Data IN** | `HTTP 200` response payload |
| **Data OUT** | Updated UI state; committed invoice removed from Review Queue; recipe cards show updated cost-per-portion values |
| **Decision point** | NO |
| **Error case** | `500` from API → display "Commit failed — no changes were saved. Please try again." with retry button. |

---

**End state of Flow 3:**  
`Invoice.status = 'COMMITTED'`  
N × `IngredientPriceHistory` records created (immutable, append-only; `source = 'InvoiceScan'`; ordered by `committed_at`)  
Affected `Recipe.costPerPortion` values updated for all recipes where cascade succeeded; per-recipe cascade failures logged to `CascadeErrorLog` (non-fatal to price commit)  
Telegram price-spike alerts sent to `Owner` + `Procurement` users where threshold exceeded  
**Atomic DB transaction scope**: Steps 3.3–3.4 only (`IngredientPriceHistory` INSERTs + `Invoice.status` UPDATE). Auto-cascade (Step 3.6) runs outside this transaction boundary and can never roll it back.

---

---

## Flow 4 — Audit Trail

**Scope**: The complete, traversable chain from original invoice image (S3) through every entity to the final committed price record. Satisfies the Procurement team's audit requirement: cross-period COGS investigation, accounts payable verification, and regulatory audit support.

---

### Step 4.1 — Search Invoices

| Field | Detail |
|---|---|
| **Actor** | Owner or Procurement (`User`) |
| **Action** | Open Audit / Invoice History view in Vue.js. Apply filters: `Supplier`, date range (`invoiceDate`), `Outlet`, `Invoice.status`, `invoiceReferenceNumber`. |
| **System component** | Vue.js + .NET 10 Web API + PostgreSQL |
| **Data IN** | JWT; optional filters: `supplierId`, `outletId`, `dateFrom`, `dateTo`, `status`, `invoiceReferenceNumber` |
| **Data OUT** | `Invoice[]` summary list: `{ id, invoiceReferenceNumber, Supplier.name, Outlet.name, invoiceDate, uploadedAt, uploadedByUser.name, committedAt, committedByUser.name, status, lineItemCount }` |
| **Decision point** | **YES** — Is role authorised to view audit data? Chef and Viewer may have `lineItemCount` visible but `unit_price` redacted per role policy. Owner and Procurement see full data. |
| **Error case** | No results matching filters → empty state with "No invoices found". Long-running query → paginate results (max 50 per page). |

---

### Step 4.2 — Inspect an Invoice

| Field | Detail |
|---|---|
| **Actor** | Procurement or Owner (`User`) |
| **Action** | Click an `Invoice` record to expand detail view. |
| **System component** | Vue.js + .NET 10 Web API + PostgreSQL |
| **Data IN** | `invoiceId` |
| **Data OUT** | `Invoice` detail: `{ id, invoiceReferenceNumber, supplierId, Supplier.name, Outlet.name, invoiceDate, s3Key, uploadedByUser.name, uploadedAt, committedByUser.name, committedAt, status }` + `InvoiceLineItem[]` summary |
| **Decision point** | NO |
| **Error case** | Invoice not found → `404`. |

---

### Step 4.3 — View Line Items

| Field | Detail |
|---|---|
| **Actor** | Procurement or Owner (`User`) |
| **Action** | View the `InvoiceLineItem` list for the `Invoice`. Each item shows current committed (or rejected) state. |
| **System component** | Vue.js + .NET 10 Web API + PostgreSQL |
| **Data IN** | `invoiceId` |
| **Data OUT** | `InvoiceLineItem[]`: `{ id, confidence_level, status, matched_ingredient_id, Ingredient.name, quantity, Unit.name, unit_price, total_price, supplier_item_code, raw_ocr_text (visible for audit comparison) }` |
| **Decision point** | NO |
| **Error case** | None. `raw_ocr_text` is stored immutably at the time of OCR; it never changes regardless of edits made during review. This enables direct comparison of "what OCR extracted" vs. "what was committed". |

---

### Step 4.4 — View Review Queue Item (Who Approved, Who Edited)

| Field | Detail |
|---|---|
| **Actor** | Procurement or Owner (`User`) |
| **Action** | Click an `InvoiceLineItem` to expand the linked `ReviewQueueItem` full audit record. |
| **System component** | Vue.js + .NET 10 Web API + PostgreSQL |
| **Data IN** | `invoiceLineItemId` |
| **Data OUT** | `ReviewQueueItem`: `{ id, status, confidence_level, wasEdited, editedByUser.name, editedAt, approvedByUser.name, approvedAt, rejectedByUser.name, rejectedAt, rejectionReason, original OCR values vs. committed values (field-by-field diff if wasEdited = true) }` |
| **Decision point** | **YES** — Is `wasEdited = true`? If YES → display a "Changes made during review" diff panel showing original OCR-extracted values alongside the user-corrected values that were committed. |
| **Error case** | None. All fields are stored at the time of action; no reconstruction required. |

---

### Step 4.5 — Navigate to IngredientPriceHistory Record

| Field | Detail |
|---|---|
| **Actor** | Procurement or Owner (`User`) |
| **Action** | From an APPROVED `InvoiceLineItem`, click "View Price Record" to navigate to the `IngredientPriceHistory` record that was created during the price commit. |
| **System component** | Vue.js + .NET 10 Web API + PostgreSQL |
| **Data IN** | `invoiceLineItemId` (FK on `IngredientPriceHistory.invoiceLineItemId`) |
| **Data OUT** | `IngredientPriceHistory`: `{ id, ingredientId, Ingredient.name, supplierId, Supplier.name, unitPrice, Unit.name, effective_date, source = 'InvoiceScan', invoiceLineItemId (FK — the exact line item that justified this price), recordedByUserId, User.name, committed_at }` |
| **Decision point** | **YES** — Is `InvoiceLineItem.status = 'REJECTED'`? If YES → no `IngredientPriceHistory` record exists (by design); display "Rejected — no price committed" in the detail panel instead of a link. |
| **Error case** | `IngredientPriceHistory` record missing for an APPROVED item → data integrity violation; surface as a system alert. |

---

### Step 4.6 — View Full Ingredient Price History Timeline

| Field | Detail |
|---|---|
| **Actor** | Procurement or Owner (`User`) |
| **Action** | Navigate to Ingredient detail view. View the full, immutable `IngredientPriceHistory` timeline for an `Ingredient`. Each record is a point-in-time snapshot. Records are never deleted or updated. |
| **System component** | Vue.js + .NET 10 Web API + PostgreSQL |
| **Data IN** | `ingredientId`; optional `supplierId` filter; optional date range |
| **Data OUT** | `IngredientPriceHistory[]` sorted by `committed_at DESC`: `{ id, unitPrice, unitId, Unit.name, effective_date, source, supplierId, Supplier.name, invoiceLineItemId (link back to source invoice line), recordedByUser.name, committed_at }` — complete immutable price tape |
| **Decision point** | NO — Every record is an append. No UPDATE or DELETE is permitted on `IngredientPriceHistory`. This invariant must be enforced at the application layer (no UPDATE/DELETE routes exist for `IngredientPriceHistory`) and may optionally be enforced at the DB layer via a trigger. |
| **Error case** | None. Historical records are immutable by design. |

---

### Step 4.7 — Retrieve Original Invoice Image

| Field | Detail |
|---|---|
| **Actor** | Procurement or Owner (`User`) |
| **Action** | Click "View Original Invoice Image" on the `Invoice` detail page. Vue.js requests a pre-signed URL from `.NET 10 Web API`. |
| **System component** | Vue.js + .NET 10 Web API + S3-compatible storage |
| **Data IN** | `invoiceId` (API resolves `Invoice.s3Key` from DB) |
| **Data OUT** | `.NET 10 Web API` generates a time-limited S3 pre-signed GET URL (TTL: 15 minutes). Returns URL to Vue.js. Vue.js renders image inline in an `<img>` tag or opens in a new tab. Pre-signed URL is never stored; it is generated on demand. |
| **Decision point** | **YES** — Is `Invoice.s3Key` present and non-null? If NO → `404 Not Found` (should never occur for committed invoices). **S3 objects for committed invoices must never be deleted** — this is an operational constraint, not a code constraint. Recommendation: enable S3 Object Lock (Compliance mode) on the invoice bucket for regulatory environments. |
| **Error case** | S3 unreachable → `502`; user sees "Image temporarily unavailable". Expired pre-signed URL (user leaves browser open > 15 min) → `403` from S3; Vue.js detects S3 `403` and auto-fetches a new pre-signed URL. |

---

### Audit Trail — Entity Linkage Map

```
S3-compatible storage (invoice image)
    │  s3Key
    ▼
Invoice
    │  id (FK on InvoiceLineItem.invoiceId)
    ▼
InvoiceLineItem  ◄──────── raw_ocr_text (immutable, original OCR output)
    │  id (FK on ReviewQueueItem.invoiceLineItemId)
    │  id (FK on IngredientPriceHistory.invoiceLineItemId)
    ▼
ReviewQueueItem
    │  approvedByUserId / editedByUserId / rejectedByUserId
    │  approvedAt / editedAt / rejectedAt
    │  wasEdited (boolean)
    │  original vs. corrected values (field-level audit)
    ▼
IngredientPriceHistory  (APPROVED items only — REJECTED items produce no record)
    │  ingredientId → Ingredient
    │  supplierId → Supplier
    │  invoiceLineItemId → back to InvoiceLineItem (bidirectional audit link)
    │  recordedByUserId → User
    │  effective_date, committed_at (immutable timestamps)
```

**Every price change in the system is traceable back to exactly one `Invoice`, one original S3 image, and one human who approved it. No price can enter `IngredientPriceHistory` without passing through this chain.**

---

---

## Cross-Flow Shared Mechanism Reference

| Mechanism | Defined in | Referenced by |
|---|---|---|
| Auto-cascade: `ICostCascadeService.RecalculateForIngredient(ingredientId)` — outside DB transaction | Flow 3, Step 3.6 | Ingredient Management flow (manual price edit); Recipe Costing flow (manual recalculate) |
| Canonical cost formula: `SUM((current_price / unit_size) * quantity * (1 / yield_percentage)) / portion_count` | Flow 3, Step 3.6 | Canonical formula — identical across all calling flows; enforced inside `ICostCascadeService` |
| `IngredientPriceHistory` append-only invariant | Flow 3, Step 3.3 | Ingredient Management flow (any price write); enforced system-wide |
| Telegram threshold alert | Flow 3, Steps 3.7–3.8 | Ingredient Management flow (manual price exceeds threshold) |
| S3 pre-signed URL generation (read-only, TTL-bound) | Flow 4, Step 4.7 | Any future feature accessing stored documents |

---

*End of document. Next flow for review: Ingredient Management and Recipe Costing Engine flows.*
