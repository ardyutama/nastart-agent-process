# Flow 01 - Single-User Auth & Telegram Linking

> **Track:** `business-flows/v1/`
> **Authority:** Active implementation guidance for the current single-user v1 build. Read this file with `AGENTS.md` and `.github/context/v1-constraints.md`.
>
> **Service owner:** Nastart.Api + Python FastAPI (bot)
> **Entities:** User, TelegramLink
>
> **v1 scope:** One registered account, no company setup, no outlet selection, no team invitations, and no role-scoped JWTs.

---

## Active Rules For This Flow

- JWT contains only `userId` and `email`
- Protected endpoints use `.RequireAuthorization()` with no role policy name
- Telegram linking uses `TelegramLink.codeHash`, `expiresAt`, `status`, `telegramUserId`, and `telegramUsername`
- Telegram identity is stored on `TelegramLink`, never on `User`

---

## Flow 1: User Registration & Email Verification

**Actor:** New User
**Entry point:** Public sign-up page in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Submits registration form | Vue.js | email, password | - | - | Client-side validation blocks malformed input |
| 2 | Nastart.Api | Validates request shape and email uniqueness | Nastart.Api -> PostgreSQL | email, password | - | Email exists -> 409 Conflict | Validation failure -> 400 |
| 3 | Nastart.Api | Creates User record | PostgreSQL | email, passwordHash, isEmailVerified=false | User.id | - | DB error -> 500 |
| 4 | Nastart.Api | Creates verification token and sends verification email | Email service | User.id, email | verification email queued | Email delivery can retry without rolling back user creation | Background delivery failure logged |
| 5 | User | Opens verification link | Browser | token | - | Token expired or invalid -> reject | Show verification error |
| 6 | Nastart.Api | Confirms token and marks email verified | PostgreSQL | token | User.isEmailVerified=true | - | Invalid token -> 400 or 410 |
| 7 | Vue.js | Shows verified state and routes user to login | Vue.js | success response | login screen | - | Allow resend flow if verification failed |

**Result:** The user has a verified account and can authenticate. No company, outlet, invitation, or role records are created.

---

## Flow 2: User Login & Authenticated Session

**Actor:** Verified User
**Entry point:** Public login page in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Submits credentials | Vue.js | email, password | - | - | Client-side validation |
| 2 | Nastart.Api | Validates credentials | Nastart.Api -> PostgreSQL | email, password | User | Email not found or password mismatch -> 401 | Return auth error |
| 3 | Nastart.Api | Checks email verification state | PostgreSQL | User.id | isEmailVerified | Not verified -> block login or require verification flow | 403 or validation error |
| 4 | Nastart.Api | Generates JWT | Nastart.Api | userId, email | JWT | - | Token generation failure -> 500 |
| 5 | Nastart.Api | Returns auth response | Nastart.Api | JWT | access token + user summary | - | - |
| 6 | Vue.js | Stores token and opens the authenticated app shell | Vue.js | JWT | authenticated session | - | Token storage failure -> sign out |

**JWT contract:**

```json
{
  "sub": "<userId>",
  "email": "<email>",
  "exp": 1234567890
}
```

**Not allowed in v1:** `outletId`, `role`, `companyId`

---

## Flow 3: Generate Telegram Link Code

**Actor:** Authenticated User
**Entry point:** Settings or Telegram linking page in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens Telegram linking screen | Vue.js | JWT | link screen | - | Unauthenticated user redirected to login |
| 2 | Nastart.Api | Authorizes request | Nastart.Api | JWT | User.id from claims | Missing auth -> 401 | Access denied |
| 3 | Nastart.Api | Invalidates older pending link codes for this user | PostgreSQL | User.id | pending links -> unlinked | - | DB error -> 500 |
| 4 | Nastart.Api | Generates a one-time plaintext code and SHA-256 hash | Nastart.Api | User.id | plaintext code, codeHash | - | Generation failure -> 500 |
| 5 | Nastart.Api | Creates TelegramLink record | PostgreSQL | userId, codeHash, expiresAt=NOW()+15 minutes, status='pending' | TelegramLink.id | - | DB error -> 500 |
| 6 | Nastart.Api | Returns plaintext code to the UI | Nastart.Api | TelegramLink.id | code + expiry timestamp | - | - |
| 7 | Vue.js | Displays `/link CODE` instruction and countdown | Vue.js | code, expiresAt | visible code prompt | Code expiry -> user must generate a new one | - |

