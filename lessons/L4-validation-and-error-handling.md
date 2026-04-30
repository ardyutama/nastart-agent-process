# Lesson 4 — Validation, Error Handling & the Result Pattern

**← Previous:** L3 — Vertical Slice Architecture with MediatR  
**→ Next:** L5 — JWT Authentication & Authorization  
**→ Used in:** L6 (full ingredient CRUD slices), L7 (cascade command slice), L8 (recipe costing slices)

---

> **What you'll learn:**
> - Why you should never throw exceptions for business logic errors
> - The `ErrorOr<T>` result pattern — return errors as data, not exceptions
> - FluentValidation — declarative validation rules for every Command and Query
> - MediatR Pipeline Behaviors — middleware that validates requests before handlers run
> - Global error handling middleware for unhandled exceptions

> ⚠️ **v1 Solopreneur Amendments (April 9, 2026)**
>
> This lesson teaches FluentValidation + ErrorOr + Pipeline Behaviors using a `CreateIngredient` slice. The patterns are identical between v1 and v2 — only scoping differs.
>
> **Scope — User-scoped ingredients (not outlet-scoped):**
>
> | v2 pattern (teaching concept) | v1 replacement |
> |---|---|
> | `Guid OutletId` in `CreateIngredientCommand` | `Guid UserId` |
> | `i.OutletId == cmd.OutletId` duplicate check | `i.UserId == cmd.UserId` |
> | `outletId` from route parameter | `userId` extracted from JWT claim |
>
> **Note:** The code examples in this lesson already use `UserId` — they are v1-ready. The table above documents the conceptual difference for reference.
>
> **Endpoint dependency:** Endpoints in this lesson call `httpContext.User.GetUserId()`. This extension method is **defined in L5**. See the prerequisite note in Section 6.
>
> **Teaching scaffold:** The `CreateIngredient` slice here is a simplified teaching example. **L6 replaces it** with the production-ready version (full CRUD, price history). After completing L6, delete this L4 scaffold.
>
> The validation pattern (FluentValidation + MediatR pipeline behavior + ErrorOr) applies unchanged.

---

## 1. The Problem with Exceptions for Business Logic

Consider this handler:

```csharp
// BAD — using exceptions for expected business cases
public async Task<CreateIngredientResponse> Handle(CreateIngredientCommand cmd, CancellationToken ct)
{
    var exists = await _db.Ingredients.AnyAsync(i => i.UserId == cmd.UserId && i.Name == cmd.Name, ct);
    if (exists)
        throw new ConflictException("Ingredient already exists"); // This is NOT exceptional

    // ...
}
```

A duplicate ingredient name is a **normal business case**, not an unexpected error. Exceptions should be reserved for truly unexpected situations (database connection lost, null reference, out of memory).

Using exceptions for business logic means:
- Stack trace overhead for every validation failure
- Try-catch blocks scattered everywhere
- No type safety — you catch `Exception` and hope it's the right type

---

## 2. ErrorOr — Return Errors as Data

`ErrorOr<T>` is a library that makes handlers return either a **success value** or a **list of errors** — no exceptions needed.

```bash
dotnet add src/Nastart.Application package ErrorOr
```

How it works:

```csharp
// Handler returns ErrorOr<CreateIngredientResponse>
// On success: the response object
// On failure: one or more Error objects
public async Task<ErrorOr<CreateIngredientResponse>> Handle(...)
{
    var exists = await _db.Ingredients.AnyAsync(...);
    if (exists)
        return Error.Conflict("Ingredient.Duplicate", "An ingredient with this name already exists.");

    // ... create and return success
    return new CreateIngredientResponse(ingredient.Id, ingredient.Name);
}
```

`ErrorOr<T>` is **implicitly convertible** — you return either `Error.XXX(...)` or the success value directly. No wrapping needed.

### Error types

| Method | HTTP Status | When to use |
|---|---|---|
| `Error.Validation(...)` | 400 | Input fails validation rules |
| `Error.NotFound(...)` | 404 | Requested resource doesn't exist |
| `Error.Conflict(...)` | 409 | Duplicate, already exists |
| `Error.Unauthorized(...)` | 401 | Not authenticated |
| `Error.Forbidden(...)` | 403 | Authenticated but wrong role |
| `Error.Failure(...)` | 500 | General failure |

---

## 3. Building a Complete Command Slice with ErrorOr

Let's build `CreateIngredient` — the first full slice with validation and error handling.

```bash
mkdir -p src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient
```

### The Command

