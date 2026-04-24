**← Previous:** L5 — JWT Authentication & Authorization  
**→ Next:** L7 — Cost Cascade Service & Price Spike Alerts  
**→ Used in:** L7 (cascade trigger), L8 (recipe costing inputs)

---

# Lesson 6 — Ingredient Management: CRUD + Manual Price History

> **What you'll learn:**
> - Building a full CRUD feature for Ingredients using VSA + MediatR
> - How to append `IngredientPriceHistory` records (manual price entry)
> - Reading current price via the C-3 pattern: `ORDER BY committed_at DESC LIMIT 1`
> - User-scoped authorization: single authenticated user manages all their own ingredients
> - Querying price history with full audit trail
>
> **Out of scope for this lesson:**
> - **Price spike detection & cascade** — deferred to L7. `// TODO (L7)` comments mark where alerts and recipe recalculation will be wired.
> - **Invoice-scanned prices** — Phase 3 (L12–L14). `PriceSource.InvoiceScan` is defined but not used yet.
>
> **Note on L4's `CreateIngredient`:**  
> L4 introduced a simplified `CreateIngredient` slice as a teaching example. **This lesson replaces that slice** with the production-ready version. If you built L4's version, delete it — L6's slice is the authoritative one.

> ⚠️ **v1 Solopreneur Scope (April 9, 2026)**
>
> This lesson is already written for v1. Key differences from a multi-tenant v2 design:
>
> | Concept | v1 (this lesson) | v2 (future) |
> |---|---|---|
> | Ownership field | `Guid UserId` everywhere | `Guid OutletId` |
> | Routes | `/api/ingredients` (no `outletId` segment) | `/api/outlets/{outletId}/ingredients` |
> | Authorization | `RequireAuthorization()` — no policy name | Named policies: `"OwnerOnly"`, `"OwnerOrProcurement"` |
> | JWT claims | `userId` + `email` only | + `outletId`, `role` |
>
> The Authorization Policies Reference at the end of this lesson documents the v2 patterns for reference — do not build them in v1.

---

## Prerequisites

Complete these lessons before starting L6:

- **L1** — Clean Architecture scaffold (solution structure in place)
- **L2** — EF Core schema (`Unit`, `Category`, `Ingredient`, `IngredientPriceHistory` entities, `IAppDbContext`, `IngredientPriceHistoryConfiguration` with `HasDefaultValueSql("NOW()")` on `CommittedAt`)
- **L3** — VSA pattern, MediatR registered, `ValidationBehavior` pipeline wired
- **L4** — FluentValidation, `ErrorOr<T>`, `ResultExtensions` (`ToApiResult()`, `ToCreatedResult()`)
- **L5** — JWT auth, `User` entity, `ClaimsPrincipalExtensions.GetUserId()`, running PostgreSQL

---

## 1. Why Price History Is Append-Only

**Canonical Decisions C-3, C-4, C-5** form the core of how ingredient pricing works.

### The Problem with Mutable Prices

A naive system stores `Ingredient.current_price` directly:
- "Update the price to $5.00" → `UPDATE Ingredient SET current_price = 5.00`
- Simple, but **irreversible** — the old price is lost
- If a recipe was costed yesterday at $3.00 and repriced to $5.00 today, you can't trace the cascade

### The Append-Only Pattern (C-3, C-5)

Instead, we **never update or delete** `IngredientPriceHistory`:
- "New price is $5.00" → `INSERT INTO IngredientPriceHistory (...)`
- The current price is **always derived**: `SELECT price FROM IngredientPriceHistory ... ORDER BY committed_at DESC LIMIT 1`
- Old prices stay visible for audit, reporting, and historical cost analysis

### Two Timestamps (C-4)

Each price record has **two independent timestamps:**

| Timestamp | Set by | Purpose |
|---|---|---|
| `committed_at` | System (`HasDefaultValueSql("NOW()")`) | **Ordering field** — determines "current price" |
| `effective_date` | User | Business date — "this price was effective on Monday even though I entered it Tuesday" |

Example: You scan an invoice dated 2024-01-15 on 2024-01-16. `effective_date = 2024-01-15`, `committed_at = 2024-01-16T14:32:00Z`. The system always orders by `committed_at` to determine the current price.

---

## 2. Feature Slices to Build

Seven handlers cover all ingredient operations. Each follows Clean Architecture: Request → Validator → Handler → Response.

| Slice | Type | Purpose |
|---|---|---|
| **Slice 1** — GetIngredients | Query | List all ingredients for this user, including derived current price |
| **Slice 2** — GetIngredientById | Query | Fetch single ingredient with unit, category, and current price |
| **Slice 3** — CreateIngredient | Command | Create new ingredient; optionally insert first price record |
| **Slice 4** — UpdateIngredient | Command | Update ingredient metadata (name, unit, category, threshold) |
| **Slice 5** — DeleteIngredient | Command | Delete ingredient; blocked by L7's `OnDelete(Restrict)` constraint |
| **Slice 6** — AddIngredientPrice | Command | Append a new price record (primary price-update path) |
| **Slice 7** — GetIngredientPriceHistory | Query | Return full price history for an ingredient, newest first |

---

# Slice 1: GetIngredients — Query

**Folder:** `Application/Features/Ingredients/Queries/GetIngredients/`

Returns all ingredients for this user, including each ingredient's **current price** (derived via C-3).

```
mkdir -p src/Nastart.Application/Features/Ingredients/Queries/GetIngredients
```

