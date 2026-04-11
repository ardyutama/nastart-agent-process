# Lesson 8 — Recipe Builder & Costing Engine

> **What you'll learn:**
> - Why `CostPerPortion` is server-authoritative and never computed by the client
> - The cascade architecture: triggering automatic recalculation when ingredient prices change
> - A unified response DTO: single user sees all fields including derived sell price
> - 7 complete vertical slices for recipe creation, modification, and versioning
> - How to implement the canonical cost formula (C-2) and the sell price formula
>
> **Out of scope for this lesson:**
> - **Async/deferred cascade** — Phase 4+. Currently cascade runs synchronously. Future: queue-based async with retries.
> - **Telegram threshold alerts** — deferred to L15. CostCascadeService dispatches alerts; how the bot receives them is Phase 4.

> ⚠️ **v1 Solopreneur Amendments (April 9, 2026)**
>
> **Routes — no `{outletId}`:**
> - `GET /api/outlets/{outletId}/recipes` → `GET /api/recipes`
> - `POST /api/outlets/{outletId}/recipes` → `POST /api/recipes`
> - All recipe endpoints follow the same pattern
>
> **Commands and queries — replace `Guid OutletId` with `Guid UserId`:**
> - `CreateRecipeCommand(Guid OutletId, ...)` → `CreateRecipeCommand(Guid UserId, ...)`
> - `GetRecipesByOutletQuery(Guid OutletId, string Role)` → `GetRecipesQuery(Guid UserId)` (remove role parameter entirely)
> - Same for all other recipe commands
>
> **Response DTOs — delete both enterprise DTOs, use one unified DTO:**
> ```csharp
> // DELETE RecipeOwnerResponse — no longer needed
> // DELETE RecipeStandardResponse — no longer needed
>
> // v1: single DTO, single user sees everything
> public record RecipeResponse(
>     Guid Id,
>     string Name,
>     int PortionCount,
>     decimal CostPerPortion,
>     decimal PackagingCost,       // new v1 field
>     decimal TargetMargin,        // new v1 field (e.g. 0.40 = 40%)
>     decimal? DerivedSellPrice,   // computed: (CostPerPortion + PackagingCost) / (1 - TargetMargin); null if TargetMargin == 1
>     decimal? FoodCostPct,        // computed: (CostPerPortion / DerivedSellPrice) * 100; null if no sell price
>     int VersionNumber,
>     string VersionLabel,
>     Guid VersionGroupId
> );
> ```
>
> **Sell price computation in handler (derived at read time, never stored):**
> ```csharp
> decimal? derivedSellPrice = recipe.TargetMargin < 1m
>     ? (recipe.CostPerPortion + recipe.PackagingCost) / (1m - recipe.TargetMargin)
>     : null;
> decimal? foodCostPct = derivedSellPrice.HasValue && derivedSellPrice > 0
>     ? (recipe.CostPerPortion / derivedSellPrice.Value) * 100m
>     : null;
> ```
>
> **Authorization — replace all role-based policies:**
> - `"OwnerOrChef"`, `"CanViewRecipes"` → `RequireAuthorization()`

---

## 1. Why CostPerPortion is Server-Authoritative

Imagine a Vue.js client receives a Recipe with `CostPerPortion = $5.00`. The user sees a form with editable fields: Name, SellingPrice, RecipeItems. 

**Question: What if the client computes `CostPerPortion` and sends it back in the PUT request?**

**Answer: Never accept it.**

- **Ingredient prices change constantly** — from invoices, market updates, cascade recalculations
- **The client has stale data** — it fetched the recipe 10 seconds ago, prices changed 5 seconds ago
- **Multiple versions are live simultaneously** — all sharing the same ingredient prices; if one client sends a computed cost, it conflicts with others
- **Audit trail breaks** — who computed the cost? Was it correct? A database-computed, timestamped value is auditable; a client-sent value is not

**The pattern:**

```
Client sends:
  {
    "name": "Pasta Carbonara",
    "recipeItems": [
      { "ingredientId": "...", "quantity": 200, "yieldPercentage": 0.95 }
    ]
  }

Server:
  1. Persists recipe and items
  2. Calls ICostCascadeService.RecalculateForIngredient() for each ingredient
  3. Cascade fetches CURRENT prices from IngredientPriceHistory (C-3)
  4. Applies formula (C-2) → NEW cost_per_portion
  5. Updates Recipe.CostPerPortion in database
  6. Returns response with authoritative cost

Client receives:
  {
    "id": "...",
    "costPerPortion": 3.47,   // Set by server ONLY
    "packagingCost": 0.45,
    "targetMargin": 0.35,
    "derivedSellPrice": 6.03, // Computed at read time
    "foodCostPct": 57.5,      // Computed at read time
    ...
  }
```

---

## 2. Unified Response DTO — One User, One View

> **v1 note:** The role-split DTO pattern (RecipeOwnerResponse vs RecipeStandardResponse) was designed for multi-role systems. v1 is single-user: there are no roles, and the single user sees all fields. Use one unified DTO.

**Bad approach — nullable fields:**

```csharp
public record RecipeResponse(
    Guid Id,
    string Name,
    decimal CostPerPortion,
    decimal? FoodCostPct,   // null for non-Owner — client confusion: null or absent?
    decimal? GrossMargin    // null for non-Owner
);
```

Why this is bad (same reasoning applies in both single-user and multi-user designs):
- JSON includes `"foodCostPct": null` — the client receives null and wonders if the field exists
- Inconsistent UI: sometimes the field renders, sometimes it's null. Not a clean API contract
- Harder to debug: the developer sees two code paths for no benefit in a single-user system

**v1 approach — one unified DTO, all fields always present:**

```csharp
// v1: single authenticated user sees all fields
public record RecipeResponse(
    Guid Id,
    string Name,
    int PortionCount,
    decimal CostPerPortion,
    decimal PackagingCost,       // flat packaging add-on per portion
    decimal TargetMargin,        // user-set margin (e.g. 0.40 = 40%)
    decimal? DerivedSellPrice,   // (CostPerPortion + PackagingCost) / (1 - TargetMargin)
    decimal? FoodCostPct,        // (CostPerPortion / DerivedSellPrice) * 100
    int VersionNumber,
    string VersionLabel,
    Guid VersionGroupId
);
```

**In the handler — compute derived fields at read time (never store them):**

```csharp
decimal? derivedSellPrice = recipe.TargetMargin < 1m
    ? (recipe.CostPerPortion + recipe.PackagingCost) / (1m - recipe.TargetMargin)
    : null;

decimal? foodCostPct = derivedSellPrice.HasValue && derivedSellPrice.Value > 0
    ? (recipe.CostPerPortion / derivedSellPrice.Value) * 100m
    : null;

return new RecipeResponse(
    recipe.Id,
    recipe.Name,
    recipe.PortionCount,
    recipe.CostPerPortion,
    recipe.PackagingCost,
    recipe.TargetMargin,
    derivedSellPrice,
    foodCostPct,
    recipe.VersionNumber,
    recipe.VersionLabel,
    recipe.VersionGroupId
);
```

**JSON response — single user always sees:**
```json
{
  "id": "...",
  "name": "Chocolate Brownie Box",
  "costPerPortion": 3.20,
  "packagingCost": 0.45,
  "targetMargin": 0.40,
  "derivedSellPrice": 6.08,
  "foodCostPct": 52.6
}
```

**Canonical Decision C-10:** Owner sees all financial fields; Chef/Procurement/Viewer do NOT. // C-10 is v2-only — applies when multi-user roles are introduced

**Bad approach:**

```csharp
// ❌ REMOVED IN v1 — do not build these:
// RecipeOwnerResponse — no roles in v1, no need for owner-specific DTO
// RecipeStandardResponse — no non-owner roles exist
//
// See the v1 amendment block at the top of this lesson for the unified RecipeResponse DTO.
```

---

## 3. Feature Slice 1 — CreateRecipe

Creates a new recipe for this user. Initializes VersionGroupId and VersionNumber.

### Folder structure

```bash
mkdir -p src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipe
```

