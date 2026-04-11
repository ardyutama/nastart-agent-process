# Business Process Flow — Telegram Bot Interaction
**Application**: Recipe Cost Management SaaS
**Flow group**: 04 — Telegram Bot
**Date**: 2026-04-02
**Status**: Revised — defects 5-A through 5-F corrected

---

## Architectural Prerequisites

| Concern | Decision |
|---|---|
| All Telegram bot logic | Lives exclusively in Python FastAPI (python-telegram-bot, webhook mode) |
| Bot access mode | Read-only and alert-only — zero write operations permitted via Telegram |
| Service-to-service auth | Internal calls between .NET 10 API and Python FastAPI use a shared `X-Internal-API-Key` header (not user JWT) |
| User resolution | Every bot command starts by resolving `telegramUserId` → `TelegramLink` (status=`confirmed`) → `User` → `Role` → `Outlet` via .NET 10 API |
| Role-based filtering | Applied in .NET 10 API before data leaves the backend; Python FastAPI is a formatting/delivery layer only |
| `TelegramLink` entity | Separate entity with fields: `id`, `userId` (FK→User), `codeHash` (SHA-256 of the plaintext code, stored only as hash), `expiresAt`, `status` enum (`pending`/`confirmed`/`unlinked`), `telegramUserId`, `telegramUsername`, `linkedAt`. **No** `telegram_user_id` column exists on `User`. |
| Outlet scoping | A User belongs to exactly one Outlet; `outlet_id` is always sourced from the User record, never from bot input |
| Telegram rate limits | Python FastAPI respects Telegram API limits: 30 messages/second globally, 1 message/second per chat |

---

## Role-Based Data Visibility Matrix

| Data Field | Owner | Chef | Procurement | Viewer |
|---|---|---|---|---|
| cost_per_portion | ✅ | ✅ | ✅ | ✅ |
| food_cost_pct | ✅ | ❌ | ❌ | ❌ |
| margin | ✅ | ❌ | ❌ | ❌ |
| ingredient breakdown (name + qty) | ✅ | ✅ | ✅ | ❌ |
| supplier name + unit price | ✅ | ❌ | ✅ | ❌ |
| flagged high-cost status (% shown) | ✅ | ⚠️ label only | ❌ | ❌ |
| recent price change count | ✅ | ❌ | ✅ | ❌ |

---

## Flow 1: Account Linking (`/link XXXX`)

**Pre-condition**: User is authenticated in the Vue.js web app. `User` record exists in PostgreSQL. No `TelegramLink` with `status = 'confirmed'` exists for this user.
**Post-condition**: `TelegramLink.status = 'confirmed'`, `TelegramLink.telegramUserId` and `TelegramLink.linkedAt` are set. Nothing is written to the `User` entity. User can issue bot commands.

---

### Step 1
| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Navigates to Settings → Telegram Linking page in the Vue.js web app |
| **Component** | Vue.js |
| **Data IN** | Authenticated session (JWT token in browser memory) |
| **Data OUT** | Telegram Linking page rendered |
| **Decision point** | No |
| **Error case** | JWT invalid or expired → middleware redirects to login |

---

### Step 2
| Field | Value |
|---|---|
| **Actor** | Vue.js |
| **Action** | On page mount, calls `GET /api/users/me/telegram-status` to check current linking state |
| **Component** | Vue.js → .NET 10 API → PostgreSQL |
| **Data IN** | `Authorization: Bearer {jwt}` |
| **Data OUT** | `{ linked: bool, telegramUsername?: string (masked), outlet_name: string }` — `telegramUsername` is present only when `linked = true`, sourced from `TelegramLink.telegramUsername` |
| **Decision point** | YES — if `linked = true`: show current linked status + "Unlink" option and do not proceed to code generation. If `linked = false`: render "Generate Code" button and proceed to Step 3. |
| **Error case** | 401 → redirect to login; 500 → show generic error banner |

---

### Step 3
| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Clicks "Generate Link Code" button |
| **Component** | Vue.js → .NET 10 API |
| **Data IN** | `POST /api/users/me/telegram-link` with `Authorization: Bearer {jwt}` |
| **Data OUT** | `{ code: "A3K9MX", expires_at: "2026-04-02T10:15:00Z" }` |
| **Decision point** | No |
| **Error case** | 429 (rate limit — code generated within last 60 seconds) → Vue.js shows "Please wait before generating a new code" with countdown |

---

### Step 4
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Generates cryptographically random 6-character alphanumeric code using `RandomNumberGenerator`. Computes `codeHash = SHA-256(code)`. Invalidates (sets `status = 'unlinked'`) any existing `pending` `TelegramLink` for this `userId`. Creates new `TelegramLink` record. Plaintext code is discarded after returning it to the caller — only `codeHash` is stored. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `userId` from JWT claims |
| **Data OUT** | `TelegramLink { id, userId, codeHash, expiresAt: now + 15 min, status: 'pending' }`. Returns `{ code: "A3K9MX", expires_at }` to Vue.js. |
| **Decision point** | No |
| **Error case** | DB write failure → 500; Vue.js shows generic error. Code not returned to user. |

---

