# Business Process Flow: Ingredient Management & Price History

**Document version:** 1.1 (Revised — defects 2-A, 2-B, 2-C corrected)  
**Date:** 2026-04-02  
**Status:** Revised — defects 2-A, 2-B, 2-C corrected; resubmitted for lead architect review  
**Scope:** Covers four flows. The Auto-Cascade Mechanism defined in Flow 2 is the canonical shared mechanism referenced by the Recipe Costing flow and the Invoice Scanning flow.

---

## Canonical Entity Names Used

`Company` · `Outlet` · `User` · `Role` · `TelegramLink` · `Ingredient` · `IngredientPriceHistory` · `Unit` · `Category` · `Recipe` · `RecipeItem` · `Invoice` · `InvoiceLineItem` · `ReviewQueueItem` · `Supplier`

---

## Shared Data Model Assumptions

These assumptions apply across all four flows and must be consistent with dependent flow documents.

### `Ingredient` table (key columns)
| Column | Type | Notes |
|---|---|---|
| `Id` | UUID PK | |
| `OutletId` | UUID FK → Outlet | Ingredient belongs to one outlet only |
| `Name` | varchar NOT NULL | Unique constraint: (OutletId, Name) |
| `CategoryId` | UUID FK → Category | |
| `UnitId` | UUID FK → Unit | Base unit for price (e.g., kg, litre, each) |
| `PriceCeiling` | decimal(18,4) nullable | Alert threshold; null = no ceiling set |
| `unit_size` | decimal(18,4) NOT NULL DEFAULT 1 | Pack/purchase size in the ingredient's base unit (e.g., `5.0` = a 5 kg bag); divides into `Price` to yield cost-per-base-unit in the cost formula |
| `CreatedByUserId` | UUID FK → User | |
| `CreatedAt` | timestamptz | |
| `UpdatedAt` | timestamptz | |

> **No `CurrentPrice` column on `Ingredient`.** Current price is always the most recent `IngredientPriceHistory` record for that ingredient ordered by `committed_at DESC`. This avoids dual-write inconsistency. For query performance, a PostgreSQL index on `(ingredient_id, committed_at DESC)` satisfies fast latest-price lookups (C-3).

### `IngredientPriceHistory` table (key columns)
| Column | Type | Notes |
|---|---|---|
| `Id` | UUID PK | |
| `IngredientId` | UUID FK → Ingredient NOT NULL | |
| `SupplierId` | UUID FK → Supplier nullable | Null if supplier unknown |
| `Price` | decimal(18,4) NOT NULL | Purchase price for `Ingredient.unit_size` units |
| `committed_at` | timestamptz NOT NULL | System timestamp of row insertion; used for ordering (C-3, C-4) |
| `effective_date` | date NOT NULL | User-editable date when this price took effect; defaults to UTC today (C-4) |
| `RecordedByUserId` | UUID FK → User NOT NULL | Human who triggered the record |
| `Source` | varchar NOT NULL | Exactly `'Manual'` or `'InvoiceScan'` — no other values (C-13) |
| `InvoiceLineItemId` | UUID FK → InvoiceLineItem nullable | Set only when Source = InvoiceScan |

> **Records are immutable.** No UPDATE is ever issued on `IngredientPriceHistory`. Every price change appends a new row.

---

## Auto-Cascade Mechanism (Canonical — Shared Across All Flows)

> This section defines the single authoritative cascade algorithm. The Recipe Costing flow and the Invoice Scanning flow MUST reference this definition verbatim — do not redefine it elsewhere.

**Label:** `ICostCascadeService` (method: `RecalculateForIngredient(ingredientId)`)  
**Trigger:** Any insertion of a new `IngredientPriceHistory` record, regardless of source.  
**Location:** .NET 10 Web API, synchronous service call within the same HTTP request lifecycle.

### Algorithm

