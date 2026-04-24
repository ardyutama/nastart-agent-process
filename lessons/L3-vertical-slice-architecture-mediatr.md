# Lesson 3 — Vertical Slice Architecture with MediatR

> **What you'll learn:**
> - What CQRS (Command Query Responsibility Segregation) means and why it matters
> - How MediatR dispatches requests to handlers without controllers knowing anything about them
> - How to structure feature folders as vertical slices
> - How to build your first Query slice and wire it to a Minimal API endpoint
> - How to apply primary constructor syntax to MediatR handlers
> - How v1 (user-scoped) differs from v2 (outlet-scoped) slice shapes

---

**← Previous:** L2 — EF Core + PostgreSQL Schema  
**→ Next:** L4 — Validation, Error Handling & ErrorOr  
**→ Used in:** L6 (full ingredient CRUD slices), L7 (cascade command slice), L8 (recipe costing slices)

---

> ⚠️ **v1 Solopreneur Amendments**
>
> This lesson's teaching examples use outlet slices. **Outlets are v2-only — do not build them in v1.**  
> Follow the structural pattern (folder layout, handler shape, endpoint registration) but apply it to the ingredient slice instead.
>
> | v2-only artifact (teaching example) | v1 replacement / action |
> |---|---|
> | `GetOutletsQuery` / `GetOutletsHandler` — `Application/Features/Outlets/Queries/GetOutlets/` | Build `GetIngredientsQuery` / `GetIngredientsHandler` in L6 |
> | `CreateOutletCommand` / `CreateOutletHandler` — `Application/Features/Outlets/Commands/CreateOutlet/` | Build `CreateIngredientCommand` / `CreateIngredientHandler` in L6 |
> | `OutletEndpoints.cs` | Delete this file — do not create it |
> | `app.MapOutletEndpoints()` in `Program.cs` | Remove this line |
>
> **The first real v1 vertical slice is `GetIngredients`, built in L6.** It is user-scoped (`userId` from JWT claims), not outlet-scoped.

---

## 1. CQRS — Commands vs Queries

CQRS splits every operation into two categories:

| Type | Purpose | Changes data? | Example |
|---|---|---|---|
| **Command** | Do something | Yes — writes, updates, deletes | `CreateIngredientCommand`, `UpdatePriceCommand` |
| **Query** | Ask something | No — read-only | `GetIngredientsQuery`, `GetIngredientByIdQuery` |

Why separate them?
- Commands carry validation, side effects, and event triggers
- Queries are pure reads — no side effects, safe to cache
- Reads are typically 10× more frequent than writes; separating them enables independent scaling
- Each slice is self-contained — one folder per feature, no shared "service" class with 30 methods

---

## 2. MediatR — The Dispatcher

MediatR decouples "who sends a request" from "who handles it." An API endpoint sends a request object into MediatR; MediatR routes it to the correct handler. The endpoint never imports or knows about the handler class.

```
Endpoint → MediatR.Send(request) → Handler → Database → Response
   ↑                                                        │
   └────────────────────────────────────────────────────────┘
```

Three MediatR types you'll use:

| Type | Purpose |
|---|---|
| `IRequest<TResponse>` | A request object (Command or Query) that returns `TResponse` |
| `IRequestHandler<TRequest, TResponse>` | The handler that processes the request |
| `IPipelineBehavior<TRequest, TResponse>` | A middleware that runs before/after every handler (validation, logging) — covered in L4 |

---

## 3. Install MediatR

```bash
dotnet add src/Nastart.Application package MediatR
dotnet add src/Nastart.API package MediatR
```

Register MediatR in DI. Create a DI extension for the Application layer:

**File:** `src/Nastart.Application/DependencyInjection.cs`

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace Nastart.Application;

/// <summary>
/// Wires up all Application-layer services into the DI container.
/// Call <see cref="AddApplication"/> from <c>Program.cs</c>.
/// </summary>
public static class DependencyInjection
{
    /// <summary>
    /// Scans the Application assembly and registers every
    /// <see cref="MediatR.IRequestHandler{TRequest,TResponse}"/> implementation with MediatR.
    /// </summary>
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        services.AddMediatR(cfg =>
            cfg.RegisterServicesFromAssembly(typeof(DependencyInjection).Assembly));