### Request

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredients/GetIngredientsQuery.cs`

```csharp
using MediatR;

namespace Nastart.Application.Features.Ingredients.Queries.GetIngredients;

// C-3: Current price is derived from latest IngredientPriceHistory record
public record GetIngredientsQuery(Guid UserId) : IRequest<GetIngredientsResponse>;
```

### Response

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredients/GetIngredientsResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Queries.GetIngredients;

// Summary DTO — only fields needed for ingredient list view
public record IngredientSummary(
    Guid Id,
    string Name,
    string UnitAbbreviation,
    string CategoryName,
    decimal UnitSize,
    decimal? CurrentPrice  // Null if no price history yet
);

public record GetIngredientsResponse(List<IngredientSummary> Ingredients);
```

### Handler

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredients/GetIngredientsHandler.cs`

```csharp
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Ingredients.Queries.GetIngredients;

public class GetIngredientsHandler(IAppDbContext db)
    : IRequestHandler<GetIngredientsQuery, GetIngredientsResponse>
{
    public async Task<GetIngredientsResponse> Handle(
        GetIngredientsQuery query, CancellationToken ct)
    {
        // Fetch all ingredients for this user, joined with Unit and Category.
        // For each ingredient, derive the current price via C-3:
        //   SELECT price ORDER BY committed_at DESC LIMIT 1
        var ingredients = await db.Ingredients
            .AsNoTracking()
            .Where(i => i.UserId == query.UserId)
            .Select(i => new IngredientSummary(
                i.Id,
                i.Name,
                i.Unit.Abbreviation,
                i.Category != null ? i.Category.Name : "Uncategorized",
                i.UnitSize,
                // C-3: Latest price ordered by committed_at descending
                // Returns null if no price history exists yet
                i.PriceHistory
                    .OrderByDescending(p => p.CommittedAt)
                    .Select(p => (decimal?)p.Price)
                    .FirstOrDefault()
            ))
            .OrderBy(s => s.Name)
            .ToListAsync(ct);

        return new GetIngredientsResponse(ingredients);
    }
}
```

---

# Slice 2: GetIngredientById — Query

**Folder:** `Application/Features/Ingredients/Queries/GetIngredientById/`

Returns a single ingredient with full detail: unit, category, all metadata, and current price.

```
mkdir -p src/Nastart.Application/Features/Ingredients/Queries/GetIngredientById
```

### Request

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredientById/GetIngredientByIdQuery.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Ingredients.Queries.GetIngredientById;

// Verifies user ownership before returning detail
public record GetIngredientByIdQuery(Guid IngredientId, Guid UserId)
    : IRequest<ErrorOr<GetIngredientByIdResponse>>;
```

### Response

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredientById/GetIngredientByIdResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Queries.GetIngredientById;

public record UnitDto(Guid Id, string Name, string Abbreviation);
public record CategoryDto(Guid Id, string Name);

public record GetIngredientByIdResponse(
    Guid Id,
    string Name,
    UnitDto Unit,
    CategoryDto? Category,
    decimal UnitSize,
    decimal PriceSpikeThresholdPct,
    decimal? CurrentPrice,
    DateTime? LastPriceDate
);
```

### Handler

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredientById/GetIngredientByIdHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Ingredients.Queries.GetIngredientById;

public class GetIngredientByIdHandler(IAppDbContext db)
    : IRequestHandler<GetIngredientByIdQuery, ErrorOr<GetIngredientByIdResponse>>
{
    public async Task<ErrorOr<GetIngredientByIdResponse>> Handle(
        GetIngredientByIdQuery query, CancellationToken ct)
    {
        var ingredient = await db.Ingredients
            .AsNoTracking()
            .Include(i => i.Unit)
            .Include(i => i.Category)
            .FirstOrDefaultAsync(i => i.Id == query.IngredientId, ct);

        if (ingredient is null || ingredient.UserId != query.UserId)
            return Error.Forbidden("Ingredient.AccessDenied", "You do not have access to this ingredient.");

        // C-3: Get latest price record by committed_at descending
        var latestPrice = await db.IngredientPriceHistories
            .AsNoTracking()
            .Where(p => p.IngredientId == query.IngredientId)
            .OrderByDescending(p => p.CommittedAt)
            .FirstOrDefaultAsync(ct);

        return new GetIngredientByIdResponse(
            ingredient.Id,
            ingredient.Name,
            new UnitDto(ingredient.Unit.Id, ingredient.Unit.Name, ingredient.Unit.Abbreviation),
            ingredient.Category is not null
                ? new CategoryDto(ingredient.Category.Id, ingredient.Category.Name)
                : null,
            ingredient.UnitSize,
            ingredient.PriceSpikeThresholdPct,
            latestPrice?.Price,
            latestPrice?.CommittedAt
        );
    }
}
```

---

# Slice 3: CreateIngredient — Command

**Folder:** `Application/Features/Ingredients/Commands/CreateIngredient/`

Creates a new ingredient for this user. Optionally inserts the first price history record.

```
mkdir -p src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient
```

### Request

**File:** `src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient/CreateIngredientCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Ingredients.Commands.CreateIngredient;

public record CreateIngredientCommand(
    Guid UserId,
    string Name,
    Guid UnitId,
    Guid? CategoryId,
    decimal UnitSize,
    decimal PriceSpikeThresholdPct,
    decimal? InitialPrice = null,
    DateOnly? InitialEffectiveDate = null
) : IRequest<ErrorOr<CreateIngredientResponse>>;
```

### Response