```
INPUT: ingredientId (UUID)   -- no price parameter; service fetches current price internally (C-3)

STEP A — Find affected RecipeItems:
  SELECT ri.*
  FROM RecipeItem ri
  JOIN Recipe r ON r.Id = ri.RecipeId
  WHERE ri.IngredientId = @ingredientId
    AND r.OutletId = (SELECT OutletId FROM Ingredient WHERE Id = @ingredientId)

STEP B — Collect unique RecipeIds from results.
  If empty → log "0 recipes affected, cascade skipped" → EXIT.

STEP C — For each RecipeId, recalculate cost_per_portion:
  a. Fetch ALL RecipeItems for that Recipe (include Ingredient.unit_size for each).
  b. For EVERY RecipeItem, fetch current price via C-3:
       linePrice = (
         SELECT price FROM IngredientPriceHistory
         WHERE ingredient_id = RecipeItem.IngredientId
         ORDER BY committed_at DESC LIMIT 1
       )
       -- returns 0.00 if no history exists; logged as a warning
  c. lineCost = (linePrice / ingredient.unit_size) * RecipeItem.quantity * (1.0 / RecipeItem.yield_percentage)
       -- unit_size: decimal from Ingredient.unit_size (e.g. 5.0 for a 5 kg bag)
       -- yield_percentage: decimal 0 < yp ≤ 1.0; stored on RecipeItem; defaults to 1.0
  d. cost_per_portion = SUM(all lineCosts) / Recipe.portion_count
       -- canonical formula (C-2): SUM((price / unit_size) * quantity * (1 / yield_percentage)) / portion_count

STEP D — Persist:
  UPDATE Recipe
  SET cost_per_portion = @calculated,
      last_cost_updated_at = NOW()
  WHERE Id = @RecipeId

STEP E — Repeat STEP C–D for all affected RecipeIds.

OUTPUT: affectedRecipeCount (int), list of {RecipeId, new_cost_per_portion}
```

**Error handling within cascade:**
- One recipe's recalculation failing does not rollback the price insert.
- Each recipe failure is logged with RecipeId, IngredientId, error message.
- Partial success is returned to the caller (`affectedRecipeCount` = successfully updated count only).
- The price insert itself is committed regardless of cascade outcome.

---

## Flow 1: Manual Ingredient Creation

**Actors:** User (Owner | Chef | Procurement), .NET 10 API, PostgreSQL, Vue.js  
**Trigger:** Authenticated user opens the Add Ingredient form in the web app.  
**End state:** `Ingredient` record persisted; optional initial `IngredientPriceHistory` record persisted; ingredient available for use in `RecipeItem` creation.

---

### Step 1 — Load Reference Data for Form

| Field | Value |
|---|---|
| **Actor** | User (Owner / Chef / Procurement) |
| **Action** | Navigate to Ingredients page; Vue.js loads dropdown data required to populate the creation form |
| **Component** | Vue.js frontend → .NET 10 API → PostgreSQL |
| **Data IN** | JWT Bearer token; `outletId` from authenticated user's session context |
| **Data OUT** | `GET /api/outlets/{outletId}/categories` → `Category[]`<br>`GET /api/units` → `Unit[]`<br>`GET /api/suppliers?outletId={outletId}` → `Supplier[]` |
| **Decision** | **Yes.** If caller's role is `Viewer`: "Add Ingredient" button is hidden; form route is inaccessible (route guard). Flow terminates. |
| **Error** | JWT expired or invalid → Vue.js route guard intercepts → redirect to `/login`.<br>Reference data fetch fails → form renders with empty dropdowns + inline error; user cannot submit. |

---

### Step 2 — User Completes and Submits Creation Form

| Field | Value |
|---|---|
| **Actor** | User (Owner / Chef / Procurement) |
| **Action** | Fill ingredient name; select `Category`; select `Unit`; enter initial price (required, must be > 0); optionally select `Supplier`; optionally set `PriceCeiling`; click "Create Ingredient" |
| **Component** | Vue.js frontend (IngredientCreateForm component) |
| **Data IN** | Form fields: `name` (string), `categoryId` (UUID), `unitId` (UUID), `initialPrice` (decimal > 0), `supplierId` (UUID, optional), `priceCeiling` (decimal > 0, optional) |
| **Data OUT** | HTTP `POST /api/outlets/{outletId}/ingredients` with JSON body |
| **Decision** | **Yes.** Client-side validation runs before HTTP call:<br>• `name` is empty → inline error, no request sent<br>• `initialPrice` ≤ 0 or non-numeric → inline error, no request sent<br>• `priceCeiling` set but ≤ `initialPrice` → show advisory warning (not a block) |
| **Error** | No HTTP call is made while validation errors exist. |

---

### Step 3 — API Authentication and Authorization

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientController) |
| **Action** | Parse request; validate JWT signature and expiry; extract `UserId`, `OutletId`, `Role` from JWT claims; verify `OutletId` in claim matches route `{outletId}`; verify `Role` ∈ { Owner, Chef, Procurement } |
| **Component** | .NET 10 Web API — Auth middleware → IngredientController |
| **Data IN** | `Authorization: Bearer <token>`, route `{outletId}`, request body |
| **Data OUT** | Validated `ClaimsPrincipal`; authorized request proceeds to validation |
| **Decision** | **Yes.**<br>• Token invalid / expired → `401 Unauthorized`<br>• Role is `Viewer` → `403 Forbidden`<br>• `OutletId` claim ≠ route `{outletId}` → `403 Forbidden` |
| **Error** | `401` / `403` returned; no DB interaction occurs. |