### Step 5
| Field | Value |
|---|---|
| **Actor** | Vue.js |
| **Action** | Renders the link code in large display text with a 15-minute countdown timer. Renders a "Open Telegram Bot" deep link (`https://t.me/{BotUsername}`). Displays instruction: "Send `/link A3K9MX` to the bot before the timer expires." |
| **Component** | Vue.js |
| **Data IN** | `{ code: "A3K9MX", expires_at }` |
| **Data OUT** | UI showing code, countdown, deep link button |
| **Decision point** | No |
| **Error case** | None (purely presentational) |

---

### Step 6
| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Opens Telegram, navigates to the bot, sends the message `/link A3K9MX` |
| **Component** | Telegram (external) |
| **Data IN** | User keyboard input |
| **Data OUT** | Telegram message delivered to bot webhook |
| **Decision point** | No |
| **Error case** | User sends expired or wrong code — handled in Step 9 |

---

### Step 7
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Telegram webhook handler) |
| **Action** | Telegram delivers `Update` object to the webhook endpoint. Bot parses message text. Extracts `telegramUserId` from `update.message.from.id`. Extracts `telegramUsername` from `update.message.from.username`. Extracts `code` from message text after `/link `. |
| **Component** | Python FastAPI |
| **Data IN** | Telegram `Update` object: `{ message.from.id (telegramUserId), message.from.username, message.text: "/link A3K9MX" }` |
| **Data OUT** | `{ telegramUserId: string, telegramUsername: string, code: "A3K9MX" }` |
| **Decision point** | YES — if message text does not match `/link {code}` pattern (e.g., `/link` with no code, or extra tokens) → bot replies: "Usage: `/link CODE` — generate your code from the web app." Stop. |
| **Error case** | Malformed `Update` payload (missing `from.id`) → discard silently; Telegram guarantees `from` on private messages |

---

### Step 8
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Calls `POST /api/telegram/link` on .NET 10 API to validate and redeem the code |
| **Component** | Python FastAPI → .NET 10 API |
| **Data IN** | `{ code: "A3K9MX", telegramUserId: "987654321", telegramUsername: "john_doe" }` + `X-Internal-API-Key` header |
| **Data OUT** | HTTP response (200 with user data, or 4xx with error code) |
| **Decision point** | No |
| **Error case** | .NET 10 API unreachable → bot replies: "Service temporarily unavailable. Please try again in a moment." |

---

### Step 9
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Computes `hash = SHA-256(code)`. Queries `TelegramLink` WHERE `codeHash = hash`. Validates all conditions in order. If all pass: sets `TelegramLink.status = 'confirmed'`, `TelegramLink.telegramUserId = telegramUserId`, `TelegramLink.telegramUsername = telegramUsername`, `TelegramLink.linkedAt = NOW()`. Nothing is written to the `User` entity. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `{ code: "A3K9MX", telegramUserId: "987654321", telegramUsername: "john_doe" }` |
| **Data OUT** | On success: `{ user_id, role_name: "Owner", outlet_name: "The Bistro" }` |
| **Decision point** | YES — validation branches (evaluated in order): <br>① `codeHash` not found → 404 `invalid_code` <br>② `expiresAt < now()` → 410 `code_expired` <br>③ `status != 'pending'` (already confirmed or unlinked) → 409 `code_already_used` <br>④ `User.is_active = false` (via join) → 403 `user_inactive` <br>⑤ Another `TelegramLink` with `telegramUserId = this value` AND `status = 'confirmed'` exists → 409 `telegram_id_already_linked` <br>⑥ All valid → proceed to write |
| **Error case** | DB write failure after validation → 500; `TelegramLink` state not changed |

---

### Step 10
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Telegram Bot) |
| **Action** | Receives response from Step 9. Sends confirmation or error message to the user via Telegram Bot API `sendMessage`. |
| **Component** | Python FastAPI → Telegram Bot API |
| **Data IN** | HTTP response from .NET 10 API, `telegramUserId` |
| **Data OUT** | Telegram message delivered to user |
| **Decision point** | YES — maps each error code to a user-friendly message: <br>`invalid_code` → "That code is not valid. Please generate a new one from the web app." <br>`code_expired` → "That code has expired (15 min limit). Generate a new one from the web app." <br>`code_already_used` → "That code has already been used. Generate a new one if needed." <br>`user_inactive` → "Your account is inactive. Contact your administrator." <br>`telegram_id_already_linked` → "This Telegram account is already linked to another user." <br>Success → "✅ Account linked! Role: Owner \| Outlet: The Bistro. Send `/cost [dish name]` to get started." |
| **Error case** | Telegram API delivery failure → log error; the `TelegramLink` was already confirmed in Step 9 — linking succeeded even if confirmation message failed |

---

## Flow 2: Query — `/cost [dish name]`

**Pre-condition**: User has a linked Telegram account — a `TelegramLink` record with `status = 'confirmed'` and matching `telegramUserId` exists. User belongs to exactly one `Outlet`.
**Post-condition**: User receives role-filtered cost data for the named dish, or a not-found message with suggestions.

---

### Step 1
| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Sends `/cost pasta bake` in Telegram |
| **Component** | Telegram (external) |
| **Data IN** | Keyboard input |
| **Data OUT** | Telegram `Update` delivered to bot webhook |
| **Decision point** | No |
| **Error case** | None |

---