**File:** `src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient/CreateIngredientCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Ingredients.Commands.CreateIngredient;

// Returns ErrorOr — handler can return success OR typed errors
public record CreateIngredientCommand(
    string Name,
    Guid UserId,
    Guid? CategoryId,
    Guid UnitId,
    decimal UnitSize,
    decimal PriceSpikeThresholdPct = 10m,
    decimal? InitialPrice = null,
    DateOnly? EffectiveDate = null
) : IRequest<ErrorOr<CreateIngredientResponse>>;
```

### The Response

**File:** `src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient/CreateIngredientResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Commands.CreateIngredient;

public record CreateIngredientResponse(Guid Id, string Name);
```

### The Handler

**File:** `src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient/CreateIngredientHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Application.Features.Ingredients.Commands.CreateIngredient;

// Primary constructor — db parameter is captured and accessible in all methods (C# 12 / .NET 8+)
public class CreateIngredientHandler(IAppDbContext db)
    : IRequestHandler<CreateIngredientCommand, ErrorOr<CreateIngredientResponse>>
{
    public async Task<ErrorOr<CreateIngredientResponse>> Handle(
        CreateIngredientCommand command, CancellationToken ct)
    {
        // Business rule: ingredient name must be unique within the user's ingredients
        var exists = await db.Ingredients
            .AnyAsync(i => i.UserId == command.UserId && i.Name == command.Name, ct)
            .ConfigureAwait(false);

        if (exists)
            return Error.Conflict("Ingredient.Duplicate",
                "An ingredient with this name already exists.");

        // Verify the unit exists
        var unitExists = await db.Units
            .AnyAsync(u => u.Id == command.UnitId, ct)
            .ConfigureAwait(false);
        if (!unitExists)
            return Error.NotFound("Unit.NotFound", "The specified unit does not exist.");

        var ingredient = new Ingredient
        {
            Id = Guid.NewGuid(),
            Name = command.Name,
            UserId = command.UserId,
            CategoryId = command.CategoryId,
            UnitId = command.UnitId,
            UnitSize = command.UnitSize,
            PriceSpikeThresholdPct = command.PriceSpikeThresholdPct
        };

        db.Ingredients.Add(ingredient);

        // If initial price provided, create the first price history record
        // C-3: Current price is always derived from IngredientPriceHistory
        if (command.InitialPrice.HasValue)
        {
            var priceRecord = new IngredientPriceHistory
            {
                Id = Guid.NewGuid(),
                IngredientId = ingredient.Id,
                Price = command.InitialPrice.Value,
                UnitSize = command.UnitSize,
                Source = PriceSource.Manual,
                // C-4: CommittedAt is NOT set here — the DB sets it via HasDefaultValueSql("NOW()")
                EffectiveDate = command.EffectiveDate ?? DateOnly.FromDateTime(DateTime.UtcNow)
            };
            db.IngredientPriceHistories.Add(priceRecord);
        }

        await db.SaveChangesAsync(ct).ConfigureAwait(false);

        return new CreateIngredientResponse(ingredient.Id, ingredient.Name);
    }
}
```

---

## 4. FluentValidation — Declarative Input Rules

FluentValidation lets you define validation rules as a class, separate from your handler. Rules fire before the handler executes.

```bash
dotnet add src/Nastart.Application package FluentValidation
dotnet add src/Nastart.Application package FluentValidation.DependencyInjectionExtensions
```

### Validator for CreateIngredient

**File:** `src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient/CreateIngredientCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Ingredients.Commands.CreateIngredient;

public class CreateIngredientCommandValidator : AbstractValidator<CreateIngredientCommand>
{
    public CreateIngredientCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Ingredient name is required.")
            .MaximumLength(255).WithMessage("Name cannot exceed 255 characters.");

        RuleFor(x => x.UserId)
            .NotEmpty().WithMessage("UserId is required.");

        RuleFor(x => x.UnitId)
            .NotEmpty().WithMessage("UnitId is required.");

        RuleFor(x => x.UnitSize)
            .GreaterThan(0).WithMessage("Unit size must be greater than zero.");

        RuleFor(x => x.PriceSpikeThresholdPct)
            .InclusiveBetween(1, 100).WithMessage("Spike threshold must be between 1% and 100%.");

        RuleFor(x => x.InitialPrice)
            .GreaterThan(0).When(x => x.InitialPrice.HasValue)
            .WithMessage("Initial price must be greater than zero.");
    }
}
```

---

## 5. MediatR Pipeline Behavior — Auto-Validation

A Pipeline Behavior is middleware that intercepts every MediatR request before the handler runs. We'll create one that finds the validator for the current request and runs it automatically.

**File:** `src/Nastart.Application/Common/Behaviors/ValidationBehavior.cs`