---

### Step 4 — API Business Validation

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientService) |
| **Action** | Validate that `categoryId`, `unitId`, and (if provided) `supplierId` exist in PostgreSQL and belong to the caller's outlet or are global records. Check uniqueness: no `Ingredient` with the same `Name` already exists for this `OutletId`. |
| **Component** | .NET 10 Web API (IngredientService) → PostgreSQL |
| **Data IN** | `{outletId, name, categoryId, unitId, supplierId?, initialPrice, priceCeiling?}` |
| **Data OUT** | Validation result; proceeds or returns error |
| **Decision** | **Yes.**<br>• `categoryId` not found → `422 Unprocessable Entity`, message: "Category not found"<br>• `unitId` not found → `422`, message: "Unit not found"<br>• `supplierId` provided but not found → `422`, message: "Supplier not found"<br>• `(OutletId, Name)` already exists → `409 Conflict`, message: "Ingredient name already exists in this outlet" |
| **Error** | Validation failures return structured error response `{errorCode, field, message}`. No DB writes occur. |

---

### Step 5 — Persist `Ingredient` and Initial `IngredientPriceHistory` (Single Transaction)

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientRepository, IngredientPriceHistoryRepository) |
| **Action** | Open a database transaction. INSERT `Ingredient` record. INSERT `IngredientPriceHistory` record for the initial price. Commit. |
| **Component** | .NET 10 Web API → PostgreSQL |
| **Data IN** | **Ingredient:** `{Id = newUUID(), OutletId, Name, CategoryId, UnitId, PriceCeiling, CreatedByUserId, CreatedAt = UTC.Now, UpdatedAt = UTC.Now}`<br>**IngredientPriceHistory:** `{Id = newUUID(), IngredientId = new Ingredient.Id, SupplierId, Price = initialPrice, committed_at = UTC.Now, effective_date = UTC.Today, RecordedByUserId, Source = 'Manual', InvoiceLineItemId = null}` |
| **Data OUT** | Persisted `Ingredient.Id`, persisted `IngredientPriceHistory.Id`; transaction committed |
| **Decision** | **Yes.** `initialPrice` = 0 or omitted: skip `IngredientPriceHistory` insert. Ingredient is created with no price history. It can be used in recipes but will contribute 0.00 to cost-per-portion until a price is recorded. |
| **Error** | Any DB constraint violation (unique, FK) within transaction → full rollback → `409` or `422` returned to caller.<br>DB connection failure → `500 Internal Server Error`; transaction never committed. |

---

### Step 6 — API Returns Created Resource

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientController) |
| **Action** | Compose response DTO; return `HTTP 201 Created` with `Location` header pointing to the new ingredient |
| **Component** | .NET 10 Web API |
| **Data IN** | Persisted `Ingredient` + `IngredientPriceHistory` records |
| **Data OUT** | `HTTP 201 Created`, `Location: /api/outlets/{outletId}/ingredients/{ingredientId}`<br>Body: `{id, name, outletId, category: {id, name}, unit: {id, name, abbreviation}, priceCeiling, currentPrice: initialPrice \| null, supplier: {id, name} \| null, createdAt}` |
| **Decision** | No |
| **Error** | Serialisation failure (extremely rare) → `500`; ingredient is persisted but response fails; client can recover by re-querying. |

---

### Step 7 — Vue.js Updates UI State

| Field | Value |
|---|---|
| **Actor** | System (Vue.js) |
| **Action** | Append new ingredient to Pinia store ingredient list; close creation modal; display success toast "Ingredient created"; if the ingredient list is paginated, navigate to the page containing the new entry |
| **Component** | Vue.js frontend (Pinia IngredientStore, IngredientList component) |
| **Data IN** | `HTTP 201` response body |
| **Data OUT** | Updated ingredient list in UI; new ingredient immediately available in RecipeItem ingredient picker |
| **Decision** | No |
| **Error** | `HTTP 4xx/5xx` → display error toast with server message; no Pinia state change; form remains open with entered data preserved. |

---

## Flow 2: Manual Price Update