### Request

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipe/CreateRecipeCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace RecipeCost.Application.Features.Recipes.Commands.CreateRecipe;

public record CreateRecipeCommand(
    Guid UserId,
    string Name,
    int PortionCount,
    decimal PackagingCost = 0m,
    decimal TargetMargin = 0m,
    string VersionLabel = "Standard",
    List<CreateRecipeItemDto> RecipeItems = null!
) : IRequest<ErrorOr<CreateRecipeResponse>>;

public record CreateRecipeItemDto(Guid IngredientId, decimal Quantity, decimal YieldPercentage);
```

### Response

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipe/CreateRecipeResponse.cs`

```csharp
namespace RecipeCost.Application.Features.Recipes.Commands.CreateRecipe;

public record CreateRecipeResponse(
    Guid Id,
    string Name,
    int PortionCount,
    decimal CostPerPortion,
    Guid VersionGroupId,
    int VersionNumber
);
```

### Validator

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipe/CreateRecipeCommandValidator.cs`

```csharp
using FluentValidation;

namespace RecipeCost.Application.Features.Recipes.Commands.CreateRecipe;

public class CreateRecipeCommandValidator : AbstractValidator<CreateRecipeCommand>
{
    public CreateRecipeCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Recipe name is required.")
            .MaximumLength(200).WithMessage("Recipe name cannot exceed 200 characters.");

        RuleFor(x => x.UserId)
            .NotEmpty().WithMessage("UserId is required.");

        RuleFor(x => x.PortionCount)
            .GreaterThanOrEqualTo(1).WithMessage("Portion count must be at least 1.");

        RuleFor(x => x.PackagingCost).GreaterThanOrEqualTo(0);
        RuleFor(x => x.TargetMargin).InclusiveBetween(0m, 0.99m).WithMessage("Target margin must be between 0% and 99%.");

        RuleFor(x => x.RecipeItems)
            .NotEmpty().WithMessage("Recipe must have at least one item.")
            .Must(items => items.All(i => i.Quantity > 0))
                .WithMessage("Each recipe item quantity must be greater than zero.")
            .Must(items => items.All(i => i.YieldPercentage > 0 && i.YieldPercentage <= 1.0m))
                .WithMessage("Each recipe item yield percentage must be between 0 and 100%.");
    }
}
```

### Handler

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipe/CreateRecipeHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Common.Interfaces;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Application.Features.Recipes.Commands.CreateRecipe;

public class CreateRecipeHandler : IRequestHandler<CreateRecipeCommand, ErrorOr<CreateRecipeResponse>>
{
    private readonly IAppDbContext _db;
    private readonly ICostCascadeService _cascade;

    public CreateRecipeHandler(IAppDbContext db, ICostCascadeService cascade)
    {
        _db = db;
        _cascade = cascade;
    }

    public async Task<ErrorOr<CreateRecipeResponse>> Handle(
        CreateRecipeCommand command, CancellationToken ct)
    {
        // Business rule: Prevent duplicate recipe names within the user's scope
        var isDuplicate = await _db.Recipes
            .AnyAsync(r => r.UserId == command.UserId && r.Name == command.Name, ct);

        if (isDuplicate)
            return Error.Conflict("Recipe.Duplicate",
                "A recipe with this name already exists.");

        // Business rule: All ingredients must belong to this user
        // C-1: Cascade service interface requires ingredients to be valid per user
        var ingredientIds = command.RecipeItems.Select(r => r.IngredientId).Distinct().ToList();
        var validCount = await _db.Ingredients
            .CountAsync(i => i.UserId == command.UserId && ingredientIds.Contains(i.Id), ct);

        if (validCount != ingredientIds.Count)
            return Error.NotFound("Ingredient.NotFound",
                "One or more ingredients do not belong to this user or do not exist.");

        // C-1: Initialize VersionGroupId as new UUID shared across all versions
        var versionGroupId = Guid.NewGuid();

        var recipe = new Recipe
        {
            Id = Guid.NewGuid(),
            UserId = command.UserId,
            Name = command.Name,
            PortionCount = command.PortionCount,
            PackagingCost = command.PackagingCost,
            TargetMargin = command.TargetMargin,
            CostPerPortion = 0m, // Will be set by cascade below
            VersionGroupId = versionGroupId,
            VersionNumber = 1,
            VersionLabel = command.VersionLabel
        };

        _db.Recipes.Add(recipe);

        // Create all RecipeItems
        var recipeItems = command.RecipeItems.Select(r => new RecipeItem
        {
            Id = Guid.NewGuid(),
            RecipeId = recipe.Id,
            IngredientId = r.IngredientId,
            Quantity = r.Quantity,
            YieldPercentage = r.YieldPercentage
        }).ToList();

        foreach (var item in recipeItems)
            _db.RecipeItems.Add(item);

        await _db.SaveChangesAsync(ct);

        // C-1: Trigger cascade for each unique ingredient to compute CostPerPortion
        // Cascade will:
        //   1. Fetch current ingredient prices (C-3)
        //   2. Apply formula (C-2) for all recipes using these ingredients
        //   3. Update Recipe.CostPerPortion in database
        //   4. Evaluate threshold alerts (C-12), dispatch if exceeded
        foreach (var ingredientId in ingredientIds)
        {
            await _cascade.RecalculateForIngredientAsync(ingredientId, ct);
        }

        // Re-fetch recipe to get the updated CostPerPortion from cascade
        // AsNoTracking — read-only re-fetch for return value only
        var updatedRecipe = await _db.Recipes
            .AsNoTracking()
            .FirstOrDefaultAsync(r => r.Id == recipe.Id, ct)
            ?? throw new InvalidOperationException("Recipe not found after creation.");

        return new CreateRecipeResponse(
            updatedRecipe.Id,
            updatedRecipe.Name,
            updatedRecipe.PortionCount,
            updatedRecipe.CostPerPortion,
            updatedRecipe.VersionGroupId,
            updatedRecipe.VersionNumber
        );
    }
}
```

---

## 4. Feature Slice 2 — GetRecipes

Query all recipes for the authenticated user. Returns unified RecipeResponse DTO (v1: single user sees all fields).

### Folder structure

```bash
mkdir -p src/RecipeCost.Application/Features/Recipes/Queries/GetRecipes
```

### Request

**File:** `src/RecipeCost.Application/Features/Recipes/Queries/GetRecipes/GetRecipesQuery.cs`

```csharp
using ErrorOr;
using MediatR;

namespace RecipeCost.Application.Features.Recipes.Queries.GetRecipes;

// v1: single user — no role parameter needed
public record GetRecipesQuery(Guid UserId)
    : IRequest<ErrorOr<List<RecipeResponse>>>;
```

### Responses (Role-split)

**File:** `src/RecipeCost.Application/Features/Recipes/Queries/GetRecipes/RecipeResponse.cs`

```csharp
namespace RecipeCost.Application.Features.Recipes.Queries.GetRecipes;

// ❌ v2-only — RecipeOwnerResponse removed in v1. Use unified RecipeResponse below.

// ❌ v2-only — RecipeStandardResponse removed in v1. Use unified RecipeResponse below.

// v1: All fields visible to the single authenticated user
public record RecipeResponse(
    Guid Id,
    string Name,
    int PortionCount,
    decimal CostPerPortion,
    decimal PackagingCost,
    decimal TargetMargin,
    decimal? DerivedSellPrice,
    decimal? FoodCostPct,
    int VersionNumber,
    string VersionLabel,
    Guid VersionGroupId
);
```

### Handler