**File:** `src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient/CreateIngredientResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Commands.CreateIngredient;

public record CreateIngredientResponse(Guid Id, string Name);
```

### Validator

**File:** `src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient/CreateIngredientCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Ingredients.Commands.CreateIngredient;

public class CreateIngredientCommandValidator : AbstractValidator<CreateIngredientCommand>
{
    public CreateIngredientCommandValidator()
    {
        RuleFor(x => x.UserId).NotEmpty();

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Ingredient name is required.")
            .MaximumLength(255).WithMessage("Name cannot exceed 255 characters.");

        RuleFor(x => x.UnitId).NotEmpty().WithMessage("UnitId is required.");

        RuleFor(x => x.UnitSize)
            .GreaterThan(0).WithMessage("Unit size must be greater than zero.");

        RuleFor(x => x.PriceSpikeThresholdPct)
            .InclusiveBetween(0, 100)
            .WithMessage("Spike threshold must be between 0 and 100.");
        // 0 is valid: means "no spike alerting". Alert logic is wired in L7.

        RuleFor(x => x.InitialPrice)
            .GreaterThan(0).When(x => x.InitialPrice.HasValue)
            .WithMessage("Initial price must be greater than zero.");

        RuleFor(x => x.InitialEffectiveDate)
            .LessThanOrEqualTo(DateOnly.FromDateTime(DateTime.UtcNow))
            .When(x => x.InitialEffectiveDate.HasValue)
            .WithMessage("Effective date cannot be in the future.");
    }
}
```

### Handler

**File:** `src/Nastart.Application/Features/Ingredients/Commands/CreateIngredient/CreateIngredientHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Application.Features.Ingredients.Commands.CreateIngredient;

public class CreateIngredientHandler(IAppDbContext db)
    : IRequestHandler<CreateIngredientCommand, ErrorOr<CreateIngredientResponse>>
{
    public async Task<ErrorOr<CreateIngredientResponse>> Handle(
        CreateIngredientCommand command, CancellationToken ct)
    {
        // Verify Unit exists
        var unitExists = await db.Units.AnyAsync(u => u.Id == command.UnitId, ct);
        if (!unitExists)
            return Error.NotFound("Unit.NotFound", "The specified unit does not exist.");

        // Verify Category exists (if provided)
        if (command.CategoryId.HasValue)
        {
            var categoryExists = await db.Categories
                .AnyAsync(c => c.Id == command.CategoryId.Value, ct);
            if (!categoryExists)
                return Error.NotFound("Category.NotFound", "The specified category does not exist.");
        }

        // Check ingredient name is unique for this user (case-insensitive, PostgreSQL ILIKE)
        var nameExists = await db.Ingredients
            .AnyAsync(i => i.UserId == command.UserId &&
                           EF.Functions.ILike(i.Name, command.Name), ct);
        if (nameExists)
            return Error.Conflict("Ingredient.Duplicate",
                "An ingredient with this name already exists.");

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
        // C-3: Current price always comes from latest PriceHistory record
        // C-4: CommittedAt is NOT set here — the DB inserts NOW() via HasDefaultValueSql
        // C-5: IngredientPriceHistory is append-only — never update or delete
        // C-13: Source = PriceSource.Manual; EF Core stores this as the string "Manual"
        //        via .HasConversion<string>() in IngredientPriceHistoryConfiguration (see L2)
        if (command.InitialPrice.HasValue)
        {
            var priceRecord = new IngredientPriceHistory
            {
                Id = Guid.NewGuid(),
                IngredientId = ingredient.Id,
                Price = command.InitialPrice.Value,
                // C-3 + C-4: Snapshot the current unit size at the time of price entry.
                // If unit size is later changed, this record still reflects the original size.
                // The cost formula (C-2) uses each record's own UnitSize for accurate historical costing.
                UnitSize = command.UnitSize,
                Source = PriceSource.Manual,
                EffectiveDate = command.InitialEffectiveDate ?? DateOnly.FromDateTime(DateTime.UtcNow),
                // CommittedAt: NOT set — DB default handles this via HasDefaultValueSql("NOW()")
                InvoiceLineItemId = null
            };

            db.IngredientPriceHistories.Add(priceRecord);
        }

        await db.SaveChangesAsync(ct);

        return new CreateIngredientResponse(ingredient.Id, ingredient.Name);
    }
}
```

---

# Slice 4: UpdateIngredient — Command

**Folder:** `Application/Features/Ingredients/Commands/UpdateIngredient/`

Updates ingredient metadata (name, category, unit, threshold). Does **not** update price — price changes go through `AddIngredientPrice`.

```
mkdir -p src/Nastart.Application/Features/Ingredients/Commands/UpdateIngredient
```

### Request

**File:** `src/Nastart.Application/Features/Ingredients/Commands/UpdateIngredient/UpdateIngredientCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Ingredients.Commands.UpdateIngredient;

public record UpdateIngredientCommand(
    Guid IngredientId,
    Guid UserId,
    string Name,
    Guid UnitId,
    Guid? CategoryId,
    decimal UnitSize,
    decimal PriceSpikeThresholdPct
) : IRequest<ErrorOr<UpdateIngredientResponse>>;
```

### Response

**File:** `src/Nastart.Application/Features/Ingredients/Commands/UpdateIngredient/UpdateIngredientResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Commands.UpdateIngredient;

public record UpdateIngredientResponse(Guid Id, string Name);
```

### Validator