**Actors:** User (Owner | Chef | Procurement), .NET 10 API, PostgreSQL, Vue.js  
**Trigger:** Authenticated user edits an ingredient's price in the web app.  
**End state:** New `IngredientPriceHistory` record persisted; all `Recipe.CostPerPortion` values referencing this ingredient are recalculated via the **Auto-Cascade Mechanism**; price ceiling checked as a side effect.

> The cascade steps here (Steps 5–6) are identical in structure to the cascade triggered by the Invoice Scanning flow after an `InvoiceLineItem` price is confirmed. Both invoke `ICostCascadeService.RecalculateForIngredient(ingredientId)` — no price parameter; the service fetches current price internally via C-3.

---

### Step 1 — User Opens Price Edit on Ingredient

| Field | Value |
|---|---|
| **Actor** | User (Owner / Chef / Procurement) |
| **Action** | Navigate to Ingredient detail page; click "Update Price" button |
| **Component** | Vue.js frontend (IngredientDetailView) |
| **Data IN** | `ingredientId` from route params; current displayed price from Pinia state or from `GET /api/outlets/{outletId}/ingredients/{ingredientId}` |
| **Data OUT** | Price update form displayed with current price pre-filled |
| **Decision** | **Yes.** Role is `Viewer` → "Update Price" button not rendered; direct URL access to edit form redirected by route guard. |
| **Error** | JWT expired → redirect to `/login`. |

---

### Step 2 — User Submits New Price

| Field | Value |
|---|---|
| **Actor** | User (Owner / Chef / Procurement) |
| **Action** | Enter new price value; optionally select/change `Supplier`; optionally override recorded date (defaults to today); submit |
| **Component** | Vue.js frontend (PriceUpdateForm component) |
| **Data IN** | `newPrice` (decimal > 0), `supplierId` (UUID, optional), `effectiveDate` (ISO date, optional — user-editable price effective date; defaults to `UTC.Today` server-side) |
| **Data OUT** | HTTP `POST /api/ingredients/{ingredientId}/prices` with JSON body |
| **Decision** | **Yes.** `newPrice` ≤ 0 or non-numeric → inline validation error, no HTTP call. |
| **Error** | Network failure on submit → Vue.js shows "Request failed, please try again"; no state change. |

---

### Step 3 — API Authentication, Authorization, and Resource Verification

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientPriceController) |
| **Action** | Validate JWT; extract `UserId`, `OutletId`, `Role`; verify role ∈ { Owner, Chef, Procurement }; load `Ingredient` by `ingredientId`; verify `Ingredient.OutletId` == JWT `OutletId` |
| **Component** | .NET 10 Web API — Auth middleware → IngredientPriceController → IngredientRepository |
| **Data IN** | JWT, route `{ingredientId}`, body `{newPrice, supplierId?, effectiveDate?}` |
| **Data OUT** | Loaded `Ingredient` entity (including `PriceCeiling`, `OutletId`) |
| **Decision** | **Yes.**<br>• Token invalid → `401`<br>• Role = Viewer → `403`<br>• `ingredientId` not found → `404`<br>• `Ingredient.OutletId` ≠ JWT `OutletId` → `404` (do not leak existence of other outlets' ingredients) |
| **Error** | Auth/resource errors return before any write. |

---

### Step 4 — Insert New `IngredientPriceHistory` Record

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientPriceHistoryRepository) |
| **Action** | INSERT a new `IngredientPriceHistory` row. **Never UPDATE an existing row.** Previous price history is immutable. |
| **Component** | .NET 10 Web API → PostgreSQL |
| **Data IN** | `{Id = newUUID(), IngredientId, SupplierId, Price = newPrice, committed_at = UTC.Now, effective_date = requestedDate?.Date ?? UTC.Today, RecordedByUserId = JWT.UserId, Source = 'Manual', InvoiceLineItemId = null}` |
| **Data OUT** | Persisted `IngredientPriceHistory` record with generated `Id` |
| **Decision** | No. Append-only; no duplicate check needed (multiple prices on the same date are valid — e.g., two suppliers quote same ingredient same day). |
| **Error** | FK violation (`IngredientId` or `SupplierId` not found) → `422`.<br>DB connection failure → `500`; no price persisted, cascade not triggered. |

---