### Step 2
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Telegram webhook handler) |
| **Action** | Parses `Update`. Identifies command = `cost`. Extracts `dish_name` = everything after `/cost ` (trimmed). Extracts `telegramUserId` from `update.message.from.id`. |
| **Component** | Python FastAPI |
| **Data IN** | `Update { message.from.id, message.text: "/cost pasta bake" }` |
| **Data OUT** | `{ telegramUserId, dish_name: "pasta bake" }` |
| **Decision point** | YES — if `dish_name` is empty after trim (user sent just `/cost`) → bot replies: "Please specify a dish name: `/cost [dish name]`". Stop. |
| **Error case** | None |

---

### Step 3
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Calls `GET /api/telegram/users/{telegramUserId}` on .NET 10 API to resolve user context |
| **Component** | Python FastAPI → .NET 10 API → PostgreSQL |
| **Data IN** | `telegramUserId` + `X-Internal-API-Key` header |
| **Data OUT** | `{ user_id, telegramLinkId, role: "Chef", outlet_id, outlet_name: "The Bistro" }` |
| **Decision point** | YES — if no `TelegramLink` with `status = 'confirmed'` AND `telegramUserId = this value` is found → bot replies: "Your Telegram account is not linked. Visit the web app to link your account." Stop. If user found but no `outlet_id` assigned → bot replies: "Your account is not assigned to an outlet. Contact your administrator." Stop. |
| **Error case** | .NET 10 API unreachable → bot replies: "Service temporarily unavailable. Please try again." |

---

### Step 4
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Queries `TelegramLink` WHERE `telegramUserId = @telegramUserId` AND `status = 'confirmed'`. Joins to `User` (via `TelegramLink.userId`), `Role` (via `User.role_id`), and `Outlet` (via `User.outlet_id`). Returns user context. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `telegramUserId` |
| **Data OUT** | `{ user_id, telegramLinkId, role_name, outlet_id, outlet_name }` |
| **Decision point** | No |
| **Error case** | DB error → 500 |

---

### Step 5
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Calls `GET /api/telegram/recipes/cost?outlet_id={outlet_id}&dish_name=pasta+bake&user_id={user_id}` on .NET 10 API |
| **Component** | Python FastAPI → .NET 10 API |
| **Data IN** | `{ outlet_id, dish_name: "pasta bake", user_id }` + `X-Internal-API-Key` |
| **Data OUT** | HTTP response: `CostResponse` DTO or 4xx error |
| **Decision point** | No |
| **Error case** | .NET 10 API unreachable → bot replies with generic error |

---

### Step 6
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Executes case-insensitive partial-match query: `Recipe` WHERE `outlet_id = ?` AND `LOWER(name) LIKE '%pasta bake%'` AND `is_active = true`. If multiple matches, returns the closest match by Levenshtein distance. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `{ outlet_id, dish_name_query: "pasta bake" }` |
| **Data OUT** | `Recipe { id, name, portion_count, sell_price, category_id, cost_threshold_percentage }` |
| **Decision point** | YES — if zero matches: query for top 3 `Recipe.name` values in the outlet with shortest Levenshtein distance to the query string. Return `404` with `{ matched: false, dish_name_query, suggestions: ["Pasta Bake Classic", "Baked Pasta al Forno", ...] }`. Python FastAPI (Step 9) handles the 404. |
| **Error case** | DB error → 500 |

---

### Step 7
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Calculates `cost_per_portion` using the canonical formula. For each `RecipeItem` belonging to the `Recipe`: fetches the latest `IngredientPriceHistory` record for `ingredient_id` using `SELECT price FROM IngredientPriceHistory WHERE ingredient_id = @id ORDER BY committed_at DESC LIMIT 1`. Applies formula per item: `item_cost = (unit_price / unit_size) * quantity * (1 / yield_pct)`. Sums all item costs. `cost_per_portion = SUM(item_costs) / Recipe.portion_count`. Calculates `food_cost_pct = cost_per_portion / sell_price × 100`. Calculates `gross_margin = sell_price − cost_per_portion`. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `RecipeItem[]` (each with `quantity`, `yield_pct`), `Ingredient` (each with `unit_size`), `IngredientPriceHistory` (latest per ingredient ordered by `committed_at DESC`), `Recipe.portion_count`, `Recipe.sell_price` |
| **Data OUT** | `{ cost_per_portion: decimal, food_cost_pct: decimal\|null, gross_margin: decimal\|null, items: [{ ingredient_id, ingredient_name, unit_price, unit_size, supplier_name, supplier_id, quantity, unit, yield_pct, line_cost, price_missing: bool }] }` |
| **Decision point** | YES — if any `RecipeItem` has no matching `IngredientPriceHistory` row → set `line_cost = 0`, `price_missing = true` for that item; include `partial_cost_warning: true` in response. If `sell_price = NULL` → `food_cost_pct = null`, `gross_margin = null`. If `Recipe.portion_count = 0` → return 422 `invalid_recipe_portion_count`. |
| **Error case** | DB error → 500 |

---

