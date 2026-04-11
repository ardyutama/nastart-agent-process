# Flow 05 — Telegram Bot Interactions

> **Status:** ACCEPTED (Round 2, after defect corrections) — Lead Architect validated
> **Service owner:** Python FastAPI + .NET 10 Web API (data/auth authority)
> **Bot framework:** python-telegram-bot
> **Rule:** Bot is READ-ONLY and ALERT-ONLY. No data can be created or modified via Telegram.

---

## Prerequisites / Canonical Entity Reference

```
TelegramLink {
  id               UUID
  userId           UUID        FK → User.id
  codeHash         varchar(64) SHA-256 of one-time code — NEVER plaintext
  expiresAt        timestamptz 15-minute TTL
  status           enum        'pending' | 'confirmed' | 'unlinked'
  telegramUserId   bigint      Telegram user_id (= chat_id for private chats)
  telegramUsername varchar(255) nullable
  linkedAt         timestamptz nullable
}
```

**User entity has NO `telegram_user_id` column.**
All Telegram identity is on `TelegramLink` only.

---

## Flow 1: Account Linking (`/link XXXX`)

**Entry:** User opens "Connect Telegram" in Vue.js web app

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens Telegram linking page | Vue.js | JWT | Link code form | Already linked (confirmed TelegramLink exists) → show "Re-link" option | - |
| 2 | .NET 10 API | Invalidates any existing pending TelegramLink for user | PostgreSQL | User.id | Previous pending TelegramLink.status='unlinked' | - | - |
| 3 | .NET 10 API | Generates 6-char alphanumeric code | .NET 10 | - | plaintext_code (shown once to user) | - | - |
| 4 | .NET 10 API | Stores hashed code in TelegramLink | PostgreSQL | userId, codeHash=SHA256(plaintext_code), expiresAt=NOW()+15min, status='pending' | TelegramLink.id | - | - |
| 5 | Vue.js | Displays plaintext_code + countdown timer | Vue.js | plaintext_code | Code display (e.g. "A3FKW2") | Timer expires → re-generate prompt | - |
| 6 | User | Opens Telegram, sends `/link A3FKW2` to bot | Telegram Bot | message text | - | Malformed → bot replies "Format: /link CODE" | - |
| 7 | Python FastAPI | Parses command, extracts plaintext_code, captures telegram_user_id | Python FastAPI | update.message | code, telegram_user_id | Not a /link command → ignore | - |
| 8 | Python FastAPI | Calls .NET 10 API with bot-secret header | .NET 10 API | codeHash=SHA256(code), telegram_user_id, telegram_username, bot_secret | - | Invalid bot_secret → 401 | Bot sends "Error, try again" |
| 9 | .NET 10 API | Validates codeHash, status='pending', expiresAt | PostgreSQL | codeHash | TelegramLink record | Expired (410) → send "Code expired, generate new" | Bot sends error |
| | | | | | | Already confirmed (409) → send "Already linked" | - |
| | | | | | | telegramUserId already on another user's TelegramLink → 409 | Bot sends conflict message |
| 10 | .NET 10 API | Updates TelegramLink | PostgreSQL | TelegramLink.id | status='confirmed', telegramUserId=telegram_user_id, telegramUsername, linkedAt=NOW() | - | DB error → 500 |
| 11 | Python FastAPI | Sends confirmation message to user | Telegram Bot API | telegramUserId | "✓ Linked as [Name] — Role: [Role] at [Outlet]" | Delivery failure → log (link already committed; do not rollback) | - |
| 12 | Vue.js | Detects confirmed state (SSE/polling) | Vue.js → .NET 10 | User.id | TelegramLink.status='confirmed' | - | Timeout after 15min → prompt re-generate |

---

## Flow 2: `/cost [dish name]`

**Trigger:** User sends `/cost pasta bake` or `/cost Risotto`
**Rule:** role-masked response — server-side only

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sends `/cost [name]` | Telegram Bot | message text | - | - | - |
| 2 | Python FastAPI | Validates user is linked | .NET 10 API | telegram_user_id | User.id, outlet_id, role | Not linked → "Use /link to connect your account" | Bot reply |
| 3 | Python FastAPI | Calls .NET 10 API: search recipe | .NET 10 API | dish_name (partial, case-insensitive), outlet_id | Recipe[] matching | No match → return top-3 Levenshtein suggestions | "Did you mean: X, Y, Z?" |
| 4 | .NET 10 API | Fetches cost data, applies role mask | PostgreSQL | recipe_id, role | Owner: {cost_per_portion, food_cost_pct, gross_margin}; Chef/Procurement/Viewer: {cost_per_portion} | Any ingredient with no IngredientPriceHistory → add partial_cost_warning flag | - |
| 5 | .NET 10 API | Returns role-filtered response | Python FastAPI | Role-filtered cost object | - | - | - |
| 6 | Python FastAPI | Formats message based on role | Python FastAPI | cost object | Owner: "🍝 Pasta Bake — Cost: $4.20/portion | Food Cost: 28% | Margin: $10.80"; Chef: "🍝 Pasta Bake — Cost: $4.20/portion" | partial_cost_warning → add "⚠️ Some ingredient prices not set" | - |
| 7 | Python FastAPI | Sends message | Telegram Bot API | telegramUserId, message | Delivered | 400/403 → PATCH /api/telegram-links/{id}/deactivate | status='unlinked' |