### Step 5 — Price Ceiling Check (Non-Blocking Side Effect)

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (PriceCeilingService) |
| **Action** | Compare `newPrice` against `Ingredient.PriceCeiling`. If exceeded, dispatch Price Ceiling Alert (Flow 3). This runs **after** the price insert is committed and **does not block** the cascade. |
| **Component** | .NET 10 Web API (PriceCeilingService) |
| **Data IN** | `Ingredient.PriceCeiling` (from entity loaded in Step 3), `newPrice`, `IngredientId`, `source = 'Manual'`, `recordedByUserId` |
| **Data OUT** | If `PriceCeiling` is set AND `newPrice > PriceCeiling`: invoke Flow 3 asynchronously (fire-and-forget with internal retry). |
| **Decision** | **Yes.**<br>• `PriceCeiling` is null → skip; no alert.<br>• `newPrice` ≤ `PriceCeiling` → skip; no alert.<br>• `newPrice` > `PriceCeiling` → invoke Flow 3. |
| **Error** | Alert dispatch failure is caught, logged, and does not affect the response to the client. Price is already persisted before this step. |

---

### Step 6 — Trigger Auto-Cascade (`ICostCascadeService.RecalculateForIngredient`)

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (`ICostCascadeService`) |
| **Action** | Call `ICostCascadeService.RecalculateForIngredient(ingredientId)`. The service executes the **Auto-Cascade Mechanism** as defined in the canonical algorithm above, fetching the current price for each ingredient internally via C-3. Finds all `RecipeItem` records referencing this `IngredientId`. Recalculates `cost_per_portion` for each affected `Recipe` using formula C-2. Persists updates. |
| **Component** | .NET 10 Web API (`ICostCascadeService` implementation) → PostgreSQL |
| **Data IN** | `ingredientId` (UUID) — no price parameter |
| **Data OUT** | `affectedRecipeCount` (int); each affected `Recipe.cost_per_portion` and `Recipe.last_cost_updated_at` updated in PostgreSQL |
| **Decision** | **Yes.** If `affectedRecipeCount` = 0 → log informational "No recipes reference ingredient {IngredientId}; cascade skipped." |
| **Error** | Per-recipe cascade error → log `{RecipeId, error}` and continue; partial success; do not rollback price insert. Final count reflects only successfully updated recipes. |

---

### Step 7 — API Returns Response

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientPriceController) |
| **Action** | Return `HTTP 200 OK` with price confirmation and cascade summary |
| **Component** | .NET 10 Web API |
| **Data IN** | Persisted `IngredientPriceHistory`, `affectedRecipeCount`, `ceilingExceeded` boolean |
| **Data OUT** | `HTTP 200` body: `{ingredientId, newPrice, effectiveDate, supplierId, affectedRecipeCount, ceilingExceeded: bool}` |
| **Decision** | No |
| **Error** | Cascade partial failure is reflected in `affectedRecipeCount` but does not change HTTP status code; errors are logged server-side. |

---

### Step 8 — Vue.js Updates UI

| Field | Value |
|---|---|
| **Actor** | System (Vue.js) |
| **Action** | Update displayed price on IngredientDetailView; show toast: "Price updated — {N} recipe(s) recalculated"; if `ceilingExceeded = true`, display inline warning badge on ingredient card: "⚠ Price exceeds ceiling" |
| **Component** | Vue.js frontend (Pinia IngredientStore, IngredientDetailView, RecipeListView) |
| **Data IN** | `HTTP 200` response body |
| **Data OUT** | Refreshed ingredient price in UI; recipe cost values on RecipeListView stale until user navigates or refreshes (recipes are not pushed via WebSocket in Phase 1–2; polling or navigation refresh is acceptable) |
| **Decision** | No |
| **Error** | `HTTP 4xx/5xx` → toast error with server message; no Pinia state mutation; form remains open with entered values. |

---

## Flow 3: Price Ceiling Alert

**Actors:** .NET 10 API, Python FastAPI, Telegram Bot API, PostgreSQL  
**Trigger:** `PriceCeilingService` is invoked after any `IngredientPriceHistory` insert (from manual update OR invoice scan) where `newPrice > Ingredient.PriceCeiling` and `PriceCeiling IS NOT NULL`.  
**End state:** Telegram message delivered to all Outlet-linked Owner and Procurement users.

> This flow is always invoked as a **fire-and-forget side effect** from Flow 2 (Step 5) or the Invoice Scanning flow. It does not block or affect the outcome of the triggering flow.

---

