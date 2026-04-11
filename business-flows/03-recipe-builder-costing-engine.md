# Flow 03 — Recipe Builder & Costing Engine

> **Status:** ACCEPTED (Round 1) — Lead Architect validated, no defects. Canonical reference for cost formula and cascade interface.
> **Service owner:** .NET 10 Web API
> **Entities:** Recipe, RecipeItem, Ingredient, IngredientPriceHistory

---

## Canonical Formula

```
cost_per_portion = SUM(
  (current_price / unit_size) * quantity * (1 / yield_percentage)
) / portion_count
```

Where `current_price` = `SELECT price FROM IngredientPriceHistory WHERE ingredient_id = @id ORDER BY committed_at DESC LIMIT 1`

---

## Flow 1: Creating a New Recipe

**Actors:** Owner or Chef
**Entry point:** Recipe management page → "New Recipe"

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens new recipe form | Vue.js | - | Empty form | - | - |
| 2 | User | Fills recipe header | Vue.js | name, category, portion_count, version_label, cost_threshold_percentage, selling_price (optional) | - | - | portion_count must be ≥ 1 |
| 3 | User | Adds ingredients (RecipeItems) one by one | Vue.js | ingredient_id, quantity, unit, yield_percentage | RecipeItem list | - | ingredient must belong to same outlet |
| 4 | Vue.js | Computes live preview cost (client-side estimate) | Vue.js | ingredient prices from cache | Estimated cost_per_portion | - | - |
| 5 | User | Saves recipe | Vue.js → .NET 10 | Recipe + RecipeItems[] | - | - | - |
| 6 | .NET 10 API | Validates role + outlet scope | .NET 10 | JWT (role, outlet_id) | - | Role not in {Owner,Chef} → 403 | 403 |
| 7 | .NET 10 API | Persists Recipe record | PostgreSQL | name, outlet_id, portion_count, version_group_id (new UUID), version_number=1, version_label, cost_threshold_percentage | Recipe.id | Duplicate name in outlet → 409 | 409 |
| 8 | .NET 10 API | Persists RecipeItem records | PostgreSQL | recipe_id, ingredient_id, quantity, yield_percentage | RecipeItem[] | - | - |
| 9 | .NET 10 API | Calls ICostCascadeService.RecalculateForIngredient for each unique ingredient | .NET 10 | ingredient_id[] | cost_per_portion (authoritative server calculation) | - | Per-ingredient errors logged, skipped |
| 10 | .NET 10 API | Updates Recipe.cost_per_portion (server-authoritative) | PostgreSQL | recipe_id, calculated cost_per_portion | Updated Recipe | - | - |
| 11 | .NET 10 API | Calculates food_cost_pct (if selling_price set) | .NET 10 | cost_per_portion, selling_price | food_cost_pct = (cost/selling_price) × 100 | - | - |
| 12 | Vue.js | Displays recipe with live, server-authoritative cost | Vue.js | Recipe (role-filtered) | Recipe card | Owner gets food_cost_pct; others do not | - |

---

## Flow 2: Auto-Cascade Cost Recalculation

**Trigger:** A new `IngredientPriceHistory` record has just been committed to PostgreSQL (from ANY source)
**Interface:** `ICostCascadeService.RecalculateForIngredient(ingredientId)`

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | .NET 10 API | Fetches current price for ingredient | PostgreSQL | `SELECT price FROM IngredientPriceHistory WHERE ingredient_id = @id ORDER BY committed_at DESC LIMIT 1` | current_price | No history → skip cascade | Log warning |
| 2 | .NET 10 API | Fetches all RecipeItems using this ingredient | PostgreSQL | `SELECT ri.*, r.* FROM RecipeItem ri JOIN Recipe r ON r.id = ri.recipe_id WHERE ri.ingredient_id = @id` | RecipeItem[] grouped by Recipe | No recipes → cascade complete | - |
| 3 | .NET 10 API | For each Recipe, fetches all RecipeItems + current prices for ALL ingredients | PostgreSQL | recipe_id, ingredient_ids[] | Full recipe cost context | - | - |
| 4 | .NET 10 API | Applies cost formula per recipe | .NET 10 | RecipeItems[], current_prices[], portion_count | new_cost_per_portion | - | Per-recipe error → log to CascadeErrorLog, skip |
| 5 | .NET 10 API | Updates Recipe.cost_per_portion | PostgreSQL | recipe_id, new_cost_per_portion | Updated Recipe | - | Error → log, skip, NEVER rollback IngredientPriceHistory |
| 6 | .NET 10 API | Evaluates threshold | .NET 10 | new_cost_per_portion, selling_price, cost_threshold_percentage | food_cost_pct | food_cost_pct > cost_threshold_percentage → trigger alert | - |
| 7 | .NET 10 API | Dispatches threshold alert (fire-and-forget) | Python FastAPI | {recipe_id, recipe_name, food_cost_pct, cost_per_portion, outlet_id} | HTTP 202 | FastAPI unreachable → retry 3x | Dead-letter log |

---

## Flow 3: Cost Threshold Alert (routed from Flow 2 Step 7)

**Alert recipients:** Owner + Chef (role-split payloads per C-10/C-12)

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | .NET 10 API | Queries Telegram recipients for outlet | PostgreSQL | `SELECT tl.telegramUserId, u.Name, u.Role FROM TelegramLink tl JOIN User u ON u.Id = tl.UserId WHERE u.OutletId = @outletId AND u.Role IN ('Owner','Chef') AND tl.status = 'confirmed'` | [{telegramUserId, name, role}] | No recipients → log and stop | - |
| 2 | .NET 10 API | Builds role-split message payloads | .NET 10 | recipients[], recipe_name, food_cost_pct, gross_margin, cost_per_portion | Owner payload: {recipe_name, food_cost_pct, gross_margin, cost_per_portion}; Chef payload: {recipe_name, cost_per_portion} | - | - |
| 3 | Python FastAPI | Formats messages per role and sends via Telegram Bot API | Telegram | [{telegramUserId, payload}] | Delivery status | 400/403 → PATCH /api/telegram-links/{id}/deactivate | Status='unlinked' |

---

## Flow 4: Recipe Versioning

**Actors:** Owner or Chef
**Entry point:** Recipe detail → "New Version" or "Duplicate"

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Clicks "New Version" on an existing recipe | Vue.js | recipe_id (source) | - | - | - |
| 2 | .NET 10 API | Fetches source recipe + RecipeItems | PostgreSQL | recipe_id | Recipe + RecipeItem[] | - | - |
| 3 | .NET 10 API | Creates new Recipe record | PostgreSQL | name (same), outlet_id (same), version_group_id (SAME as source), version_number = source.version_number + 1, version_label (user input, e.g. "Premium"), cost_threshold_percentage | New Recipe.id | - | - |
| 4 | .NET 10 API | Copies RecipeItems to new version | PostgreSQL | new_recipe_id, RecipeItems[] from source | New RecipeItem[] | User may edit before saving | - |
| 5 | .NET 10 API | Calculates independent cost for new version | ICostCascadeService per ingredient | new Recipe.id | cost_per_portion for new version | - | - |
| 6 | Vue.js | Shows version comparison side by side | Vue.js | Recipe versions grouped by version_group_id | Side-by-side cost cards | - | - |

**Versioning schema on Recipe:**
```
version_group_id    UUID    Groups related recipe versions (same UUID shared across versions)
version_number      int     Sequential within a group (1, 2, 3...)
version_label       text    Human label e.g. "Standard", "Premium", "Winter Edition"
```

All versions are independent recipes with their own RecipeItems — auto-cascade keeps all versions live-costed automatically.