**File:** `src/Nastart.Application/Features/Ingredients/Commands/UpdateIngredient/UpdateIngredientCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Ingredients.Commands.UpdateIngredient;

public class UpdateIngredientCommandValidator : AbstractValidator<UpdateIngredientCommand>
{
    public UpdateIngredientCommandValidator()
    {
        RuleFor(x => x.IngredientId).NotEmpty();
        RuleFor(x => x.UserId).NotEmpty();

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Ingredient name is required.")
            .MaximumLength(255).WithMessage("Name cannot exceed 255 characters.");

        RuleFor(x => x.UnitId).NotEmpty().WithMessage("UnitId is required.");

        RuleFor(x => x.UnitSize)
            .GreaterThan(0).WithMessage("Unit size must be greater than zero.");

        RuleFor(x => x.PriceSpikeThresholdPct)
            .InclusiveBetween(0, 100)
            .WithMessage("Spike threshold must be between 0 and 100.");
    }
}
```

### Handler

**File:** `src/Nastart.Application/Features/Ingredients/Commands/UpdateIngredient/UpdateIngredientHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Ingredients.Commands.UpdateIngredient;

public class UpdateIngredientHandler(IAppDbContext db)
    : IRequestHandler<UpdateIngredientCommand, ErrorOr<UpdateIngredientResponse>>
{
    public async Task<ErrorOr<UpdateIngredientResponse>> Handle(
        UpdateIngredientCommand command, CancellationToken ct)
    {
        var ingredient = await db.Ingredients
            .FirstOrDefaultAsync(i => i.Id == command.IngredientId, ct);

        if (ingredient is null || ingredient.UserId != command.UserId)
            return Error.Forbidden("Ingredient.AccessDenied", "You do not have access to this ingredient.");

        // Check for duplicate name when name is being changed (case-insensitive)
        if (!EF.Functions.ILike(ingredient.Name, command.Name))
        {
            var duplicateName = await db.Ingredients
                .AnyAsync(i =>
                    i.UserId == command.UserId &&
                    i.Id != command.IngredientId &&
                    EF.Functions.ILike(i.Name, command.Name), ct);

            if (duplicateName)
                return Error.Conflict("Ingredient.Duplicate",
                    "An ingredient with this name already exists.");
        }

        // Verify Unit exists
        var unitExists = await db.Units.AnyAsync(u => u.Id == command.UnitId, ct);
        if (!unitExists)
            return Error.NotFound("Unit.NotFound", "The specified unit does not exist.");

        // Verify Category exists (if provided)
        if (command.CategoryId.HasValue)
        {
            var categoryExists = await db.Categories
                .AnyAsync(c => c.Id == command.CategoryId.Value, ct);
            if (!categoryExists)
                return Error.NotFound("Category.NotFound", "The specified category does not exist.");
        }

        ingredient.Name = command.Name;
        ingredient.UnitId = command.UnitId;
        ingredient.CategoryId = command.CategoryId;
        ingredient.UnitSize = command.UnitSize;
        ingredient.PriceSpikeThresholdPct = command.PriceSpikeThresholdPct;

        await db.SaveChangesAsync(ct);

        return new UpdateIngredientResponse(ingredient.Id, ingredient.Name);
    }
}
```

---

# Slice 5: DeleteIngredient — Command

**Folder:** `Application/Features/Ingredients/Commands/DeleteIngredient/`

Deletes an ingredient. In L7, `RecipeItem.IngredientId` gains `OnDelete(Restrict)` — at that point any deletion of an ingredient used in a recipe will fail at the DB level.

```
mkdir -p src/Nastart.Application/Features/Ingredients/Commands/DeleteIngredient
```

### Request

**File:** `src/Nastart.Application/Features/Ingredients/Commands/DeleteIngredient/DeleteIngredientCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Ingredients.Commands.DeleteIngredient;

public record DeleteIngredientCommand(Guid IngredientId, Guid UserId)
    : IRequest<ErrorOr<DeleteIngredientResponse>>;
```

### Response

**File:** `src/Nastart.Application/Features/Ingredients/Commands/DeleteIngredient/DeleteIngredientResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Commands.DeleteIngredient;

public record DeleteIngredientResponse(Guid Id);
```

### Validator

**File:** `src/Nastart.Application/Features/Ingredients/Commands/DeleteIngredient/DeleteIngredientCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Ingredients.Commands.DeleteIngredient;

public class DeleteIngredientCommandValidator : AbstractValidator<DeleteIngredientCommand>
{
    public DeleteIngredientCommandValidator()
    {
        RuleFor(x => x.IngredientId).NotEmpty();
        RuleFor(x => x.UserId).NotEmpty();
    }
}
```

### Handler

**File:** `src/Nastart.Application/Features/Ingredients/Commands/DeleteIngredient/DeleteIngredientHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Ingredients.Commands.DeleteIngredient;

public class DeleteIngredientHandler(IAppDbContext db)
    : IRequestHandler<DeleteIngredientCommand, ErrorOr<DeleteIngredientResponse>>
{
    public async Task<ErrorOr<DeleteIngredientResponse>> Handle(
        DeleteIngredientCommand command, CancellationToken ct)
    {
        var ingredient = await db.Ingredients
            .FirstOrDefaultAsync(i => i.Id == command.IngredientId, ct);

        if (ingredient is null || ingredient.UserId != command.UserId)
            return Error.Forbidden("Ingredient.AccessDenied", "You do not have access to this ingredient.");

        // L7 note: RecipeItem.IngredientId will use OnDelete(Restrict).
        // After L7, deleting an ingredient used in a recipe will throw a DbUpdateException.
        // Upgrade this handler in L7 to query RecipeItems first and return
        // 409 Conflict with a descriptive message rather than letting the DB throw.
        db.Ingredients.Remove(ingredient);
        await db.SaveChangesAsync(ct);

        return new DeleteIngredientResponse(ingredient.Id);
    }
}
```