**File:** `src/RecipeCost.Application/Features/Recipes/Queries/GetRecipes/GetRecipesHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Common.Interfaces;

namespace RecipeCost.Application.Features.Recipes.Queries.GetRecipes;

public class GetRecipesHandler
    : IRequestHandler<GetRecipesQuery, ErrorOr<List<RecipeResponse>>>
{
    private readonly IAppDbContext _db;

    public GetRecipesHandler(IAppDbContext db)
    {
        _db = db;
    }

    public async Task<ErrorOr<List<RecipeResponse>>> Handle(
        GetRecipesQuery query, CancellationToken ct)
    {
        var recipes = await _db.Recipes
            .AsNoTracking()
            .Where(r => r.UserId == query.UserId)
            .OrderBy(r => r.Name)
            .ToListAsync(ct);

        var responses = recipes.Select(recipe =>
        {
            decimal? derivedSellPrice = recipe.TargetMargin < 1m
                ? (recipe.CostPerPortion + recipe.PackagingCost) / (1m - recipe.TargetMargin)
                : null;
            decimal? foodCostPct = derivedSellPrice.HasValue && derivedSellPrice.Value > 0
                ? (recipe.CostPerPortion / derivedSellPrice.Value) * 100m
                : null;
            return new RecipeResponse(
                recipe.Id, recipe.Name, recipe.PortionCount,
                recipe.CostPerPortion, recipe.PackagingCost, recipe.TargetMargin,
                derivedSellPrice, foodCostPct,
                recipe.VersionNumber, recipe.VersionLabel, recipe.VersionGroupId);
        }).ToList();

        return responses;
    }
}
```

---

## 5. Feature Slice 3 — GetRecipeById

Query a single recipe with RecipeItems details. Also role-split.

### Folder structure

```bash
mkdir -p src/RecipeCost.Application/Features/Recipes/Queries/GetRecipeById
```

### Request

**File:** `src/RecipeCost.Application/Features/Recipes/Queries/GetRecipeById/GetRecipeByIdQuery.cs`

```csharp
using ErrorOr;
using MediatR;

namespace RecipeCost.Application.Features.Recipes.Queries.GetRecipeById;

public record GetRecipeByIdQuery(Guid RecipeId, Guid UserId)
    : IRequest<ErrorOr<RecipeResponse>>;
```

### Responses

**File:** `src/RecipeCost.Application/Features/Recipes/Queries/GetRecipeById/RecipeItemDetail.cs`

```csharp
namespace RecipeCost.Application.Features.Recipes.Queries.GetRecipeById;

public record RecipeItemDetail(
    Guid RecipeItemId,
    Guid IngredientId,
    string IngredientName,
    decimal Quantity,
    decimal YieldPercentage,
    decimal? CurrentPrice  // Fetched from IngredientPriceHistory
);
```

**File:** `src/RecipeCost.Application/Features/Recipes/Queries/GetRecipeById/RecipeByIdOwnerResponse.cs`

```csharp
namespace RecipeCost.Application.Features.Recipes.Queries.GetRecipeById;

// ❌ v2-only — RecipeByIdOwnerResponse removed in v1. Use unified RecipeResponse below.

// ❌ v2-only — RecipeByIdStandardResponse removed in v1. Use unified RecipeResponse below.
```

### Handler

**File:** `src/RecipeCost.Application/Features/Recipes/Queries/GetRecipeById/GetRecipeByIdHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Common.Interfaces;

namespace RecipeCost.Application.Features.Recipes.Queries.GetRecipeById;

public class GetRecipeByIdHandler : IRequestHandler<GetRecipeByIdQuery, ErrorOr<RecipeResponse>>
{
    private readonly IAppDbContext _db;

    public GetRecipeByIdHandler(IAppDbContext db)
    {
        _db = db;
    }

    public async Task<ErrorOr<RecipeResponse>> Handle(
        GetRecipeByIdQuery query, CancellationToken ct)
    {
        var recipe = await _db.Recipes
            .AsNoTracking()
            .Include(r => r.RecipeItems)
            .ThenInclude(ri => ri.Ingredient)
            .FirstOrDefaultAsync(r => r.Id == query.RecipeId, ct);

        if (recipe is null || recipe.UserId != query.UserId)
            return Error.NotFound("Recipe.NotFound",
                "Recipe not found or doesn't belong to this user.");

        // Build RecipeItemDetails with current prices from IngredientPriceHistory (C-3)
        var itemDetails = new List<RecipeItemDetail>();

        // Batch-load latest prices for all ingredients in this recipe — avoids N+1 queries.
        // Without this, the foreach below would fire one DB query per recipe item.
        var ingredientIds = recipe.RecipeItems.Select(ri => ri.IngredientId).Distinct().ToList();
        var latestPrices = await _db.IngredientPriceHistories
            .Where(ph => ingredientIds.Contains(ph.IngredientId))
            .GroupBy(ph => ph.IngredientId)
            .Select(g => new
            {
                IngredientId = g.Key,
                Price = (decimal?)g.OrderByDescending(ph => ph.CommittedAt).First().Price
            })
            .ToDictionaryAsync(x => x.IngredientId, x => x.Price, ct);

        foreach (var recipeItem in recipe.RecipeItems)
        {
            // C-3: current price from pre-loaded batch (avoids N+1)
            latestPrices.TryGetValue(recipeItem.IngredientId, out var currentPrice);

            itemDetails.Add(new RecipeItemDetail(
                recipeItem.Id,
                recipeItem.IngredientId,
                recipeItem.Ingredient.Name,
                recipeItem.Quantity,
                recipeItem.YieldPercentage,
                currentPrice
            ));
        }

        decimal? derivedSellPrice = recipe.TargetMargin < 1m
            ? (recipe.CostPerPortion + recipe.PackagingCost) / (1m - recipe.TargetMargin)
            : null;
        decimal? foodCostPct = derivedSellPrice.HasValue && derivedSellPrice.Value > 0
            ? (recipe.CostPerPortion / derivedSellPrice.Value) * 100m
            : null;

        return new RecipeResponse(
            recipe.Id, recipe.Name, recipe.PortionCount,
            recipe.CostPerPortion, recipe.PackagingCost, recipe.TargetMargin,
            derivedSellPrice, foodCostPct,
            recipe.VersionNumber, recipe.VersionLabel, recipe.VersionGroupId);
    }
}
```

---

## 6. Feature Slice 4 — AddRecipeItem

Adds an ingredient to an existing recipe.

### Folder structure

```bash
mkdir -p src/RecipeCost.Application/Features/Recipes/Commands/AddRecipeItem
```

### Request & Response

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/AddRecipeItem/AddRecipeItemCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace RecipeCost.Application.Features.Recipes.Commands.AddRecipeItem;

public record AddRecipeItemCommand(
    Guid RecipeId,
    Guid UserId,
    Guid IngredientId,
    decimal Quantity,
    decimal YieldPercentage
) : IRequest<ErrorOr<AddRecipeItemResponse>>;
```

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/AddRecipeItem/AddRecipeItemResponse.cs`

```csharp
namespace RecipeCost.Application.Features.Recipes.Commands.AddRecipeItem;

public record AddRecipeItemResponse(
    Guid RecipeItemId,
    Guid RecipeId,
    Guid IngredientId,
    decimal NewCostPerPortion
);
```

### Validator

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/AddRecipeItem/AddRecipeItemCommandValidator.cs`

```csharp
using FluentValidation;

namespace RecipeCost.Application.Features.Recipes.Commands.AddRecipeItem;

