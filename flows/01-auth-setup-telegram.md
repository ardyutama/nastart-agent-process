# Business Process Flows: Authentication, Company/Outlet Setup & Telegram Linking

**System:** Recipe Cost Management SaaS  
**Author:** Senior Software Architect  
**Reviewed against:** Canonical entity model, locked design decisions  
**Date:** 2026-04-02

---

## Canonical Entities Referenced

`Company` · `Outlet` · `User` · `Role` · `TelegramLink` · `Ingredient` · `IngredientPriceHistory` · `Unit` · `Category` · `Recipe` · `RecipeItem` · `Invoice` · `InvoiceLineItem` · `ReviewQueueItem` · `Supplier`

---

## Flow 1 — Company & First Outlet Registration

**Scope:** A brand-new user registers an account, creates their `Company`, configures the first `Outlet`, and receives a working JWT session.  
**Actor:** Prospective Owner (no existing account)

---

### Step 1

| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Navigates to the registration page and submits the sign-up form. |
| **Component** | Vue.js |
| **Data IN** | `email`, `password` (min 12 chars, 1 upper, 1 digit, 1 symbol), `fullName` |
| **Data OUT** | HTTP POST body to `.NET 10 API` → `POST /auth/register` |
| **Decision Point** | No |
| **Error Case** | Client-side validation fails → inline field errors shown; form not submitted. |

---

### Step 2

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Validates the registration payload: checks email format, password policy, and that the email does not already exist in `User`. |
| **Component** | .NET 10 API |
| **Data IN** | `email`, `password`, `fullName` |
| **Data OUT** | Validation result (pass / fail) |
| **Decision Point** | **Yes** — if email already exists → return `409 Conflict` with error `EMAIL_ALREADY_REGISTERED`. If password policy fails → return `422 Unprocessable Entity`. |
| **Error Case** | Duplicate email → Vue.js displays "An account with this email already exists." Password policy failure → Vue.js highlights the specific rule that was not met. |

---

### Step 3

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Hashes the password (BCrypt, cost factor ≥ 12), creates a new `User` record with `isVerified = false`, `isActive = false`, and assigns `Role = Owner` for this user's future outlet scope. Generates a secure email-verification token (cryptographically random, 32 bytes, stored hashed). |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | Validated `email`, hashed password, `fullName` |
| **Data OUT** | New `User` row: `userId`, `email`, `passwordHash`, `fullName`, `isVerified = false`, `isActive = false`, `createdAt` |
| **Decision Point** | No |
| **Error Case** | Database write fails → return `500 Internal Server Error`; no partial state left (transaction rolled back). |

---

### Step 4

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Sends a verification email containing a time-limited link (24-hour expiry) with the verification token embedded as a URL parameter. |
| **Component** | .NET 10 API (SMTP / transactional email service) |
| **Data IN** | `email`, verification token |
| **Data OUT** | Email delivered; `201 Created` returned to Vue.js with body `{ "message": "VERIFY_EMAIL_SENT" }` |
| **Decision Point** | No |
| **Error Case** | Email delivery failure → log the error server-side; still return `201` to avoid leaking user existence. Provide a "Resend verification email" option in the UI. |

---

### Step 5

| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Clicks the verification link in their email. |
| **Component** | Vue.js (verification landing page) |
| **Data IN** | URL query param: `token` |
| **Data OUT** | HTTP GET to `.NET 10 API` → `GET /auth/verify-email?token=XXXX` |
| **Decision Point** | No |
| **Error Case** | Token missing from URL → Vue.js shows "Invalid verification link." |

---

### Step 6

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Looks up the hashed token, checks it has not expired and has not been used. Sets `User.isVerified = true`, `User.isActive = true`, invalidates the token. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | Verification token (raw) |
| **Data OUT** | `User` record updated: `isVerified = true`, `isActive = true`; returns `200 OK` with `{ "message": "EMAIL_VERIFIED" }` |
| **Decision Point** | **Yes** — token not found or expired → return `400 Bad Request` with `INVALID_OR_EXPIRED_TOKEN`. |
| **Error Case** | Expired token → Vue.js shows "Link expired" with a "Resend verification email" button. |

---

### Step 7

| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Vue.js redirects user to the login page. User submits email + password. |
| **Component** | Vue.js |
| **Data IN** | `email`, `password` |
| **Data OUT** | HTTP POST to `.NET 10 API` → `POST /auth/login` |
| **Decision Point** | No |
| **Error Case** | Network failure → Vue.js shows generic "Unable to connect" message. |

---

### Step 8

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Looks up `User` by email, verifies `isActive = true` and `isVerified = true`, validates BCrypt hash. Issues a signed JWT (access token, 15-minute expiry) and a refresh token (opaque, 7-day expiry, stored hashed in DB). |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `email`, `password` |
| **Data OUT** | `accessToken` (JWT containing `userId`, `email`, `roles[]`), `refreshToken` (HttpOnly cookie); `200 OK` |
| **Decision Point** | **Yes** — `isVerified = false` → `403 Forbidden` with `EMAIL_NOT_VERIFIED`. Wrong password (after BCrypt compare) → `401 Unauthorized` with `INVALID_CREDENTIALS`. Account inactive → `403 Forbidden` with `ACCOUNT_DISABLED`. |
| **Error Case** | Any credential failure → increment failed-attempt counter; after 5 failures lock account for 15 minutes and return `429 Too Many Requests`. |

---

### Step 9

| Field | Value |
|---|---|
| **Actor** | User (now authenticated as Owner) |
| **Action** | Vue.js detects that the authenticated user has no `Company` associated. Redirects to the "Create Your Company" onboarding wizard (step 1 of 2). User submits company details. |
| **Component** | Vue.js |
| **Data IN** | `companyName`, `country`, `currency` (ISO 4217), `timezone` (IANA) |
| **Data OUT** | HTTP POST to `.NET 10 API` → `POST /companies` with Bearer `accessToken` |
| **Decision Point** | No |
| **Error Case** | Unauthenticated request (expired token) → `.NET 10 API` returns `401`; Vue.js triggers token refresh flow. |

---

### Step 10

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Validates JWT. Creates a new `Company` record. Associates the authenticated `User` as the Company owner (sets `User.companyId`). |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | JWT (`userId`), `companyName`, `country`, `currency`, `timezone` |
| **Data OUT** | New `Company` row: `companyId`, `companyName`, `country`, `currency`, `timezone`, `createdAt`. `User.companyId` set. Returns `201 Created` with `{ companyId }`. |
| **Decision Point** | **Yes** — `User.companyId` already set → return `409 Conflict` with `COMPANY_ALREADY_EXISTS` (prevents duplicate companies per owner). |
| **Error Case** | Validation failure (e.g. unsupported currency code) → `422 Unprocessable Entity`. |

---

### Step 11

| Field | Value |
|---|---|
| **Actor** | User (Owner) |
| **Action** | Wizard advances to step 2. User submits first outlet details. |
| **Component** | Vue.js |
| **Data IN** | `outletName`, `address` (optional), `companyId` (from previous step response) |
| **Data OUT** | HTTP POST to `.NET 10 API` → `POST /outlets` with Bearer `accessToken` |
| **Decision Point** | No |
| **Error Case** | Network timeout → Vue.js shows retry option; no partial state created yet. |

---

### Step 12

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Validates JWT and confirms `User` owns the referenced `Company`. Creates the `Outlet` record. Creates an `OutletUser` join record linking the `User` to the `Outlet` with `role = Owner`. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | JWT (`userId`), `companyId`, `outletName`, `address` |
| **Data OUT** | New `Outlet` row: `outletId`, `companyId`, `outletName`, `address`, `createdAt`. New `OutletUser` row: `outletId`, `userId`, `role = Owner`. Returns `201 Created` with `{ outletId }`. |
| **Decision Point** | **Yes** — `User` does not own the `Company` → return `403 Forbidden`. |
| **Error Case** | Database write failure → transaction rolled back; `500 Internal Server Error`. |

---

### Step 13

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Re-issues a fresh JWT that now includes the `outletId` and `role = Owner` in the claims. |
| **Component** | .NET 10 API |
| **Data IN** | `userId`, `outletId`, `role` |
| **Data OUT** | New `accessToken` (JWT containing `userId`, `outletId`, `role = Owner`); returned in `201` response body or via a follow-up `POST /auth/refresh`. |
| **Decision Point** | No |
| **Error Case** | Token signing failure → `500`. Client retries token refresh. |