### Step 8
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Looks up `User.role_id → Role.name`. Applies role-based field masking to the full cost data before returning the response. |
| **Component** | .NET 10 API |
| **Data IN** | Full cost data from Step 7, `User.role` |
| **Data OUT** | Role-filtered `CostResponse` DTO: <br>**Owner**: `cost_per_portion`, `food_cost_pct`, `gross_margin`, `items[]` (with `supplier_name`, `unit_price`, `line_cost`), `partial_cost_warning` <br>**Chef**: `cost_per_portion`, `items[]` (with `ingredient_name`, `quantity`, `unit`, `line_cost`; `supplier_name`, `unit_price`, `food_cost_pct`, `gross_margin` omitted), `partial_cost_warning` <br>**Procurement**: `cost_per_portion`, `items[]` (with `ingredient_name`, `supplier_name`, `unit_price`, `quantity`, `unit`; `line_cost` included; `food_cost_pct` and `gross_margin` omitted), `partial_cost_warning` <br>**Viewer**: `cost_per_portion` only, no `items[]` |
| **Decision point** | NO — unknown role defaults to Viewer (minimum visibility) |
| **Error case** | None (pure transformation) |

---

### Step 9
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Telegram Bot) |
| **Action** | Receives `CostResponse`. Formats into a Telegram message. Sends via `sendMessage` to `telegramUserId`. |
| **Component** | Python FastAPI → Telegram Bot API |
| **Data IN** | `CostResponse` DTO, `telegramUserId`, `role`, `outlet_name` |
| **Data OUT** | Telegram message sent to user |
| **Decision point** | YES — if 404 received from Step 6 → reply: "Dish 'pasta bake' not found in The Bistro. Did you mean: Pasta Bake Classic, Baked Pasta al Forno?" <br>If `partial_cost_warning = true` → append: "⚠️ Note: one or more ingredient prices are missing — cost shown may be incomplete." <br>If `food_cost_pct = null` (no sell price) → omit food cost and margin fields without error. <br>**Example message (Chef)**: "🍽 *Pasta Bake Classic* \| The Bistro\nCost/portion: $4.20\n— Semolina flour 200g: $0.84\n— Cream 100ml: $0.55\n— Parmesan 30g: $1.20\n_(prices as of today)_" <br>**Example message (Owner)**: "🍽 *Pasta Bake Classic* \| The Bistro\nCost/portion: $4.20 \| Food cost: 28.0% \| Margin: $10.80" |
| **Error case** | Telegram API failure → log; no retry (stateless) |

---

## Flow 3: Natural Language Query (LLM Fallback)

**Pre-condition**: User has a linked Telegram account. User sends a free-text message (not a `/command`).
**Post-condition**: User receives the same role-filtered response as the matching command flow, or a helpful command guide if intent is unresolvable.

---

### Step 1
| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Sends free-text message: "how much does the pasta bake cost tonight?" |
| **Component** | Telegram (external) |
| **Data IN** | User keyboard input |
| **Data OUT** | Telegram `Update` delivered to bot webhook |
| **Decision point** | No |
| **Error case** | None |

---

### Step 2
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Telegram webhook handler) |
| **Action** | Receives `Update`. Checks whether `message.text` starts with `/`. It does not. Routes to the NLP/LLM handler. |
| **Component** | Python FastAPI |
| **Data IN** | `Update { message.from.id, message.text: "how much does the pasta bake cost tonight?" }` |
| **Data OUT** | `{ telegramUserId, raw_message_text }` routed to LLM handler |
| **Decision point** | YES — if message starts with `/` → route to command dispatcher (not this flow). If message is empty or contains only whitespace → discard. Otherwise → continue to Step 3. |
| **Error case** | None |

---

### Step 3
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Calls `GET /api/telegram/users/{telegramUserId}` on .NET 10 API to resolve user context (identical to Flow 2 Steps 3–4) |
| **Component** | Python FastAPI → .NET 10 API → PostgreSQL |
| **Data IN** | `telegramUserId` + `X-Internal-API-Key` |
| **Data OUT** | `{ user_id, telegramLinkId, role, outlet_id, outlet_name, active_recipe_names: string[] }` — .NET 10 API includes the list of active `Recipe.name` values for the outlet to support LLM entity extraction |
| **Decision point** | YES — if not linked → "Your Telegram account is not linked. Visit the web app to link your account." Stop. |
| **Error case** | .NET 10 API unreachable → generic error |

---

### Step 4
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (LLM module) |
| **Action** | Constructs an LLM prompt with system context: available intent types, `outlet_name`, `role` name (not sensitive data), and `active_recipe_names[]` (to ground entity extraction against real dish names). Sends the user's raw message text to the configured LLM. Expects a structured JSON response. |
| **Component** | Python FastAPI → LLM service (e.g., OpenAI, Anthropic, or locally hosted model) |
| **Data IN** | `{ message_text, outlet_name, role_name, active_recipe_names[] }` |
| **Data OUT** | `{ intent: "cost_query" \| "ingredients_query" \| "summary_query" \| "alerts_query" \| "unknown", entity: { dish_name?: string }, confidence: float (0–1) }` |
| **Decision point** | No (LLM returns structured JSON forced via function calling or JSON mode) |
| **Error case** | LLM service timeout or non-parseable response → fall through to Step 6 (clarification message). Do NOT bubble LLM errors to user as raw text. |

---