public class AddRecipeItemCommandValidator : AbstractValidator<AddRecipeItemCommand>
{
    public AddRecipeItemCommandValidator()
    {
        RuleFor(x => x.RecipeId).NotEmpty();
        RuleFor(x => x.UserId).NotEmpty();
        RuleFor(x => x.IngredientId).NotEmpty();
        RuleFor(x => x.Quantity).GreaterThan(0).WithMessage("Quantity must be greater than zero.");
        RuleFor(x => x.YieldPercentage)
            .GreaterThan(0).LessThanOrEqualTo(1.0m)
            .WithMessage("Yield percentage must be between 0 and 100%.");
    }
}
```

### Handler

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/AddRecipeItem/AddRecipeItemHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Common.Interfaces;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Application.Features.Recipes.Commands.AddRecipeItem;

public class AddRecipeItemHandler : IRequestHandler<AddRecipeItemCommand, ErrorOr<AddRecipeItemResponse>>
{
    private readonly IAppDbContext _db;
    private readonly ICostCascadeService _cascade;

    public AddRecipeItemHandler(IAppDbContext db, ICostCascadeService cascade)
    {
        _db = db;
        _cascade = cascade;
    }

    public async Task<ErrorOr<AddRecipeItemResponse>> Handle(
        AddRecipeItemCommand command, CancellationToken ct)
    {
        // Verify recipe belongs to user (404 if not)
        // AsNoTracking — access-control check only; recipe is never modified in this handler
        var recipe = await _db.Recipes
            .AsNoTracking()
            .FirstOrDefaultAsync(r => r.Id == command.RecipeId && r.UserId == command.UserId, ct);

        if (recipe is null)
            return Error.NotFound("Recipe.NotFound",
                "Recipe not found or doesn't belong to this user.");

        // Verify ingredient belongs to user (404 if not)
        // AsNoTracking — access-control check only; ingredient is never modified in this handler
        var ingredient = await _db.Ingredients
            .AsNoTracking()
            .FirstOrDefaultAsync(i => i.Id == command.IngredientId && i.UserId == command.UserId, ct);

        if (ingredient is null)
            return Error.NotFound("Ingredient.NotFound",
                "Ingredient not found or doesn't belong to this user.");

        // Check ingredient not already in recipe (409 conflict)
        var alreadyExists = await _db.RecipeItems
            .AnyAsync(ri => ri.RecipeId == command.RecipeId && ri.IngredientId == command.IngredientId, ct);

        if (alreadyExists)
            return Error.Conflict("RecipeItem.Duplicate",
                "This ingredient is already in the recipe. Use UpdateRecipeItem to change quantity.");

        // Create and add RecipeItem
        var recipeItem = new RecipeItem
        {
            Id = Guid.NewGuid(),
            RecipeId = command.RecipeId,
            IngredientId = command.IngredientId,
            Quantity = command.Quantity,
            YieldPercentage = command.YieldPercentage
        };

        _db.RecipeItems.Add(recipeItem);
        await _db.SaveChangesAsync(ct);

        // C-1: Trigger cascade to recalculate cost for this ingredient's affected recipes
        await _cascade.RecalculateForIngredientAsync(command.IngredientId, ct);

        // Re-fetch recipe to get updated cost
        // AsNoTracking — read-only re-fetch for return value only
        var updatedRecipe = await _db.Recipes
            .AsNoTracking()
            .FirstOrDefaultAsync(r => r.Id == command.RecipeId, ct)
            ?? throw new InvalidOperationException("Recipe not found after item addition.");

        return new AddRecipeItemResponse(
            recipeItem.Id,
            command.RecipeId,
            command.IngredientId,
            updatedRecipe.CostPerPortion
        );
    }
}
```

---

## 7. Feature Slice 5 — RemoveRecipeItem

Removes an ingredient from a recipe.

### Folder structure

```bash
mkdir -p src/RecipeCost.Application/Features/Recipes/Commands/RemoveRecipeItem
```

### Request & Response

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/RemoveRecipeItem/RemoveRecipeItemCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace RecipeCost.Application.Features.Recipes.Commands.RemoveRecipeItem;

public record RemoveRecipeItemCommand(
    Guid RecipeItemId,
    Guid RecipeId,
    Guid UserId
) : IRequest<ErrorOr<RemoveRecipeItemResponse>>;
```

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/RemoveRecipeItem/RemoveRecipeItemResponse.cs`

```csharp
namespace RecipeCost.Application.Features.Recipes.Commands.RemoveRecipeItem;

public record RemoveRecipeItemResponse(
    Guid RecipeId,
    decimal NewCostPerPortion
);
```

### Handler

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/RemoveRecipeItem/RemoveRecipeItemHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Common.Interfaces;

namespace RecipeCost.Application.Features.Recipes.Commands.RemoveRecipeItem;

public class RemoveRecipeItemHandler
    : IRequestHandler<RemoveRecipeItemCommand, ErrorOr<RemoveRecipeItemResponse>>
{
    private readonly IAppDbContext _db;
    private readonly ICostCascadeService _cascade;

    public RemoveRecipeItemHandler(IAppDbContext db, ICostCascadeService cascade)
    {
        _db = db;
        _cascade = cascade;
    }

    public async Task<ErrorOr<RemoveRecipeItemResponse>> Handle(
        RemoveRecipeItemCommand command, CancellationToken ct)
    {
        // Verify recipe belongs to user
        // AsNoTracking — access-control check only; recipeItem (not recipe) is the entity being removed
        var recipe = await _db.Recipes
            .AsNoTracking()
            .FirstOrDefaultAsync(r => r.Id == command.RecipeId && r.UserId == command.UserId, ct);

        if (recipe is null)
            return Error.NotFound("Recipe.NotFound",
                "Recipe not found or doesn't belong to this user.");

        // Verify RecipeItem belongs to this recipe
        // Change-tracked — recipeItem is passed to _db.RecipeItems.Remove() below
        var recipeItem = await _db.RecipeItems
            .FirstOrDefaultAsync(ri => ri.Id == command.RecipeItemId && ri.RecipeId == command.RecipeId, ct);

        if (recipeItem is null)
            return Error.NotFound("RecipeItem.NotFound",
                "Recipe item not found in this recipe.");

        // Store ingredient ID for cascade BEFORE deleting
        var ingredientId = recipeItem.IngredientId;

        // Delete RecipeItem
        _db.RecipeItems.Remove(recipeItem);
        await _db.SaveChangesAsync(ct);

        // C-1: Trigger cascade — removal of item may affect cost
        await _cascade.RecalculateForIngredientAsync(ingredientId, ct);

        // Re-fetch recipe to get updated cost
        // AsNoTracking — read-only re-fetch for return value only
        var updatedRecipe = await _db.Recipes
            .AsNoTracking()
            .FirstOrDefaultAsync(r => r.Id == command.RecipeId, ct)
            ?? throw new InvalidOperationException("Recipe not found after item removal.");

        return new RemoveRecipeItemResponse(
            command.RecipeId,
            updatedRecipe.CostPerPortion
        );
    }
}
```

---

## 8. Feature Slice 6 — CreateRecipeVersion

Creates a new version of a recipe, copying all RecipeItems. Same VersionGroupId, incremented VersionNumber.

### Folder structure

```bash
mkdir -p src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipeVersion
```

### Request & Response

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipeVersion/CreateRecipeVersionCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace RecipeCost.Application.Features.Recipes.Commands.CreateRecipeVersion;

public record CreateRecipeVersionCommand(
    Guid SourceRecipeId,
    Guid UserId,
    string VersionLabel
) : IRequest<ErrorOr<CreateRecipeVersionResponse>>;
```

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipeVersion/CreateRecipeVersionResponse.cs`

```csharp
namespace RecipeCost.Application.Features.Recipes.Commands.CreateRecipeVersion;