---

# Slice 6: AddIngredientPrice — Command

**Folder:** `Application/Features/Ingredients/Commands/AddIngredientPrice/`

Appends a new price history record. This is the **only** way to update a price — records are permanent once committed.

```
mkdir -p src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice
```

### Request

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

public record AddIngredientPriceCommand(
    Guid IngredientId,
    Guid UserId,
    decimal Price,
    DateOnly? EffectiveDate = null
) : IRequest<ErrorOr<AddIngredientPriceResponse>>;
```

### Response

> ⚠️ **Superseded by L7 Section 12.** After applying L7, this response DTO is replaced entirely.
> The L7 shape is: `AddIngredientPriceResponse(Guid PriceHistoryId, decimal Price, int AffectedRecipes, int FailedRecipes)`.
> The fields below (`IngredientId`, `EffectiveDate`, `CommittedAt`) are removed; cascade counts are added.

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

public record AddIngredientPriceResponse(
    Guid IngredientId,
    decimal Price,
    DateOnly EffectiveDate,
    DateTime CommittedAt  // DB-generated — reflects the actual INSERT timestamp
);
```

### Validator

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

public class AddIngredientPriceCommandValidator : AbstractValidator<AddIngredientPriceCommand>
{
    public AddIngredientPriceCommandValidator()
    {
        RuleFor(x => x.IngredientId).NotEmpty();
        RuleFor(x => x.UserId).NotEmpty();
        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than zero.");

        RuleFor(x => x.EffectiveDate)
            .LessThanOrEqualTo(DateOnly.FromDateTime(DateTime.UtcNow))
            .When(x => x.EffectiveDate.HasValue)
            .WithMessage("Effective date cannot be in the future.");
    }
}
```

### Handler

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

// ⚠️ This handler is replaced in full by L7 Section 12.
// The TODO comments at the bottom (spike check + cascade) are resolved there.
public class AddIngredientPriceHandler(IAppDbContext db)
    : IRequestHandler<AddIngredientPriceCommand, ErrorOr<AddIngredientPriceResponse>>
{
    public async Task<ErrorOr<AddIngredientPriceResponse>> Handle(
        AddIngredientPriceCommand command, CancellationToken ct)
    {
        // Verify ownership and snapshot current unit size
        var ingredient = await db.Ingredients
            .FirstOrDefaultAsync(i => i.Id == command.IngredientId, ct);

        if (ingredient is null || ingredient.UserId != command.UserId)
            return Error.Forbidden("Ingredient.AccessDenied", "You do not have access to this ingredient.");

        var effectiveDate = command.EffectiveDate ?? DateOnly.FromDateTime(DateTime.UtcNow);

        // C-5: Append-only — never update existing records
        // C-4: CommittedAt is NOT set here; the DB sets it via HasDefaultValueSql("NOW()")
        // C-13: PriceSource.Manual stored as the string "Manual" via .HasConversion<string>() (L2)
        // C-3 + C-4: Snapshot UnitSize at the time of this price entry.
        //   If the ingredient's unit size is later changed, this record retains the original size.
        //   The cost formula (C-2) uses each record's own UnitSize for accurate historical costing.
        var priceRecord = new IngredientPriceHistory
        {
            Id = Guid.NewGuid(),
            IngredientId = command.IngredientId,
            Price = command.Price,
            UnitSize = ingredient.UnitSize,
            Source = PriceSource.Manual,
            EffectiveDate = effectiveDate,
            InvoiceLineItemId = null
            // CommittedAt: NOT set — DB default inserts NOW()
        };

        db.IngredientPriceHistories.Add(priceRecord);
        await db.SaveChangesAsync(ct);

        // EF Core refreshes ValueGeneratedOnAdd properties after SaveChangesAsync.
        // priceRecord.CommittedAt now contains the actual DB-inserted timestamp from HasDefaultValueSql.

        // TODO (L7): After price is committed, wire:
        // 1. ICostCascadeService.RecalculateForIngredient(command.IngredientId)
        //    → recalculates CostPerPortion on all recipes using this ingredient
        // 2. Check price spike against ingredient.PriceSpikeThresholdPct
        //    → send alert to the single user via IAlertDispatcher

        return new AddIngredientPriceResponse(
            command.IngredientId,
            command.Price,
            effectiveDate,
            priceRecord.CommittedAt
        );
    }
}
```

---

# Slice 7: GetIngredientPriceHistory — Query

**Folder:** `Application/Features/Ingredients/Queries/GetIngredientPriceHistory/`

Returns the complete price history for an ingredient, ordered from newest to oldest.

```
mkdir -p src/Nastart.Application/Features/Ingredients/Queries/GetIngredientPriceHistory
```

### Request

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredientPriceHistory/GetIngredientPriceHistoryQuery.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Ingredients.Queries.GetIngredientPriceHistory;

public record GetIngredientPriceHistoryQuery(Guid IngredientId, Guid UserId)
    : IRequest<ErrorOr<GetIngredientPriceHistoryResponse>>;