---

### Step 14

| Field | Value |
|---|---|
| **Actor** | Vue.js |
| **Action** | Stores the new `accessToken` in memory (not localStorage). Redirects Owner to the main dashboard scoped to the new `Outlet`. Onboarding complete. |
| **Component** | Vue.js |
| **Data IN** | `accessToken`, `outletId` |
| **Data OUT** | Rendered dashboard for `Outlet`; Owner can now create ingredients, recipes, and invite users. |
| **Decision Point** | No |
| **Error Case** | None — purely client-side redirect. |

---

## Flow 2 — User Invitation & Role Assignment

**Scope:** An authenticated Owner invites a new team member (Chef, Procurement, or Viewer) to a specific `Outlet`. The invitee creates an account (or logs in) and gains role-scoped access to that outlet only.  
**Actor:** Owner (authenticated), then Invitee (new or existing User)

---

### Step 1

| Field | Value |
|---|---|
| **Actor** | Owner |
| **Action** | Navigates to Outlet Settings → Team Members → Invite User. Submits the invite form. |
| **Component** | Vue.js |
| **Data IN** | `email` (invitee), `role` (one of: `Chef`, `Procurement`, `Viewer`), `outletId` (from current outlet context in JWT) |
| **Data OUT** | HTTP POST to `.NET 10 API` → `POST /outlets/{outletId}/invitations` with Bearer `accessToken` |
| **Decision Point** | No |
| **Error Case** | Unauthenticated or expired token → `401`; Vue.js triggers refresh. |

---

### Step 2

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Validates JWT. Confirms the caller holds `role = Owner` on the specified `outletId`. Validates that `role` in payload is one of the permitted non-Owner roles. Checks that the invitee email is not already an active member of this `Outlet`. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | JWT (`userId`, `outletId`, `role = Owner`), `inviteeEmail`, `role` |
| **Data OUT** | Validation pass/fail |
| **Decision Point** | **Yes** — caller is not Owner on this outlet → `403 Forbidden`. Role is `Owner` (disallowed via invite) → `422`. Invitee already active on this outlet → `409 Conflict` with `ALREADY_A_MEMBER`. |
| **Error Case** | Any validation failure → descriptive error code returned; Vue.js displays inline message. |

---

### Step 3

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Generates a secure invitation token (cryptographically random, 32 bytes, stored hashed). Creates an `Invitation` record. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `outletId`, `inviteeEmail`, `role`, `invitedByUserId` |
| **Data OUT** | New `Invitation` row: `invitationId`, `outletId`, `inviteeEmail`, `role`, `tokenHash`, `expiresAt` (72 hours), `acceptedAt = null`, `createdAt`. |
| **Decision Point** | No |
| **Error Case** | DB write fails → `500`; no email sent. |

---

### Step 4

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Sends an invitation email to `inviteeEmail` containing an accept link with the raw token. Link format: `https://app.domain/accept-invite?token=XXXX`. Returns `201 Created` to Vue.js with `{ invitationId }`. |
| **Component** | .NET 10 API (SMTP / transactional email) |
| **Data IN** | `inviteeEmail`, raw invitation token, `outletName`, `role`, inviting Owner's name |
| **Data OUT** | Email delivered; `201 Created` response to Vue.js. Team Member list shows the invite as "Pending". |
| **Decision Point** | No |
| **Error Case** | Email delivery failure → log server-side; invitation record persists; Owner can resend via `POST /invitations/{invitationId}/resend`. |

---

### Step 5

| Field | Value |
|---|---|
| **Actor** | Invitee |
| **Action** | Clicks the invitation link in the email. |
| **Component** | Vue.js (accept-invite landing page) |
| **Data IN** | URL query param: `token` |
| **Data OUT** | Vue.js calls `.NET 10 API` → `GET /invitations/validate?token=XXXX` to check token state before rendering the form. |
| **Decision Point** | No |
| **Error Case** | Token missing → "Invalid invitation link" page shown. |

---