### Step 1 — Ceiling Threshold Evaluation

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (PriceCeilingService) |
| **Action** | Receive trigger payload; evaluate whether a ceiling breach exists |
| **Component** | .NET 10 Web API (PriceCeilingService) |
| **Data IN** | `{ingredientId, ingredientName, outletId, outletName, newPrice, previousPrice (latest IngredientPriceHistory before this one), priceCeiling, source: 'Manual'|'InvoiceScan', recordedByUserId, recordedByUserName, committed_at, invoiceRef (string|null — only when source = InvoiceScan)}` |
| **Data OUT** | `PriceCeilingBreachEvent` object ready for dispatch |
| **Decision** | **Yes.**<br>• `PriceCeiling` is null → EXIT; log "No ceiling configured for ingredient {id}".<br>• `newPrice` ≤ `PriceCeiling` → EXIT; no alert.<br>• `newPrice` > `PriceCeiling` → continue to Step 2. |
| **Error** | DB read failure when fetching previous price → use `null` for `previousPrice` in notification; continue. |

---

### Step 2 — Identify Alert Recipients via `TelegramLink`

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (PriceCeilingService) |
| **Action** | Query `TelegramLink` for all users linked to this outlet who hold the Owner or Procurement role |
| **Component** | .NET 10 Web API → PostgreSQL |
| **Query** | `SELECT tl.telegramUserId, u.Name FROM TelegramLink tl JOIN User u ON u.Id = tl.UserId WHERE u.OutletId = @outletId AND u.Role IN ('Owner', 'Procurement') AND tl.status = 'confirmed'` |
| **Data IN** | `outletId` |
| **Data OUT** | `List<{telegramUserId, UserName}>` |
| **Decision** | **Yes.** If list is empty → log "No active TelegramLink recipients for outlet {outletId}; alert suppressed." → EXIT. |
| **Error** | DB failure → log warning; EXIT without dispatching. Price insert is already committed. |

---

### Step 3 — .NET 10 API Calls FastAPI Notification Endpoint

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (NotificationService) |
| **Action** | HTTP `POST` to FastAPI internal endpoint `/internal/notify/price-ceiling`. This is a service-to-service call on the internal Docker network; not exposed publicly. Retry: up to 3 attempts with exponential backoff (1s, 2s, 4s). |
| **Component** | .NET 10 Web API (NotificationService) → Python FastAPI |
| **Data IN** | Request body: `{ingredientName, outletName, newPrice, previousPrice|null, priceCeiling, percentageOver: ((newPrice - priceCeiling) / priceCeiling * 100), source, recordedByUserName, committed_at, invoiceRef|null, recipients: [{telegramUserId, userName}]}` |
| **Data OUT** | `HTTP 202 Accepted` from FastAPI; delivery is async from FastAPI's side |
| **Decision** | **Yes.** FastAPI not reachable after 3 retries → log error `{ingredientId, outletId, newPrice, timestamp, error}`; raise internal alert (log to monitoring); return to caller without error. |
| **Error** | Network timeout → retry as above. `HTTP 5xx` from FastAPI → retry. `HTTP 4xx` from FastAPI → log as misconfiguration; do not retry; alert operations team. |

---

### Step 4 — FastAPI Formats and Dispatches Telegram Messages

| Field | Value |
|---|---|
| **Actor** | Python FastAPI (TelegramNotificationService) |
| **Action** | Format one Telegram message per breach event (same message sent to all recipients). Call Telegram Bot API `sendMessage` for each `telegramUserId` in the recipients list. |
| **Component** | Python FastAPI → Telegram Bot API |
| **Data IN** | Payload from .NET API (Step 3) |
| **Data OUT** | Telegram messages delivered to each `TelegramLink.telegramUserId` |
| **Message format** | `⚠️ Price Ceiling Breach — {outletName}`<br>`Ingredient: {ingredientName}`<br>`New Price: ${newPrice}/{unit} (+{percentageOver}% above ceiling of ${priceCeiling})`<br>`Previous Price: ${previousPrice}` (omitted if null)<br>`Source: {Manual / Invoice Scan (Ref: {invoiceRef})}`<br>`Recorded by: {recordedByUserName}`<br>`Time: {committed_at}` |
| **Decision** | **Yes (per recipient).** If `sendMessage` returns an error for a specific `telegramUserId`:<br>• `400 Bad Request` (invalid `chat_id`) → `PATCH /api/telegram-links/{id}/deactivate` → sets `TelegramLink.status = 'unlinked'`; log.<br>• `403 Forbidden` (bot blocked by user) → `PATCH /api/telegram-links/{id}/deactivate` → sets `TelegramLink.status = 'unlinked'`; log.<br>• `429 Too Many Requests` → apply Telegram's `retry_after` delay; retry once.<br>• Other error → log `{telegramUserId, error}`; continue with remaining recipients. |
| **Error** | Telegram API globally unreachable → log all failed `telegramUserIds`; FastAPI returns `HTTP 207 Multi-Status` to .NET API with per-recipient statuses. |

---

