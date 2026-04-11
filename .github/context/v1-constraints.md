# v1 Constraints — Solopreneur Scope

> **Load this file at the start of every coding session.**
> This is the most important context file. It defines exactly what v1 IS and what it IS NOT.
> Agents that ignore this file will build enterprise code that doesn't belong in v1.

---

## The Single Most Important Rule

**v1 is a single-user application.** One registered account. One kitchen. Many recipes.

There is no company hierarchy. There are no roles. There is no team. Any code that introduces `Company`, `Outlet`, `OutletUser`, `Invitation`, or `Role` in v1 is wrong.

---

## Entity Shapes — v1 (build these)

```
User {
  Id             UUID PK
  Email          varchar(255) UNIQUE NOT NULL
  PasswordHash   varchar(255) NOT NULL
  IsEmailVerified bool DEFAULT false
  CreatedAt      timestamptz
  UpdatedAt      timestamptz
}

TelegramLink {
  Id              UUID PK
  UserId          UUID FK → User.Id
  CodeHash        varchar(64)   ← SHA-256 hash ONLY, never plaintext (C-6)
  ExpiresAt       timestamptz   ← 15-minute TTL
  Status          enum('pending','confirmed','unlinked')
  TelegramUserId  bigint        ← Telegram user_id
  TelegramUsername varchar(255) nullable
  LinkedAt        timestamptz   nullable
}

Ingredient {
  Id                    UUID PK
  UserId                UUID FK → User.Id   ← NOT OutletId
  Name                  varchar(255) NOT NULL
  CategoryId            UUID FK → Category.Id nullable
  UnitId                UUID FK → Unit.Id NOT NULL
  UnitSize              decimal(10,4) NOT NULL
  PriceSpikeThreshold   decimal(5,2) nullable
  CreatedAt / UpdatedAt timestamptz
  UNIQUE(UserId, Name)
}

IngredientPriceHistory {
  Id            UUID PK
  IngredientId  UUID FK → Ingredient.Id
  Price         decimal(10,4) NOT NULL
  Source        enum('Manual','InvoiceScan') NOT NULL   ← C-13: case-sensitive
  CommittedAt   timestamptz NOT NULL DEFAULT NOW()      ← C-4: system timestamp, ordering field
  EffectiveDate date NOT NULL                           ← C-4: user-editable business date
}

Unit { Id, Name, Abbreviation }
Category { Id, UserId FK, Name }

Recipe {
  Id               UUID PK
  UserId           UUID FK → User.Id   ← NOT OutletId
  Name             varchar(255) NOT NULL
  PortionCount     int NOT NULL DEFAULT 1
  CostPerPortion   decimal(10,4)        ← server-authoritative via CostCascadeService
  PackagingCost    decimal(10,4) NOT NULL DEFAULT 0    ← v1 NEW
  TargetMargin     decimal(5,4) NOT NULL DEFAULT 0     ← v1 NEW (e.g. 0.40 = 40%)
  VersionGroupId   UUID NOT NULL
  VersionNumber    int NOT NULL DEFAULT 1
  VersionLabel     varchar(100) NOT NULL DEFAULT 'Standard'
  CreatedAt / UpdatedAt timestamptz
  -- NO SellingPrice column
  -- NO CostThresholdPercentage column
}

RecipeItem {
  Id              UUID PK
  RecipeId        UUID FK → Recipe.Id
  IngredientId    UUID FK → Ingredient.Id
  Quantity        decimal(10,4) NOT NULL
  YieldPercentage decimal(5,4) NOT NULL DEFAULT 1.0
}

CascadeErrorLog {
  Id            UUID PK
  IngredientId  UUID NOT NULL
  RecipeId      UUID NOT NULL
  ErrorMessage  text NOT NULL
  CreatedAt     timestamptz   ← serves as error timestamp
}
```

---

## Entities NOT in v1 (v2-only)

| Entity | Why excluded |
|---|---|
| `Company` | No company hierarchy — single user owns everything directly |
| `Outlet` | No outlet scoping — one kitchen per user |
| `OutletUser` | No team assignments |
| `Invitation` | No invitations — single user registers directly |
| `Supplier` | Not needed in v1 — ingredient prices entered manually or via receipt scan |

---

## Enums NOT in v1 (v2-only)

| Enum | Why excluded |
|---|---|
| `Role` (Owner/Chef/Procurement/Viewer) | No roles in single-user app |
| `InvitationStatus` | No invitations |

---

## JWT Claims — v1

```
{
  "sub": "<userId>",        ← ClaimTypes.NameIdentifier
  "email": "<email>",       ← ClaimTypes.Email
  "exp": <timestamp>
}
```

**Not in v1 JWT:** `outletId`, `role`, `companyId`

**ITokenService signature:**
```csharp
string GenerateToken(Guid userId, string email);
```

**ClaimsPrincipalExtensions — v1 only has:**
```csharp
GetUserId()   → Guid
GetEmail()    → string
// GetOutletId() and GetRole() are v2-only
```

---

## Authorization — v1

```csharp
// Single policy: authenticated user
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// All protected endpoints:
.RequireAuthorization()   // ← no policy name
```

**Never use in v1:** `"OwnerOnly"`, `"OwnerOrChef"`, `"OwnerOrProcurement"`, `"CanManageIngredients"` — these are v2 role policies.