### Step 6

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Looks up the hashed token. Validates it has not expired and has not been accepted. Returns the invitation metadata (non-sensitive: `outletName`, `role`, `inviteeEmail`). |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | Raw token |
| **Data OUT** | `{ outletName, role, inviteeEmail }` for display; `200 OK`. |
| **Decision Point** | **Yes** — not found or expired → `404` with `INVALID_OR_EXPIRED_INVITATION`. Already accepted → `409` with `INVITATION_ALREADY_USED`. |
| **Error Case** | Expired → Vue.js shows "This invitation has expired. Ask your Owner to resend it." |

---

### Step 7

| Field | Value |
|---|---|
| **Actor** | Invitee |
| **Action** | Vue.js renders the accept page. **Branch A:** Invitee already has an account → shown a "Log in to accept" form. **Branch B:** Invitee does not have an account → shown a "Create account to accept" form with email pre-filled and read-only. |
| **Component** | Vue.js |
| **Data IN** | `inviteeEmail`, `outletName`, `role` (from Step 6 response) |
| **Data OUT** | User input: password (Branch B also collects `fullName`). |
| **Decision Point** | **Yes** — determine branch by whether `inviteeEmail` exists in `User` table. Vue.js asks the API → `GET /users/exists?email=XXXX` (returns boolean only — no other user data). |
| **Error Case** | API check fails → default to Branch B (safe fallback). |

---

### Step 8A — Branch A (Existing User)

| Field | Value |
|---|---|
| **Actor** | Invitee |
| **Action** | Submits login credentials on the accept page. |
| **Component** | Vue.js → .NET 10 API → `POST /auth/login` (same as Flow 1 Step 8) |
| **Data IN** | `email`, `password`, `invitationToken` (passed along) |
| **Data OUT** | `accessToken`, `refreshToken` |
| **Decision Point** | No (standard login; see Flow 1 Step 8 for failure cases) |
| **Error Case** | Wrong credentials → `401`; invite token preserved in Vue.js state. |

---

### Step 8B — Branch B (New User)

| Field | Value |
|---|---|
| **Actor** | Invitee |
| **Action** | Submits registration form. .NET 10 API creates the `User` record with `isVerified = true` (invitation email already validates the address — no separate email verification required). |
| **Component** | Vue.js → .NET 10 API → `POST /auth/register-via-invite` |
| **Data IN** | `fullName`, `password`, `invitationToken` |
| **Data OUT** | `User` created with `isVerified = true`, `isActive = true`. `accessToken`, `refreshToken` returned. |
| **Decision Point** | No |
| **Error Case** | Token consumed between page load and submit (race condition) → `400 BAD_REQUEST` with `INVITATION_ALREADY_USED`. |

---

### Step 9

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Accepts the invitation: validates the token, confirms the authenticated `User.email` matches `Invitation.inviteeEmail`, creates an `OutletUser` record, marks `Invitation.acceptedAt = now()`. All in a single DB transaction. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `invitationToken` (raw), authenticated `userId` |
| **Data OUT** | New `OutletUser` row: `outletId`, `userId`, `role` (as specified in invitation). `Invitation.acceptedAt` set. Returns `200 OK` with `{ outletId, role }`. |
| **Decision Point** | **Yes** — `User.email ≠ Invitation.inviteeEmail` → `403 Forbidden` with `EMAIL_MISMATCH` (prevents token hijacking). |
| **Error Case** | Transaction fails → rollback; invitation remains open for retry. |

---

### Step 10

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Re-issues JWT to include the new `outletId` and `role` in claims. |
| **Component** | .NET 10 API |
| **Data IN** | `userId`, `outletId`, `role` |
| **Data OUT** | Updated `accessToken` containing the new outlet scope. |
| **Decision Point** | No |
| **Error Case** | Token signing failure → `500`; client can force login to get fresh token. |

---

### Step 11

| Field | Value |
|---|---|
| **Actor** | Vue.js |
| **Action** | Stores new `accessToken`. Redirects Invitee to the `Outlet` dashboard scoped to their assigned `role`. Role-specific UI is rendered (e.g., Chef does not see invoice upload; Procurement does not see recipe margin %). |
| **Component** | Vue.js |
| **Data IN** | `accessToken` (`outletId`, `role` in claims) |
| **Data OUT** | Role-scoped dashboard rendered. Invitation flow complete. |
| **Decision Point** | No |
| **Error Case** | None — purely client-side redirect. |

---

## Flow 3 — Telegram Account Linking

