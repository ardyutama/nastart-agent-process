# Flow 02 - Ingredient Management & Price History

> **Track:** `business-flows/v1/`
> **Authority:** Active implementation guidance for the current single-user v1 build. Read this file with `AGENTS.md`, `.github/context/v1-constraints.md`, and the shared decisions in `business-flows/00-canonical-decisions.md`.
>
> **Service owner:** Nastart.Api
> **Entities:** Ingredient, IngredientPriceHistory, Unit, Category, TelegramLink
>
> **v1 scope:** Ingredient ownership is user-scoped. Price history is append-only. The current price is derived from the latest `IngredientPriceHistory` row.

---

## Active Rules For This Flow

- Ingredient records belong directly to `UserId`
- `Supplier` is not part of v1 ingredient management
- Manual price updates append new `IngredientPriceHistory` rows; they do not update a `current_price` column
- `ICostCascadeService.RecalculateForIngredientAsync(ingredientId, cancellationToken)` is the only cascade entry point
- `IngredientPriceHistory.Source` is exactly `Manual` or `InvoiceScan`
- Price spike alerts route to the single confirmed Telegram link for the current user

---

## Shared Pricing Mechanism

**Current price lookup:**

```sql
SELECT price
FROM IngredientPriceHistory
WHERE ingredient_id = @ingredientId
ORDER BY committed_at DESC
LIMIT 1
```

**Price history rule:** Every new price creates a new history row with:
- `price`
- `unit_size` snapshot
- `source`
- `committed_at`
- `effective_date`

There is no `Ingredient.current_price` field in v1.

---

## Flow 1: List Ingredients With Derived Current Price

**Actor:** Authenticated User
**Entry point:** Ingredient list page in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens ingredient list | Vue.js | JWT | request for ingredients | - | Unauthenticated -> redirect to login |
| 2 | Nastart.Api | Resolves `UserId` from claims | Nastart.Api | JWT | User.id | Invalid token -> 401 | Reject |
| 3 | Nastart.Api | Queries user-owned ingredients | PostgreSQL | User.id | Ingredient rows | - | DB error -> 500 |
| 4 | Nastart.Api | Derives each ingredient's current price from latest price history | PostgreSQL | Ingredient.id | currentPrice or null | No history -> currentPrice=null | - |
| 5 | Nastart.Api | Returns ingredient summaries | Nastart.Api | ingredient list | response DTOs | - | - |
| 6 | Vue.js | Renders list with current prices and thresholds | Vue.js | ingredient summaries | ingredient table/cards | - | Empty state when no ingredients exist |

---

## Flow 2: Create Ingredient With Optional Initial Price

**Actor:** Authenticated User
**Entry point:** Ingredient creation form in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens create ingredient form | Vue.js | - | empty form | - | - |
| 2 | User | Submits ingredient details | Vue.js | name, categoryId, unitId, unitSize, priceSpikeThresholdPct, optional initialPrice, optional effectiveDate | - | - | Client-side validation |
| 3 | Nastart.Api | Validates request and resolves `UserId` | Nastart.Api | JWT + request body | User.id | Invalid request -> 400 | Reject |
| 4 | Nastart.Api | Checks uniqueness for `(UserId, Name)` | PostgreSQL | User.id, name | uniqueness decision | Duplicate -> 409 Conflict | Reject |
| 5 | Nastart.Api | Creates Ingredient record | PostgreSQL | UserId, name, categoryId, unitId, unitSize, priceSpikeThresholdPct | Ingredient.id | - | DB error -> 500 |
| 6 | Nastart.Api | Optionally inserts first `IngredientPriceHistory` row | PostgreSQL | ingredientId, initialPrice, unitSize snapshot, source='Manual', effectiveDate | IngredientPriceHistory.id | No initial price -> skip this step | DB error -> 500 |
| 7 | Nastart.Api | Returns created ingredient with derived current price | Nastart.Api | Ingredient + latest price | created response | - | - |
| 8 | Vue.js | Shows new ingredient in the list | Vue.js | created response | refreshed list | - | - |

**Result:** Ingredient metadata and initial price history stay in sync without introducing mutable current-price storage.

---

## Flow 3: Update Ingredient Metadata

**Actor:** Authenticated User
**Entry point:** Ingredient detail or edit form in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens edit form | Vue.js | ingredientId | current ingredient data | - | Ingredient missing -> not found state |
| 2 | User | Updates metadata | Vue.js | name, categoryId, unitId, unitSize, priceSpikeThresholdPct | - | - | Validation error |
| 3 | Nastart.Api | Loads ingredient and checks ownership | PostgreSQL | ingredientId, User.id | Ingredient | Not found or unauthorized user -> 404 or 403 | Reject |
| 4 | Nastart.Api | Updates ingredient metadata only | PostgreSQL | edited fields | updated Ingredient | - | DB error -> 500 |
| 5 | Nastart.Api | Returns updated ingredient summary | Nastart.Api | updated record | response DTO | - | - |
| 6 | Vue.js | Refreshes ingredient detail view | Vue.js | response DTO | updated UI | - | - |

