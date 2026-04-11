# Canonical Decisions — Recipe Cost Management App

> These are the locked, authoritative design decisions validated by the Lead Architect across all 5 business flows. Every implementation file, API contract, and database schema MUST conform to these. Do not deviate without updating all flows accordingly.

---

## C-1 — Cascade Service Interface

**Value:** `ICostCascadeService.RecalculateForIngredient(ingredientId)`

- No price parameter — the service fetches current price internally via C-3
- Single entry point called by ALL sources: manual price update, invoice scan confirmation
- Never implement a separate recalculation outside this interface

---

## C-2 — Cost-Per-Portion Formula

```
cost_per_portion = SUM(
  (current_price / unit_size) * quantity * (1 / yield_percentage)
) / portion_count
```

- `current_price` — fetched via C-3
- `unit_size` — from `Ingredient.unit_size`
- `quantity` — from `RecipeItem.quantity`
- `yield_percentage` — from `RecipeItem.yield_percentage`
- `portion_count` — from `Recipe.portion_count`

---

## C-3 — Current Price Lookup

```sql
SELECT price
FROM IngredientPriceHistory
WHERE ingredient_id = @id
ORDER BY committed_at DESC
LIMIT 1
```

Current price is ALWAYS derived from the most recent `IngredientPriceHistory` record. There is NO `current_price` column on `Ingredient`.

---

## C-4 — IngredientPriceHistory Timestamps

| Field | Type | Description |
|---|---|---|
| `committed_at` | `timestamptz` | System-set on INSERT (`NOW()`). Used for ordering. **Non-editable.** |
| `effective_date` | `date` | User-editable. Defaults to `Invoice.invoiceDate ?? NOW()`. Represents the business date the price was effective. |

Use `ORDER BY committed_at DESC` for all "current price" lookups. Never conflate with `effective_date`.

---

## C-5 — Cascade Failure Isolation

- Per-recipe cascade errors **NEVER** roll back `IngredientPriceHistory`
- Price history is append-only and permanent once committed
- Failed recipe recalculations are caught, logged to `CascadeErrorLog { ingredientId, recipeId, error, timestamp }`, and skipped
- The cascade runs **OUTSIDE** the price commit transaction

---

## C-6 — TelegramLink Entity Schema

```
TelegramLink {
  id               UUID          PK
  userId           UUID          FK → User.id
  codeHash         varchar(64)   SHA-256 hash of the one-time link code (NOT plaintext)
  expiresAt        timestamptz   15-minute TTL from code generation
  status           enum          'pending' | 'confirmed' | 'unlinked'
  telegramUserId   bigint        Telegram user_id (= chat_id for private chats)
  telegramUsername varchar(255)  nullable
  linkedAt         timestamptz   nullable — set when status → 'confirmed'
}
```

**There is NO `telegram_user_id` column on the `User` entity.**

---

## C-7 — Telegram Deactivation Trigger

Both error types trigger deactivation:
- `400 Bad Request` (invalid chat_id)
- `403 Forbidden` (user blocked the bot)

**Action:** `PATCH /api/telegram-links/{id}/deactivate` → sets `TelegramLink.status = 'unlinked'`

Nothing is written to the `User` entity.

---

## C-8 — Canonical Roles

Exactly four roles, no others:

| Role | Permissions Summary |
|---|---|
| `Owner` | Full access: cost_per_portion, food_cost_pct, gross_margin, margins, all alerts |
| `Chef` | cost_per_portion only — no margins, no food_cost_pct |
| `Procurement` | cost_per_portion + supplier prices — no food_cost_pct |
| `Viewer` | cost_per_portion only — read-only |

Role stripping is **server-side only**. Vue.js never receives `food_cost_pct` or `gross_margin` for non-Owner roles.

---

## C-9 — Alert Threshold Location

`Recipe.cost_threshold_percentage` — **per recipe, not per outlet.**

Threshold check: `Recipe.food_cost_pct > Recipe.cost_threshold_percentage`

There is NO `food_cost_alert_threshold` on `Outlet`.

---

## C-10 — Role-Based Data Visibility (All Responses & Alerts)

| Role | cost_per_portion | food_cost_pct | gross_margin | supplier_prices |
|---|---|---|---|---|
| Owner | ✅ | ✅ | ✅ | ✅ |
| Chef | ✅ | ❌ | ❌ | ❌ |
| Procurement | ✅ | ❌ | ❌ ✅ (own context) | ✅ |
| Viewer | ✅ | ❌ | ❌ | ❌ |

Push alert payloads must be built separately per role — absent fields are not null, they are not present in the object.

---

## C-11 — Price Spike Alert Recipients

`Owner` + `Procurement`

---

## C-12 — Food Cost Threshold Alert Recipients

`Owner` + `Chef`

With role-split payloads:
- Owner payload: `{ recipe_name, food_cost_pct, gross_margin, cost_per_portion }`
- Chef payload: `{ recipe_name, cost_per_portion }` — food_cost_pct and gross_margin absent

---

## C-13 — IngredientPriceHistory.source Enum

Exactly two values (case-sensitive):
- `'Manual'`
- `'InvoiceScan'`

---

## C-14 — TelegramLink Chat ID Column

`telegramUserId` — this is Telegram's `user_id` (which equals `chat_id` for private chats).

Not `chatId`, not `TelegramChatId`, not `telegram_chat_id`. Use `telegramUserId` everywhere.

---

## Advisory Notes (non-blocking — implementation team awareness)

1. **Recipient resolution ownership**: .NET 10 API resolves TelegramLink recipients and passes `[{telegramUserId, userName}]` to Python FastAPI. FastAPI does NOT query the database directly for recipients.

2. **Alert scope consistency**: Price ceiling and food cost threshold alerts are outlet-scoped. Verify price-spike notifications use `outletId` (not `companyId`) for consistent routing.

3. **Duplicate cascade calls**: When an invoice has two line items for the same ingredient, de-duplicate the `ingredientId` list before iterating cascade calls to avoid redundant recalculations.

4. **Synchronous cascade latency**: The cascade runs synchronously in v1. Acceptable for MVP. Plan async background processing for Phase 4+.

5. **`price_spike_threshold_pct` column**: Stored on `Ingredient` entity. Default 10%, configurable per ingredient.