**Scope:** An authenticated User links their Telegram account to their app identity so the Telegram Bot can respond with outlet-scoped, role-aware data.  
**Actor:** Authenticated User (any role), Telegram Bot (Python FastAPI), .NET 10 API

---

### Step 1

| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Navigates to Account Settings → Telegram → "Link Telegram Account". Clicks "Generate Link Code". |
| **Component** | Vue.js |
| **Data IN** | Bearer `accessToken` |
| **Data OUT** | HTTP POST to `.NET 10 API` → `POST /users/me/telegram-link-code` |
| **Decision Point** | No |
| **Error Case** | Unauthenticated → `401`; Vue.js triggers refresh. |

---

### Step 2

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Validates JWT. Checks that the `User` does not already have an active confirmed `TelegramLink` (prevents accidental re-link without explicit unlink). Generates a cryptographically random 6-character alphanumeric link code (uppercase, unambiguous characters — excludes O, 0, I, 1). Stores a `TelegramLink` record with `status = pending`, the hashed code, and a 15-minute expiry. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | JWT (`userId`) |
| **Data OUT** | New `TelegramLink` row: `telegramLinkId`, `userId`, `codeHash`, `expiresAt` (now + 15 min), `status = pending`, `telegramUserId = null`, `telegramUsername = null`, `linkedAt = null`. Returns `200 OK` with raw `code` (e.g. `"A3FKW2"`) to Vue.js. |
| **Decision Point** | **Yes** — existing confirmed `TelegramLink` found → return `409 Conflict` with `ALREADY_LINKED`. User must explicitly unlink first. |
| **Error Case** | DB write fails → `500`. |

---

### Step 3

| Field | Value |
|---|---|
| **Actor** | Vue.js |
| **Action** | Displays the 6-character link code prominently with a 15-minute countdown timer. Shows instruction: "Open Telegram, find our bot @RecipeCostBot, and send: `/link A3FKW2`". |
| **Component** | Vue.js |
| **Data IN** | `code` from Step 2 response |
| **Data OUT** | Code displayed to user. Timer begins. |
| **Decision Point** | No |
| **Error Case** | If timer expires before linking, Vue.js prompts "Code expired — generate a new one." (Detected via polling `GET /users/me/telegram-link-status` every 5 seconds, or via SSE.) |

---

### Step 4

| Field | Value |
|---|---|
| **Actor** | User |
| **Action** | Opens Telegram, navigates to `@RecipeCostBot`, and sends the message `/link A3FKW2`. |
| **Component** | Telegram (external) → Telegram Bot (Python FastAPI) |
| **Data IN** | Telegram Update object: `message.text = "/link A3FKW2"`, `message.from.id` (Telegram user ID), `message.from.username` |
| **Data OUT** | Telegram Bot receives the update via webhook. |
| **Decision Point** | No |
| **Error Case** | User sends malformed command (e.g. `/link` with no code, or `/link abc` with wrong length) → Bot replies "Invalid code format. Please send `/link XXXXXX` using the 6-character code from the app." No API call made. |

---

### Step 5

| Field | Value |
|---|---|
| **Actor** | Telegram Bot (Python FastAPI) |
| **Action** | Parses the `/link` command. Extracts the code (`A3FKW2`). Calls the `.NET 10 API` to redeem the code. |
| **Component** | Python FastAPI (Telegram Bot handler) → .NET 10 API → `POST /telegram/link` |
| **Data IN** | `code = "A3FKW2"`, `telegramUserId` (integer), `telegramUsername` (string, may be null) |
| **Data OUT** | HTTP POST request to internal `.NET 10 API` endpoint (bot-to-api service-to-service call, authenticated with a shared bot secret header — not a user JWT). |
| **Decision Point** | No |
| **Error Case** | `.NET 10 API` unreachable → Bot replies "Service temporarily unavailable. Please try again in a moment." Request is not retried automatically. |

---