---

## Flow 3: Natural Language Fallback (LLM)

**Trigger:** User sends free text (not a /command)
**Example:** "how much does the lamb shoulder cost tonight?"

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sends free text | Telegram Bot | message text | - | - | - |
| 2 | Python FastAPI | Validates user is linked | .NET 10 API | telegram_user_id | User.id, outlet_id, role | Not linked → reply error | - |
| 3 | Python FastAPI | Fetches minimal outlet context (no costs, no prices) | .NET 10 API | outlet_id | {outlet_name, role_name, active_recipe_names[]} ONLY — NO cost data sent to LLM | - | - |
| 4 | Python FastAPI | Sends to LLM for intent classification | LLM | user_message + outlet context | {intent, extracted_entity, confidence} | confidence < 0.70 → "I didn't understand that. Try /cost [dish name]" | Bot reply |
| 5 | Python FastAPI | Routes to appropriate command sub-flow | .NET 10 API | intent + extracted_entity | Same flow as direct command | intent not recognised → graceful fallback | - |
| 6 | Python FastAPI | Returns role-filtered response | Telegram | Same as resolved command | Bot message | - | - |

---

## Flow 4: Proactive Push Alert (Outbound)

**Trigger:** .NET 10 API detects food cost threshold breach OR price spike (from cascade or price commit)
**Pattern:** "threshold crossing" — fires only when `old_pct ≤ threshold AND new_pct > threshold` (not on every recalculation)

**Alert type routing:**
- `food_cost_threshold` alert → Recipients: Owner + Chef
- `price_spike` alert → Recipients: Owner + Procurement

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | .NET 10 API | Detects threshold crossing | .NET 10 | old_pct, new_pct, threshold, outlet_id, alert_type | Trigger confirmed | old_pct > threshold already → skip (already notified) | - |
| 2 | .NET 10 API | Queries recipients by alert type | PostgreSQL | `SELECT tl.telegramUserId, tl.id AS telegramLinkId, u.Name, u.Role FROM TelegramLink tl JOIN User u ON u.Id = tl.UserId WHERE u.OutletId = @outletId AND u.Role IN (<role_list by alert_type>) AND tl.status = 'confirmed'` | [{telegramUserId, telegramLinkId, name, role}] | No recipients → log and stop | - |
| 3 | .NET 10 API | Builds role-split payloads per recipient | .NET 10 | recipient[], alert_data | Owner payload: {recipe_name, food_cost_pct, gross_margin, cost_per_portion}; Chef payload: {recipe_name, cost_per_portion} | - | - |
| 4 | .NET 10 API | Sends alert batch to Python FastAPI | Python FastAPI | [{telegramUserId, telegramLinkId, payload}] | HTTP 202 | FastAPI unreachable → retry 3x + exponential backoff → dead-letter persist | - |
| 5 | Python FastAPI | Formats message per payload shape | Python FastAPI | payload (role-determined shape) | Owner: "⚠️ [Recipe] food cost reached 38% (threshold: 32%). Cost: $4.20. Margin: $8.60"; Chef: "⚠️ [Recipe] cost has increased to $4.20/portion" | - | - |
| 6 | Python FastAPI | Sends to each Telegram user | Telegram Bot API | telegramUserId, message | Delivered | 400 (invalid chatId) → PATCH /api/telegram-links/{id}/deactivate (status='unlinked') | - |
| | | | | | | 403 (bot blocked) → PATCH /api/telegram-links/{id}/deactivate (status='unlinked') | - |

---

## Flow 5: `/summary` Command

**Trigger:** User sends `/summary`
**Output:** Cost overview for the user's outlet, sorted by risk (highest food_cost_pct first)

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sends `/summary` | Telegram Bot | - | - | - | - |
| 2 | Python FastAPI | Validates user linked, gets outlet_id + role | .NET 10 API | telegram_user_id | outlet_id, role | Not linked → reply error | - |
| 3 | Python FastAPI | Calls .NET 10 API for outlet recipe summary | .NET 10 API | outlet_id, role | Recipes[] with cost_per_portion (+ food_cost_pct for Owner) sorted by food_cost_pct DESC within Category | - | - |
| 4 | .NET 10 API | Applies role mask before returning | .NET 10 | Recipes[], role | Owner: full data; Chef/Procurement/Viewer: cost_per_portion only | - | - |
| 5 | Python FastAPI | Flags recipes over threshold | Python FastAPI | Recipes[], Recipe.cost_threshold_percentage | Flagged recipes: `recipe.food_cost_pct > recipe.cost_threshold_percentage` | No recipes → "No recipes found for this outlet" | - |
| 6 | Python FastAPI | Builds summary message | Python FastAPI | flagged + unflagged recipes | Formatted list (flagged first, ⚠️ prefix) | Message > 4096 chars → send flagged items only + "See full report: [web app link]" | - |
| 7 | Python FastAPI | Delivers message | Telegram Bot API | telegramUserId, message | Delivered | 400/403 → PATCH /api/telegram-links/{id}/deactivate | status='unlinked' |