**Important:** Changing metadata does not rewrite historical price rows. New price history is added only through price-entry flows.

---

## Flow 4: Add Manual Price & Trigger Cascade

**Actor:** Authenticated User
**Entry point:** Ingredient detail page -> add price form

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Submits new manual price | Vue.js | ingredientId, price, optional effectiveDate | - | price must be > 0 | Client-side validation |
| 2 | Nastart.Api | Loads ingredient and checks ownership | PostgreSQL | ingredientId, User.id | Ingredient | Missing ingredient -> 404 | Reject |
| 3 | Nastart.Api | Inserts new `IngredientPriceHistory` row | PostgreSQL | ingredientId, price, unitSize snapshot, source='Manual', committedAt=NOW(), effectiveDate | IngredientPriceHistory.id | - | DB error -> 500 |
| 4 | Nastart.Api | Calculates price change against previous history row | PostgreSQL | latest two history rows, priceSpikeThresholdPct | changePct | No previous price -> skip spike check | - |
| 5 | Nastart.Api | Calls `ICostCascadeService.RecalculateForIngredientAsync(ingredientId, cancellationToken)` | Application service | ingredientId | affected recipe count | Per-recipe failure -> log `CascadeErrorLog` and continue | Service error handling per canonical decision |
| 6 | Nastart.Api | Optionally dispatches price spike alert to confirmed Telegram link | Nastart.Api -> Python FastAPI | ingredient name, old price, new price, changePct, telegram recipient | async alert request | No confirmed link or threshold not crossed -> skip | Delivery failure can deactivate link |
| 7 | Nastart.Api | Returns updated price history summary | Nastart.Api | latest history, cascade result | success response | - | - |
| 8 | Vue.js | Shows latest price and cascade result | Vue.js | response DTO | refreshed ingredient detail | - | - |

**Cascade failure rule:** New price history is never rolled back because a downstream recipe recalculation failed.

---

## Flow 5: View Ingredient Price History

**Actor:** Authenticated User
**Entry point:** Ingredient detail page -> history tab

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens history tab | Vue.js | ingredientId | history request | - | - |
| 2 | Nastart.Api | Loads ingredient and checks ownership | PostgreSQL | ingredientId, User.id | Ingredient | Not found or unauthorized -> 404 or 403 | Reject |
| 3 | Nastart.Api | Queries full price history newest first | PostgreSQL | ingredientId | IngredientPriceHistory[] | - | DB error -> 500 |
| 4 | Nastart.Api | Returns audit-friendly history DTOs | Nastart.Api | price history rows | response DTO | - | - |
| 5 | Vue.js | Renders history table or trend chart | Vue.js | history DTOs | history UI | - | Empty state if only metadata exists |

**History fields:** `price`, `unitSize`, `source`, `committedAt`, `effectiveDate`

---

## Flow 6: Price Spike Alert To Single Linked User

**Trigger:** Runs after any new `IngredientPriceHistory` row when the threshold is exceeded.

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | Nastart.Api | Compares latest two price rows | PostgreSQL | previous price, new price, threshold | changePct | Change does not exceed threshold -> stop | - |
| 2 | Nastart.Api | Looks up confirmed Telegram link for the ingredient owner | PostgreSQL | User.id | TelegramLink | No confirmed link -> stop | - |
| 3 | Nastart.Api | Sends alert request to Python FastAPI | Nastart.Api -> Python FastAPI | telegramUserId, ingredientName, oldPrice, newPrice, changePct | HTTP 202 | Service unreachable -> log retryable failure | Log |
| 4 | Python FastAPI | Delivers Telegram message | Telegram Bot API | telegramUserId, rendered message | delivery status | 400 or 403 -> call deactivate path | Link deactivated |

**v1 recipient model:** The only recipient is the single linked user. There is no team recipient query.

---

## Route Summary

| Capability | Route pattern | Auth |
|---|---|---|
| List ingredients | `GET /api/ingredients` | `.RequireAuthorization()` |
| Get ingredient by id | `GET /api/ingredients/{id}` | `.RequireAuthorization()` |
| Create ingredient | `POST /api/ingredients` | `.RequireAuthorization()` |
| Update ingredient metadata | `PUT /api/ingredients/{id}` | `.RequireAuthorization()` |
| Add manual price | `POST /api/ingredients/{id}/prices` | `.RequireAuthorization()` |
| View price history | `GET /api/ingredients/{id}/prices` | `.RequireAuthorization()` |

---

## Not In This v1 Flow

- Outlet-scoped ingredient routes
- Role-based ingredient permissions
- Supplier comparison views
- Team-wide price-spike recipient routing
- Mutable `current_price` on `Ingredient`