```csharp
using ErrorOr;
using FluentValidation;
using MediatR;

namespace Nastart.Application.Common.Behaviors;

// This behavior intercepts every MediatR request.
// If a validator exists for the request type, it runs BEFORE the handler.
// If validation fails, the handler is never called — errors return immediately.
//
// Primary constructor: validator parameter is optional — if no IValidator<TRequest> is
// registered in DI for this request type, the parameter is null and validation is skipped.
public class ValidationBehavior<TRequest, TResponse>(IValidator<TRequest>? validator = null)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
    where TResponse : IErrorOr
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        // No validator registered for this request type — skip validation
        if (validator is null)
            return await next().ConfigureAwait(false);

        var validationResult = await validator
            .ValidateAsync(request, cancellationToken)
            .ConfigureAwait(false);

        if (validationResult.IsValid)
            return await next().ConfigureAwait(false);

        // Convert FluentValidation errors to ErrorOr errors
        var errors = validationResult.Errors
            .Select(e => Error.Validation(e.PropertyName, e.ErrorMessage))
            .ToList();

        // Cast through object to leverage ErrorOr's implicit List<Error> conversion.
        // (TResponse)(object) is safer than (dynamic) — keeps compile-time type visibility.
        return (TResponse)(object)errors;
    }
}
```

### Register validators and behavior in DI

**Update** `src/Nastart.Application/DependencyInjection.cs` (extends the version created in L3 — do not replace the file from scratch):

```csharp
using FluentValidation;
using Microsoft.Extensions.DependencyInjection;
using Nastart.Application.Common.Behaviors;

namespace Nastart.Application;

public static class DependencyInjection
{
    public static IServiceCollection AddApplication(this IServiceCollection services)
    {
        var assembly = typeof(DependencyInjection).Assembly;

        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(assembly);

            // Register the validation pipeline behavior
            // This runs BEFORE every handler automatically
            cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
        });

        // Scan and register all validators in the Application assembly
        services.AddValidatorsFromAssembly(assembly);

        return services;
    }
}
```

**Flow summary:**

```
HTTP Request
  → Endpoint creates Command
    → MediatR receives Command
      → ValidationBehavior intercepts
        → Finds CreateIngredientCommandValidator
          → Runs rules
            → PASS: calls the Handler
            → FAIL: returns Error list (handler never runs)
```

---

## 6. Endpoint Error Mapping

Endpoints need to translate `ErrorOr<T>` results into proper HTTP responses. Create a helper:

**File:** `src/Nastart.API/Extensions/ResultExtensions.cs`

```csharp
using ErrorOr;

namespace Nastart.API.Extensions;

public static class ResultExtensions
{
    // Maps ErrorOr<T> to appropriate HTTP result
    public static IResult ToApiResult<T>(this ErrorOr<T> result)
    {
        if (!result.IsError)
            return Results.Ok(result.Value);

        return result.FirstError.Type switch
        {
            ErrorType.Validation => Results.BadRequest(new
            {
                errors = result.Errors.Select(e => new { e.Code, e.Description })
            }),
            ErrorType.NotFound => Results.NotFound(new
            {
                error = result.FirstError.Description
            }),
            ErrorType.Conflict => Results.Conflict(new
            {
                error = result.FirstError.Description
            }),
            ErrorType.Unauthorized => Results.Unauthorized(),
            ErrorType.Forbidden => Results.Forbid(),
            _ => Results.Problem(detail: result.FirstError.Description, statusCode: 500)
        };
    }

    // Same but returns 201 Created on success (for POST endpoints)
    public static IResult ToCreatedResult<T>(this ErrorOr<T> result, string location)
    {
        if (!result.IsError)
            return Results.Created(location, result.Value);

        return result.ToApiResult();
    }
}
```

### Update the Ingredient endpoint

> ⚠️ **Prerequisite — `ClaimsPrincipalExtensions.GetUserId()`**
>
> The calls to `httpContext.User.GetUserId()` below require `ClaimsPrincipalExtensions`, which is **defined in L5**. If you haven't completed L5 yet, use this temporary stub in `src/Nastart.API/Extensions/ClaimsPrincipalExtensions.cs`:
> ```csharp
> public static class ClaimsPrincipalExtensions
> {
>     // L5 replaces this with a real JWT claim extraction
>     public static Guid GetUserId(this System.Security.Claims.ClaimsPrincipal user)
>         => Guid.Parse("c0000000-0000-0000-0000-000000000001");
> }
> ```
> Delete this stub when you complete L5.

**File:** `src/Nastart.API/Endpoints/IngredientEndpoints.cs`