        return services;
    }
}
```

Update `Program.cs`:

**File:** `src/Nastart.API/Program.cs`

```csharp
using Nastart.Application;
using Nastart.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration, builder.Environment);

var app = builder.Build();

app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));

// Endpoint groups will be mapped here
app.Run();
```

---

## 4. Feature Folder Convention

Every feature slice lives inside `Application/Features/`. The convention:

```
Application/
  Features/
    {DomainArea}/
      Commands/
        {ActionName}/
          {ActionName}Command.cs              ← the request (input)
          {ActionName}Handler.cs              ← the business logic
          {ActionName}CommandValidator.cs     ← input validation (L4)
          {ActionName}Response.cs             ← the output DTO
      Queries/
        {ActionName}/
          {ActionName}Query.cs
          {ActionName}Handler.cs
          {ActionName}QueryValidator.cs       ← input validation (L4, where needed)
          {ActionName}Response.cs
```

Each folder is a self-contained unit. When you open a feature folder, every file needed to understand that feature is right there — no jumping between `Services/`, `DTOs/`, `Repositories/`.

---

> ⚠️ **v2-only — do not build in v1.** Outlet management is introduced when the product expands to multi-outlet teams. Skip this slice entirely. The first real vertical slice you build is `GetIngredients` in L6.

## 5. Your First Query — GetOutlets

> ⚠️ **v2-only — do not build in v1.** Outlet management is a v2 multi-outlet feature. The first real vertical slice you build in v1 is `GetIngredients` in L6.

Let's build a complete query slice. This returns all outlets for a company.

> **Note:** This lesson builds endpoints WITHOUT authentication. L5 will add JWT middleware and `RequireAuthorization()` to all endpoints. We separate concerns: first learn the CQRS pattern, then layer security on top.

### Create the folder structure

```bash
mkdir -p src/Nastart.Application/Features/Outlets/Queries/GetOutlets
```

### The Query (request object)

**File:** `src/Nastart.Application/Features/Outlets/Queries/GetOutlets/GetOutletsQuery.cs`

```csharp
using MediatR;

namespace Nastart.Application.Features.Outlets.Queries.GetOutlets;

// This is a Query (read-only) — it returns a list of outlets for a given company.
// Implements IRequest<T> so MediatR knows the return type.
public record GetOutletsQuery(Guid CompanyId) : IRequest<List<GetOutletsResponse>>;
```

### The Response DTO

**File:** `src/Nastart.Application/Features/Outlets/Queries/GetOutlets/GetOutletsResponse.cs`

```csharp
namespace Nastart.Application.Features.Outlets.Queries.GetOutlets;

// Response DTO — only the data the API consumer needs.
// This is NOT the Outlet entity. We never return domain entities directly.
public record GetOutletsResponse(Guid Id, string Name, DateTimeOffset CreatedAt);
```

### The Handler

**File:** `src/Nastart.Application/Features/Outlets/Queries/GetOutlets/GetOutletsHandler.cs`

```csharp
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Outlets.Queries.GetOutlets;

/// <summary>Handles <see cref="GetOutletsQuery"/> — returns all outlets for a company.</summary>
// Constructor injection — MediatR resolves db from DI; primary constructor captures it as a field.
public class GetOutletsHandler(IAppDbContext db) : IRequestHandler<GetOutletsQuery, List<GetOutletsResponse>>
{
    public async Task<List<GetOutletsResponse>> Handle(
        GetOutletsQuery request, CancellationToken cancellationToken)
    {
        // Query filters outlets by CompanyId — each outlet belongs to one company
        var outlets = await db.Outlets
            .AsNoTracking()
            .Where(o => o.CompanyId == request.CompanyId)
            .Select(o => new GetOutletsResponse(o.Id, o.Name, o.CreatedAt))
            .ToListAsync(cancellationToken)
            .ConfigureAwait(false);

        return outlets;
    }
}
```

**What to notice:**
- The handler depends on `IAppDbContext` (the interface), not `AppDbContext` (the implementation)
- It uses `Select()` to project directly into the response DTO — no intermediate object
- It receives `CancellationToken` so the query can be cancelled if the HTTP request is aborted

---

## 6. Your First Command — CreateOutlet

> ⚠️ **v2-only — do not build in v1.** Outlet management is a v2 multi-outlet feature. Use `CreateIngredientCommand` from L6 as the v1 equivalent.

Now a write operation. Commands change state — they insert, update, or delete data.

```bash
mkdir -p src/Nastart.Application/Features/Outlets/Commands/CreateOutlet
```

### The Command

**File:** `src/Nastart.Application/Features/Outlets/Commands/CreateOutlet/CreateOutletCommand.cs`

```csharp
using MediatR;

