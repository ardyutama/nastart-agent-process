# Flow 02 — Ingredient Management & Price History

> **Status:** ACCEPTED (Round 2, after defect corrections) — Lead Architect validated
> **Service owner:** .NET 10 Web API
> **Entities:** Ingredient, IngredientPriceHistory, Unit, Category, Supplier, TelegramLink

---

## Shared Cascade Algorithm (referenced by this flow, Recipe Costing, and Invoice Scanning)

**Interface:** `ICostCascadeService.RecalculateForIngredient(ingredientId)`

```
INPUT:  ingredientId (UUID)

STEPS:
  A. Fetch current price:
     SELECT price FROM IngredientPriceHistory
     WHERE ingredient_id = @ingredientId
     ORDER BY committed_at DESC LIMIT 1

  B. Fetch all RecipeItems using this ingredient:
     SELECT ri.*, r.portion_count, r.cost_threshold_percentage
     FROM RecipeItem ri
     JOIN Recipe r ON r.id = ri.recipe_id
     WHERE ri.ingredient_id = @ingredientId

  C. For each Recipe found:
     FOR each RecipeItem in Recipe:
       item_cost = (price / ingredient.unit_size) * ri.quantity * (1 / ri.yield_percentage)
     cost_per_portion = SUM(item_costs) / r.portion_count
     UPDATE Recipe SET cost_per_portion = <calculated>

  D. After each Recipe update:
     IF recipe.food_cost_pct > recipe.cost_threshold_percentage:
       Dispatch alert (fire-and-forget) to Python FastAPI alert endpoint

OUTPUT: affectedRecipeCount, new_cost_per_portion[]

ERROR HANDLING:
  - Per-recipe failure → log to CascadeErrorLog{ingredientId, recipeId, error, timestamp} → skip
  - IngredientPriceHistory is NEVER rolled back on cascade failure
```

---

## Flow 1: Manual Ingredient Creation

**Actors:** Owner, Chef, or Procurement
**Entry point:** Ingredient management page → "Add Ingredient"

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens ingredient form | Vue.js | - | Empty form | - | - |
| 2 | User | Fills ingredient details | Vue.js | name, category_id, unit_id, unit_size, price_spike_threshold_pct | - | - | Validation on submit |
| 3 | .NET 10 API | Validates outlet scope + role | .NET 10 | JWT (outlet_id, role) | - | Role not in {Owner,Chef,Procurement} → 403 | 403 |
| 4 | .NET 10 API | Creates Ingredient record | PostgreSQL | name, outlet_id, category_id, unit_id, unit_size, price_spike_threshold_pct | Ingredient.id | Duplicate name in outlet → 409 | 409 Conflict |
| 5 | User | Enters initial price | Vue.js | price, effective_date (optional) | - | - | - |
| 6 | .NET 10 API | Inserts first IngredientPriceHistory | PostgreSQL | ingredient_id, price, source='Manual', committed_at=NOW(), effective_date=input??NOW(), invoiceLineItemId=null | IngredientPriceHistory.id | - | DB error → 500 |
| 7 | Vue.js | Shows ingredient with current price | Vue.js | Ingredient + latest price | Ingredient card | - | - |

---

## Flow 2: Manual Price Update → Auto-Cascade

**Actors:** Owner or Procurement
**Entry point:** Ingredient detail page → "Update Price"

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens price update form | Vue.js | - | Current price displayed | - | - |
| 2 | User | Enters new price + optional effective date | Vue.js | new_price, effective_date | - | - | price must be > 0 |
| 3 | .NET 10 API | Validates role | .NET 10 | JWT (role) | - | Role not in {Owner,Procurement} → 403 | 403 |
| 4 | .NET 10 API | Inserts new IngredientPriceHistory | PostgreSQL | ingredient_id, price=new_price, source='Manual', committed_at=NOW(), effective_date=input??NOW(), invoiceLineItemId=null | IngredientPriceHistory.id | - | DB error → 500 |
| 5 | .NET 10 API | Checks price spike threshold | PostgreSQL | previous price (ORDER BY committed_at DESC LIMIT 1 OFFSET 1), new_price, Ingredient.price_spike_threshold_pct | price_change_pct | change_pct > threshold → trigger Flow 3 | - |
| 6 | .NET 10 API | Calls ICostCascadeService | .NET 10 | ingredientId | affectedRecipeCount, new costs | - | Per-recipe errors logged, skipped |
| 7 | Vue.js | Shows updated cost across affected recipes | Vue.js | updated Recipe list | Dashboard refresh | - | - |

---

## Flow 3: Price Ceiling / Spike Alert

**Trigger:** Called after any new IngredientPriceHistory record (manual OR invoice scan)

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | .NET 10 API | Computes price change % | .NET 10 | new_price, previous_price, Ingredient.price_spike_threshold_pct | price_change_pct | change_pct ≤ threshold → stop | - |
| 2 | .NET 10 API | Queries alert recipients | PostgreSQL | `SELECT tl.telegramUserId, u.Name FROM TelegramLink tl JOIN User u ON u.Id = tl.UserId WHERE u.OutletId = @outletId AND u.Role IN ('Owner','Procurement') AND tl.status = 'confirmed'` | [{telegramUserId, userName}] | No recipients → log and stop | - |
| 3 | .NET 10 API | Sends recipient list + alert payload to FastAPI | Python FastAPI | [{telegramUserId}, ingredient_name, old_price, new_price, change_pct, outlet_name] | HTTP 202 | FastAPI unreachable → retry 3x with backoff | Dead-letter log |
| 4 | Python FastAPI | Formats and sends Telegram messages | Telegram Bot API | [{telegramUserId, message}] | Delivery status | 400 (invalid chatId) OR 403 (bot blocked) → call PATCH /api/telegram-links/{id}/deactivate | Set TelegramLink.status='unlinked' |

---

## Flow 4: Supplier Price History & Comparison

**Actors:** Owner, Procurement (read access)
**Entry point:** Ingredient detail → "Price History" tab

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Navigates to ingredient price history | Vue.js | ingredient_id | - | - | - |
| 2 | .NET 10 API | Fetches price history for ingredient | PostgreSQL | `SELECT iph.*, s.name AS supplier_name, i.unit FROM IngredientPriceHistory iph LEFT JOIN InvoiceLineItem ili ON ili.id = iph.invoiceLineItemId LEFT JOIN Invoice inv ON inv.id = ili.invoice_id LEFT JOIN Supplier s ON s.id = inv.supplier_id WHERE iph.ingredient_id = @id ORDER BY committed_at DESC` | IngredientPriceHistory[] with supplier context | - | - |
| 3 | Vue.js | Renders price trend chart + supplier comparison | Vue.js | history[] | Line chart grouped by Supplier.name | - | - |
| 4 | User | Filters by supplier or date range | Vue.js | supplier_id (optional), date_from, date_to | Filtered records | - | - |

**Database index:** Composite index on `IngredientPriceHistory(ingredient_id, committed_at DESC)` for performance.

**IngredientPriceHistory schema:**
```
id                UUID        PK
ingredient_id     UUID        FK → Ingredient.id
price             decimal(18,4)
unit_size         decimal(18,4)  snapshot at time of recording
source            varchar(20) 'Manual' | 'InvoiceScan'
committed_at      timestamptz  system timestamp — non-editable — ordering field
effective_date    date         user-editable — business effective date
invoiceLineItemId UUID         FK → InvoiceLineItem.id — null for Manual entries
```
