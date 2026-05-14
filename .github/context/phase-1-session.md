# Phase 1 Session Context Pack

> Load this file when starting any Phase 1 coding work.
> Also load: `AGENTS.md` + `.github/context/v1-constraints.md`
> For flow-level behavior in Phase 1, load `business-flows/00-index.md`, `business-flows/v1/01-auth-telegram-linking.md`, and `business-flows/v1/02-ingredient-price-management.md`.
> For .NET backend work, also load the relevant `.github/skills/dotnet-skills/` docs.
> For business logic changes, also load `.github/skills/test-driven-development/SKILL.md` and follow TDD: failing test first, then implementation.
> Lesson to follow: `lessons/L1-clean-architecture-and-solution-scaffold.md` then `lessons/L2-ef-core-postgresql-schema.md`
> If this file conflicts with `AGENTS.md` or `.github/context/v1-constraints.md`, those files win.

---

## Phase 1 Working Rules

- Use the `Nastart` solution and `Nastart.*` project names. If older lesson snippets still show `RecipeCost.*` or `Nastart.API`, translate them to `Nastart.*` and `Nastart.Api`.
- For auth behavior in Phase 1, use `business-flows/v1/01-auth-telegram-linking.md` as the current flow spec.
- For ingredient CRUD and price-history behavior in Phase 1, use `business-flows/v1/02-ingredient-price-management.md` as the current flow spec.
- Before .NET backend work, load `.github/skills/dotnet-skills/dotnet-best-practices/SKILL.md` plus the relevant specialized .NET skill for the task.
- Keep `Nastart.Api` focused on HTTP mapping, auth context, request validation, and result translation. Put business logic in `Nastart.Application` and `Nastart.Domain`.
- For business logic, follow TDD: write the lowest-level failing test first, confirm it fails for the right reason, implement the smallest change to pass, then refactor.
- If no suitable test project exists yet, ask before adding test packages or scaffolding new test infrastructure.

---

## What Phase 1 Delivers

A running .NET 10 API with:
- PostgreSQL schema: all v1 entities migrated and live
- JWT auth: register, verify email, login → receive token
- Ingredient CRUD + price history API
- Vue.js login screen + ingredient management page

**Phase 1 Checkpoint:** Log in, create an ingredient, manually update its price, see price history.

---

## Repo to Create

```bash
mkdir nastart && cd nastart
dotnet new sln --format sln -n Nastart
dotnet new webapi -n Nastart.Api -o src/Nastart.Api --framework net10.0
dotnet new classlib -n Nastart.Domain -o src/Nastart.Domain --framework net10.0
dotnet new classlib -n Nastart.Application -o src/Nastart.Application --framework net10.0
dotnet new classlib -n Nastart.Infrastructure -o src/Nastart.Infrastructure --framework net10.0
dotnet sln add src/**/*.csproj
```

**Project references (inner rings only):**
```bash
dotnet add src/Nastart.Application reference src/Nastart.Domain
dotnet add src/Nastart.Infrastructure reference src/Nastart.Domain
dotnet add src/Nastart.Infrastructure reference src/Nastart.Application
dotnet add src/Nastart.Api reference src/Nastart.Domain
dotnet add src/Nastart.Api reference src/Nastart.Application
dotnet add src/Nastart.Api reference src/Nastart.Infrastructure
```

---

## v1 Entities to Scaffold (Phase 1 subset)

Build these in Phase 1. Recipe, RecipeItem, CascadeErrorLog come in Phase 2 (L7–L8).

```
User
TelegramLink
Unit
Category
Ingredient
IngredientPriceHistory
```

**Entity shapes:** See `.github/context/v1-constraints.md` → "Entity Shapes — v1"

---

## NuGet Packages to Install

```bash
# Infrastructure
dotnet add src/Nastart.Infrastructure package Microsoft.EntityFrameworkCore
dotnet add src/Nastart.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL

# API (design-time tools for migrations)
dotnet add src/Nastart.Api package Microsoft.EntityFrameworkCore.Design

# Application (IAppDbContext uses DbSet<T>)
dotnet add src/Nastart.Application package Microsoft.EntityFrameworkCore

# MediatR
dotnet add src/Nastart.Application package MediatR
dotnet add src/Nastart.Api package MediatR

# Validation + result pattern
dotnet add src/Nastart.Application package FluentValidation
dotnet add src/Nastart.Application package FluentValidation.DependencyInjectionExtensions
dotnet add src/Nastart.Application package ErrorOr

# Auth
dotnet add src/Nastart.Api package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add src/Nastart.Infrastructure package Microsoft.IdentityModel.JsonWebTokens
dotnet add src/Nastart.Infrastructure package BCrypt.Net-Next

# EF Core CLI (global)
dotnet tool install --global dotnet-ef
```

---

## File Locations