```

### Response

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredientPriceHistory/GetIngredientPriceHistoryResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Queries.GetIngredientPriceHistory;

public record PriceHistoryEntry(
    Guid Id,
    decimal Price,
    string Source,      // "Manual" or "InvoiceScan" (Phase 3)
    DateTime CommittedAt,
    DateOnly EffectiveDate
);

public record GetIngredientPriceHistoryResponse(
    Guid IngredientId,
    string IngredientName,
    List<PriceHistoryEntry> PriceHistory  // Empty list if no prices recorded yet
);
```

### Handler

**File:** `src/Nastart.Application/Features/Ingredients/Queries/GetIngredientPriceHistory/GetIngredientPriceHistoryHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Ingredients.Queries.GetIngredientPriceHistory;

public class GetIngredientPriceHistoryHandler(IAppDbContext db)
    : IRequestHandler<GetIngredientPriceHistoryQuery, ErrorOr<GetIngredientPriceHistoryResponse>>
{
    public async Task<ErrorOr<GetIngredientPriceHistoryResponse>> Handle(
        GetIngredientPriceHistoryQuery query, CancellationToken ct)
    {
        // Verify ownership
        var ingredient = await db.Ingredients
            .AsNoTracking()
            .FirstOrDefaultAsync(i => i.Id == query.IngredientId, ct);

        if (ingredient is null || ingredient.UserId != query.UserId)
            return Error.Forbidden("Ingredient.AccessDenied", "You do not have access to this ingredient.");

        // C-3 (full history): Return all records ordered newest committed_at first
        // Returns empty list if no prices have been recorded yet — not an error
        var priceHistory = await db.IngredientPriceHistories
            .AsNoTracking()
            .Where(p => p.IngredientId == query.IngredientId)
            .OrderByDescending(p => p.CommittedAt)
            .Select(p => new PriceHistoryEntry(
                p.Id,
                p.Price,
                p.Source.ToString(),  // "Manual" or "InvoiceScan" — C-13
                p.CommittedAt,
                p.EffectiveDate
            ))
            .ToListAsync(ct);

        return new GetIngredientPriceHistoryResponse(
            ingredient.Id,
            ingredient.Name,
            priceHistory
        );
    }
}
```

---

# The Endpoint File

One unified file for all ingredient routes. This follows the `MapGroup` pattern from L5.

**File:** `src/Nastart.API/Endpoints/IngredientEndpoints.cs`

```csharp
using MediatR;
using Nastart.API.Extensions;
using Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;
using Nastart.Application.Features.Ingredients.Commands.CreateIngredient;
using Nastart.Application.Features.Ingredients.Commands.DeleteIngredient;
using Nastart.Application.Features.Ingredients.Commands.UpdateIngredient;
using Nastart.Application.Features.Ingredients.Queries.GetIngredientById;
using Nastart.Application.Features.Ingredients.Queries.GetIngredientPriceHistory;
using Nastart.Application.Features.Ingredients.Queries.GetIngredients;

namespace Nastart.API.Endpoints;

public static class IngredientEndpoints
{
    public static void MapIngredientEndpoints(this WebApplication app)
    {
        // All ingredient routes under /api/ingredients.
        // RequireAuthorization() at group level covers all child routes — no need to repeat per endpoint.
        var group = app.MapGroup("/api/ingredients")
            .WithTags("Ingredients")
            .RequireAuthorization();

        // GET /api/ingredients
        group.MapGet("/", async (HttpContext httpContext, ISender sender, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            var result = await sender.Send(new GetIngredientsQuery(UserId: userId), ct);
            return Results.Ok(result);
        })
        .WithName("GetIngredients")
        .WithOpenApi();

        // GET /api/ingredients/{ingredientId}
        group.MapGet("/{ingredientId:guid}", async (
            Guid ingredientId, HttpContext httpContext, ISender sender, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            var result = await sender.Send(new GetIngredientByIdQuery(ingredientId, UserId: userId), ct);
            return result.ToApiResult();
        })
        .WithName("GetIngredientById")
        .WithOpenApi();

        // POST /api/ingredients
        group.MapPost("/", async (
            CreateIngredientCommand command, HttpContext httpContext, ISender sender, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            // Override UserId from JWT — client-provided UserId in the body is always ignored
            var cmd = command with { UserId = userId };
            var result = await sender.Send(cmd, ct);
            return result.ToCreatedResult($"/api/ingredients/{result.Value?.Id}");
        })
        .WithName("CreateIngredient")
        .WithOpenApi();

        // PUT /api/ingredients/{ingredientId}
        group.MapPut("/{ingredientId:guid}", async (
            Guid ingredientId, UpdateIngredientCommand command,
            HttpContext httpContext, ISender sender, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            var cmd = command with { UserId = userId, IngredientId = ingredientId };
            var result = await sender.Send(cmd, ct);
            return result.ToApiResult();
        })
        .WithName("UpdateIngredient")
        .WithOpenApi();

        // DELETE /api/ingredients/{ingredientId}
        group.MapDelete("/{ingredientId:guid}", async (
            Guid ingredientId, HttpContext httpContext, ISender sender, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            var result = await sender.Send(new DeleteIngredientCommand(ingredientId, UserId: userId), ct);
            return result.ToApiResult();
        })
        .WithName("DeleteIngredient")
        .WithOpenApi();

        // Price sub-routes nested inside the ingredients group to inherit RequireAuthorization()
        var pricesGroup = group.MapGroup("/{ingredientId:guid}/prices")
            .WithTags("Ingredient Prices");

        // POST /api/ingredients/{ingredientId}/prices
        pricesGroup.MapPost("/", async (
            Guid ingredientId, AddIngredientPriceCommand command,
            HttpContext httpContext, ISender sender, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            var cmd = command with { UserId = userId, IngredientId = ingredientId };
            var result = await sender.Send(cmd, ct);
            return result.ToCreatedResult($"/api/ingredients/{ingredientId}/prices");
        })
        .WithName("AddIngredientPrice")
        .WithOpenApi();

        // GET /api/ingredients/{ingredientId}/prices
        pricesGroup.MapGet("/", async (
            Guid ingredientId, HttpContext httpContext, ISender sender, CancellationToken ct) =>
        {
            var userId = httpContext.User.GetUserId();
            var result = await sender.Send(new GetIngredientPriceHistoryQuery(ingredientId, UserId: userId), ct);
            return result.ToApiResult();
        })
        .WithName("GetIngredientPriceHistory")
        .WithOpenApi();
    }
}
```