public record CreateRecipeVersionResponse(
    Guid NewRecipeId,
    Guid VersionGroupId,
    int VersionNumber,
    string VersionLabel
);
```

### Handler

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/CreateRecipeVersion/CreateRecipeVersionHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Common.Interfaces;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Application.Features.Recipes.Commands.CreateRecipeVersion;

public class CreateRecipeVersionHandler
    : IRequestHandler<CreateRecipeVersionCommand, ErrorOr<CreateRecipeVersionResponse>>
{
    private readonly IAppDbContext _db;
    private readonly ICostCascadeService _cascade;

    public CreateRecipeVersionHandler(IAppDbContext db, ICostCascadeService cascade)
    {
        _db = db;
        _cascade = cascade;
    }

    public async Task<ErrorOr<CreateRecipeVersionResponse>> Handle(
        CreateRecipeVersionCommand command, CancellationToken ct)
    {
        // Fetch source recipe with RecipeItems (404 if wrong user)
        // AsNoTracking — source recipe is read-only (items are copied to new recipe, never modified here)
        var sourceRecipe = await _db.Recipes
            .AsNoTracking()
            .Include(r => r.RecipeItems)
            .FirstOrDefaultAsync(r => r.Id == command.SourceRecipeId && r.UserId == command.UserId, ct);

        if (sourceRecipe is null)
            return Error.NotFound("Recipe.NotFound",
                "Source recipe not found or doesn't belong to this user.");

        // C-1: Create new recipe with same VersionGroupId, incremented VersionNumber
        var newRecipe = new Recipe
        {
            Id = Guid.NewGuid(),
            UserId = sourceRecipe.UserId,
            Name = sourceRecipe.Name,
            PortionCount = sourceRecipe.PortionCount,
            PackagingCost = sourceRecipe.PackagingCost,
            TargetMargin = sourceRecipe.TargetMargin,
            CostPerPortion = 0m,  // Will be set by cascade
            VersionGroupId = sourceRecipe.VersionGroupId,  // SAME as source
            VersionNumber = sourceRecipe.VersionNumber + 1,
            VersionLabel = command.VersionLabel
        };

        _db.Recipes.Add(newRecipe);

        // Copy all RecipeItems verbatim from source
        var copiedItems = sourceRecipe.RecipeItems.Select(ri => new RecipeItem
        {
            Id = Guid.NewGuid(),
            RecipeId = newRecipe.Id,
            IngredientId = ri.IngredientId,
            Quantity = ri.Quantity,
            YieldPercentage = ri.YieldPercentage
        }).ToList();

        foreach (var item in copiedItems)
            _db.RecipeItems.Add(item);

        await _db.SaveChangesAsync(ct);

        // C-1: Trigger cascade for each unique ingredient to compute independent CostPerPortion
        var ingredientIds = sourceRecipe.RecipeItems
            .Select(ri => ri.IngredientId)
            .Distinct()
            .ToList();

        foreach (var ingredientId in ingredientIds)
        {
            await _cascade.RecalculateForIngredientAsync(ingredientId, ct);
        }

        // Re-fetch new recipe to get computed CostPerPortion
        // AsNoTracking — read-only re-fetch for return value only
        var updatedNewRecipe = await _db.Recipes
            .AsNoTracking()
            .FirstOrDefaultAsync(r => r.Id == newRecipe.Id, ct)
            ?? throw new InvalidOperationException("New recipe not found after creation.");

        return new CreateRecipeVersionResponse(
            updatedNewRecipe.Id,
            updatedNewRecipe.VersionGroupId,
            updatedNewRecipe.VersionNumber,
            updatedNewRecipe.VersionLabel
        );
    }
}
```

---

## 9. Feature Slice 7 — UpdateRecipe

Updates recipe metadata (name, portion count, selling price, threshold). Triggers cascade if PortionCount changes.

### Folder structure

```bash
mkdir -p src/RecipeCost.Application/Features/Recipes/Commands/UpdateRecipe
```

### Request & Response

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/UpdateRecipe/UpdateRecipeCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace RecipeCost.Application.Features.Recipes.Commands.UpdateRecipe;

public record UpdateRecipeCommand(
    Guid RecipeId,
    Guid UserId,
    string Name,
    int PortionCount,
    decimal PackagingCost,
    decimal TargetMargin,
    string VersionLabel
) : IRequest<ErrorOr<UpdateRecipeResponse>>;
```

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/UpdateRecipe/UpdateRecipeResponse.cs`

```csharp
namespace RecipeCost.Application.Features.Recipes.Commands.UpdateRecipe;

public record UpdateRecipeResponse(
    Guid Id,
    string Name,
    decimal CostPerPortion
);
```

### Validator

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/UpdateRecipe/UpdateRecipeCommandValidator.cs`

```csharp
using FluentValidation;

namespace RecipeCost.Application.Features.Recipes.Commands.UpdateRecipe;

public class UpdateRecipeCommandValidator : AbstractValidator<UpdateRecipeCommand>
{
    public UpdateRecipeCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Recipe name is required.")
            .MaximumLength(200).WithMessage("Recipe name cannot exceed 200 characters.");

        RuleFor(x => x.PortionCount)
            .GreaterThanOrEqualTo(1).WithMessage("Portion count must be at least 1.");

        RuleFor(x => x.PackagingCost).GreaterThanOrEqualTo(0);
        RuleFor(x => x.TargetMargin).InclusiveBetween(0m, 0.99m).WithMessage("Target margin must be between 0% and 99%.");
    }
}
```

### Handler

**File:** `src/RecipeCost.Application/Features/Recipes/Commands/UpdateRecipe/UpdateRecipeHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Common.Interfaces;

namespace RecipeCost.Application.Features.Recipes.Commands.UpdateRecipe;

public class UpdateRecipeHandler : IRequestHandler<UpdateRecipeCommand, ErrorOr<UpdateRecipeResponse>>
{
    private readonly IAppDbContext _db;
    private readonly ICostCascadeService _cascade;

    public UpdateRecipeHandler(IAppDbContext db, ICostCascadeService cascade)
    {
        _db = db;
        _cascade = cascade;
    }

    public async Task<ErrorOr<UpdateRecipeResponse>> Handle(
        UpdateRecipeCommand command, CancellationToken ct)
    {
        var recipe = await _db.Recipes
            .Include(r => r.RecipeItems)
            .FirstOrDefaultAsync(r => r.Id == command.RecipeId && r.UserId == command.UserId, ct);

        if (recipe is null)
            return Error.NotFound("Recipe.NotFound",
                "Recipe not found or doesn't belong to this user.");

        // Check for duplicate name if name changed
        if (recipe.Name != command.Name)
        {
            var isDuplicate = await _db.Recipes
                .AnyAsync(r => r.UserId == command.UserId && r.Name == command.Name && r.Id != command.RecipeId, ct);

            if (isDuplicate)
                return Error.Conflict("Recipe.Duplicate",
                    "A recipe with this name already exists.");
        }

        // Store old portion count to detect changes
        var portionCountChanged = recipe.PortionCount != command.PortionCount;

        // Update fields
        recipe.Name = command.Name;
        recipe.PortionCount = command.PortionCount;
        recipe.PackagingCost = command.PackagingCost;
        recipe.TargetMargin = command.TargetMargin;
        recipe.VersionLabel = command.VersionLabel;

        await _db.SaveChangesAsync(ct);

        // C-1 & C-2: If PortionCount changed, trigger cascade for all ingredients
        // because cost formula includes portion_count — change requires recalculation
        if (portionCountChanged)
        {
            var ingredientIds = recipe.RecipeItems
                .Select(ri => ri.IngredientId)
                .Distinct()
                .ToList();

            foreach (var ingredientId in ingredientIds)
            {
                await _cascade.RecalculateForIngredientAsync(ingredientId, ct);
            }

            // Re-fetch to get updated CostPerPortion
            // AsNoTracking — read-only re-fetch; the previous tracked entity has already been saved
            recipe = await _db.Recipes
                .AsNoTracking()
                .FirstOrDefaultAsync(r => r.Id == command.RecipeId, ct)
                ?? throw new InvalidOperationException("Recipe not found after update.");
        }

        return new UpdateRecipeResponse(
            recipe.Id,
            recipe.Name,
            recipe.CostPerPortion
        );
    }
}
```

---

## 10. RecipeEndpoints

All 7 slices wired to Minimal API endpoints. Extract role from JWT claims.

### Request DTOs (for minimal API binding)

**File:** `src/RecipeCost.API/Contracts/Recipe/CreateRecipeRequest.cs`

```csharp
namespace RecipeCost.API.Contracts.Recipe;

public record CreateRecipeRequest(
    string Name,
    int PortionCount,
    decimal PackagingCost = 0m,
    decimal TargetMargin = 0m,
    string VersionLabel = "Standard",
    List<CreateRecipeItemRequest> RecipeItems = null!
);

public record CreateRecipeItemRequest(Guid IngredientId, decimal Quantity, decimal YieldPercentage);

public record AddRecipeItemRequest(Guid IngredientId, decimal Quantity, decimal YieldPercentage);

public record UpdateRecipeRequest(
    string Name,
    int PortionCount,
    decimal PackagingCost,
    decimal TargetMargin,
    string VersionLabel
);