```
src/Nastart.Domain/
├── Common/
│   └── BaseEntity.cs
├── Entities/
│   ├── User.cs
│   ├── TelegramLink.cs
│   ├── Unit.cs
│   ├── Category.cs
│   ├── Ingredient.cs
│   └── IngredientPriceHistory.cs
└── Enums/
    ├── TelegramLinkStatus.cs
    └── PriceSource.cs             ← 'Manual' | 'InvoiceScan' (C-13)

src/Nastart.Application/
├── Common/
│   ├── Interfaces/
│   │   ├── IAppDbContext.cs
│   │   ├── ITokenService.cs
│   │   └── ICostCascadeService.cs  ← Interface only; implementation comes in L7
│   └── Behaviors/
│       └── ValidationBehavior.cs  ← MediatR pipeline: runs FluentValidation before handlers
├── Features/
│   ├── Auth/
│   │   ├── Commands/Register/
│   │   ├── Commands/Login/
│   │   └── Commands/VerifyEmail/
│   └── Ingredients/               ← Full CRUD + price history (L6)
└── DependencyInjection.cs

src/Nastart.Infrastructure/
├── Persistence/
│   ├── AppDbContext.cs
│   ├── Configurations/
│   │   ├── UserConfiguration.cs
│   │   ├── TelegramLinkConfiguration.cs
│   │   ├── IngredientConfiguration.cs
│   │   ├── IngredientPriceHistoryConfiguration.cs
│   │   ├── UnitConfiguration.cs
│   │   └── CategoryConfiguration.cs
│   └── Migrations/
├── Services/
│   └── JwtTokenService.cs
└── DependencyInjection.cs

src/Nastart.Api/
├── Program.cs
├── Endpoints/
│   ├── AuthEndpoints.cs
│   └── IngredientEndpoints.cs
├── Middleware/
│   └── ExceptionHandlingMiddleware.cs
└── appsettings.json
```

---

## Docker Compose (local PostgreSQL)

```yaml
# docker-compose.yml at repo root
services:
  postgres:
    image: postgres:18-alpine
    environment:
      POSTGRES_DB: nastart_dev
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
```

**Connection string (appsettings.json):**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=nastart_dev;Username=dev;Password=dev_password"
  },
  "JwtSettings": {
    "Secret": "REPLACE_WITH_32+_CHAR_SECRET_IN_USER_SECRETS",
    "Issuer": "Nastart",
    "Audience": "NastartApp",
    "ExpirationHours": 24
  }
}
```

> **Security:** Never commit real JWT secrets. Use `dotnet user-secrets` for local dev.

---

## ITokenService — v1 Signature

```csharp
public interface ITokenService
{
    string GenerateToken(Guid userId, string email);
}
```

**JWT claims emitted (v1 only):**
```csharp
new Claim("sub", userId.ToString()),
new Claim("email", email)
// exp set automatically via SecurityTokenDescriptor.Expires
```

---

## IAppDbContext — v1 (Phase 1 subset)

```csharp
public interface IAppDbContext
{
    DbSet<User> Users { get; }
    DbSet<TelegramLink> TelegramLinks { get; }
    DbSet<Ingredient> Ingredients { get; }
    DbSet<IngredientPriceHistory> IngredientPriceHistories { get; }
    DbSet<Unit> Units { get; }
    DbSet<Category> Categories { get; }
    // Recipe, RecipeItem, CascadeErrorLog added in Phase 2 (L7)
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

---

## Authorization Setup

```csharp
// Program.cs — v1 single-user auth
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["JwtSettings:Issuer"],
            ValidAudience = builder.Configuration["JwtSettings:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["JwtSettings:Secret"]!))
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// All protected endpoints: .RequireAuthorization()
// Public endpoints (login, register): .AllowAnonymous()
```

---

## Phase 1 API Contract

```
POST   /api/auth/register           → { email, password }
POST   /api/auth/login              → { token }
POST   /api/auth/verify-email       → { token: emailVerificationToken }

GET    /api/ingredients             → IngredientListItem[]     (current price via C-3)
POST   /api/ingredients             → IngredientResponse
GET    /api/ingredients/{id}        → IngredientResponse
PUT    /api/ingredients/{id}        → IngredientResponse
DELETE /api/ingredients/{id}        → 204

POST   /api/ingredients/{id}/prices → { price, effectiveDate, source:'Manual' }
GET    /api/ingredients/{id}/prices → PriceHistoryItem[]

GET    /health                      → { status: "healthy" }
```

---

## Known Gotchas

1. **`committed_at` is set by the server, not the client.** In `AddIngredientPrice` handler: never accept `committed_at` from request body. Set it as `DateTime.UtcNow` on INSERT.

2. **`PriceSource` is stored as a string.** Configure EF: `entity.Property(p => p.Source).HasConversion<string>()`. Values must serialize as `"Manual"` or `"InvoiceScan"` — not `0` or `1`.

3. **Ingredient name uniqueness is per-user.** Unique index: `(UserId, Name)` — not global, not per-outlet.

4. **Category is user-scoped.** `Category.UserId FK → User.Id` — users create their own categories.

5. **Unit is global reference data.** `Unit` has no `UserId` — it's a shared lookup table.

6. **Password hashing.** Use BCrypt: `BCrypt.Net.BCrypt.HashPassword(password)` and `BCrypt.Net.BCrypt.Verify(password, hash)`. Never store plaintext.

7. **Email verification token.** Generate with `Convert.ToBase64String(RandomNumberGenerator.GetBytes(32))`. Hash it for storage. The raw token goes in the email link; the hash is stored in the database.

---

## Phase 1 Complete When

- [ ] `docker compose up -d` starts PostgreSQL
- [ ] `dotnet ef database update` runs all migrations without error
- [ ] Any business logic added in Phase 1 was introduced through a failing test first, or a test-infrastructure blocker was explicitly documented
- [ ] `POST /api/auth/register` creates a user
- [ ] `POST /api/auth/login` returns a valid JWT
- [ ] `GET /api/ingredients` (with JWT) returns empty array
- [ ] `POST /api/ingredients` creates an ingredient
- [ ] `POST /api/ingredients/{id}/prices` appends a price
- [ ] `GET /api/ingredients` returns the ingredient with its current price
- [ ] Vue.js login screen authenticates and displays the ingredient management page