### Step 6

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Validates the bot secret header. Hashes the received code and looks up a matching `TelegramLink` with `status = pending`. Checks expiry. Checks that `telegramUserId` is not already linked to another `User` (prevents one Telegram account linking to multiple app accounts). |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `code` (raw), `telegramUserId`, `telegramUsername`, bot secret header |
| **Data OUT** | Validation result (pass / fail) |
| **Decision Point** | **Yes — multiple branches:** (A) Code not found or `status ≠ pending` → `404` with `CODE_NOT_FOUND`. (B) Code expired → `410 Gone` with `CODE_EXPIRED`. (C) `telegramUserId` already linked to a different `User` → `409 Conflict` with `TELEGRAM_ALREADY_LINKED_TO_ANOTHER_ACCOUNT`. Bot receives the error code in the response body and sends the appropriate user-facing message. |
| **Error Case** | Invalid bot secret → `401 Unauthorized`; bot logs the error silently; no message sent to the Telegram user. |

---

### Step 7

| Field | Value |
|---|---|
| **Actor** | .NET 10 API |
| **Action** | Updates the `TelegramLink` record: sets `telegramUserId`, `telegramUsername`, `status = confirmed`, `linkedAt = now()`. Invalidates any other pending codes for this `User` (prevents replay). All in a single DB transaction. |
| **Component** | .NET 10 API → PostgreSQL |
| **Data IN** | `telegramLinkId`, `telegramUserId`, `telegramUsername` |
| **Data OUT** | Updated `TelegramLink` row: `status = confirmed`, `telegramUserId`, `telegramUsername`, `linkedAt`. Returns `200 OK` with `{ userId, outletId, role }` to Python FastAPI. |
| **Decision Point** | No |
| **Error Case** | Transaction fails → rollback; `TelegramLink` remains in `pending` state; Bot replies "Linking failed. Please try again." |

---

### Step 8

| Field | Value |
|---|---|
| **Actor** | Telegram Bot (Python FastAPI) |
| **Action** | Receives the `200 OK` response containing `userId`, `outletId`, `role`. Caches this mapping in memory (or Redis if available) for fast lookup on subsequent bot commands. Sends a confirmation message to the user in Telegram. |
| **Component** | Python FastAPI (Telegram Bot) |
| **Data IN** | `200 OK` response: `{ userId, outletId, role }` |
| **Data OUT** | Telegram message sent to user: "✅ Your account is now linked! You are connected as **[role]** for **[outletName]**. Try `/cost [dish name]` to get started." |
| **Decision Point** | No |
| **Error Case** | Telegram API fails to deliver the confirmation message → linking is still complete; user will discover it works on next command. |

---

### Step 9

| Field | Value |
|---|---|
| **Actor** | Vue.js |
| **Action** | Polling (`GET /users/me/telegram-link-status`) or SSE detects that `TelegramLink.status` has changed to `confirmed`. Updates the UI to show "Telegram account linked ✓" with the linked `telegramUsername`. Countdown timer stops. |
| **Component** | Vue.js → .NET 10 API |
| **Data IN** | `GET /users/me/telegram-link-status` response: `{ status: "confirmed", telegramUsername }` |
| **Data OUT** | Settings page reflects confirmed link state. "Generate Code" button replaced by "Unlink Telegram" button. |
| **Decision Point** | **Yes** — if timer reaches 0 and status is still `pending` → display "Code expired" and re-enable "Generate Link Code" button. |
| **Error Case** | Polling fails → degrade gracefully; user can manually refresh the settings page to check link status. |

---

## Cross-Flow Notes for Lead Architect Review

| Concern | Decision |
|---|---|
| JWT claims scope | Token always carries `userId`, `outletId[]`, `role` per outlet. Multi-outlet users receive an array; the active outlet is selected by the frontend and passed as a request header or path param. |
| Outlet isolation | All queries are filtered by `outletId`. No cross-outlet data access is possible at the API layer regardless of `Company` membership. |
| Role enforcement layer | `.NET 10 API` enforces role checks at the controller/policy level, not just in business logic. |
| `TelegramLink` to outlet scope | `TelegramLink` is linked to a `User`, not an `Outlet`. The bot resolves the active outlet from the `OutletUser` join. If a user belongs to multiple outlets, the bot defaults to the first outlet and allows `/switchoutlet` (out of scope for this flow). |
| Password reset flow | Not covered in this document — separate flow. |
| Invitation cleanup | Expired, unaccepted invitations are soft-deleted by a background job (not described here). |
| Re-linking Telegram | User must explicitly unlink (`DELETE /users/me/telegram-link`) before generating a new code. This sets `TelegramLink.status = unlinked` — the row is retained for audit. |