public record CreateRecipeVersionRequest(string VersionLabel);
```

### Endpoints

**File:** `src/RecipeCost.API/Endpoints/RecipeEndpoints.cs`

```csharp
using System.Security.Claims;
using MediatR;
using RecipeCost.API.Contracts.Recipe;
using RecipeCost.API.Extensions;
using RecipeCost.Application.Features.Recipes.Commands.AddRecipeItem;
using RecipeCost.Application.Features.Recipes.Commands.CreateRecipe;
using RecipeCost.Application.Features.Recipes.Commands.CreateRecipeVersion;
using RecipeCost.Application.Features.Recipes.Commands.RemoveRecipeItem;
using RecipeCost.Application.Features.Recipes.Commands.UpdateRecipe;
using RecipeCost.Application.Features.Recipes.Queries.GetRecipeById;
using RecipeCost.Application.Features.Recipes.Queries.GetRecipes;

namespace RecipeCost.API.Endpoints;

public static class RecipeEndpoints
{
    public static void MapRecipeEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/recipes")
            .WithTags("Recipes")
            .RequireAuthorization(); // Any authenticated user

        // GET /api/recipes — all recipes for user
        group.MapGet("/", GetRecipes)
            .WithName("GetRecipes")
            .WithOpenApi();

        // POST /api/recipes — create recipe
        group.MapPost("/", CreateRecipe)
            .RequireAuthorization()
            .WithName("CreateRecipe")
            .WithOpenApi();

        // GET /api/recipes/{recipeId} — single recipe with items
        group.MapGet("/{recipeId}", GetRecipeById)
            .WithName("GetRecipeById")
            .WithOpenApi();

        // PUT /api/recipes/{recipeId} — update metadata
        group.MapPut("/{recipeId}", UpdateRecipe)
            .RequireAuthorization()
            .WithName("UpdateRecipe")
            .WithOpenApi();

        // POST /api/recipes/{recipeId}/items — add item
        group.MapPost("/{recipeId}/items", AddRecipeItem)
            .RequireAuthorization()
            .WithName("AddRecipeItem")
            .WithOpenApi();

        // DELETE /api/recipes/{recipeId}/items/{recipeItemId} — remove item
        group.MapDelete("/{recipeId}/items/{recipeItemId}", RemoveRecipeItem)
            .RequireAuthorization()
            .WithName("RemoveRecipeItem")
            .WithOpenApi();

        // POST /api/recipes/{recipeId}/versions — create version
        group.MapPost("/{recipeId}/versions", CreateRecipeVersion)
            .RequireAuthorization()
            .WithName("CreateRecipeVersion")
            .WithOpenApi();
    }

    // Handler implementations

    private static async Task<IResult> GetRecipes(
        HttpContext httpContext, ISender sender, CancellationToken ct)
    {
        var userId = httpContext.User.GetUserId();

        var query = new GetRecipesQuery(userId);
        var result = await sender.Send(query, ct);

        return result.ToApiResult();
    }

    private static async Task<IResult> CreateRecipe(
        CreateRecipeRequest request, HttpContext httpContext, ISender sender, CancellationToken ct)
    {
        var userId = httpContext.User.GetUserId();
        var command = new CreateRecipeCommand(
            userId,
            request.Name,
            request.PortionCount,
            request.PackagingCost,
            request.TargetMargin,
            request.VersionLabel,
            request.RecipeItems.Select(r => new CreateRecipeItemDto(
                r.IngredientId, r.Quantity, r.YieldPercentage)).ToList()
        );

        var result = await sender.Send(command, ct);
        return result.ToCreatedResult($"/api/recipes/{result.Value?.Id}");
    }

    private static async Task<IResult> GetRecipeById(
        Guid recipeId, HttpContext httpContext, ISender sender, CancellationToken ct)
    {
        var userId = httpContext.User.GetUserId();

        var query = new GetRecipeByIdQuery(recipeId, userId);
        var result = await sender.Send(query, ct);

        return result.ToApiResult();
    }

    private static async Task<IResult> UpdateRecipe(
        Guid recipeId, UpdateRecipeRequest request, HttpContext httpContext, ISender sender, CancellationToken ct)
    {
        var userId = httpContext.User.GetUserId();
        var command = new UpdateRecipeCommand(
            recipeId,
            userId,
            request.Name,
            request.PortionCount,
            request.PackagingCost,
            request.TargetMargin,
            request.VersionLabel
        );

        var result = await sender.Send(command, ct);
        return result.ToApiResult();
    }

    private static async Task<IResult> AddRecipeItem(
        Guid recipeId, AddRecipeItemRequest request, HttpContext httpContext, ISender sender, CancellationToken ct)
    {
        var userId = httpContext.User.GetUserId();
        var command = new AddRecipeItemCommand(
            recipeId,
            userId,
            request.IngredientId,
            request.Quantity,
            request.YieldPercentage
        );

        var result = await sender.Send(command, ct);
        return result.ToCreatedResult($"/api/recipes/{recipeId}/items/{result.Value?.RecipeItemId}");
    }

    private static async Task<IResult> RemoveRecipeItem(
        Guid recipeId, Guid recipeItemId, HttpContext httpContext, ISender sender, CancellationToken ct)
    {
        var userId = httpContext.User.GetUserId();
        var command = new RemoveRecipeItemCommand(recipeItemId, recipeId, userId);
        var result = await sender.Send(command, ct);

        return result.ToApiResult();
    }

    private static async Task<IResult> CreateRecipeVersion(
        Guid recipeId, CreateRecipeVersionRequest request, HttpContext httpContext, ISender sender, CancellationToken ct)
    {
        var userId = httpContext.User.GetUserId();
        var command = new CreateRecipeVersionCommand(recipeId, userId, request.VersionLabel);
        var result = await sender.Send(command, ct);

        return result.ToApiResult();
    }
}
```

### Register endpoints and policies in Program.cs

Update `src/RecipeCost.API/Program.cs` to add authorization policies:

```csharp
// After builder.Services.AddAuthorization():
builder.Services.AddAuthorization();

// Register endpoint handler:
app.MapRecipeEndpoints();
```

---

## 11. Recipe Endpoints Summary

| Method | Path | Auth | Handler | Purpose |
|---|---|---|---|---|
| **GET** | `/api/recipes` | RequireAuthorization | GetRecipes | List all recipes for user |
| **POST** | `/api/recipes` | RequireAuthorization | CreateRecipe | Create new recipe, trigger cascade |
| **GET** | `/api/recipes/{recipeId}` | RequireAuthorization | GetRecipeById | Single recipe with items |
| **PUT** | `/api/recipes/{recipeId}` | RequireAuthorization | UpdateRecipe | Update metadata, cascade if portion count changed |
| **POST** | `/api/recipes/{recipeId}/items` | RequireAuthorization | AddRecipeItem | Add ingredient to recipe, trigger cascade |
| **DELETE** | `/api/recipes/{recipeId}/items/{recipeItemId}` | RequireAuthorization | RemoveRecipeItem | Remove ingredient from recipe, trigger cascade |
| **POST** | `/api/recipes/{recipeId}/versions` | RequireAuthorization | CreateRecipeVersion | Create new version, copy items, trigger cascade |

---

## 12. How to Verify — Full Scenario

Assume the API is running with `.MapRecipeEndpoints()` registered. You're logged in as a verified user with a valid JWT token. Your account has ingredients created (from L6).

### Step 1: Get your ingredient IDs

```bash
TOKEN="<your-jwt-from-login>"

# List your recipes
curl -X GET "http://localhost:5000/api/recipes" \
  -H "Authorization: Bearer $TOKEN"
```

### Step 2: Create a recipe with 3 ingredients

First, verify you have ingredient IDs from L6:

```bash
curl -X GET "http://localhost:5000/api/recipes" \
  -H "Authorization: Bearer $TOKEN"
# Expected: List of ingredients with IDs
# Save ingredient IDs as: ING1="...", ING2="...", ING3="..."
```

Create recipe:

```bash
curl -X POST "http://localhost:5000/api/recipes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Pasta Carbonara",
    "portionCount": 4,
    "packagingCost": 0.45,
    "targetMargin": 0.35,
    "versionLabel": "v1",
    "recipeItems": [
      {"ingredientId": "'$ING1'", "quantity": 200, "yieldPercentage": 0.95},
      {"ingredientId": "'$ING2'", "quantity": 100, "yieldPercentage": 0.90},
      {"ingredientId": "'$ING3'", "quantity": 50, "yieldPercentage": 1.0}
    ]
  }'