### Wire in Program.cs

Add this line to `src/Nastart.API/Program.cs` after middleware configuration:

```csharp
// --- Endpoints ---
app.MapIngredientEndpoints();  // ← L6

app.Run();
```

---

## Authorization Policies Reference

> ⚠️ **v2-only — do not implement in v1.** Role-based authorization policies (`OwnerOnly`, `ChefOrAbove`, etc.) are introduced when the product supports multi-user teams in v2. In v1, all authenticated requests belong to the single owner — use `.RequireAuthorization()` with no policy name.

| Route | v2 policy name | v1 |
|---|---|---|
| GET `/api/ingredients` | `"OwnerOrProcurement"` | `RequireAuthorization()` |
| POST `/api/ingredients` | `"OwnerOnly"` | `RequireAuthorization()` |
| PUT `/api/ingredients/{id}` | `"OwnerOnly"` | `RequireAuthorization()` |
| DELETE `/api/ingredients/{id}` | `"OwnerOnly"` | `RequireAuthorization()` |
| POST `.../prices` | `"OwnerOrProcurement"` | `RequireAuthorization()` |
| GET `.../prices` | `"OwnerOrProcurement"` | `RequireAuthorization()` |

---

# How to Test

**Before running:** ensure `app.MapIngredientEndpoints()` is in `Program.cs`, then:

```bash
docker-compose up -d   # Start PostgreSQL
dotnet run --project src/Nastart.API
```

## Test Scenario: Complete Ingredient CRUD Workflow

