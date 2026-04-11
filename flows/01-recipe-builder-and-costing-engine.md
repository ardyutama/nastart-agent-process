# Business Process Flows — Recipe Builder & Cost-Per-Portion Calculation Engine

**Application:** Recipe Cost Management SaaS  
**Document type:** Implementation-ready process flow  
**Review status:** Draft — for lead architect review  
**Last updated:** 2026-04-02

---

## Architecture Reference

| Service | Technology | Responsibility |
|---|---|---|
| .NET 10 API | C#, .NET 10 Web API | Business logic, auth, recipes, ingredients, outlets, cascade engine |
| Python FastAPI | Python, FastAPI | OCR, LLM, Telegram bot, alert delivery |
| Vue.js | Vite + TypeScript | Frontend SPA |
| PostgreSQL | PostgreSQL | Persistent data store |
| Telegram Bot API | External | Push notifications |

**Auth:** JWT tokens — `user_id`, `role`, `outlet_id` embedded in every API request.  
**Roles:** Owner, Chef, Procurement, Viewer  
**Visibility rules (system-wide):**
- Owner: sees `cost_per_portion`, `food_cost_pct`, `gross_margin`
- Chef, Procurement, Viewer: sees `cost_per_portion` only — `food_cost_pct` and `gross_margin` are **always** stripped from API responses for these roles

---

## Canonical Entity Names

`Company` · `Outlet` · `User` · `Role` · `TelegramLink` · `Ingredient` · `IngredientPriceHistory` · `Unit` · `Category` · `Recipe` · `RecipeItem` · `Invoice` · `InvoiceLineItem` · `ReviewQueueItem` · `Supplier`

---

## Cost Formula (canonical — do not vary)

```
cost_per_portion = SUM(
  (ingredient.current_price / ingredient.unit_size) * recipe_item.quantity * (1 / recipe_item.yield_percentage)
) / recipe.portion_count
```

Where `ingredient.current_price` = the `price` field of the **most-recent** `IngredientPriceHistory` record for that `ingredient_id`, ordered by `committed_at DESC`.

---

## Recipe Schema Fields Relevant to These Flows

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | PK |
| `outlet_id` | UUID | FK to Outlet — every recipe belongs to ONE outlet |
| `name` | varchar | Recipe display name |
| `version_group_id` | UUID | Groups all versions of the "same" recipe — assigned on first save |
| `version_number` | int | 1-based, auto-incremented within a version group |
| `version_label` | varchar | e.g., "Standard", "Premium — truffle oil" — required when version_number > 1 |
| `portion_count` | int | Number of portions this recipe yields |
| `sale_price` | decimal | Selling price per portion — used for food cost % |
| `cost_threshold_percentage` | decimal | Alert fires when food_cost_pct exceeds this |
| `cost_per_portion` | decimal | Calculated and stored — always refreshed by CostCalculationService |
| `cost_last_updated_at` | timestamptz | Set whenever cost_per_portion is recalculated |
| `has_incomplete_pricing` | bool | True if any RecipeItem's ingredient has no IngredientPriceHistory |
| `created_at` | timestamptz | Immutable |
| `created_by_user_id` | UUID | FK to User |

---
---

# Flow 1 — Creating a New Recipe

**Actors:** Owner or Chef  
**Starting point:** User is authenticated, has a valid JWT, and navigates to the Recipe Builder  
**End state:** Recipe persisted in PostgreSQL with a calculated `cost_per_portion`; recipe detail view displayed

---

### Step 1

| Field | Value |
|---|---|
| **Step** | 1 |
| **Actor** | Owner or Chef |
| **Action** | Navigate to Recipe Builder page |
| **System component** | Vue.js |
| **Data IN** | None (user is already authenticated; JWT present in session store) |
| **Data OUT** | Empty recipe form rendered; `outlet_id` pre-populated from JWT claims |
| **Decision point** | No |
| **Error case** | If JWT is expired → redirect to login |

---

### Step 2

| Field | Value |
|---|---|
| **Step** | 2 |
| **Actor** | Owner or Chef |
| **Action** | Fill in recipe metadata: `name`, `portion_count`, `sale_price`, `cost_threshold_percentage`, `version_label` (optional for first version) |
| **System component** | Vue.js (local form state — no API call yet) |
| **Data IN** | User keyboard input |
| **Data OUT** | Local form state: `{ name, portion_count, sale_price, cost_threshold_percentage }` |
| **Decision point** | No |
| **Error case** | Client-side validation: `name` required; `portion_count` must be integer > 0; `sale_price` must be decimal ≥ 0; `cost_threshold_percentage` must be 0–100 |