# Expected: 201 Created
# Response:
# {
#   "id": "recipe-uuid-1",
#   "name": "Test Pasta Carbonara",
#   "portionCount": 4,
#   "costPerPortion": 3.47,
#   "versionGroupId": "version-group-uuid",
#   "versionNumber": 1
# }
```

### Step 3: Verify CostPerPortion was computed by cascade

```bash
RECIPE_ID="<recipe-uuid-1>"
curl -X GET "http://localhost:5000/api/recipes/$RECIPE_ID" \
  -H "Authorization: Bearer $TOKEN"

# Expected: 200 OK
# Unified response (single user sees all fields):
# {
#   "id": "recipe-uuid-1",
#   "name": "Test Pasta Carbonara",
#   "portionCount": 4,
#   "costPerPortion": 3.47,
#   "packagingCost": 0.45,
#   "targetMargin": 0.35,
#   "derivedSellPrice": 6.03,
#   "foodCostPct": 57.5,
#   "versionNumber": 1,
#   "versionLabel": "v1",
#   "versionGroupId": "version-group-uuid"
# }
```

### Step 4: Add an ingredient to the recipe

```bash
curl -X POST "http://localhost:5000/api/recipes/$RECIPE_ID/items" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "ingredientId": "'$ING_NEW'",
    "quantity": 25,
    "yieldPercentage": 0.85
  }'

# Expected: 201 Created
# {
#   "recipeItemId": "item-uuid-new",
#   "recipeId": "recipe-uuid-1",
#   "ingredientId": "'$ING_NEW'",
#   "newCostPerPortion": 3.62
# }
# Note: CostPerPortion updated via cascade (C-1)
```

### Step 6: Verify updated cost propagated

```bash
curl -X GET "http://localhost:5000/api/recipes/$RECIPE_ID" \
  -H "Authorization: Bearer $TOKEN"

# Expected: costPerPortion now 3.62 (reflects new ingredient cost-per-portion share)
```

### Step 7: Remove a recipe item

```bash
RECIPE_ITEM_ID="<item-uuid-from-recipe-detail>"
curl -X DELETE "http://localhost:5000/api/recipes/$RECIPE_ID/items/$RECIPE_ITEM_ID" \
  -H "Authorization: Bearer $TOKEN"

# Expected: 200 OK
# {
#   "recipeId": "recipe-uuid-1",
#   "newCostPerPortion": 3.42
# }
# Note: Cost recalculated after removal
```

### Step 8: Create a new recipe version

```bash
curl -X POST "http://localhost:5000/api/recipes/$RECIPE_ID/versions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"versionLabel": "premium"}'

# Expected: 201 Created
# {
#   "newRecipeId": "recipe-uuid-2",
#   "versionGroupId": "version-group-uuid",
#   "versionNumber": 2,
#   "versionLabel": "premium"
# }
# Note: versionGroupId is SAME as v1, versionNumber incremented
```

### Step 9: Update recipe portion count — verify cascade triggers

```bash
curl -X PUT "http://localhost:5000/api/recipes/$RECIPE_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Pasta Carbonara",
    "portionCount": 6,
    "packagingCost": 0.45,
    "targetMargin": 0.35,
    "versionLabel": "v1"
  }'

# Expected: 200 OK
# {
#   "id": "recipe-uuid-1",
#   "name": "Test Pasta Carbonara",
#   "costPerPortion": 2.31
# }
# Note: costPerPortion decreased because portion_count increased (C-2 formula)

# Verify v2 (the version we created) is INDEPENDENT:
curl -X GET "http://localhost:5000/api/recipes/recipe-uuid-2" \
  -H "Authorization: Bearer $TOKEN"
# v2 still has its own cost — not affected by v1 update (proved independence)
```

---

## 13. Diagram: Flow of Cascade Trigger

```
CreateRecipe Handler
  ├─ Save Recipe + RecipeItems to DB
  └─ For each unique IngredientId:
      └─ Call ICostCascadeService.RecalculateForIngredientAsync()
          ├─ Fetch current price (C-3)
          ├─ Find all Recipes using this IngredientId
          ├─ Apply formula (C-2) per recipe
          ├─ Update Recipe.CostPerPortion
          └─ Dispatch price spike alert if threshold exceeded
                └─ Alert target: single user (userId) via IAlertDispatcher
                   (v1: no role-split; C-10/C-12 are v2-only)

Result: Recipe.CostPerPortion is ALWAYS server-authoritative, computed post-save.
        No client-side estimation is ever persisted.
        No stale cache issues.
```

---

## 14. Canonical Decisions Implemented

| Decision | Implementation |
|---|---|
| **C-1** | CreateRecipe, AddRecipeItem, RemoveRecipeItem, CreateRecipeVersion, UpdateRecipe (if portionCount changes) all call `ICostCascadeService.RecalculateForIngredientAsync` |
| **C-2** | Cascade service implements the canonical formula. No handlers compute cost directly. Formula: `cost_per_portion = SUM( (current_price / unit_size) * quantity * (1 / yield_percentage) ) / portion_count` |
| **C-3** | Current price_always fetched using: `SELECT price FROM IngredientPriceHistory WHERE ingredient_id = @id ORDER BY committed_at DESC LIMIT 1` |
| **C-9** | `Recipe.CostThresholdPercentage` is removed in v1 — no stored threshold; personal Telegram spike alerts added in Phase 4 (L15) |
| **C-10** | Unified `RecipeResponse` — single user sees all fields. // C-10 is v2-only — applies when multi-user roles are introduced |
| **C-12** | Cost threshold alert removed in v1 — no stored threshold. Spike alerts dispatched via `IAlertDispatcher.SendPriceSpikeAlertAsync(userId, ...)` |

---

## Phase 2 Complete — Recipe Management & Costing

Across L6–L8, you now have:

| Feature | Status |
|---|---|
| Ingredient CRUD + validation | ✅ (L6) |
| Ingredient price history + current price lookup (C-3) | ✅ (L7) |
| CostCascadeService + canonical formula (C-2) | ✅ (L7) |
| Recipe creation with automatic cost computation | ✅ (L8) |
| Recipe queries with role-split responses (C-10) | ✅ (L8) |
| Recipe items: add, remove (cascade triggered) | ✅ (L8) |
| Recipe versioning (independent live versions) | ✅ (L8) |
| 7 complete vertical slices + REST endpoints | ✅ (L8) |
| All canonical decisions (C-1, C-2, C-3, C-9, C-10, C-12) enforced | ✅ |

### Deferred to later phases

| Feature | Phase | Lesson |
|---|---|---|
| Async cascade with queue + idempotent retries | Phase 4 | L16+ |
| Telegram threshold alert dispatch to bot | Phase 4 | L15 |
| Invoice OCR + price extraction | Phase 3 | L12–L14 |
| Recipe UI: diff/compare versions, rollback | Phase 3 | L10–L11 |

---

## 15. Testing Recipe Handlers

> Use MSTest 3.x with the EF Core In-Memory provider. Assert on **response values** (`CostPerPortion`, recipe name, `DerivedSellPrice`) — not on mock call counts.

### Key Test Cases

**File:** `tests/RecipeCost.Application.Tests/Features/Recipes/GetRecipesHandlerTests.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Features.Recipes.Queries.GetRecipes;
using RecipeCost.Domain.Entities;
using RecipeCost.Infrastructure.Data;

namespace RecipeCost.Application.Tests.Features.Recipes;

[TestClass]
public sealed class GetRecipesHandlerTests
{
    [TestMethod]
    public async Task Handle_ReturnsOnlyCurrentUserRecipes()
    {
        // Arrange
        using var db = new AppDbContext(new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString()).Options);