### Step 5
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Evaluates `intent` and `confidence`. Routes to the appropriate sub-flow. |
| **Component** | Python FastAPI |
| **Data IN** | `{ intent, entity, confidence }` |
| **Data OUT** | Routed to sub-flow with extracted entity, OR to Step 6 |
| **Decision point** | YES — routing table: <br>① `intent = "cost_query"` AND `confidence ≥ 0.70` AND `entity.dish_name` present → execute **Flow 2 Steps 5–9** using `entity.dish_name` as `dish_name`. Role-based filtering applies identically. <br>② `intent = "ingredients_query"` AND `confidence ≥ 0.70` → execute `/ingredients` sub-flow (not detailed in this document) with `entity.dish_name`. <br>③ `intent = "summary_query"` AND `confidence ≥ 0.70` → execute **Flow 5 Steps 4–7**. <br>④ `intent = "alerts_query"` AND `confidence ≥ 0.70` → execute `/alerts` sub-flow (not detailed in this document). <br>⑤ `intent = "unknown"` OR `confidence < 0.70` → continue to Step 6. |
| **Error case** | None — decision is deterministic on LLM output shape |

---

### Step 6 (fallback only — unresolvable intent)
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Telegram Bot) |
| **Action** | Sends a structured help message to the user |
| **Component** | Python FastAPI → Telegram Bot API |
| **Data IN** | `telegramUserId` |
| **Data OUT** | Telegram message: "I'm not sure what you're looking for. Try one of these commands:\n`/cost [dish name]` — get cost for a dish\n`/ingredients [dish name]` — list ingredients and quantities\n`/summary` — food cost overview for your outlet\n`/alerts` — see active threshold alerts" |
| **Decision point** | No |
| **Error case** | Telegram delivery failure → log |

> **Access control note**: The LLM fallback path applies **identical** role-based field masking as the direct command paths. The LLM receives only `role_name` (a non-sensitive label) and `active_recipe_names` (dish names only, not costs). No financial data is passed to the LLM.

---

## Flow 4: Proactive Push Alert

**Trigger sources** (consistent with Recipe Costing and Invoice Scanning flows):
- **Trigger A** — Food cost threshold breach: a `Recipe`'s `food_cost_pct` exceeds `Recipe.cost_threshold_percentage` (per-recipe threshold) after a recipe cost recalculation cascade initiated by an `IngredientPriceHistory` insert
- **Trigger B** — Ingredient price spike: a new `IngredientPriceHistory` record's price deviates from the previous record by more than `Ingredient.price_spike_threshold_pct`

Both triggers fire from within the .NET 10 API **after** it processes a price update webhook from Python FastAPI (Invoice Scanning flow). This makes Flow 4 a downstream continuation of the Invoice Scanning flow.

**Pre-condition**: At least one `User` in the affected `Outlet` has a `TelegramLink` with `status = 'confirmed'` and holds a role configured to receive the alert type.
**Post-condition**: All eligible, Telegram-linked `User`s in the `Outlet` receive a role-filtered alert message. Delivery is logged.

---

### Step 1
| Field | Value |
|---|---|
| **Actor** | .NET 10 API (Recipe Cost Recalculation engine) |
| **Action** | Following an `IngredientPriceHistory` insert (triggered by the invoice scan webhook from Python FastAPI), calls `ICostCascadeService.RecalculateForIngredient(ingredientId)`. This recalculates `cost_per_portion` and `food_cost_pct` for every active `Recipe` in the affected `Outlet` that references the updated `Ingredient` via `RecipeItem`. Current price is sourced from `SELECT price FROM IngredientPriceHistory WHERE ingredient_id = @id ORDER BY committed_at DESC LIMIT 1`. Formula applied per recipe item: `item_cost = (unit_price / unit_size) * quantity * (1 / yield_pct)`. `cost_per_portion = SUM(item_costs) / Recipe.portion_count`. Updated values are stored. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `ingredientId` from the `IngredientPriceHistory` insert event |
| **Data OUT** | `RecipeCostSnapshot[] { recipe_id, recipe_name, outlet_id, old_cost_per_portion, new_cost_per_portion, old_food_cost_pct, new_food_cost_pct, sell_price, old_gross_margin, new_gross_margin, cost_threshold_percentage }` |
| **Decision point** | No — recalculation runs for all affected recipes unconditionally |
| **Error case** | Recalculation failure for a specific `Recipe` → log error for that recipe; continue processing remaining recipes; **per-recipe errors never roll back the `IngredientPriceHistory` insert** |

---

### Step 2
| Field | Value |
|---|---|
| **Actor** | .NET 10 API (Alert Evaluation engine) |
| **Action** | For each recipe in the snapshot: evaluates Trigger A using `Recipe.cost_threshold_percentage`. For the updated ingredient itself: evaluates Trigger B. Records each qualifying breach as an `AlertEvent`. |
| **Component** | .NET 10 API → PostgreSQL (reads `Recipe.cost_threshold_percentage` per recipe, `Ingredient.price_spike_threshold_pct`) |
| **Data IN** | `RecipeCostSnapshot[]` (each includes `cost_threshold_percentage`), `IngredientPriceHistory` (new + old price), `Ingredient.price_spike_threshold_pct` |
| **Data OUT** | `AlertEvent[] { alert_type: "food_cost_threshold" \| "price_spike", outlet_id, recipe_id?, recipe_name?, ingredient_id, ingredient_name, supplier_id, supplier_name, old_value, new_value, threshold, invoice_id }` |
| **Decision point** | YES — Trigger A: `new_food_cost_pct > recipe.cost_threshold_percentage` AND `old_food_cost_pct ≤ recipe.cost_threshold_percentage` (alert only on the crossing event, not on every subsequent recalculation). Trigger B: `\|new_price − old_price\| / old_price > Ingredient.price_spike_threshold_pct`. If neither condition is met for any recipe or ingredient → exit flow (no alert generated). |
| **Error case** | Missing `Ingredient.price_spike_threshold_pct` → use system default (10%); log warning. `Recipe.cost_threshold_percentage` is a required non-null field; if missing for a recipe → skip Trigger A for that recipe and log warning. |

