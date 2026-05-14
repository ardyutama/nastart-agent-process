# Flow 05 - Telegram Bot Interactions

> **Track:** `business-flows/v1/`
> **Authority:** Active implementation guidance for the current single-user v1 build. Read this file with `AGENTS.md`, `.github/context/v1-constraints.md`, and the v1 auth, ingredient, and recipe flows.
>
> **Service owner:** Python FastAPI + Nastart.Api
> **Bot rule:** Telegram is read-only and alert-only. It never creates or edits ingredients, recipes, or prices directly.
>
> **v1 scope:** Telegram resolves to one linked user through `TelegramLink`. There is no role masking, no outlet context, and no team recipient routing.

---

## Active Rules For This Flow

- Account linking uses the `/link CODE` flow defined in `v1/01-auth-telegram-linking.md`
- `TelegramLink.telegramUserId` is the only Telegram identity key used for delivery
- Telegram commands read from Nastart.Api; the bot does not query the database directly for business data
- The user sees the unified v1 recipe response, not a role-filtered subset
- Push alerts target the single confirmed Telegram link for the user

---

## Prerequisite: Account Linking

Before any command or alert can work, the user must complete the link flow from `v1/01-auth-telegram-linking.md`.

Required linked state:
- `TelegramLink.status='confirmed'`
- `TelegramLink.telegramUserId` is populated
- `TelegramLink.userId` points to the authenticated app user

If the link is missing or unlinked, the bot responds with instructions to reconnect using `/link CODE` from the web app.

---

## Flow 1: `/cost [recipe name]`

**Actor:** Linked User
**Trigger:** User sends `/cost banana bread` or `/cost pandan cake`

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sends `/cost [recipe name]` | Telegram Bot | message text | command request | - | Malformed command -> usage reply |
| 2 | Python FastAPI | Resolves linked app user from `telegramUserId` | Nastart.Api | telegramUserId | linked User.id | No confirmed link -> reply with reconnect instructions | Bot reply |
| 3 | Python FastAPI | Calls Nastart.Api recipe search endpoint | Nastart.Api | User.id, recipe name query | recipe match list | No match -> suggest close names | Bot reply |
| 4 | Nastart.Api | Loads user-owned recipe and computes read-time fields | PostgreSQL + handler logic | recipeId, User.id | unified `RecipeResponse` | Recipe not owned by user -> 404 or 403 | Reject |
| 5 | Python FastAPI | Formats the response for Telegram | Python FastAPI | `RecipeResponse` | message text | Derived sell price null -> show cost only and explain margin issue | - |
| 6 | Python FastAPI | Sends message to Telegram user | Telegram Bot API | telegramUserId, message | delivered message | 400 or 403 -> deactivate link | Link unlinked |

**Typical response:** recipe name, cost per portion, packaging cost, target margin, derived sell price, and food cost percentage.

---

## Flow 2: Natural Language Fallback

**Actor:** Linked User
**Trigger:** Free-text question such as `what does my brownie cost now?`

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sends free text | Telegram Bot | message text | raw user message | - | - |
| 2 | Python FastAPI | Resolves linked user | Nastart.Api | telegramUserId | User.id | No confirmed link -> reply with reconnect instructions | Bot reply |
| 3 | Python FastAPI | Uses minimal app context for intent classification | Python FastAPI + LLM | user message + recipe names only | intent + extracted recipe name | Low confidence -> fallback help text | Bot reply |
| 4 | Python FastAPI | Routes to the same lookup path as `/cost` | Python FastAPI -> Nastart.Api | parsed recipe query | `RecipeResponse` | Intent not recognized -> suggest `/cost [recipe]` or `/summary` | Bot reply |
| 5 | Python FastAPI | Sends the final message | Telegram Bot API | formatted message | delivered message | 400 or 403 -> deactivate link | Link unlinked |

**LLM boundary:** Do not send hidden financial data beyond what is needed to resolve the recipe name. Authoritative costs still come from Nastart.Api.

---

## Flow 3: `/summary`

**Actor:** Linked User
**Trigger:** User sends `/summary`

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sends `/summary` | Telegram Bot | message text | summary request | - | - |
| 2 | Python FastAPI | Resolves linked user | Nastart.Api | telegramUserId | User.id | No confirmed link -> reply with reconnect instructions | Bot reply |
| 3 | Python FastAPI | Calls Nastart.Api for recipe list | Nastart.Api | User.id | `RecipeResponse[]` | No recipes -> reply with empty-state message | Bot reply |
| 4 | Nastart.Api | Returns user-owned recipes with computed values | PostgreSQL + handler logic | User.id | unified recipe list | - | API error -> graceful failure |
| 5 | Python FastAPI | Sorts and formats summary message | Python FastAPI | recipe list | short summary text | Message too long -> send top items plus web-app reminder | Split or shorten |
| 6 | Python FastAPI | Delivers summary message | Telegram Bot API | telegramUserId, message | delivered message | 400 or 403 -> deactivate link | Link unlinked |

**Suggested summary content:** top recipes by highest food cost percentage, plus current cost per portion and derived sell price.

---

## Flow 4: Price Spike Alert

**Trigger:** The ingredient flow detects a manual or invoice-based price spike that exceeds `Ingredient.PriceSpikeThresholdPct`.

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | Nastart.Api | Looks up confirmed Telegram link for the owning user | PostgreSQL | User.id | TelegramLink | No confirmed link -> stop | - |
| 2 | Nastart.Api | Sends outbound alert request to Python FastAPI | Nastart.Api -> Python FastAPI | telegramUserId, ingredientName, oldPrice, newPrice, changePct | HTTP 202 | Service unreachable -> log and retry if configured | Log |
| 3 | Python FastAPI | Formats personal alert message | Python FastAPI | alert payload | message text | - | - |
| 4 | Python FastAPI | Delivers alert to Telegram | Telegram Bot API | telegramUserId, message | delivery status | 400 or 403 -> deactivate link | Link unlinked |

**v1 recipient model:** exactly one recipient, the linked user.

---

## Flow 5: Link Deactivation On Delivery Failure

**Trigger:** Telegram delivery returns `400 Bad Request` or `403 Forbidden`.

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | Python FastAPI | Detects failed Telegram delivery | Telegram Bot API | response code, telegramUserId | failure event | Only 400/403 trigger deactivation | Other failures can retry |
| 2 | Python FastAPI | Calls the deactivation path in Nastart.Api | Nastart.Api | TelegramLink.id or telegramUserId lookup result | deactivation request | Invalid link -> log and stop | Log |
| 3 | Nastart.Api | Sets `TelegramLink.status='unlinked'` | PostgreSQL | TelegramLink.id | updated link | - | DB error -> 500 |
| 4 | System | Stops future Telegram sends until the user relinks | System behavior | updated link status | muted delivery path | - | - |

---

## Command Summary

| Command | Purpose | Output |
|---|---|---|
| `/link CODE` | Confirm Telegram account linking | Link confirmation |
| `/cost [recipe]` | Show one recipe's current costing | Single recipe summary |
| `/summary` | Show quick overview of current recipes | Multi-recipe summary |

---

## Not In This v1 Flow

- Role-based data masking
- Outlet-scoped recipient resolution
- Team-wide push alert fan-out
- Telegram writes for ingredient or recipe edits
- Threshold alerts based on `Recipe.CostThresholdPercentage`