        var userId = Guid.NewGuid();
        var otherUserId = Guid.NewGuid();
        db.Recipes.AddRange(
            new Recipe { Id = Guid.NewGuid(), UserId = userId, Name = "My Brownie", PortionCount = 4, VersionGroupId = Guid.NewGuid() },
            new Recipe { Id = Guid.NewGuid(), UserId = otherUserId, Name = "Other Brownie", PortionCount = 4, VersionGroupId = Guid.NewGuid() }
        );
        await db.SaveChangesAsync();

        var handler = new GetRecipesHandler(db);

        // Act
        var result = await handler.Handle(new GetRecipesQuery(userId), CancellationToken.None);

        // Assert — only the current user's recipe is returned
        Assert.IsFalse(result.IsError);
        Assert.HasCount(1, result.Value);
        Assert.AreEqual("My Brownie", result.Value[0].Name);
    }
}
```

**File:** `tests/RecipeCost.Application.Tests/Features/Recipes/CreateRecipeHandlerTests.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using NSubstitute;
using RecipeCost.Application.Common.Interfaces;
using RecipeCost.Application.Features.Recipes.Commands.CreateRecipe;
using RecipeCost.Domain.Entities;
using RecipeCost.Infrastructure.Data;

namespace RecipeCost.Application.Tests.Features.Recipes;

[TestClass]
public sealed class CreateRecipeHandlerTests
{
    [TestMethod]
    public async Task Handle_NewRecipe_HasZeroCostPerPortionBeforeCascadeRuns()
    {
        // Arrange
        using var db = new AppDbContext(new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString()).Options);

        var userId = Guid.NewGuid();
        var ingredientId = Guid.NewGuid();
        db.Ingredients.Add(new Ingredient { Id = ingredientId, UserId = userId, Name = "Flour", UnitSize = 1000m });
        await db.SaveChangesAsync();

        // Stub cascade — does nothing; isolates handler logic from cascade logic
        var cascadeStub = Substitute.For<ICostCascadeService>();
        cascadeStub.RecalculateForIngredientAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
            .Returns(new CascadeResult(0, 0));

        var handler = new CreateRecipeHandler(db, cascadeStub);
        var command = new CreateRecipeCommand(
            userId, "My Brownie Box", 12, 0.30m, 0.35m, "Standard",
            [new CreateRecipeItemDto(ingredientId, 500m, 1.0m)]);

        // Act
        var result = await handler.Handle(command, CancellationToken.None);

        // Assert — CostPerPortion starts at 0; cascade (not the handler) sets the real value
        Assert.IsFalse(result.IsError);
        Assert.AreEqual("My Brownie Box", result.Value.Name);
        Assert.AreEqual(0m, result.Value.CostPerPortion,
            "CostPerPortion must be 0 before cascade runs — handler never computes it directly.");
    }
}
```

**File:** `tests/RecipeCost.Application.Tests/Features/Recipes/DerivedSellPriceTests.cs`

```csharp
namespace RecipeCost.Application.Tests.Features.Recipes;

// Tests the sell price formula in isolation — no DB, no handlers, pure arithmetic
[TestClass]
public sealed class DerivedSellPriceTests
{
    [TestMethod]
    [DataRow(5.00, 0.50, 0.40, 9.17)]  // (5.00+0.50)/(1-0.40) = 9.1̅ → 9.17
    [DataRow(3.20, 0.45, 0.30, 5.21)]  // (3.20+0.45)/(1-0.30) = 5.214... → 5.21
    [DataRow(2.00, 0.00, 0.50, 4.00)]  // (2.00+0.00)/(1-0.50) = 4.00
    [DataRow(5.00, 0.50, 0.00, 5.50)]  // 0% margin → sell price = cost + packaging
    public void DerivedSellPrice_Formula_MatchesExpected(
        double costPerPortion, double packagingCost, double targetMargin, double expectedSellPrice)
    {
        // Arrange
        decimal cost = (decimal)costPerPortion;
        decimal packaging = (decimal)packagingCost;
        decimal margin = (decimal)targetMargin;

        // Act — replica of the formula used in all recipe query handlers
        decimal? derivedSellPrice = margin < 1m
            ? (cost + packaging) / (1m - margin)
            : null;

        // Assert
        Assert.IsNotNull(derivedSellPrice);
        Assert.AreEqual((decimal)expectedSellPrice, Math.Round(derivedSellPrice.Value, 2));
    }

    [TestMethod]
    public void DerivedSellPrice_WhenTargetMarginIsOne_ReturnsNull()
    {
        // Arrange — 100% margin would cause division by zero
        decimal cost = 5.00m;
        decimal packaging = 0.50m;
        decimal margin = 1.0m;

        // Act
        decimal? derivedSellPrice = margin < 1m
            ? (cost + packaging) / (1m - margin)
            : null;

        // Assert — formula returns null, never throws
        Assert.IsNull(derivedSellPrice);
    }
}
```

> **Key principles:**
> - **Sealed test classes** — `sealed` is required per MSTest 3.x best practices
> - **`[DataRow]` for formula tests** — same formula, different inputs: data-driven over copy-paste tests
> - **Assert on values** — check `CostPerPortion`, `DerivedSellPrice`, and recipe scoping; never assert on cascade mock invocation counts
> - **Isolate formula tests** — `DerivedSellPriceTests` has no DB dependency; it's fast and deterministic
> - **One In-Memory DB per test** — `Guid.NewGuid().ToString()` as database name prevents inter-test state pollution

---

## 16. Phase 2 Capstone — Final `Program.cs`

After completing L6, L7, and L8, your `Program.cs` should look like this. Every service registration, middleware, and endpoint map accumulated across L3–L8 is assembled here.

**File:** `src/RecipeCost.API/Program.cs`

```csharp
using RecipeCost.Application;
using RecipeCost.Infrastructure;
using RecipeCost.API.Endpoints;
using RecipeCost.API.Middleware;

var builder = WebApplication.CreateBuilder(args);

// ──────────────────────────────────────────────
// DI Registrations (accumulated across L3–L8)
// ──────────────────────────────────────────────

// L3: MediatR + all IRequestHandler implementations
// L4: ValidationBehavior pipeline + FluentValidation scanner
// L7: ICostCascadeService
builder.Services.AddApplication();

// L2: AppDbContext + Npgsql
// L5: IPasswordHasher → BcryptPasswordHasher, ITokenService → JwtTokenService, IEmailService → ConsoleEmailService
// L7: IAlertDispatcher → ConsoleAlertDispatcher
builder.Services.AddInfrastructure(builder.Configuration);

// L4: Registers GlobalExceptionHandler (IExceptionHandler)
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// L5: JWT Authentication + Authorization policies
builder.Services.AddAuthentication(Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // Loaded from AddInfrastructure — see L5 for full TokenValidationParameters setup
    });

// v1: no role-based policies — single authenticated user uses RequireAuthorization() only
// v2-only: C-8 role policies (OwnerOnly, OwnerOrChef, etc.) will be re-introduced in v2 when multi-user roles are added
builder.Services.AddAuthorization();

var app = builder.Build();

// ──────────────────────────────────────────────
// Middleware pipeline (ORDER MATTERS — see L5)
// ──────────────────────────────────────────────

app.UseExceptionHandler();          // L4: Global exception handler (must be first)
app.UseAuthentication();            // L5: Reads JWT from Authorization header
app.UseAuthorization();             // L5: Enforces RequireAuthorization() on endpoints

// ──────────────────────────────────────────────
// Endpoints (accumulated across L3, L6, L8)
// ──────────────────────────────────────────────

app.MapGet("/health", () => Results.Ok(new { status = "healthy", phase = "Phase 2 complete" }));

app.MapAuthEndpoints();             // L5: /api/auth/register, /login, /verify-email
app.MapOutletEndpoints();           // L3+L5: /api/outlets — ⚠️ v2-only: skip in v1
app.MapIngredientEndpoints();       // L6: /api/ingredients + /prices
app.MapRecipeEndpoints();           // L8: /api/recipes + /items + /versions

app.Run();
```

> **Note:** `AddJwtBearer` options are defined inside `AddInfrastructure()` (see L5's `Infrastructure/DependencyInjection.cs`) — they are not duplicated here. `Program.cs` only calls the extension methods; each layer owns its own configuration.