---

### Step 3

| Field | Value |
|---|---|
| **Step** | 3 |
| **Actor** | Owner or Chef |
| **Action** | Search for an ingredient to add to the recipe |
| **System component** | Vue.js → .NET 10 API |
| **Data IN** | `GET /api/outlets/{outlet_id}/ingredients?search={term}` — query string from search input |
| **Data OUT** | Array of `Ingredient` objects: `{ id, name, unit_id, unit_label, unit_size, current_price }` — `current_price` is the `price` from the latest `IngredientPriceHistory` record joined server-side |
| **Decision point** | YES — if result set is empty → Vue.js displays "No ingredients found. Create one first?" with link to Ingredient Management |
| **Error case** | Network failure → show retry toast; `outlet_id` in JWT does not match requested outlet → .NET API returns HTTP 403 |

---

### Step 4

| Field | Value |
|---|---|
| **Step** | 4 |
| **Actor** | Owner or Chef |
| **Action** | Select an ingredient and enter `quantity` and `yield_percentage` for this `RecipeItem` |
| **System component** | Vue.js (local state only) |
| **Data IN** | Selected `ingredient_id`; user-entered `quantity` (in the ingredient's unit); `yield_percentage` (0 < x ≤ 100, defaults to 100) |
| **Data OUT** | `RecipeItem` appended to local `items[]` array in form state |
| **Decision point** | YES — if `yield_percentage` not entered → default to 100. If `quantity` = 0 → block add |
| **Error case** | `quantity` must be > 0; `yield_percentage` must be between 0 (exclusive) and 100 (inclusive) — enforce with field-level validation before appending |

---

### Step 5

| Field | Value |
|---|---|
| **Step** | 5 |
| **Actor** | System (Vue.js reactive computation) |
| **Action** | Recalculate and display live cost preview after each RecipeItem add/edit/remove |
| **System component** | Vue.js (pure client-side, using `current_price` fetched in Step 3) |
| **Data IN** | `items[]` (each with `ingredient.current_price`, `ingredient.unit_size`, `item.quantity`, `item.yield_percentage`); `recipe.portion_count` |
| **Data OUT** | Live `estimated_cost_per_portion` displayed in the form header |
| **Formula** | `cost_per_portion = SUM((current_price / unit_size) * quantity * (1 / yield_percentage)) / portion_count` |
| **Decision point** | YES — if any ingredient in `items[]` has no `current_price` (null) → exclude that item from the live preview AND display a yellow warning badge: "Incomplete pricing — [Ingredient Name] has no price on record" |
| **Error case** | `portion_count` = 0 → display "—" instead of cost; prevent divide-by-zero silently |

---

### Step 6

| Field | Value |
|---|---|
| **Step** | 6 |
| **Actor** | Owner or Chef |
| **Action** | Click "Save Recipe" |
| **System component** | Vue.js |
| **Data IN** | Final form state |
| **Data OUT** | `POST /api/recipes` request body: `{ name, outlet_id, portion_count, sale_price, cost_threshold_percentage, version_label, items: [{ ingredient_id, quantity, yield_percentage }] }` |
| **Decision point** | YES — Vue.js runs final client-side validation before submitting. If any required field missing → block submit and highlight errors. If `items[]` is empty → block submit with "Add at least one ingredient" |
| **Error case** | Validation block prevents empty or incomplete submissions |

---

### Step 7

| Field | Value |
|---|---|
| **Step** | 7 |
| **Actor** | .NET 10 API |
| **Action** | Authenticate and authorise the request |
| **System component** | .NET 10 API (AuthMiddleware + RecipePolicy) |
| **Data IN** | JWT from `Authorization: Bearer` header; request body |
| **Data OUT** | Extracted claims: `user_id`, `role`, `outlet_id` |
| **Decision point** | YES — Role must be `Owner` or `Chef`. If role is `Procurement` or `Viewer` → return `HTTP 403 Forbidden` with body `{ error: "RECIPE_CREATE_FORBIDDEN" }`. `outlet_id` from JWT must match `outlet_id` in request body → if mismatch → HTTP 403 |
| **Error case** | Missing or invalid JWT → HTTP 401 |

---

### Step 8

| Field | Value |
|---|---|
| **Step** | 8 |
| **Actor** | .NET 10 API |
| **Action** | Validate all `ingredient_id` values in `items[]` exist and belong to the same outlet |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | `items[].ingredient_id[]` + `outlet_id` |
| **Data OUT** | Confirmed list of valid `Ingredient` records |
| **Decision point** | YES — if any `ingredient_id` is not found in `Ingredients` table for this outlet → return `HTTP 422 Unprocessable Entity` with `{ error: "INGREDIENT_NOT_FOUND", ingredient_id: "..." }` |
| **Error case** | PostgreSQL query failure → HTTP 500; all individual 422 errors returned as an array in a single response |

---

### Step 9

| Field | Value |
|---|---|
| **Step** | 9 |
| **Actor** | .NET 10 API |
| **Action** | Persist `Recipe` and all `RecipeItem` records in a single database transaction |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | Validated recipe payload |
| **Data OUT** | `Recipe` record created: `{ id (new UUID), outlet_id, name, portion_count, sale_price, cost_threshold_percentage, version_group_id (new UUID), version_number=1, version_label, cost_per_portion=null, has_incomplete_pricing=false, created_at, created_by_user_id }` + one `RecipeItem` record per item |
| **Decision point** | YES — `version_group_id` is generated as a new UUID here (this is version 1 of a new group). For existing version groups (Flow 4), it will be supplied. `version_number` = 1 |
| **Error case** | Transaction rollback on any constraint violation → HTTP 422. Duplicate `(outlet_id, name, version_label)` combination → HTTP 409 `{ error: "DUPLICATE_RECIPE_VERSION" }` |

---

### Step 10

| Field | Value |
|---|---|
| **Step** | 10 |
| **Actor** | .NET 10 API (CostCalculationService) |
| **Action** | Calculate `cost_per_portion` for the newly saved recipe |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | `recipe_id`; all `RecipeItem` records for this recipe; latest `IngredientPriceHistory.price` for each ingredient (fetched via `SELECT ... ORDER BY committed_at DESC LIMIT 1`) |
| **Data OUT** | `cost_per_portion` (decimal, 4dp stored, 2dp displayed); `has_incomplete_pricing` flag |
| **Formula** | `cost_per_portion = SUM((current_price / unit_size) * quantity * (1 / yield_percentage)) / portion_count` |
| **Decision point** | YES — if any ingredient has NO `IngredientPriceHistory` record → set `has_incomplete_pricing = true`; calculate cost using available ingredients only. If ALL ingredients have no price → `cost_per_portion = null` |
| **Error case** | `yield_percentage = 0` in a RecipeItem (should be blocked at Step 4/8 but double-check here) → skip that item, flag `has_incomplete_pricing = true`, log warning. `portion_count = 0` → set `cost_per_portion = null`, log error |

---

### Step 11

| Field | Value |
|---|---|
| **Step** | 11 |
| **Actor** | .NET 10 API |
| **Action** | Persist calculated `cost_per_portion` and return response to Vue.js |
| **System component** | .NET 10 API → PostgreSQL → Vue.js |
| **Data IN** | Calculated `cost_per_portion`, `has_incomplete_pricing` |
| **Data OUT** | `Recipe` record updated with `cost_per_portion`, `cost_last_updated_at = NOW()`. HTTP 201 response body: `{ id, name, outlet_id, version_number, version_label, version_group_id, portion_count, sale_price, cost_per_portion, has_incomplete_pricing, items: [...] }` — **Owner only**: `food_cost_pct`, `gross_margin` appended. Stripped for all other roles. |
| **Decision point** | YES — Role check: if `role == Owner` → include `food_cost_pct = (cost_per_portion / sale_price) * 100` and `gross_margin = sale_price - cost_per_portion` in response. All other roles → omit these fields entirely |
| **Error case** | PostgreSQL write failure → return HTTP 500; recipe is persisted at this point but `cost_per_portion` may be stale — a background retry should recalculate (note this as a known gap to address) |

---

### Step 12

| Field | Value |
|---|---|
| **Step** | 12 |
| **Actor** | System (Vue.js) |
| **Action** | Navigate to recipe detail view and display the saved recipe |
| **System component** | Vue.js |
| **Data IN** | HTTP 201 response body |
| **Data OUT** | Recipe detail page rendered: recipe name, ingredient table, `cost_per_portion`, and food cost % / margin (Owner only). If `has_incomplete_pricing = true` → display yellow banner: "Pricing incomplete — some ingredients have no price on record" |
| **Decision point** | No |
| **Error case** | HTTP 201 not received → Vue.js remains on the Recipe Builder form, displays error toast with the server's error message |

---
---

# Flow 2 — Auto-Cascade Cost Recalculation

**This is the canonical, reusable cascade mechanism.** It is triggered by **exactly two sources**:
1. A manual ingredient price update (user edits an ingredient price via the UI)
2. Confirmation of an `InvoiceLineItem` in the Review Queue

Both sources commit a new `IngredientPriceHistory` record and then hand off to this exact flow. The flow is identical regardless of source.

**Starting point:** A new `IngredientPriceHistory` record has just been committed to PostgreSQL.  
**End state:** Every `Recipe` that contains this ingredient has a refreshed `cost_per_portion`. Cost threshold check (Flow 3) has been evaluated for each affected recipe.

---

### Step 1

| Field | Value |
|---|---|
| **Step** | 1 |
| **Actor** | System |
| **Action** | New `IngredientPriceHistory` record committed to database |
| **System component** | PostgreSQL |
| **Data IN** | `IngredientPriceHistory { id, ingredient_id, price, effective_date, source (manual \| invoice), committed_at }` |
| **Data OUT** | Record persisted. `ingredient_id` available as the cascade trigger key |
| **Decision point** | No |
| **Error case** | This step is the responsibility of the caller (Manual Update flow or Invoice Scan flow). If this commit fails, the cascade never starts — correct behaviour |

---

### Step 2

| Field | Value |
|---|---|
| **Step** | 2 |
| **Actor** | .NET 10 API (IngredientPriceService) |
| **Action** | Dispatch cascade after price commit |
| **System component** | .NET 10 API |
| **Data IN** | `ingredient_id` of the newly committed `IngredientPriceHistory` record |
| **Data OUT** | `CostCascadeJob` initiated — in v1, executed **synchronously inline** within the same request context. Implementation note: this is designed as a seam; the call site is `ICostCascadeService.RecalculateForIngredient(ingredient_id)`. In a future async version, this becomes a background job enqueue without changing the interface. |
| **Decision point** | No — cascade always runs; it is not optional |
| **Error case** | If cascade throws an uncaught exception → **do NOT roll back the `IngredientPriceHistory` record**. The price update is permanent. Log the cascade failure with `ingredient_id`, `committed_at`, and error details. Stale recipes will remain stale until a retry or manual recalculation is triggered |

---

### Step 3

| Field | Value |
|---|---|
| **Step** | 3 |
| **Actor** | .NET 10 API (CostCascadeService) |
| **Action** | Query all `RecipeItem` records that reference this ingredient |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | `ingredient_id` |
| **Data OUT** | List of distinct `recipe_id` values from `RecipeItem WHERE ingredient_id = @ingredient_id` — scoped to the ingredient's `outlet_id` implicitly (because recipes and ingredients belong to the same outlet) |
| **Decision point** | YES — if the result set is empty → **cascade ends here**. No recipes use this ingredient. Log "cascade skipped — no recipes affected" at DEBUG level |
| **Error case** | PostgreSQL query failure → log error with `ingredient_id`; abort cascade; do not re-throw (price record is already committed) |

---

### Step 4

| Field | Value |
|---|---|
| **Step** | 4 |
| **Actor** | .NET 10 API (CostCascadeService) |
| **Action** | For each affected `recipe_id`: fetch ALL of that recipe's `RecipeItem` records and resolve the current price for each ingredient |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | `recipe_id` |
| **Data OUT** | Full list of `RecipeItem { ingredient_id, quantity, yield_percentage }` + for each ingredient: `current_price` (from `SELECT price FROM IngredientPriceHistory WHERE ingredient_id = @id ORDER BY committed_at DESC LIMIT 1`) + `unit_size` from `Ingredient` |
| **Decision point** | YES — if an ingredient within this recipe has NO `IngredientPriceHistory` record → its contribution to cost = 0, and `has_incomplete_pricing = true` for this recipe. Log at WARN level |
| **Error case** | Query failure for a specific recipe → log error, skip this recipe, continue cascade for remaining recipes. Errors are isolated per recipe |

---

### Step 5

| Field | Value |
|---|---|
| **Step** | 5 |
| **Actor** | .NET 10 API (CostCalculationService) |
| **Action** | Apply the canonical cost formula to compute new `cost_per_portion` |
| **System component** | .NET 10 API (pure in-memory computation — no DB call) |
| **Data IN** | RecipeItems with resolved prices, `recipe.portion_count` |
| **Data OUT** | `new_cost_per_portion` (decimal, 4dp) |
| **Formula** | `cost_per_portion = SUM((current_price / unit_size) * quantity * (1 / yield_percentage)) / portion_count` |
| **Decision point** | No |
| **Error case** | `yield_percentage = 0` on any item → skip that item, set `has_incomplete_pricing = true`, log at ERROR (this should have been caught at data entry). `portion_count = 0` → skip cost update for this recipe entirely, log at ERROR |

---

### Step 6

| Field | Value |
|---|---|
| **Step** | 6 |
| **Actor** | .NET 10 API (CostCascadeService) |
| **Action** | Persist the updated `cost_per_portion` on the `Recipe` record |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | `recipe_id`, `new_cost_per_portion`, `has_incomplete_pricing` |
| **Data OUT** | `Recipe.cost_per_portion = new_cost_per_portion`, `Recipe.cost_last_updated_at = NOW()`, `Recipe.has_incomplete_pricing` updated |
| **Decision point** | No |
| **Error case** | PostgreSQL write failure → log at ERROR with `recipe_id` and computed value; mark recipe as `cost_stale = true` if that field is available; do not re-throw — continue cascade for remaining recipes |

---

### Step 7

| Field | Value |
|---|---|
| **Step** | 7 |
| **Actor** | .NET 10 API (CostCascadeService) |
| **Action** | Evaluate cost threshold for this recipe |
| **System component** | .NET 10 API (in-memory computation) |
| **Data IN** | `recipe.cost_per_portion` (new), `recipe.sale_price`, `recipe.cost_threshold_percentage` |
| **Data OUT** | `food_cost_pct = (new_cost_per_portion / sale_price) * 100`; `threshold_breached = food_cost_pct > cost_threshold_percentage` |
| **Decision point** | YES — if `threshold_breached = true` → **invoke Flow 3 (Cost Threshold Alert)** for this recipe. If false → cascade for this recipe is complete |
| **Error case** | `sale_price` is null or 0 → CANNOT calculate `food_cost_pct`; skip threshold check; log at WARN "Recipe {id} has no sale_price — threshold check skipped" |

---

### Step 8

| Field | Value |
|---|---|
| **Step** | 8 |
| **Actor** | .NET 10 API (CostCascadeService) |
| **Action** | Repeat Steps 4–7 for each remaining affected `recipe_id` from Step 3 |
| **System component** | .NET 10 API |
| **Data IN** | Remaining `recipe_id` list |
| **Data OUT** | All affected recipes updated; cost threshold evaluated for each |
| **Decision point** | YES — continue until all `recipe_id` values from Step 3 are processed |
| **Error case** | Per-recipe errors are isolated and logged. One failed recipe does not abort the loop |

---

**Cascade completion:** All `Recipe` records that use the updated ingredient now carry a refreshed `cost_per_portion`. Any threshold breaches have been handed to Flow 3.

---
---

# Flow 3 — Cost Threshold Alert

**Starting point:** Flow 2, Step 7 — `threshold_breached = true` for a specific `recipe_id`.  
**End state:** Owner (and Chef, if applicable) receive a Telegram notification with role-appropriate content.

---

### Step 1

| Field | Value |
|---|---|
| **Step** | 1 |
| **Actor** | .NET 10 API (CostCascadeService) |
| **Action** | Construct alert payload |
| **System component** | .NET 10 API |
| **Data IN** | `recipe_id`, `recipe.name`, `recipe.outlet_id`, `outlet.name`, `new_cost_per_portion`, `food_cost_pct`, `cost_threshold_percentage`, `ingredient_id` that triggered the cascade (for context) |
| **Data OUT** | Alert payload: `{ recipe_id, recipe_name, outlet_id, outlet_name, new_cost_per_portion, previous_cost_per_portion, food_cost_pct, threshold_pct, triggered_by_ingredient_id, triggered_at }` |
| **Decision point** | No |
| **Error case** | None at this step |

---

### Step 2

| Field | Value |
|---|---|
| **Step** | 2 |
| **Actor** | .NET 10 API |
| **Action** | POST alert payload to Python FastAPI |
| **System component** | .NET 10 API → Python FastAPI (`POST /internal/alerts/cost-threshold`) |
| **Data IN** | Alert payload from Step 1 |
| **Data OUT** | HTTP 202 Accepted (async — fire and forget from .NET's perspective) |
| **Call pattern** | **Fire-and-forget async HTTP call.** .NET API does NOT await Telegram delivery. The cascade is considered complete once this HTTP call is dispatched. This prevents Telegram delivery latency from blocking the cost update pipeline. |
| **Decision point** | No |
| **Error case** | Python FastAPI unreachable (connection refused, timeout) → log `ALERT_DELIVERY_FAILED` with full payload and `recipe_id`; do NOT fail or re-try inline; consider a `PendingAlerts` table for retry (mark as a Phase 4 hardening task) |

---

### Step 3

| Field | Value |
|---|---|
| **Step** | 3 |
| **Actor** | Python FastAPI (AlertService) |
| **Action** | Receive and validate alert payload |
| **System component** | Python FastAPI |
| **Data IN** | Alert payload from Step 2 |
| **Data OUT** | Payload validated and accepted |
| **Decision point** | No |
| **Error case** | Payload validation failure (missing required fields, type mismatch) → return HTTP 400; .NET API logs the 400 response against the `recipe_id` for debugging |

---

### Step 4

| Field | Value |
|---|---|
| **Step** | 4 |
| **Actor** | Python FastAPI (AlertService) |
| **Action** | Retrieve Telegram recipients for the affected outlet |
| **System component** | Python FastAPI → .NET 10 API (`GET /api/outlets/{outlet_id}/telegram-links`) |
| **Data IN** | `outlet_id` |
| **Data OUT** | List of `TelegramLink` records: `{ user_id, telegram_chat_id, role }` |
| **Decision point** | YES — filter recipients by role: **Owner** receives full alert including `food_cost_pct` and `gross_margin`. **Chef** receives alert with `cost_per_portion` only — `food_cost_pct` is withheld. **Procurement** and **Viewer** receive NO cost threshold alerts. If no eligible recipients → log "no Telegram recipients for outlet {outlet_id}" and end flow. |
| **Error case** | .NET API returns error or empty list → log and exit gracefully |

---

### Step 5

| Field | Value |
|---|---|
| **Step** | 5 |
| **Actor** | Python FastAPI (AlertService) |
| **Action** | Format role-specific Telegram message for each recipient |
| **System component** | Python FastAPI |
| **Data IN** | Alert payload; recipient list with roles |
| **Data OUT** | Formatted message strings per recipient |
| **Message format — Owner** | `⚠️ Cost Alert — [Outlet Name]\n[Recipe Name] has crossed your food cost threshold.\nFood cost: [food_cost_pct]% (threshold: [threshold_pct]%)\nCost per portion: $[new_cost_per_portion]\nTriggered: [triggered_at UTC]` |
| **Message format — Chef** | `⚠️ Cost Alert — [Outlet Name]\n[Recipe Name] cost per portion has increased to $[new_cost_per_portion].\nPlease review ingredient pricing. [triggered_at UTC]` |
| **Decision point** | No |
| **Error case** | None |

---

### Step 6

| Field | Value |
|---|---|
| **Step** | 6 |
| **Actor** | Python FastAPI (TelegramBot) |
| **Action** | Send Telegram message to each recipient |
| **System component** | Python FastAPI → Telegram Bot API |
| **Data IN** | `telegram_chat_id`, formatted message string |
| **Data OUT** | Telegram `message_id` on successful delivery |
| **Decision point** | No |
| **Error case** | Telegram API error (rate limit 429, invalid `chat_id` 400, bot blocked by user) → **retry up to 3 times with exponential backoff** (1s, 3s, 9s). After 3 failures → log `TELEGRAM_DELIVERY_FAILED` with `telegram_chat_id`, `recipe_id`, and Telegram error code. A blocked bot (user has blocked the bot) should silently deactivate the `TelegramLink` record via `.NET 10 API PATCH /api/telegram-links/{id}/deactivate` |

---

### Step 7

| Field | Value |
|---|---|
| **Step** | 7 |
| **Actor** | Python FastAPI |
| **Action** | Record alert delivery audit log |
| **System component** | Python FastAPI (internal log or POST to .NET 10 API audit endpoint) |
| **Data IN** | `recipe_id`, `outlet_id`, `triggered_at`, list of `telegram_chat_id` values and their delivery status (delivered / failed) |
| **Data OUT** | Audit record persisted |
| **Decision point** | No |
| **Error case** | Audit log failure is non-fatal — log locally and continue |

---
---

# Flow 4 — Recipe Versioning

**Design note:** Versioning is implemented within the canonical `Recipe` entity. There is no separate `RecipeVersion` entity. All versions of a "recipe family" share a `version_group_id` (UUID). Each version is a fully independent `Recipe` record with its own `RecipeItem` set, its own `cost_per_portion`, and its own `version_number` and `version_label`. Auto-cascade (Flow 2) applies to **all** versions independently — every version always reflects live ingredient prices.

**Actors:** Owner or Chef  
**Sub-flow A:** Creating a new version of an existing recipe  
**Sub-flow B:** Viewing all versions side-by-side

---

## Sub-flow A — Creating a New Recipe Version

### Step 1

| Field | Value |
|---|---|
| **Step** | 1 |
| **Actor** | Owner or Chef |
| **Action** | Open an existing recipe and select "Save as New Version" |
| **System component** | Vue.js |
| **Data IN** | Existing `Recipe` record: `{ id, name, version_group_id, version_number, version_label, portion_count, sale_price, cost_threshold_percentage, items: [...] }` |
| **Data OUT** | Recipe Builder pre-populated as a copy of the existing recipe. `version_group_id` retained (same family). `version_number` and `version_label` cleared (user must set the new label). UI displays: "Creating new version of: [name] (previously v[version_number])" |
| **Decision point** | No |
| **Error case** | None — this is a read + client-side copy |

---

### Step 2

| Field | Value |
|---|---|
| **Step** | 2 |
| **Actor** | Owner or Chef |
| **Action** | Modify the recipe (change ingredients, quantities, portion count, etc.) and enter a `version_label` |
| **System component** | Vue.js |
| **Data IN** | User edits to `RecipeItem` list; `version_label` input (e.g., "Premium — truffle oil", "Summer 2026 — portion reduced") |
| **Data OUT** | Updated local form state; live cost preview recalculates (same as Flow 1, Step 5) |
| **Decision point** | No |
| **Error case** | `version_label` is required when `version_number > 1` — block save if blank. Version label must not duplicate an existing label within the same `version_group_id` (validated server-side in Step 4) |

---

### Step 3

| Field | Value |
|---|---|
| **Step** | 3 |
| **Actor** | Owner or Chef |
| **Action** | Click "Save" |
| **System component** | Vue.js → .NET 10 API |
| **Data IN** | `POST /api/recipes` body: `{ name (same as original), outlet_id, portion_count, sale_price, cost_threshold_percentage, version_group_id (from original), version_label, items: [...] }` |
| **Data OUT** | HTTP request dispatched |
| **Decision point** | YES — `version_group_id` is populated (existing group) vs. null (new recipe — handled in Flow 1). The presence of `version_group_id` tells the API this is a new version of an existing family, not a brand-new recipe. |
| **Error case** | Same as Flow 1, Step 6 |

---

### Step 4

| Field | Value |
|---|---|
| **Step** | 4 |
| **Actor** | .NET 10 API |
| **Action** | Validate and assign `version_number` |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | `version_group_id` from request |
| **Data OUT** | `version_number = SELECT MAX(version_number) FROM Recipe WHERE version_group_id = @id FOR UPDATE + 1` (pessimistic lock to avoid race condition on concurrent saves) |
| **Decision point** | YES — validate that `version_group_id` exists in `Recipe` table and belongs to `outlet_id` in JWT. If not → HTTP 404. If `(version_group_id, version_label)` combination already exists → HTTP 409 `{ error: "DUPLICATE_VERSION_LABEL" }` |
| **Error case** | Race condition on version_number assignment is resolved by the `FOR UPDATE` row lock |

---

### Step 5

| Field | Value |
|---|---|
| **Step** | 5 |
| **Actor** | .NET 10 API |
| **Action** | Persist new `Recipe` record and its `RecipeItem` records |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | Validated payload with `version_number`, `version_group_id`, `version_label` |
| **Data OUT** | New `Recipe` record created (new `id`, inherited `version_group_id`, sequential `version_number`, new `created_at`, new `RecipeItem` records). **The original recipe is untouched.** |
| **Decision point** | No |
| **Error case** | Same as Flow 1, Step 9 |

---

### Step 6

| Field | Value |
|---|---|
| **Step** | 6 |
| **Actor** | .NET 10 API (CostCalculationService) |
| **Action** | Calculate `cost_per_portion` for the new version |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | New `recipe_id`; new `RecipeItem` records; current ingredient prices |
| **Data OUT** | `cost_per_portion` stored on new `Recipe` record |
| **Note** | Identical to Flow 1, Steps 10–11. The original recipe's `cost_per_portion` is unaffected. |
| **Decision point** | Same as Flow 1, Step 10 |
| **Error case** | Same as Flow 1, Step 10 |

---

### Step 7

| Field | Value |
|---|---|
| **Step** | 7 |
| **Actor** | .NET 10 API |
| **Action** | Return HTTP 201 with new version data |
| **System component** | .NET 10 API → Vue.js |
| **Data IN** | Persisted new version |
| **Data OUT** | HTTP 201: `{ id, name, version_number, version_label, version_group_id, cost_per_portion, food_cost_pct (Owner only), ... }` |
| **Decision point** | Same role-based field stripping as Flow 1, Step 11 |
| **Error case** | Same as Flow 1, Step 11 |

---

## Sub-flow B — Viewing All Versions Side-by-Side

### Step 1

| Field | Value |
|---|---|
| **Step** | 1 |
| **Actor** | Owner or Chef (or Procurement, Viewer — read access applies to all roles) |
| **Action** | Navigate to the Version Comparison view for a recipe family |
| **System component** | Vue.js → .NET 10 API |
| **Data IN** | `GET /api/recipes/versions/{version_group_id}` |
| **Data OUT** | HTTP request dispatched with JWT |
| **Decision point** | No |
| **Error case** | If `version_group_id` is not a valid UUID → HTTP 400 |

---

### Step 2

| Field | Value |
|---|---|
| **Step** | 2 |
| **Actor** | .NET 10 API |
| **Action** | Fetch all `Recipe` records sharing the `version_group_id` |
| **System component** | .NET 10 API → PostgreSQL |
| **Data IN** | `version_group_id`; `outlet_id` from JWT |
| **Data OUT** | All `Recipe` records for this group (ordered by `version_number ASC`). For each version: full `RecipeItem` list with ingredient details and `current_price` |
| **Decision point** | YES — `outlet_id` from each `Recipe` must match `outlet_id` in JWT. If mismatch → HTTP 403. If `version_group_id` not found → HTTP 404 |
| **Error case** | PostgreSQL query failure → HTTP 500 |

---

### Step 3

| Field | Value |
|---|---|
| **Step** | 3 |
| **Actor** | .NET 10 API |
| **Action** | Apply role-based field stripping and return response |
| **System component** | .NET 10 API → Vue.js |
| **Data IN** | Raw version list |
| **Data OUT** | Array of `RecipeVersionSummary`: `{ recipe_id, version_number, version_label, cost_per_portion, item_count, portion_count, cost_last_updated_at, has_incomplete_pricing }`. **Owner only:** `food_cost_pct`, `gross_margin` appended to each version. All other roles: omit. |
| **Decision point** | YES — role check. Owner sees `food_cost_pct` and `gross_margin`. All others see `cost_per_portion` only. |
| **Error case** | None at this step |

---

### Step 4

| Field | Value |
|---|---|
| **Step** | 4 |
| **Actor** | System (Vue.js) |
| **Action** | Render side-by-side comparison table |
| **System component** | Vue.js |
| **Data IN** | Array of `RecipeVersionSummary` objects |
| **Data OUT** | Comparison UI: each column = one version; each row = one ingredient. Cells show quantity and cost contribution per ingredient per version. Differences highlighted: ingredients only in one version (yellow background), quantity differences (orange), cost delta between versions shown in header row. `cost_per_portion` for each version in the column header. `food_cost_pct` and `gross_margin` row is rendered only if role = Owner. |
| **Decision point** | YES — if only one version exists → display message "Only one version exists. Use 'Save as New Version' to create a variant." |
| **Error case** | None (display only — all data is server-side validated) |

---
---

# Cross-Flow Consistency Notes

These notes are for the lead architect review and must be verified against the Invoice Scanning, Ingredient Management, and Telegram Bot flows.

## Cascade seam (Flow 2, Step 2)
`ICostCascadeService.RecalculateForIngredient(ingredient_id)` is the **single entry point** for all cost recalculations. The Invoice Scan confirmation flow and the Manual Ingredient Price Update flow **must both call this interface** — not implement cost recalculation independently. This is the critical consistency guarantee.

## Price history immutability
`IngredientPriceHistory` records are **never updated or deleted**. The current price is always resolved as `ORDER BY committed_at DESC LIMIT 1`. Any flow that "updates" an ingredient price does so by inserting a new `IngredientPriceHistory` record, not by modifying an existing one.

## Role-based field stripping
The `food_cost_pct` and `gross_margin` fields are **stripped at the API response layer** in .NET 10 API, not in Vue.js. Vue.js should never receive these values for non-Owner roles — this is a server-side responsibility. This rule must be consistent across all recipe-related API endpoints: `GET /api/recipes`, `GET /api/recipes/{id}`, `GET /api/recipes/versions/{version_group_id}`, and the recipe creation response.

## Outlet isolation
Every query for `Recipe`, `RecipeItem`, and `Ingredient` records must include an `outlet_id` filter derived from the JWT. Cross-outlet data access must never be possible through the API.

## `version_group_id` lifecycle
- Assigned once at first save (Flow 1, Step 9)
- Never changes — it is the stable identifier for a recipe family
- Used as the key for version comparison queries (Flow 4, Sub-flow B)
- The Telegram Bot's `/cost [dish]` command should resolve by `version_group_id` + a version selector (default: most recently created version)
