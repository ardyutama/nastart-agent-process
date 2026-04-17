# Flow 01 — User Authentication, Company Setup & Telegram Linking

> **Status:** HISTORICAL REFERENCE (Accepted in Round 1 for the earlier enterprise design)
> **Service owner:** .NET 10 Web API + Python FastAPI (bot)
> **Entities:** Company, Outlet, User, TelegramLink
>
> **Update (April 17, 2026):** This file is a historical enterprise onboarding reference. Do not implement `Company`, `Outlet`, `Invitation`, role assignment, or outlet-scoped JWTs for v1. The current v1 auth path is single-user register, verify, login, then optional Telegram linking using `TelegramLink` only.

---

## Flow 1: Company & First Outlet Registration

**Actor:** New User (becomes Owner)
**Entry point:** Public sign-up page in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Submits registration form | Vue.js | email, password, name | - | - | Client-side validation |
| 2 | .NET 10 API | Validates email uniqueness | .NET 10 → PostgreSQL | email | - | If exists → 409 Conflict | Return error |
| 3 | .NET 10 API | Creates User record | PostgreSQL | email, bcrypt(password), name, isVerified=false, isActive=false | User.id | - | DB error → 500 |
| 4 | .NET 10 API | Sends verification email | Email service | User.id, email | Verification token stored | - | Queue email for retry |
| 5 | User | Clicks verification link | Browser | token | - | If expired → re-send option | Show error |
| 6 | .NET 10 API | Sets isVerified=true, isActive=true | PostgreSQL | token | User (verified) | - | - |
| 7 | User | Logs in for first time | Vue.js | email, password | - | - | 401 if wrong |
| 8 | .NET 10 API | Issues initial JWT (no outlet yet) | .NET 10 | credentials | JWT (no outlet scope) | - | - |
| 9 | Vue.js | Detects no Company — shows onboarding wizard | Vue.js | JWT claims | Wizard screen | - | - |
| 10 | User | Creates Company | Vue.js → .NET 10 API | company_name, currency, timezone | Company.id | - | - |
| 11 | User | Creates first Outlet | Vue.js → .NET 10 API | outlet_name, Company.id | Outlet.id | - | - |
| 12 | .NET 10 API | Creates OutletUser join record | PostgreSQL | User.id, Outlet.id, role=Owner | OutletUser | - | - |
| 13 | .NET 10 API | Re-issues JWT with outlet scope | .NET 10 | User.id, Outlet.id, role | JWT (outlet-scoped) | - | - |
| 14 | Vue.js | Redirects to main dashboard | Vue.js | JWT | Dashboard | - | - |

---

## Flow 2: User Invitation & Role Assignment

**Actor:** Owner (inviting a new team member)
**Entry point:** Team management page in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | Owner | Fills invitation form | Vue.js | invitee_email, role (Chef/Procurement/Viewer), Outlet.id | - | - | Role must be non-Owner |
| 2 | .NET 10 API | Creates Invitation record | PostgreSQL | email, role, Outlet.id, Owner.id, token, expiresAt(48h) | Invitation.id | - | - |
| 3 | .NET 10 API | Sends invitation email | Email service | email, invite_token, outlet_name | - | - | Queue for retry |
| 4 | Invitee | Clicks invite link | Browser | invite_token | - | If expired → show error | Re-invite from Owner |
| 5 | .NET 10 API | Validates token + email match | PostgreSQL | invite_token | Invitation record | EMAIL_MISMATCH → 403 | Security block |
| 6 | — | **Branch A** — invitee has existing account | — | — | — | User already exists → Branch A | — |
| 6A | Invitee | Logs in with existing credentials | Vue.js → .NET 10 | credentials, invite_token | - | - | 401 if wrong |
| 6B | .NET 10 API | Creates OutletUser (existing user) | PostgreSQL | User.id, Outlet.id, role | OutletUser | - | - |
| 7 | — | **Branch B** — new user | — | — | — | No account → Branch B | — |
| 7A | Invitee | Registers (email pre-filled, no verification needed) | Vue.js → .NET 10 | name, password, invite_token | - | - | - |
| 7B | .NET 10 API | Creates User (isVerified=true via invite) | PostgreSQL | name, bcrypt(password), email, isVerified=true | User.id | - | - |
| 7C | .NET 10 API | Creates OutletUser | PostgreSQL | User.id, Outlet.id, role | OutletUser | - | - |
| 8 | .NET 10 API | Marks Invitation as used | PostgreSQL | Invitation.id | status=used | - | - |
| 9 | .NET 10 API | Issues JWT with outlet+role scope | .NET 10 | User.id, Outlet.id, role | JWT | - | - |
| 10 | Vue.js | Redirects to dashboard | Vue.js | JWT | Dashboard (role-restricted view) | - | - |

---

## Flow 3: Telegram Account Linking

**Actor:** Authenticated User (any role)
**Entry point:** "Connect Telegram" page in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Navigates to Telegram linking page | Vue.js | - | Link code form | - | - |
| 2 | .NET 10 API | Invalidates any existing pending TelegramLink for this User | PostgreSQL | User.id | Previous TelegramLink.status = 'unlinked' (if pending) | - | - |
| 3 | .NET 10 API | Generates one-time code, hashes it | .NET 10 | User.id | 6-char alphanumeric code (shown to user), codeHash stored | - | - |
| 4 | .NET 10 API | Creates TelegramLink record | PostgreSQL | userId, codeHash, expiresAt=NOW()+15min, status='pending' | TelegramLink.id | - | - |
| 5 | Vue.js | Displays code + 15-min countdown | Vue.js | plaintext code | Code shown (e.g., A3FKW2) | - | Countdown expires → re-generate |
| 6 | User | Opens Telegram, sends `/link A3FKW2` to bot | Telegram Bot | plaintext message | - | - | - |
| 7 | Python FastAPI (bot) | Parses command, extracts code | Python FastAPI | update.message.text | plaintext code, telegram_user_id | Malformed → reply error | Reply to user |
| 8 | Python FastAPI (bot) | Calls .NET 10 API with bot-secret header | .NET 10 API | codeHash=SHA256(code), telegram_user_id, telegram_username, bot_secret | - | - | 401 if invalid bot-secret |
| 9 | .NET 10 API | Validates codeHash, TTL, status | PostgreSQL | codeHash | TelegramLink record | Expired → 410; Already confirmed → 409 | Reply error via bot |
| 10 | .NET 10 API | Updates TelegramLink | PostgreSQL | TelegramLink.id, telegram_user_id, telegram_username | status='confirmed', telegramUserId set, linkedAt=NOW() | - | DB error → 500 |
| 11 | Python FastAPI (bot) | Sends confirmation message to user | Telegram | telegramUserId | "✓ Linked to your account as [Name]" | - | Log if delivery fails |
| 12 | Vue.js | Detects link confirmation (SSE/polling) | Vue.js → .NET 10 | User.id | TelegramLink.status='confirmed' | - | Timeout → prompt to retry |

**Re-link rule:** A User may have only one active (`status='confirmed'`) TelegramLink at a time. Generating a new code automatically sets previous pending records to 'unlinked'.

**Unlink endpoint:** `PATCH /api/telegram-links/{id}/deactivate` → sets `status='unlinked'` (record retained for audit)