namespace Nastart.Application.Features.Outlets.Commands.CreateOutlet;

// A Command mutates state — it creates a new outlet.
// Returns the created outlet's response data.
public record CreateOutletCommand(string Name, Guid CompanyId) : IRequest<CreateOutletResponse>;
```

> **Note for v1 commands (see L4):** In v1, commands that can fail with business errors return `ErrorOr<TResponse>` instead of `TResponse` directly. This v2 teaching example uses the simpler form to focus on structure — L4 adds `ErrorOr<T>` to every command.

### The Response

**File:** `src/Nastart.Application/Features/Outlets/Commands/CreateOutlet/CreateOutletResponse.cs`

```csharp
namespace Nastart.Application.Features.Outlets.Commands.CreateOutlet;

public record CreateOutletResponse(Guid Id, string Name);
```

### The Handler

**File:** `src/Nastart.Application/Features/Outlets/Commands/CreateOutlet/CreateOutletHandler.cs`

```csharp
using MediatR;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;

namespace Nastart.Application.Features.Outlets.Commands.CreateOutlet;

/// <summary>Handles <see cref="CreateOutletCommand"/> — creates a new outlet and returns its id and name.</summary>
public class CreateOutletHandler(IAppDbContext db) : IRequestHandler<CreateOutletCommand, CreateOutletResponse>
{
    public async Task<CreateOutletResponse> Handle(
        CreateOutletCommand command, CancellationToken cancellationToken)
    {
        // Create the domain entity from command data
        var outlet = new Outlet
        {
            Id = Guid.NewGuid(),
            Name = command.Name,
            CompanyId = command.CompanyId
        };

        db.Outlets.Add(outlet);
        await db.SaveChangesAsync(cancellationToken).ConfigureAwait(false);

        // Return only what the caller needs — never the full entity
        return new CreateOutletResponse(outlet.Id, outlet.Name);
    }
}
```

---

## 7. Minimal API Endpoints — Thin Dispatchers

> ⚠️ **v2-only — do not build in v1.** The `OutletEndpoints.cs` file below is the original multi-tenant teaching example. In v1 (single-user), skip this file entirely. The v1 endpoint is `IngredientEndpoints.cs` shown in section 9.

Endpoints live in the API project. They do three things and nothing else:
1. Parse the incoming HTTP request
2. Send it to MediatR
3. Return the result

**File:** `src/Nastart.API/Endpoints/OutletEndpoints.cs`

```csharp
using MediatR;
using Nastart.Application.Features.Outlets.Commands.CreateOutlet;
using Nastart.Application.Features.Outlets.Queries.GetOutlets;

namespace Nastart.API.Endpoints;

public static class OutletEndpoints
{
    public static void MapOutletEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/outlets")
            .WithTags("Outlets");

        // GET /api/outlets?companyId=xxx
        // Will be protected with .RequireAuthorization() in L5
        group.MapGet("/", async (Guid companyId, ISender sender, CancellationToken ct) =>
        {
            // ISender is MediatR's interface for sending requests
            // CancellationToken is forwarded so the query cancels if the client disconnects
            var result = await sender.Send(new GetOutletsQuery(companyId), ct);
            return Results.Ok(result);
        });