```csharp
using MediatR;
using Nastart.API.Extensions;
using Nastart.Application.Features.Ingredients.Commands.CreateIngredient;
using Nastart.Application.Features.Ingredients.Queries.GetIngredients;

namespace Nastart.API.Endpoints;

public static class IngredientEndpoints
{
    public static void MapIngredientEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/ingredients")
            .WithTags("Ingredients");

        group.MapGet("/", async (ISender sender, HttpContext httpContext, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            var result = await sender.Send(new GetIngredientsQuery(userId), ct);
            return Results.Ok(result);
        });

        group.MapPost("/", async (CreateIngredientRequest request, ISender sender, HttpContext httpContext, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            var command = new CreateIngredientCommand(
                Name: request.Name,
                UserId: userId,
                CategoryId: request.CategoryId,
                UnitId: request.UnitId,
                UnitSize: request.UnitSize,
                PriceSpikeThresholdPct: request.PriceSpikeThresholdPct,
                InitialPrice: request.InitialPrice,
                EffectiveDate: request.EffectiveDate);

            var result = await sender.Send(command, ct);

            // Check error first — only access result.Value on the success path
            if (result.IsError)
                return result.ToApiResult();

            return Results.Created($"/api/ingredients/{result.Value!.Id}", result.Value);
        });
    }
}

// Request body shape — separate from the Command to allow JWT claim extraction (userId) + body merging
public record CreateIngredientRequest(
    string Name,
    Guid? CategoryId,
    Guid UnitId,
    decimal UnitSize,
    decimal PriceSpikeThresholdPct = 10m,
    decimal? InitialPrice = null,
    DateOnly? EffectiveDate = null);
```

---

## 7. Global Exception Handler

For truly unexpected errors (database connection lost, null reference), add a global handler as a safety net:

**File:** `src/Nastart.API/Middleware/GlobalExceptionHandler.cs`

```csharp
using System.Net;
using Microsoft.AspNetCore.Diagnostics;

namespace Nastart.API.Middleware;

public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);

        httpContext.Response.StatusCode = (int)HttpStatusCode.InternalServerError;
        await httpContext.Response.WriteAsJsonAsync(new
        {
            error = "An unexpected error occurred.",
            // Never expose exception details in production
            detail = httpContext.RequestServices
                .GetRequiredService<IHostEnvironment>().IsDevelopment()
                    ? exception.Message
                    : null
        }, cancellationToken);

        return true;
    }
}
```

Register in `Program.cs`:

```csharp
using Nastart.API.Middleware;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration, builder.Environment);
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();

app.UseExceptionHandler();

app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));
// app.MapOutletEndpoints(); // v2-only — outlets excluded from v1
app.MapIngredientEndpoints();

app.Run();
```

---

## 8. The Error Handling Stack

All three layers work together:

```
Layer 1: FluentValidation (ValidationBehavior)
  → Catches invalid input BEFORE the handler runs
  → Returns 400 Bad Request with field-level errors

Layer 2: ErrorOr<T> in Handlers
  → Handles expected business cases (duplicate, not found, forbidden)
  → Returns typed errors (409 Conflict, 404 Not Found, etc.)

Layer 3: GlobalExceptionHandler
  → Catches truly unexpected errors (DB connection, null ref)
  → Returns 500 Internal Server Error
  → Logs the full exception for debugging
```

No try-catch blocks in your handlers. No exceptions thrown for business logic. Clean, predictable error flow.

---

## Checkpoint — Verify Before Moving to L5

```bash
# 1. Build
dotnet build

# 2. Start API
dotnet run --project src/Nastart.API

# 3. Test validation — empty name should fail with 400
curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -d '{"name": "", "unitId": "00000000-0000-0000-0000-000000000000", "unitSize": 0}'
# Expected: 400 with validation errors

# 4. Test missing unit — should fail with 404
curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -d '{"name": "Chicken Breast", "unitId": "deadbeef-0000-0000-0000-000000000001", "unitSize": 1.0}'
# Expected: 404 "The specified unit does not exist."

# 5. Test success — create a unit first, then create ingredient
# Insert a unit directly:
docker exec -it $(docker compose ps -q postgres) psql -U dev -d recipe_cost_dev -c \
  "INSERT INTO units (id, name, abbreviation, created_at) VALUES ('b0000000-0000-0000-0000-000000000001', 'kilogram', 'kg', NOW());"

curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -d '{"name": "Chicken Breast", "unitId": "b0000000-0000-0000-0000-000000000001", "unitSize": 1.0, "initialPrice": 12.50}'
# Expected: 201 Created with {"id":"...","name":"Chicken Breast"}

# 6. Test duplicate — same request again
# Expected: 409 Conflict "An ingredient with this name already exists."
```

### What you now understand

- `ErrorOr<T>`: handlers return success OR errors — no exceptions for business logic
- FluentValidation: declarative rules per-command, auto-discovered by DI
- `ValidationBehavior`: MediatR pipeline middleware that validates before the handler
- `GlobalExceptionHandler`: safety net for unexpected errors only
- The three-layer error stack: Validation → Business errors → Unexpected exceptions