```bash
# --- SETUP: Get a JWT token ---

curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name": "Jane Chef", "email": "jane@test.com", "password": "password123"}'
# Expected: 201 { "userId": "...", "message": "Check your email..." }

USERID="<userId-from-register>"
curl -X POST http://localhost:5000/api/auth/verify \
  -H "Content-Type: application/json" \
  -d "{\"userId\": \"$USERID\"}"
# Expected: 200 { "message": "Email verified. You can now log in." }

curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "jane@test.com", "password": "password123"}'
# Expected: 200 { "token": "eyJhbG..." }

TOKEN="<token-from-login>"

# --- Pre-seed: Units and Categories must exist (from L2 seed data) ---
UNITID="87654321-abcd-1234-5678-abcdef000001"     # kilogram / kg
CATEGORYID="76543210-bcde-2345-6789-bcdef0000002"  # Proteins

# --- TEST 1: Create an ingredient with initial price ---

curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"name\": \"Chicken Breast\",
    \"unitId\": \"$UNITID\",
    \"categoryId\": \"$CATEGORYID\",
    \"unitSize\": 1.0,
    \"priceSpikeThresholdPct\": 10,
    \"initialPrice\": 5.50,
    \"initialEffectiveDate\": \"2024-01-15\"
  }"
# Expected: 201 { "id": "<ingredientId>", "name": "Chicken Breast" }

INGREDIENTID="<ingredientId>"

# --- TEST 2: Get all ingredients ---

curl -X GET "http://localhost:5000/api/ingredients" \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 {
#   "ingredients": [
#     { "id": "...", "name": "Chicken Breast", "unitAbbreviation": "kg",
#       "categoryName": "Proteins", "unitSize": 1.0, "currentPrice": 5.50 }
#   ]
# }

# --- TEST 3: Get single ingredient ---

curl -X GET "http://localhost:5000/api/ingredients/$INGREDIENTID" \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 {
#   "id": "...", "name": "Chicken Breast",
#   "unit": { "id": "...", "name": "kilogram", "abbreviation": "kg" },
#   "category": { "id": "...", "name": "Proteins" },
#   "unitSize": 1.0, "priceSpikeThresholdPct": 10,
#   "currentPrice": 5.50, "lastPriceDate": "2024-01-..."
# }

# --- TEST 4: Update ingredient metadata ---

curl -X PUT "http://localhost:5000/api/ingredients/$INGREDIENTID" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{
    \"name\": \"Organic Chicken Breast\",
    \"unitId\": \"$UNITID\",
    \"categoryId\": \"$CATEGORYID\",
    \"unitSize\": 1.0,
    \"priceSpikeThresholdPct\": 15
  }"
# Expected: 200 { "id": "...", "name": "Organic Chicken Breast" }

# --- TEST 5: Add a new price with effectiveDate ---

curl -X POST "http://localhost:5000/api/ingredients/$INGREDIENTID/prices" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"price\": 6.75, \"effectiveDate\": \"2024-01-20\"}"
# Expected: 201 {
#   "ingredientId": "...", "price": 6.75,
#   "effectiveDate": "2024-01-20", "committedAt": "2024-..."
# }

# --- TEST 6: Add a price without effectiveDate (defaults to today) ---

curl -X POST "http://localhost:5000/api/ingredients/$INGREDIENTID/prices" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"price\": 7.00}"
# Expected: 201 { "effectiveDate": "<today's date>", "committedAt": "..." }

# --- TEST 7: Get price history (ordered newest first) ---

curl -X GET "http://localhost:5000/api/ingredients/$INGREDIENTID/prices" \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 {
#   "ingredientId": "...", "ingredientName": "Organic Chicken Breast",
#   "priceHistory": [
#     { "price": 7.00, "source": "Manual", "effectiveDate": "<today>" },
#     { "price": 6.75, "source": "Manual", "effectiveDate": "2024-01-20" },
#     { "price": 5.50, "source": "Manual", "effectiveDate": "2024-01-15" }
#   ]
# }

# --- TEST 8: Price history on ingredient with no prices ---

curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"name\": \"Flour\", \"unitId\": \"$UNITID\", \"unitSize\": 1.0, \"priceSpikeThresholdPct\": 5}"

FLOURID="<flour-ingredient-id>"

curl -X GET "http://localhost:5000/api/ingredients/$FLOURID/prices" \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 { "ingredientId": "...", "ingredientName": "Flour", "priceHistory": [] }
# Note: empty list is NOT an error

# --- TEST 9: Delete ingredient ---

curl -X DELETE "http://localhost:5000/api/ingredients/$INGREDIENTID" \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 { "id": "..." }

# --- ERROR CASES ---

# Duplicate name — case-insensitive: "chicken breast" matches "Chicken Breast"
curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"name\": \"chicken breast\", \"unitId\": \"$UNITID\", \"unitSize\": 1.0, \"priceSpikeThresholdPct\": 10}"
# Expected: 409 Conflict { "error": "An ingredient with this name already exists." }

# Invalid Unit
curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"name\": \"Test\", \"unitId\": \"00000000-0000-0000-0000-000000000000\", \"unitSize\": 1.0, \"priceSpikeThresholdPct\": 10}"
# Expected: 404 NotFound { "error": "The specified unit does not exist." }

# Invalid Category
curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"name\": \"Test2\", \"unitId\": \"$UNITID\", \"categoryId\": \"00000000-0000-0000-0000-000000000000\", \"unitSize\": 1.0, \"priceSpikeThresholdPct\": 10}"
# Expected: 404 NotFound { "error": "The specified category does not exist." }

# Negative price (validation error)
curl -X POST http://localhost:5000/api/ingredients \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d "{\"name\": \"Bad\", \"unitId\": \"$UNITID\", \"unitSize\": 1.0, \"priceSpikeThresholdPct\": 10, \"initialPrice\": -5.00}"
# Expected: 400 BadRequest { "errors": [{ "code": "InitialPrice", "description": "..." }] }

# Cross-user access — register a second user and try to access the first user's ingredient
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name": "Bob", "email": "bob@test.com", "password": "password123"}'
# ... verify bob, login as bob to get TOKEN2 ...

curl -X GET "http://localhost:5000/api/ingredients/$FLOURID" \
  -H "Authorization: Bearer $TOKEN2"
# Expected: 403 Forbidden

curl -X DELETE "http://localhost:5000/api/ingredients/$FLOURID" \
  -H "Authorization: Bearer $TOKEN2"
# Expected: 403 Forbidden

# No JWT
curl -X GET "http://localhost:5000/api/ingredients"
# Expected: 401 Unauthorized
```

---

# What You've Built (L6 Summary)

| Component | Status |
|---|---|
| GetIngredients query | ✅ |
| GetIngredientById query | ✅ |
| CreateIngredient command | ✅ |
| UpdateIngredient command | ✅ |
| DeleteIngredient command | ✅ |
| AddIngredientPrice command | ✅ |
| GetIngredientPriceHistory query | ✅ |
| DeleteIngredientCommandValidator | ✅ |
| Category existence validation on Create/Update | ✅ |
| Case-insensitive name uniqueness check (`EF.Functions.ILike`) | ✅ |
| Endpoint authorization (`RequireAuthorization()` at group level) | ✅ |
| Nested price subgroup (inherits auth from parent) | ✅ |
| Append-only price history (C-3, C-4, C-5) | ✅ |
| User scoping on all queries/commands | ✅ |

### Key Canonical Decisions Wired

- **C-3** — Current price always derived from `ORDER BY committed_at DESC LIMIT 1`
- **C-4** — `CommittedAt` is DB-generated via `HasDefaultValueSql`; `EffectiveDate` is user-supplied
- **C-5** — `IngredientPriceHistory` is append-only; records are never updated or deleted
- **C-13** — `PriceSource.Manual` stored as the string `"Manual"` via `.HasConversion<string>()` (L2)
- **C-8 (v2-only)** — Role-based endpoint policies are deferred to v2; all endpoints use `.RequireAuthorization()` with no policy name

### Deferred to L7

- **Cascade recalculation** — `ICostCascadeService.RecalculateForIngredient(ingredientId)` called after price commit
- **Price spike alerts** — comparison against `PriceSpikeThresholdPct`, alert sent to user
- **Recipe item constraint** — `RecipeItem.IngredientId` gets `OnDelete(Restrict)`, blocking deletion of ingredients still used in recipes

---

**← Previous:** L5 — JWT Authentication & Authorization  
**→ Next:** L7 — Cost Cascade Service & Price Spike Alerts