---

## Sell Price Formula — v1

```
sell_price = (CostPerPortion + PackagingCost) / (1 - TargetMargin)
```

- **Derived at read time in the API handler** — never stored in the database
- `PackagingCost`: flat dollar add-on per portion entered by the user (e.g. $0.45 for box + label)
- `TargetMargin`: decimal 0.0–0.99 (e.g. 0.40 = 40% margin)
- If `TargetMargin == 0`, sell price = ingredient cost + packaging (no margin applied)
- If `TargetMargin >= 1`, return `null` (don't divide by zero or negative)

**In handler code:**
```csharp
decimal? derivedSellPrice = recipe.TargetMargin < 1m
    ? (recipe.CostPerPortion + recipe.PackagingCost) / (1m - recipe.TargetMargin)
    : null;

decimal? foodCostPct = derivedSellPrice.HasValue && derivedSellPrice.Value > 0
    ? (recipe.CostPerPortion / derivedSellPrice.Value) * 100m
    : null;
```

---

## Routes — v1 Pattern

| Resource | v1 Route | NOT this |
|---|---|---|
| Ingredients | `GET /api/ingredients` | ~~`GET /api/outlets/{id}/ingredients`~~ |
| Single ingredient | `GET /api/ingredients/{id}` | ~~`GET /api/outlets/{id}/ingredients/{id}`~~ |
| Price history | `POST /api/ingredients/{id}/prices` | ~~`POST /api/outlets/{id}/ingredients/{id}/prices`~~ |
| Recipes | `GET /api/recipes` | ~~`GET /api/outlets/{id}/recipes`~~ |
| Single recipe | `GET /api/recipes/{id}` | ~~`GET /api/outlets/{id}/recipes/{id}`~~ |

**There is no `{outletId}` in any v1 route.** User context comes from JWT, not from the URL.

---

## Response DTOs — v1

**Recipes:** Single unified DTO — the user sees all fields (no role-split):
```csharp
public record RecipeResponse(
    Guid Id,
    string Name,
    int PortionCount,
    decimal CostPerPortion,
    decimal PackagingCost,
    decimal TargetMargin,
    decimal? DerivedSellPrice,   // computed, not stored
    decimal? FoodCostPct,        // computed, not stored
    int VersionNumber,
    string VersionLabel,
    Guid VersionGroupId
);
```

**Never build in v1:** `RecipeOwnerResponse`, `RecipeStandardResponse`, `RecipeByIdOwnerResponse`, `RecipeByIdStandardResponse` — these are role-split DTOs from the enterprise design.

---

## Canonical Decisions — v1 Status

| Decision | v1 Status | Notes |
|---|---|---|
| C-1 Cascade interface | ✅ Applies | `RecalculateForIngredient(ingredientId)` |
| C-2 Cost formula | ✅ Applies | `SUM((price/unitSize)*qty*(1/yield))/portionCount` |
| C-3 Current price lookup | ✅ Applies | `ORDER BY committed_at DESC LIMIT 1` |
| C-4 Price timestamps | ✅ Applies | `committed_at` (system) + `effective_date` (user) |
| C-5 Cascade failure isolation | ✅ Applies | Log to `CascadeErrorLog`, never rollback price history |
| C-6 TelegramLink | ✅ Applies | SHA-256 hash, 15-min TTL, direct userId FK |
| C-13 PriceSource enum | ✅ Applies | `'Manual'` or `'InvoiceScan'` (case-sensitive) |
| C-7 Telegram deactivation | ✅ Applies | 400/403 → status = 'unlinked' |
| C-8 4-role system | ❌ v2-only | No roles in v1 |
| C-9 Alert threshold on recipe | ❌ v2-only | No `CostThresholdPercentage` on Recipe in v1 |
| C-10 Role visibility | ❌ v2-only | Single user sees all fields — no stripping |
| C-11 Price spike recipients | ❌ v2-only | Single user is the only recipient |
| C-12 Threshold alert payload | ❌ v2-only | No role-split alert payloads in v1 |
| C-14 TelegramUserId column name | ✅ Applies | `telegramUserId` not `chatId` |

---

## Confusion Flags — Read Before Implementing

When you see these patterns in lesson code, they are enterprise code that has been corrected:

| Enterprise pattern | v1 replacement |
|---|---|
| `Guid OutletId` in command/query | `Guid UserId` |
| `i.OutletId == command.OutletId` | `i.UserId == command.UserId` |
| `ingredient.OutletId != command.OutletId` | `ingredient.UserId != command.UserId` |
| `r.OutletId == command.OutletId` | `r.UserId == command.UserId` |
| `Recipe.SellingPrice` | `Recipe.TargetMargin` + derived sell price |
| `Recipe.CostThresholdPercentage` | Not present in v1 |
| `RecipeOwnerResponse` | `RecipeResponse` (unified) |
| `GetRecipesByOutletQuery(OutletId, Role)` | `GetRecipesQuery(UserId)` |
| `/api/outlets/{outletId}/...` | `/api/...` (no outletId in route) |
| `.RequireAuthorization("OwnerOnly")` | `.RequireAuthorization()` |
| JWT `outletId` or `role` claim | Not present in v1 JWT |