---

### Step 3
| Field | Value |
|---|---|
| **Actor** | .NET 10 API (Alert Routing engine) |
| **Action** | For each `AlertEvent`, determines which `Role`s are eligible recipients: `"food_cost_threshold"` → `[Owner, Chef]`; `"price_spike"` → `[Owner, Procurement]`. Queries: `SELECT u.id, tl.id AS telegramLinkId, tl.telegramUserId, r.name AS role_name, u.display_name FROM User u JOIN TelegramLink tl ON tl.userId = u.id JOIN Role r ON r.id = u.role_id WHERE u.outlet_id = @outletId AND r.name IN @eligibleRoles AND tl.status = 'confirmed' AND u.is_active = true`. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `AlertEvent[]`, `outlet_id`, `eligible_roles[]` per alert type |
| **Data OUT** | Per-alert recipient list: `[{ user_id, telegramLinkId, telegramUserId, role_name, display_name }]` |
| **Decision point** | YES — if the recipient list for an `AlertEvent` is empty (no linked users in required roles) → log: "Alert suppressed: no Telegram-linked recipients for outlet {id}, alert type {type}". Skip that `AlertEvent`. Continue processing remaining events. |
| **Error case** | DB error during User query → log; skip that `AlertEvent`; do not crash the recalculation pipeline |

---

### Step 4
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | For each `(AlertEvent, recipient)` pair: builds a role-filtered `AlertPayload` with two distinct payload shapes depending on role. Applies server-side role masking — fields absent from the Chef payload are **not included at all** (not nulled). |
| **Component** | .NET 10 API |
| **Data IN** | `AlertEvent`, `recipient.role_name`, `recipient.telegramLinkId`, full cost snapshot |
| **Data OUT** | `AlertPayload { telegramUserId, telegramLinkId, alert_type, role_name, message_data }` where `message_data` shape is strictly role-masked: <br><br>**`food_cost_threshold` alert:** <br>— Owner payload `message_data`: `{ recipe_name, food_cost_pct, gross_margin, cost_per_portion }` <br>— Chef payload `message_data`: `{ recipe_name, cost_per_portion }` ← `food_cost_pct` and `gross_margin` are **absent** (field omitted entirely, not null) <br><br>**`price_spike` alert:** <br>— Owner payload `message_data`: `{ ingredient_name, old_price, new_price, price_change_pct, supplier_name, affected_recipe_count, outlet_name }` <br>— Procurement payload `message_data`: `{ ingredient_name, old_price, new_price, price_change_pct, supplier_name, affected_recipe_count, outlet_name }` |
| **Decision point** | No |
| **Error case** | None (pure transformation) |

---

### Step 5
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Calls `POST /internal/alerts/send` on Python FastAPI with the full `AlertPayload[]` list. Uses `X-Internal-API-Key` header. Persists a pending `AlertDeliveryRecord` in PostgreSQL before the call (optimistic write — will be updated with outcome in Step 8). |
| **Component** | .NET 10 API → Python FastAPI |
| **Data IN** | `AlertPayload[]` |
| **Data OUT** | `{ accepted: true, alert_count: N }` from Python FastAPI |
| **Decision point** | No |
| **Error case** | Python FastAPI unreachable → retry up to 3 times with exponential backoff (1s, 2s, 4s). After all retries exhausted → persist payloads as `alert_status = "failed"` in `AlertDeliveryRecord` for a scheduled retry job. Log warning. Do not block the calling request thread — this must be a background/async dispatch. |

---

### Step 6
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Alert handler) |
| **Action** | Receives `AlertPayload[]`. For each payload: formats a human-readable Telegram message based on `alert_type` and `role_name` using only the fields present in `message_data` (no assumptions about absent fields). Applies Telegram MarkdownV2 formatting. |
| **Component** | Python FastAPI |
| **Data IN** | `AlertPayload[]` |
| **Data OUT** | `SendQueue[] { telegramUserId, telegramLinkId, message_text }` |
| **Decision point** | No |
| **Error case** | Malformed `AlertPayload` (missing required fields) → log and skip that payload; increment `failed_count` in response |

---

**Example formatted messages (Step 6 output):**

> **Owner — food_cost_threshold** (message_data contains: recipe_name, food_cost_pct, gross_margin, cost_per_portion):
> `🚨 Food Cost Alert — The Bistro`
> `📍 Pasta Bake Classic: food cost 38.5% (threshold: 35%)`
> `💸 Cost/portion: $5.15 | Margin: $6.95`

> **Chef — food_cost_threshold** (message_data contains: recipe_name, cost_per_portion ONLY):
> `⚠️ Cost Alert — Pasta Bake Classic`
> `Cost/portion: $5.15`