### Step 5 — FastAPI Returns Delivery Report to .NET API

| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Return delivery result summary |
| **Component** | Python FastAPI → .NET 10 API |
| **Data IN** | Per-recipient send results |
| **Data OUT** | `HTTP 202` body: `{dispatched: N, failed: M, failedTelegramUserIds: []}` |
| **Decision** | No. Delivery result is logged by .NET API. Partial delivery (some sent, some failed) is not an error condition for the triggering price update. |
| **Error** | N/A — response is informational. |

---

## Flow 4: Supplier Price History & Comparison

**Actors:** User (any authenticated role), .NET 10 API, PostgreSQL, Vue.js  
**Trigger:** Authenticated user opens the Price History tab on an Ingredient detail page.  
**End state:** Vue.js renders a chronological price history list and a per-supplier comparison table, with price ceiling reference line shown if configured.

---

### Step 1 — User Navigates to Price History View

| Field | Value |
|---|---|
| **Actor** | User (any role: Owner / Chef / Procurement / Viewer) |
| **Action** | Open Ingredient detail page; select "Price History" tab; optionally apply filters (supplier, date range) |
| **Component** | Vue.js frontend (IngredientDetailView, PriceHistoryTab component) |
| **Data IN** | `ingredientId` (from route), optional filter state: `supplierId`, `dateFrom` (ISO date), `dateTo` (ISO date) |
| **Data OUT** | `GET /api/ingredients/{ingredientId}/price-history?supplierId={}&from={}&to={}` |
| **Decision** | **Yes.** All roles can view; no role gate on this endpoint. |
| **Error** | JWT expired → redirect to `/login`. |

---

### Step 2 — API Authentication and Ownership Verification

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientPriceHistoryController) |
| **Action** | Validate JWT; load `Ingredient` by `ingredientId`; verify `Ingredient.OutletId` == JWT `OutletId` |
| **Component** | .NET 10 Web API (Auth middleware → IngredientPriceHistoryController → IngredientRepository) |
| **Data IN** | JWT, route `{ingredientId}`, query params `{supplierId?, from?, to?}` |
| **Data OUT** | Validated `Ingredient` entity (name, unitId, priceCeiling) |
| **Decision** | **Yes.**<br>• Token invalid → `401`<br>• `ingredientId` not found OR `OutletId` mismatch → `404` |
| **Error** | `401`/`404` short-circuit; no DB queries for history data. |

---

### Step 3 — Query `IngredientPriceHistory` with Optional Filters

| Field | Value |
|---|---|
| **Actor** | .NET 10 API → PostgreSQL |
| **Action** | Execute parameterized query for price history records |
| **Component** | .NET 10 Web API (IngredientPriceHistoryRepository) → PostgreSQL |
| **SQL (conceptual)** | `SELECT iph.Id, iph.Price, iph.committed_at, iph.effective_date, iph.Source, iph.InvoiceLineItemId, s.Id AS SupplierId, s.Name AS SupplierName, u.Name AS RecordedByUserName FROM IngredientPriceHistory iph LEFT JOIN Supplier s ON s.Id = iph.SupplierId LEFT JOIN "User" u ON u.Id = iph.RecordedByUserId WHERE iph.IngredientId = @id AND (@supplierId IS NULL OR iph.SupplierId = @supplierId) AND (@from IS NULL OR iph.effective_date >= @from) AND (@to IS NULL OR iph.effective_date <= @to) ORDER BY iph.committed_at DESC` |
| **Data IN** | `@id`, `@supplierId`, `@from`, `@to` (all parameterized — no string interpolation) |
| **Data OUT** | `List<IngredientPriceHistoryRecord>` ordered newest-first |
| **Decision** | No filter errors; returns empty list if no records match — not a 404. |
| **Error** | DB failure → `500`. |

---

### Step 4 — Compute Per-Supplier Aggregates

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientPriceHistoryService) |
| **Action** | Group the query result set by `SupplierId` (including null as "Unknown Supplier"). For each group, compute the comparison statistics. |
| **Component** | .NET 10 Web API — in-memory LINQ aggregation over result set |
| **Data IN** | `List<IngredientPriceHistoryRecord>` from Step 3 |
| **Data OUT** | `List<SupplierPriceSummary>`: `{supplierId, supplierName, latestPrice, latestCommittedAt, latestEffectiveDate, recordCount, minPrice, maxPrice, avgPrice, priceChangePct}` where `priceChangePct` = `(latestPrice - oldestPriceInPeriod) / oldestPriceInPeriod * 100` (null if only one record) |
| **Decision** | **Yes.** Only one supplier in result set → comparison table has one row; no functionality broken; UI renders single-supplier view. |
| **Error** | Division by zero in `priceChangePct` when `oldestPrice = 0` → return `null` for that field; do not crash. |