        // POST /api/outlets
        group.MapPost("/", async (CreateOutletCommand command, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.Send(command, ct);
            return Results.Created($"/api/outlets/{result.Id}", result);
        });
    }
}
```

> **v1 developers: skip ahead to [Section 9](#9-second-feature--ingredients-query) for the actual v1 endpoint pattern.**

Register the endpoints in `Program.cs`:

```csharp
using Nastart.Application;
using Nastart.Infrastructure;
using Nastart.API.Endpoints;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration, builder.Environment);

var app = builder.Build();

app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));
// app.MapOutletEndpoints(); // v2-only — remove this line in v1

app.Run();
```

---

## 8. How the Request Flows

> ⚠️ **v2-only — do not build in v1.** This flow diagram uses the outlet example. The same MediatR dispatch pattern applies identically to any v1 command — substitute command, handler, and DbSet accordingly.


When `POST /api/outlets` is called, here's the journey:

```
1. HTTP request hits Minimal API endpoint (API layer)
2. Endpoint creates CreateOutletCommand from request body
3. Endpoint calls sender.Send(command, ct) — hands off to MediatR (ct = CancellationToken, cancels if client disconnects)
4. MediatR finds CreateOutletHandler (Application layer)
5. Handler calls _db.Outlets.Add() — IAppDbContext (Application interface)
6. AppDbContext executes INSERT (Infrastructure layer)
7. PostgreSQL stores the row (Database)
8. Handler returns CreateOutletResponse
9. MediatR returns response to endpoint
10. Endpoint returns HTTP 201 Created
```

Notice: the endpoint never imported `CreateOutletHandler`. MediatR found it automatically because we registered `AddMediatR(cfg => cfg.RegisterServicesFromAssembly(...))` in DI.

> **`CancellationToken` pattern:** Minimal API endpoints receive a `CancellationToken` automatically from ASP.NET Core when declared as a parameter. Always forward this token to `sender.Send()`. This allows long-running queries to be cancelled when the HTTP client disconnects — preventing wasted work and DB connections.

---

## 9. Second Feature — Ingredients Query

Build one more slice to solidify the pattern. Ingredients are scoped to the logged-in user (not outlets).

```bash
mkdir -p src/Nastart.Application/Features/Ingredients/Queries/GetIngredients
```

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredients/GetIngredientsQuery.cs`

```csharp
using MediatR;

namespace Nastart.Application.Features.Ingredients.Queries.GetIngredients;

/// <summary>Returns all ingredients belonging to the specified user.</summary>
public record GetIngredientsQuery(Guid UserId) : IRequest<List<IngredientListResponse>>;
```

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredients/IngredientListResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Queries.GetIngredients;

/// <summary>Projection of a single ingredient for list display.</summary>
/// <remarks>
/// Field names match the v1 Ingredient entity columns exactly.
/// <c>CategoryName</c> and <c>PriceSpikeThresholdPct</c> are nullable because
/// CategoryId is optional and PriceSpikeThresholdPct may not be set.
/// </remarks>
public record IngredientListResponse(
    Guid Id,
    string Name,
    string? CategoryName,
    string UnitAbbreviation,
    decimal UnitSize,
    decimal? PriceSpikeThresholdPct);
```

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredients/GetIngredientsHandler.cs`

```csharp
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Ingredients.Queries.GetIngredients;

/// <summary>
/// Handles <see cref="GetIngredientsQuery"/>.
/// Returns all ingredients scoped to the requesting user.
/// </summary>
public class GetIngredientsHandler(IAppDbContext db)
    : IRequestHandler<GetIngredientsQuery, List<IngredientListResponse>>
{
    public async Task<List<IngredientListResponse>> Handle(
        GetIngredientsQuery request, CancellationToken cancellationToken)
    {
        // Ingredients are user-scoped — only return ingredients for this user
        return await db.Ingredients
            .AsNoTracking()
            .Where(i => i.UserId == request.UserId)
            .Select(i => new IngredientListResponse(
                i.Id,
                i.Name,
                i.Category != null ? i.Category.Name : null,
                i.Unit.Abbreviation,
                i.UnitSize,
                i.PriceSpikeThresholdPct))
            .ToListAsync(cancellationToken)
            .ConfigureAwait(false);
    }
}
```

> **Note:** This handler returns data directly. L4 introduces `ErrorOr<T>` for commands that can fail with business errors (e.g., duplicate ingredient names).