> **Owner / Procurement — price_spike**:
> `📦 Price Spike — Semolina Flour`
> `New price: $3.80/kg (was $3.10/kg, +22.6%)`
> `Supplier: Harvest Co.`
> `Affects 4 recipes in The Bistro.`

---

### Step 7
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Sends each message in `SendQueue` to Telegram Bot API `sendMessage`. Sends sequentially with per-chat rate limiting (≤1 msg/sec per `telegramUserId`). Records delivery outcome (success/failure) per recipient. |
| **Component** | Python FastAPI → Telegram Bot API |
| **Data IN** | `SendQueue[] { telegramUserId, telegramLinkId, message_text }` |
| **Data OUT** | `DeliveryResult[] { telegramUserId, success: bool, telegram_message_id?: int, error_code?: int }` |
| **Decision point** | YES — special cases: <br>HTTP 400 or HTTP 403 from Telegram (bot blocked or chat not found) → call `PATCH /api/telegram-links/{telegramLinkId}/deactivate` on .NET 10 API, which sets `TelegramLink.status = 'unlinked'`. Nothing is written to `User`. Log the deregistration event. Do not retry. <br>HTTP 429 (rate limited) → respect `retry_after` header value; retry that message once after the specified delay. <br>Network timeout → mark as failed; do not retry inline. |
| **Error case** | Bulk delivery failure (Telegram API down) → log all failures; return partial success summary to .NET 10 API in Step 8 |

---

### Step 8
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Calls `POST /internal/alerts/delivery-report` on .NET 10 API with delivery outcomes |
| **Component** | Python FastAPI → .NET 10 API → PostgreSQL |
| **Data IN** | `{ alert_payloads_sent: N, delivered_to: [telegramUserIds], failed: [{ telegramUserId, error_code }] }` |
| **Data OUT** | `AlertDeliveryRecord` updated with final status per recipient |
| **Decision point** | No |
| **Error case** | .NET 10 API unreachable for the delivery report → log locally in Python FastAPI; non-critical path — the alert was already sent or not; reporting failure does not affect user outcome |

---

## Flow 5: `/summary` Command

**Pre-condition**: User has a linked Telegram account — a `TelegramLink` with `status = 'confirmed'` and matching `telegramUserId` exists. User belongs to exactly one `Outlet`.
**Post-condition**: User receives a role-filtered summary of current food costs for all active recipes in their outlet, within Telegram's 4096-character message limit.

---

### Step 1
| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Sends `/summary` to the bot |
| **Component** | Telegram (external) |
| **Data IN** | User keyboard input |
| **Data OUT** | Telegram `Update` delivered to bot webhook |
| **Decision point** | No |
| **Error case** | None |

---

### Step 2
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Telegram webhook handler) |
| **Action** | Parses `Update`. Identifies command = `summary`. No additional arguments expected. Extracts `telegramUserId` from `update.message.from.id`. |
| **Component** | Python FastAPI |
| **Data IN** | `Update { message.from.id, message.text: "/summary" }` |
| **Data OUT** | `{ telegramUserId }` |
| **Decision point** | No |
| **Error case** | None |

---

### Step 3
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Calls `GET /api/telegram/users/{telegramUserId}` on .NET 10 API (identical to Flow 2 Steps 3–4) |
| **Component** | Python FastAPI → .NET 10 API → PostgreSQL |
| **Data IN** | `telegramUserId` + `X-Internal-API-Key` |
| **Data OUT** | `{ user_id, telegramLinkId, role, outlet_id, outlet_name }` |
| **Decision point** | YES — if no confirmed `TelegramLink` found → "Your Telegram account is not linked. Visit the web app to link your account." Stop. If no outlet assigned → "Your account is not assigned to an outlet." Stop. |
| **Error case** | .NET 10 API unreachable → generic error |

---

### Step 4
| Field | Value |
|---|---|
| **Actor** | Python FastAPI |
| **Action** | Calls `GET /api/telegram/summary?user_id={user_id}&outlet_id={outlet_id}` on .NET 10 API |
| **Component** | Python FastAPI → .NET 10 API |
| **Data IN** | `{ user_id, outlet_id }` + `X-Internal-API-Key` |
| **Data OUT** | `SummaryResponse` DTO or error |
| **Decision point** | No |
| **Error case** | .NET 10 API error → bot replies "Unable to retrieve summary. Please try again." |

---