---

### Step 5 — API Returns Composite Response

| Field | Value |
|---|---|
| **Actor** | .NET 10 API (IngredientPriceHistoryController) |
| **Action** | Compose and return combined response object |
| **Component** | .NET 10 Web API |
| **Data IN** | `Ingredient` entity, `List<IngredientPriceHistoryRecord>`, `List<SupplierPriceSummary>` |
| **Data OUT** | `HTTP 200 OK` |
| **Response body** | ```json { "ingredient": { "id": "...", "name": "Beef Tenderloin", "unit": {"abbreviation": "kg"}, "priceCeiling": 40.00 }, "history": [ { "id": "...", "price": 45.00, "supplierId": "...", "supplierName": "Metro Cash & Carry", "committed_at": "2026-03-28T09:14:00Z", "effective_date": "2026-03-28", "source": "InvoiceScan", "invoiceLineItemId": "...", "recordedByUserName": "Jane" } ], "supplierComparison": [ { "supplierId": "...", "supplierName": "Metro Cash & Carry", "latestPrice": 45.00, "latestCommittedAt": "2026-03-28T09:14:00Z", "latestEffectiveDate": "2026-03-28", "recordCount": 12, "minPrice": 36.00, "maxPrice": 45.00, "avgPrice": 39.80, "priceChangePct": 12.5 } ] }``` |
| **Decision** | No |
| **Error** | Empty `history` list is valid; `supplierComparison` will also be empty. UI renders empty state. |

---

### Step 6 — Vue.js Renders Price History and Supplier Comparison

| Field | Value |
|---|---|
| **Actor** | System (Vue.js) |
| **Action** | Render two sub-components side by side (or tabbed on mobile): (a) chronological history table/chart; (b) supplier comparison table. Apply visual decorators for ceiling breaches. |
| **Component** | Vue.js frontend (PriceHistoryChart, SupplierComparisonTable components) |
| **Data IN** | `HTTP 200` response body |
| **Rendered outputs** | **(a) Price History Chart/Table:** Line chart with one series per supplier (color-coded); X axis = `committed_at`, Y axis = `Price`; each data point is hoverable with tooltip showing source, user, effective_date.<br>**(b) Supplier Comparison Table:** Columns: Supplier Name, Latest Price, Avg Price, Min, Max, Price Change %, Last Updated. Sorted by `latestPrice` ascending (cheapest first) by default. |
| **Decision** | **Yes.**<br>• `Ingredient.PriceCeiling` is set → render a horizontal dashed reference line at `PriceCeiling` on the chart.<br>• For each supplier in comparison table where `latestPrice > priceCeiling` → show ⚠ icon in that row.<br>• `InvoiceLineItemId` is set on a history record → show a clickable "Invoice" link in the history table (navigates to that invoice's detail view). |
| **Error** | API error → render empty state with "Unable to load price history; please try again." No partial render. |

---

## Cross-Flow Consistency Notes

These constraints must be enforced across all flow documents in this system:

| Constraint | Details |
|---|---|
| **Auto-Cascade entry point** | Always `ICostCascadeService.RecalculateForIngredient(ingredientId)` — no price parameter; service fetches current price internally via C-3. Invoked once after any `IngredientPriceHistory` insert, regardless of source. |
| **Price history immutability** | No flow ever UPDATEs an `IngredientPriceHistory` row. Invoice Scanning flow must also follow append-only. |
| **Outlet data isolation** | All ingredient reads and writes scope to `Ingredient.OutletId = JWT.OutletId`. Cross-outlet ingredient access is never permitted. |
| **Role gate consistency** | Viewers cannot create, update, or upload. Invoice scanning (Flow in Invoice doc) is Owner and Procurement only, same restriction as manual price update. |
| **Price ceiling trigger** | `PriceCeilingService` is called identically from both the Manual Price Update flow and the Invoice Scanning flow. The check is: `newPrice > PriceCeiling AND PriceCeiling IS NOT NULL`. |
| **`TelegramLink` recipient logic** | Price ceiling alerts → Owner + Procurement only. Other alert types may differ; document in the relevant flow. |
| **`Source` field values** | Exactly `'Manual'` for all user-entered prices. Exactly `'InvoiceScan'` for all prices originating from a confirmed `InvoiceLineItem`. No other values (C-13). |