**File:** `src/Nastart.API/Endpoints/IngredientEndpoints.cs`

```csharp
using MediatR;
using Nastart.Application.Features.Ingredients.Queries.GetIngredients;

namespace Nastart.API.Endpoints;

public static class IngredientEndpoints
{
    public static void MapIngredientEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/ingredients")
            .WithTags("Ingredients");

        // GET /api/ingredients
        // Will be protected with .RequireAuthorization() in L5
        group.MapGet("/", async (ISender sender, HttpContext httpContext, CancellationToken ct) =>
        {
            // L3: hardcoded test userId — ClaimsPrincipalExtensions.GetUserId() is added in L5.
            // Replace this line in L5 with: var userId = httpContext.User.GetUserId();
            var userId = Guid.Parse("c0000000-0000-0000-0000-000000000001");
            var result = await sender.Send(new GetIngredientsQuery(userId), ct);
            return Results.Ok(result);
        });
    }
}
```

Register in `Program.cs`:

```csharp
// app.MapOutletEndpoints(); // v2-only — remove this line in v1
app.MapIngredientEndpoints();
```

---

## 10. The Vertical Slice So Far

### v1 build — what you actually create

```
src/Nastart.Application/
├── Common/
│   ├── Behaviors/          ← empty (populated in L4)
│   └── Interfaces/
│       ├── IAppDbContext.cs
│       └── IEmailService.cs
├── DependencyInjection.cs
└── Features/
    └── Ingredients/
        └── Queries/
            └── GetIngredients/
                ├── GetIngredientsQuery.cs
                ├── GetIngredientsHandler.cs
                └── IngredientListResponse.cs
```

The `Ingredients` slice is a placeholder built in this lesson to prove the pattern. Full CRUD is added in L6.

### v2 reference — teaching examples in sections 5–8, do not build

```
src/Nastart.Application/
└── Features/
    └── Outlets/              ← v2-only: do not create this folder
        ├── Commands/
        │   └── CreateOutlet/
        │       ├── CreateOutletCommand.cs
        │       ├── CreateOutletHandler.cs
        │       └── CreateOutletResponse.cs
        └── Queries/
            └── GetOutlets/
                ├── GetOutletsQuery.cs
                ├── GetOutletsHandler.cs
                └── GetOutletsResponse.cs
```

Each feature is a vertical slice — everything needed for that operation lives in one folder. The outlet examples above illustrate the shape; the ingredients folder is the only slice you ship in v1.

---

## Checkpoint — Verify Before Moving to L4

```bash
# 1. Solution builds — confirms MediatR found all handlers at startup
dotnet build
# Expected: Build succeeded, 0 Error(s)

# 2. Start the API
dotnet run --project src/Nastart.API

# 3. Test the health endpoint first
curl http://localhost:5000/health
# Expected: {"status":"healthy"}

# 4. Insert a test user directly (v1 User columns: id, email, password_hash, is_email_verified, created_at, updated_at)
docker exec -it $(docker compose ps -q postgres) psql -U dev -d recipe_cost_dev -c \
  "INSERT INTO users (id, email, password_hash, is_email_verified, created_at, updated_at) VALUES \
  ('c0000000-0000-0000-0000-000000000001', 'test@example.com', 'placeholder', false, NOW(), NOW());"

# 5. Test the ingredients endpoint
# Auth is added in L5. The endpoint currently uses a hardcoded test userId
# (c0000000-0000-0000-0000-000000000001). Make sure the INSERT above ran first.
curl http://localhost:5000/api/ingredients
# Expected: 200 OK with an empty array []
# This confirms: endpoint → ISender → GetIngredientsHandler → DB → response
# In L5: replace the hardcoded userId with httpContext.User.GetUserId() after JWT is configured.
```

### What you now understand

- CQRS: Commands change data, Queries read data
- MediatR: dispatches requests to handlers automatically
- Feature folders: each operation is self-contained in one folder
- Minimal API endpoints: thin dispatchers that do no business logic
- The entire flow: HTTP → Endpoint → MediatR → Handler → DbContext → Database → Response