### Step 5
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Queries all active `Recipe` records for `outlet_id`. For each `Recipe`: calculates `cost_per_portion` using the canonical formula — for each `RecipeItem`: `SELECT price FROM IngredientPriceHistory WHERE ingredient_id = @id ORDER BY committed_at DESC LIMIT 1`; `item_cost = (unit_price / unit_size) * quantity * (1 / yield_pct)`; `cost_per_portion = SUM(item_costs) / Recipe.portion_count`. Calculates `food_cost_pct` where `sell_price IS NOT NULL`. Calculates `gross_margin`. Applies `flagged = true` where `recipe.food_cost_pct > recipe.cost_threshold_percentage`. Groups records by `Category.name`. Within each group: sorts by `food_cost_pct DESC` (highest risk first; nulls last). Also computes `recent_price_changes_7d` (count of `IngredientPriceHistory` rows with `committed_at > now() - 7 days` for ingredients referenced by each recipe) for Procurement view. |
| **Component** | .NET 10 API → PostgreSQL (joins: `Recipe`, `RecipeItem`, `Ingredient`, `IngredientPriceHistory`, `Category`) |
| **Data IN** | `outlet_id` |
| **Data OUT** | `{ outlet_name, as_of: DateTime, recipe_count: int, categories: [{ category_name, recipes: [{ recipe_id, recipe_name, cost_per_portion, food_cost_pct: decimal\|null, gross_margin: decimal\|null, sell_price: decimal\|null, flagged: bool, partial_cost_warning: bool, recent_price_changes_7d: int, cost_threshold_percentage: decimal }] }] }` |
| **Decision point** | YES — if `recipe_count = 0` → return `{ recipe_count: 0 }`. Python FastAPI handles the empty state in Step 7. |
| **Error case** | DB error → 500 |

---

### Step 6
| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Applies role-based field masking to the full summary before returning |
| **Component** | .NET 10 API |
| **Data IN** | Full summary data, `User.role` |
| **Data OUT** | Role-filtered `SummaryResponse` DTO: <br>**Owner**: all fields per recipe — `cost_per_portion`, `food_cost_pct`, `gross_margin`, `flagged` (with exact `%` and threshold), `sell_price` <br>**Chef**: `cost_per_portion`, `flagged` shown as boolean label only (no `%` value, no `gross_margin`, no `food_cost_pct`), no `sell_price` <br>**Procurement**: `cost_per_portion`, `recent_price_changes_7d` per recipe (no `food_cost_pct`, no `gross_margin`, no `flagged`) <br>**Viewer**: `cost_per_portion` only per recipe; no flags, no grouping metadata beyond `category_name` |
| **Decision point** | No — unknown role defaults to Viewer |
| **Error case** | None (pure transformation) |

---

### Step 7
| Field | Value |
|---|---|
| **Actor** | Python FastAPI (Telegram Bot) |
| **Action** | Receives role-filtered `SummaryResponse`. Formats into Telegram message(s). Enforces Telegram's 4096-character limit per message: if the full summary exceeds the limit, sends the top N recipes per category (prioritising `flagged = true` entries first) and appends "…and X more. See full summary at [web app URL]." Sends message(s) via Telegram Bot API. |
| **Component** | Python FastAPI → Telegram Bot API |
| **Data IN** | `SummaryResponse` DTO, `telegramUserId` |
| **Data OUT** | 1–3 Telegram messages |
| **Decision point** | YES — if `recipe_count = 0` → reply: "No active recipes found for The Bistro. Start by adding recipes in the web app." <br>If summary fits in a single message → send as one message. <br>If summary exceeds 4096 characters → truncate smartly (send flagged/high-cost items first; append truncation notice with web app link). |
| **Error case** | Telegram API delivery failure → log |

---

## Cross-Flow Consistency Notes

| Topic | Decision applied across all flows |
|---|---|
| User resolution endpoint | All flows use `GET /api/telegram/users/{telegramUserId}` as the first authenticated step after receiving a bot message |
| User resolution query | `SELECT … FROM TelegramLink tl JOIN User u ON u.id = tl.userId … WHERE tl.telegramUserId = @id AND tl.status = 'confirmed'` — never looks up by a column on `User` |
| Role-based masking location | Always in .NET 10 API — Python FastAPI never makes visibility decisions |
| `outlet_id` sourcing | Always from `User.outlet_id` — never from user input in any flow |
| Internal API authentication | All Python FastAPI ↔ .NET 10 API calls use `X-Internal-API-Key`; no user JWT is passed in bot flows |
| Ingredient cost source | Latest `IngredientPriceHistory` record per ingredient: `ORDER BY committed_at DESC LIMIT 1` |
| Cost formula | `cost_per_portion = SUM((unit_price / unit_size) * quantity * (1 / yield_pct)) / Recipe.portion_count` |
| Food cost threshold | `Recipe.cost_threshold_percentage` (per-recipe) — no outlet-level threshold field exists |
| Price spike threshold | `Ingredient.price_spike_threshold_pct` (per ingredient) |
| Cascade failure isolation | Per-recipe recalculation errors never roll back the `IngredientPriceHistory` insert |
| Deactivation on 400/403 | Both HTTP 400 and 403 from Telegram → `PATCH /api/telegram-links/{telegramLinkId}/deactivate` → `TelegramLink.status = 'unlinked'`. Nothing written to `User`. |
| Bot is read-only | No bot flow writes to any `User` field. The only Telegram-initiated writes are: `TelegramLink` fields on confirmation (Flow 1) and `TelegramLink.status = 'unlinked'` on deactivation (Flow 4). |
| Unlinked user message | Identical copy used across all flows: "Your Telegram account is not linked. Visit the web app to link your account." |
| Owner alert visibility | `food_cost_threshold` alert payload: `{ recipe_name, food_cost_pct, gross_margin, cost_per_portion }` |
| Chef alert visibility | `food_cost_threshold` alert payload: `{ recipe_name, cost_per_portion }` — `food_cost_pct` and `gross_margin` fields are absent, not null |
| Price spike alert recipients | `Owner` + `Procurement` only |
| Food cost threshold alert recipients | `Owner` + `Chef` only |