**Invariant:** The plaintext code is shown once to the user and is never stored directly in the database.

---

## Flow 4: Confirm Telegram Link From Bot

**Actor:** Authenticated User using Telegram bot
**Entry point:** Telegram private chat with the bot

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Sends `/link ABC123` to bot | Telegram Bot | plaintext code | - | - | Malformed command -> bot replies with usage |
| 2 | Python FastAPI | Parses the command and extracts the code | Python FastAPI | message text, telegram user metadata | plaintext code, telegramUserId, telegramUsername | Invalid command -> stop | Bot replies with help |
| 3 | Python FastAPI | Hashes the code and calls the protected confirmation endpoint | Python FastAPI -> Nastart.Api | codeHash, telegramUserId, telegramUsername, bot secret | confirmation request | Invalid bot secret -> reject | 401 |
| 4 | Nastart.Api | Looks up pending TelegramLink by code hash | PostgreSQL | codeHash | TelegramLink record | Not found -> 404 | Bot sends failure reply |
| 5 | Nastart.Api | Validates status and expiry | PostgreSQL | TelegramLink | Expired -> 410, not pending -> 409 | Bot sends failure reply | - |
| 6 | Nastart.Api | Confirms the TelegramLink | PostgreSQL | telegramUserId, telegramUsername, linkedAt=NOW() | status='confirmed' | - | DB error -> 500 |
| 7 | Python FastAPI | Sends success message back to Telegram user | Telegram Bot | telegramUserId | confirmation message | Delivery failure -> log and mark for retry if needed | Log error |
| 8 | Vue.js | Polls or refetches link status | Vue.js -> Nastart.Api | authenticated request | `status='confirmed'` | Timeout -> keep waiting or allow retry | - |

**Result:** The single user account is linked to one Telegram identity through `TelegramLink`.

---

## Flow 5: Deactivate Telegram Link

**Actor:** System or authenticated user
**Entry point:** User disconnect action or bot delivery failure

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User or system | Requests deactivation | Nastart.Api | TelegramLink.id | - | - | Invalid link id -> 404 |
| 2 | Nastart.Api | Authorizes ownership or trusted system action | Nastart.Api | JWT or internal system request | authorized request | Unauthorized -> 401 or 403 | Reject |
| 3 | Nastart.Api | Sets `TelegramLink.status='unlinked'` | PostgreSQL | TelegramLink.id | updated link record | - | DB error -> 500 |
| 4 | Nastart.Api | Returns success to caller | Nastart.Api | updated status | success response | - | - |

**System-triggered deactivation:** A `400 Bad Request` or `403 Forbidden` response from Telegram delivery triggers the same deactivation path so the app does not keep sending to a broken link.

---

## Route Summary

| Capability | Route pattern | Auth |
|---|---|---|
| Register | `POST /api/auth/register` | Public |
| Verify email | `POST /api/auth/verify-email` | Public |
| Login | `POST /api/auth/login` | Public |
| Generate Telegram link code | `POST /api/telegram-links` | `.RequireAuthorization()` |
| Read link status | `GET /api/telegram-links/me` | `.RequireAuthorization()` |
| Deactivate Telegram link | `PATCH /api/telegram-links/{id}/deactivate` | `.RequireAuthorization()` or trusted internal path |
| Confirm Telegram link from bot | Internal API endpoint with bot secret | Service-to-service |

---

## Not In This v1 Flow

- Company registration
- Outlet creation or selection
- Team invitations
- Role assignment or role-based authorization
- Outlet-scoped JWT re-issuance