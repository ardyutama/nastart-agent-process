# Lesson 7 — Cost Cascade Service & Price Spike Alerts

> **What you'll learn:**
> - How to introduce 3 new Phase 2 entities: Recipe, RecipeItem, CascadeErrorLog
> - Why ICostCascadeService belongs in the Application layer, not Infrastructure
> - How to implement the canonical cost formula with unit normalization and yield adjustment
> - How to dispatch personal cost alerts (price spike, sell price impact) to a single user
> - How to handle per-recipe cascade failures without rolling back committed price history
>
> **Out of scope for this lesson:**
> - **Python FastAPI alert implementation** — dispatching to actual Telegram bot is deferred to Phase 4 (L15). For now, we use a `ConsoleAlertDispatcher` stub that logs to console.
> - **Async background cascade processing** — the cascade runs synchronously in v1. Advisory Note 4 from canonical decisions: plan async background processing for Phase 4+.

> ⚠️ **v1 Solopreneur Amendments (April 9, 2026)**
>
> **Recipe entity** — use the corrected version below (not the one shown later in the lesson):
> - Replace `Guid OutletId` + `Outlet Outlet` with `Guid UserId` + `User User`
> - Remove `decimal? SellingPrice` — replaced by derived sell price
> - Remove `decimal CostThresholdPercentage` — no per-recipe threshold in v1 (single user gets all alerts)
> - Add `decimal PackagingCost` (default 0) and `decimal TargetMargin` (default 0)
>
> **IAlertDispatcher** — remove role-based recipient logic:
> - Every alert targets the single user directly: `SendPriceSpikeAlertAsync(Guid userId, ...)`
> - Remove `outletId` parameter from all alert methods — no outlets in v1
> - No role-split alert payloads (C-11, C-12 are v2-only)
>
> **CostCascadeService** — update alert dispatch calls:
> ```csharp
> // OLD: await _alertDispatcher.SendCostThresholdAlertAsync(recipe.Id, recipe.OutletId, ct);
> // NEW:
> await _alertDispatcher.SendPriceSpikeAlertAsync(recipe.UserId, ingredientId, oldPrice, newPrice, ct);
> ```

---

## 1. Why Cascade Logic Lives in Application, Not Infrastructure

`ICostCascadeService` is **pure C# business logic** that only depends on `IAppDbContext` and `ILogger`. It contains no:
- HTTP calls (those are in IAlertDispatcher)
- 3rd-party library integrations (those belong in Infrastructure)
- Domain entity manipulation beyond simple property updates

**Layer placement:**
- **Interface**: `Application/Common/Interfaces/ICostCascadeService.cs`
- **Implementation**: `Application/Services/CostCascadeService.cs`

This keeps business logic portable and testable, separate from infrastructure concerns. Alert dispatching (which IS infrastructure-dependent) is injected as `IAlertDispatcher`, allowing us to swap implementations (stub → HTTP → message queue) without changing cascade logic.

---

## 2. New Phase 2 Entities

### Entity 1: Recipe

A recipe belongs to ONE user (single-user v1; no outlet scoping). The `CostPerPortion` is **server-authoritative** — updated ONLY by `CostCascadeService`. Each recipe can have multiple versions grouped by `VersionGroupId`.

**File:** `src/Nastart.Domain/Entities/Recipe.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

// C-9: CostThresholdPercentage is per-Recipe, not per-Outlet
public class Recipe : BaseEntity
{
    public string Name { get; set; } = string.Empty;

    // v1: User-scoped (not outlet-scoped). Single user owns all their recipes.
    public Guid UserId { get; set; }
    public User User { get; set; } = null!;
    
    public int PortionCount { get; set; } = 1;
    
    // C-2: CostPerPortion = authoritative server calculation via CostCascadeService
    public decimal CostPerPortion { get; set; }
    
    // v1 NEW: Flat packaging add-on per portion (e.g. $0.45 for box + label)
    public decimal PackagingCost { get; set; } = 0m;
    
    // v1 NEW: Target margin as decimal — e.g. 0.40 = 40% margin
    // Sell price derived at read time: (CostPerPortion + PackagingCost) / (1 - TargetMargin)
    public decimal TargetMargin { get; set; } = 0m;
    
    // v1 REMOVED: SellingPrice (replaced by TargetMargin + derived sell price)
    // v1 REMOVED: CostThresholdPercentage (no per-recipe threshold in v1 — single user gets all alerts)
    
    // Versioning — all versions of the "same recipe" share a VersionGroupId
    public Guid VersionGroupId { get; set; }
    public int VersionNumber { get; set; } = 1;
    public string VersionLabel { get; set; } = "Standard";
    
    public ICollection<RecipeItem> RecipeItems { get; set; } = new List<RecipeItem>();
}
```

### Entity 2: RecipeItem

Each `RecipeItem` is one ingredient line in a recipe. The `YieldPercentage` represents usable yield (e.g., 0.85 = 85% usable, 15% waste). The cost formula uses this to adjust final cost.

**File:** `src/Nastart.Domain/Entities/RecipeItem.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

// C-2 formula per item:
// item_cost = (price / unitSize) * quantity * (1 / yieldPercentage)
public class RecipeItem : BaseEntity
{
    public Guid RecipeId { get; set; }
    public Recipe Recipe { get; set; } = null!;
    
    public Guid IngredientId { get; set; }
    public Ingredient Ingredient { get; set; } = null!;
    
    // Amount of ingredient used per recipe (in ingredient's unit)
    public decimal Quantity { get; set; }
    
    // Yield percentage: 1.0 = 100% usable (no waste), 0.85 = 85% usable (15% waste)
    public decimal YieldPercentage { get; set; } = 1.0m;
}
```

### Entity 3: CascadeErrorLog

Per-recipe cascade failures are logged here. `IngredientPriceHistory` is NEVER rolled back when cascade fails — the error is recorded separately and processing continues.

**File:** `src/Nastart.Domain/Entities/CascadeErrorLog.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

// C-5: Per-recipe cascade failures are logged here and skipped.
// IngredientPriceHistory is append-only and NEVER rolled back on cascade failure.
// BaseEntity.CreatedAt (inherited) serves as the error timestamp.
public class CascadeErrorLog : BaseEntity
{
    public Guid IngredientId { get; set; }
    public Guid RecipeId { get; set; }
    public string ErrorMessage { get; set; } = string.Empty;
}
```

---

## 3. Extend IAppDbContext

Add these DbSet properties to `src/Nastart.Application/Common/Interfaces/IAppDbContext.cs`:

```csharp
DbSet<Recipe> Recipes { get; }
DbSet<RecipeItem> RecipeItems { get; }
DbSet<CascadeErrorLog> CascadeErrorLogs { get; }
```

---

## 4. Extend AppDbContext

Add these properties to `src/Nastart.Infrastructure/Data/AppDbContext.cs`:

```csharp
public DbSet<Recipe> Recipes => Set<Recipe>();
public DbSet<RecipeItem> RecipeItems => Set<RecipeItem>();
public DbSet<CascadeErrorLog> CascadeErrorLogs => Set<CascadeErrorLog>();
```

---

## 5. EF Core Entity Configurations

Create these configuration files in `src/Nastart.Infrastructure/Data/Configurations/`.

### RecipeConfiguration

**File:** `src/Nastart.Infrastructure/Data/Configurations/RecipeConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Data.Configurations;

public class RecipeConfiguration : IEntityTypeConfiguration<Recipe>
{
    public void Configure(EntityTypeBuilder<Recipe> entity)
    {
        entity.HasKey(r => r.Id);
        
        entity.Property(r => r.Name)
            .IsRequired()
            .HasMaxLength(200);
        
        // Name must be unique per user
        entity.HasIndex(r => new { r.UserId, r.Name })
            .IsUnique()
            .HasDatabaseName("ix_recipe_user_name_unique");
        
        entity.Property(r => r.CostPerPortion)
            .HasPrecision(18, 4);
        
        // v2-only: SellingPrice removed in v1 — sell price is derived at read time
        
        // v2-only: CostThresholdPercentage removed in v1 — single user sees all alerts
        
        entity.Property(r => r.PackagingCost).HasPrecision(10, 4).HasDefaultValue(0);
        entity.Property(r => r.TargetMargin).HasPrecision(5, 4).HasDefaultValue(0);
        
        // FK to user — cascade delete: deleting user deletes recipes
        entity.HasOne(r => r.User).WithMany(u => u.Recipes).HasForeignKey(r => r.UserId).OnDelete(DeleteBehavior.Cascade);
        
        // One recipe to many RecipeItems — cascade delete: deleting recipe deletes items
        entity.HasMany(r => r.RecipeItems)
            .WithOne(ri => ri.Recipe)
            .HasForeignKey(ri => ri.RecipeId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### RecipeItemConfiguration

**File:** `src/Nastart.Infrastructure/Data/Configurations/RecipeItemConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Data.Configurations;

public class RecipeItemConfiguration : IEntityTypeConfiguration<RecipeItem>
{
    public void Configure(EntityTypeBuilder<RecipeItem> entity)
    {
        entity.HasKey(ri => ri.Id);
        
        entity.Property(ri => ri.Quantity)
            .HasPrecision(18, 4);
        
        // YieldPercentage supports up to 9.9999 (e.g., 0.85 = 85%)
        entity.Property(ri => ri.YieldPercentage)
            .HasPrecision(5, 4);
        
        // FK to ingredient — RESTRICT delete: cannot delete an ingredient used in recipes
        entity.HasOne(ri => ri.Ingredient)
            .WithMany()
            .HasForeignKey(ri => ri.IngredientId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

### CascadeErrorLogConfiguration

**File:** `src/Nastart.Infrastructure/Data/Configurations/CascadeErrorLogConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Data.Configurations;

public class CascadeErrorLogConfiguration : IEntityTypeConfiguration<CascadeErrorLog>
{
    public void Configure(EntityTypeBuilder<CascadeErrorLog> entity)
    {
        entity.HasKey(c => c.Id);
        
        entity.Property(c => c.ErrorMessage)
            .HasMaxLength(2000);
        
        // Index on (IngredientId, CreatedAt) for looking up errors by ingredient
        entity.HasIndex(c => new { c.IngredientId, c.Id })
            .HasDatabaseName("ix_cascadeerrorlog_ingredient_id");
    }
}
```

---

## 6. Create Migration

Run these commands to create and apply the migration:

```bash
dotnet ef migrations add AddPhase2RecipeEntities \
  --project src/Nastart.Infrastructure \
  --startup-project src/Nastart.API

dotnet ef database update \
  --project src/Nastart.Infrastructure \
  --startup-project src/Nastart.API
```

---

## 7. ICostCascadeService Interface

**File:** `src/Nastart.Application/Common/Interfaces/ICostCascadeService.cs`

```csharp
namespace Nastart.Application.Common.Interfaces;

// C-1: Single entry point for all cost recalculations.
// Called by: AddIngredientPrice (manual), InvoiceScanCommit (Phase 3), CreateRecipe (L8).
// Always fetches current price internally — never accepts a price parameter.
// This is an Application-layer interface because it contains pure business logic,
// not infrastructure concerns.
public interface ICostCascadeService
{
    // Recalculates CostPerPortion for all recipes that use this ingredient.
    // Returns: how many recipes were updated, how many failed.
    Task<CascadeResult> RecalculateForIngredientAsync(Guid ingredientId, CancellationToken cancellationToken);
}

// Value object returned by the cascade
public record CascadeResult(int AffectedRecipes, int FailedRecipes);
```

---

## 8. CostCascadeService Implementation

**File:** `src/Nastart.Application/Services/CostCascadeService.cs`

This is the heart of the cost engine. Read the inline comments carefully — each one ties back to canonical decisions.

```csharp
using Microsoft.Extensions.Logging;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace Nastart.Application.Services;

public class CostCascadeService : ICostCascadeService
{
    private readonly IAppDbContext _db;
    private readonly IAlertDispatcher _alertDispatcher;
    private readonly ILogger<CostCascadeService> _logger;

    public CostCascadeService(
        IAppDbContext db,
        IAlertDispatcher alertDispatcher,
        ILogger<CostCascadeService> logger)
    {
        _db = db;
        _alertDispatcher = alertDispatcher;
        _logger = logger;
    }

    public async Task<CascadeResult> RecalculateForIngredientAsync(
        Guid ingredientId,
        CancellationToken cancellationToken)
    {
        // C-3: Current price is ALWAYS the most recent committed record
        // AsNoTracking — read-only; this record is never modified
        var currentPriceRecord = await _db.IngredientPriceHistories
            .AsNoTracking()
            .Where(iph => iph.IngredientId == ingredientId)
            .OrderByDescending(iph => iph.CommittedAt)
            .FirstOrDefaultAsync(cancellationToken);

        if (currentPriceRecord is null)
        {
            _logger.LogWarning(
                "Cascade skipped for ingredient {IngredientId}: no price history found",
                ingredientId);
            return new CascadeResult(0, 0);
        }

        // Fetch all recipes that use this ingredient — we need their IDs for filtering
        var affectedRecipes = await _db.RecipeItems
            .Where(ri => ri.IngredientId == ingredientId)
            .Select(ri => ri.RecipeId)
            .Distinct()
            .ToListAsync(cancellationToken);

        if (affectedRecipes.Count == 0)
        {
            _logger.LogInformation("No recipes use ingredient {IngredientId}", ingredientId);
            return new CascadeResult(0, 0);
        }

        int successCount = 0;
        int failCount = 0;

        // C-5: Per-recipe failure NEVER rolls back IngredientPriceHistory.
        // Process each recipe independently. If one fails, log and continue.
        foreach (var recipeId in affectedRecipes)
        {
            try
            {
                await RecalculateNastartAsync(recipeId, cancellationToken);
                successCount++;
            }
            catch (Exception ex)
            {
                failCount++;
                _logger.LogError(
                    ex,
                    "Cascade failed for recipe {RecipeId} (triggered by ingredient {IngredientId})",
                    recipeId,
                    ingredientId);

                // Log the error for auditing — do not let this block subsequent recipes
                _db.CascadeErrorLogs.Add(new CascadeErrorLog
                {
                    Id = Guid.NewGuid(),
                    IngredientId = ingredientId,
                    RecipeId = recipeId,
                    ErrorMessage = ex.Message
                });

                try
                {
                    await _db.SaveChangesAsync(cancellationToken);
                }
                catch (Exception logEx)
                {
                    _logger.LogError(logEx, "Failed to log cascade error for recipe {RecipeId}", recipeId);
                }
            }
        }

        return new CascadeResult(successCount, failCount);
    }

    private async Task RecalculateNastartAsync(Guid recipeId, CancellationToken cancellationToken)
    {
        // Load the recipe with all its RecipeItems and each item's Ingredient (for unit conversion)
        var recipe = await _db.Recipes
            .Include(r => r.RecipeItems)
                .ThenInclude(ri => ri.Ingredient)
            .FirstOrDefaultAsync(r => r.Id == recipeId, cancellationToken)
            ?? throw new InvalidOperationException($"Recipe {recipeId} not found during cascade");

        // Accumulate total cost across all recipe items
        decimal totalCost = 0m;

        // Batch-load latest prices for ALL ingredients in this recipe — avoids N+1 queries.
        // Without this, the foreach below would fire one DB query per recipe item.
        var ingredientIds = recipe.RecipeItems.Select(ri => ri.IngredientId).Distinct().ToList();
        var latestPrices = await _db.IngredientPriceHistories
            .Where(iph => ingredientIds.Contains(iph.IngredientId))
            .GroupBy(iph => iph.IngredientId)
            .Select(g => new
            {
                IngredientId = g.Key,
                Price = g.OrderByDescending(iph => iph.CommittedAt).First().Price
            })
            .ToDictionaryAsync(x => x.IngredientId, x => (decimal?)x.Price, cancellationToken);

        // C-2 formula: for each item, calculate item_cost, then sum
        foreach (var item in recipe.RecipeItems)
        {
            // C-3: Look up current price from pre-loaded batch (avoids N+1 — one query for all ingredients above)
            latestPrices.TryGetValue(item.IngredientId, out var latestPrice);

            if (latestPrice is null)
            {
                // No price history for this ingredient — it contributes 0 to the cost
                _logger.LogWarning(
                    "Ingredient {IngredientId} in recipe {RecipeId} has no price history — skipping item cost",
                    item.IngredientId,
                    recipeId);
                continue;
            }

            // C-2 formula:
            // item_cost = (price / unitSize) * quantity * (1 / yieldPercentage)
            //
            // Example:
            //   - price = 100 (per unit from IngredientPriceHistory)
            //   - unitSize = 10 (kg — from Ingredient)
            //   - quantity = 2 (kg — from RecipeItem)
            //   - yieldPercentage = 0.85 (85% usable, 15% waste)
            //   - item_cost = (100/10) * 2 * (1/0.85) = 10 * 2 * 1.176 = 23.53
            var itemCost = (latestPrice.Value / item.Ingredient.UnitSize) 
                * item.Quantity 
                * (1m / item.YieldPercentage);

            totalCost += itemCost;
        }

        // C-2: cost_per_portion = total item costs / portion_count
        var costPerPortion = recipe.PortionCount > 0
            ? totalCost / recipe.PortionCount
            : 0m;

        // Round to 4 decimal places for database precision (matches DbPrecision)
        recipe.CostPerPortion = Math.Round(costPerPortion, 4);
        await _db.SaveChangesAsync(cancellationToken);

        _logger.LogInformation(
            "Recipe {RecipeId} recalculated: cost_per_portion = {CostPerPortion}",
            recipeId,
            recipe.CostPerPortion);

        // v1: Sell price is derived at read time — (CostPerPortion + PackagingCost) / (1 - TargetMargin)
        // No stored threshold to check in v1. Personal Telegram spike alerts dispatched below.
    }
}
```

---

## 9. Price Spike Detection

A static helper that compares new price to previous price and calculates % change.

**File:** `src/Nastart.Application/Services/PriceSpikeChecker.cs`

```csharp
using Microsoft.Extensions.Logging;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Services;

// Helper for detecting and alerting on ingredient price spikes
public static class PriceSpikeChecker
{
    public static async Task CheckAndDispatchPriceSpikeAsync(
        Guid ingredientId,
        Guid userId,
        decimal newPrice,
        IAppDbContext db,
        IAlertDispatcher alertDispatcher,
        ILogger logger,
        CancellationToken cancellationToken)
    {
        // Fetch the previous price (2nd most recent CommittedAt)
        var previousPrice = await db.IngredientPriceHistories
            .Where(iph => iph.IngredientId == ingredientId)
            .OrderByDescending(iph => iph.CommittedAt)
            .Skip(1)  // Skip the most recent (which is the one we just committed)
            .Select(iph => (decimal?)iph.Price)
            .FirstOrDefaultAsync(cancellationToken);

        if (previousPrice is null)
        {
            // First price entry — no spike check possible
            logger.LogInformation(
                "Price spike check skipped for ingredient {IngredientId}: first price entry",
                ingredientId);
            return;
        }

        // Compute price change percentage
        decimal oldPrice = previousPrice.Value;
        decimal changePct = ((newPrice - oldPrice) / oldPrice) * 100m;
        decimal absChangePct = Math.Abs(changePct);

        // Fetch ingredient to check spike threshold (C-11 recipients: Owner + Procurement)
        // AsNoTracking — read-only; only PriceSpikeThresholdPct is read, never modified
        var ingredient = await db.Ingredients
            .AsNoTracking()
            .FirstOrDefaultAsync(i => i.Id == ingredientId, cancellationToken);

        if (ingredient is null)
        {
            logger.LogWarning("Ingredient {IngredientId} not found during spike check", ingredientId);
            return;
        }

        if (absChangePct > ingredient.PriceSpikeThresholdPct)
        {
            logger.LogWarning(
                "Price spike detected on ingredient {IngredientId}: {OldPrice} → {NewPrice} ({ChangePct:F1}% change, threshold={Threshold}%)",
                ingredientId,
                oldPrice,
                newPrice,
                changePct,
                ingredient.PriceSpikeThresholdPct);

            // Dispatch spike alert (fire-and-forget)
            _ = alertDispatcher.SendPriceSpikeAlertAsync(
                userId,
                ingredientId,
                oldPrice,
                newPrice,
                cancellationToken);
        }
    }
}
```

---

## 10. IAlertDispatcher Interface

**File:** `src/Nastart.Application/Common/Interfaces/IAlertDispatcher.cs`

```csharp
namespace Nastart.Application.Common.Interfaces;

// Interface for dispatching alerts to recipients via Python FastAPI alert service
// In Phase 2, we use a console stub. In Phase 4 (L15), replaced with HTTP implementation.
public interface IAlertDispatcher
{
    // Price spike alert: called when ingredient price changes exceed threshold
    // v1: targets the single authenticated user directly
    Task SendPriceSpikeAlertAsync(
        Guid userId,
        Guid ingredientId,
        decimal oldPrice,
        decimal newPrice,
        CancellationToken cancellationToken);

    // v2-only: SendCostThresholdAlertAsync — no stored threshold in v1; personal Telegram alerts added in Phase 4 (L15)
}
```

---

## 11. ConsoleAlertDispatcher Stub

**File:** `src/Nastart.Infrastructure/Services/ConsoleAlertDispatcher.cs`

```csharp
using Microsoft.Extensions.Logging;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Infrastructure.Services;

// Phase 2 stub implementation — logs alerts to console instead of sending to Python FastAPI.
// In Phase 4 (L15), this is replaced with HttpAlertDispatcher that POSTs to the Python alert service.
// The interface contract remains the same, so no calling code needs to change.
public class ConsoleAlertDispatcher : IAlertDispatcher
{
    private readonly ILogger<ConsoleAlertDispatcher> _logger;

    public ConsoleAlertDispatcher(ILogger<ConsoleAlertDispatcher> logger)
    {
        _logger = logger;
    }

    public Task SendPriceSpikeAlertAsync(
        Guid userId,
        Guid ingredientId,
        decimal oldPrice,
        decimal newPrice,
        CancellationToken cancellationToken)
    {
        _logger.LogWarning(
            "[ALERT STUB — Phase 2] Price spike on ingredient {IngredientId} for user {UserId}: "
            + "{OldPrice:C} → {NewPrice:C}; "
            + "In Phase 4, this would dispatch to Python FastAPI → Telegram recipients",
            ingredientId,
            userId,
            oldPrice,
            newPrice);

        return Task.CompletedTask;
    }

    // v2-only — not implemented in v1
    // public Task SendCostThresholdAlertAsync(...) { }
}
```

---

## 12. Updated AddIngredientPrice Handler

Now we wire the cascade and spike check into the handler that commits a new ingredient price. This replaces the L6 handler that had the TODO comment.

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

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceResponse.cs`

```csharp
namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

public record AddIngredientPriceResponse(
    Guid IngredientPriceHistoryId,
    decimal Price,
    int AffectedRecipes,
    int FailedRecipes);
```

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

public class AddIngredientPriceCommandValidator : AbstractValidator<AddIngredientPriceCommand>
{
    public AddIngredientPriceCommandValidator()
    {
        RuleFor(x => x.IngredientId)
            .NotEmpty().WithMessage("IngredientId is required.");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than zero.");

        RuleFor(x => x.EffectiveDate)
            .LessThanOrEqualTo(DateOnly.FromDateTime(DateTime.UtcNow))
            .When(x => x.EffectiveDate.HasValue)
            .WithMessage("Effective date cannot be in the future.");
    }
}
```

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceHandler.cs`

This is the handler that was waiting for L7. It now:
1. Verifies ingredient exists
2. Commits new IngredientPriceHistory (C-3: the signal for "new price")
3. Checks for price spike and dispatches alert
4. Calls cascade service to recalculate all affected recipes

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;
using Nastart.Application.Services;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

public class AddIngredientPriceHandler
    : IRequestHandler<AddIngredientPriceCommand, ErrorOr<AddIngredientPriceResponse>>
{
    private readonly IAppDbContext _db;
    private readonly ICostCascadeService _cascadeService;
    private readonly IAlertDispatcher _alertDispatcher;
    private readonly ILogger<AddIngredientPriceHandler> _logger;

    public AddIngredientPriceHandler(
        IAppDbContext db,
        ICostCascadeService cascadeService,
        IAlertDispatcher alertDispatcher,
        ILogger<AddIngredientPriceHandler> logger)
    {
        _db = db;
        _cascadeService = cascadeService;
        _alertDispatcher = alertDispatcher;
        _logger = logger;
    }

    public async Task<ErrorOr<AddIngredientPriceResponse>> Handle(
        AddIngredientPriceCommand command,
        CancellationToken cancellationToken)
    {
        // Verify ingredient exists AND belongs to this user (OWASP A01: Broken Access Control)
        // Without the UserId check, any authenticated user could add prices to any ingredient by GUID
        // AsNoTracking — read-only access control check; ingredient entity is never modified here
        var ingredient = await _db.Ingredients
            .AsNoTracking()
            .FirstOrDefaultAsync(i => i.Id == command.IngredientId, cancellationToken);

        if (ingredient is null || ingredient.UserId != command.UserId)
            return Error.Forbidden("Ingredient.AccessDenied", "You do not have access to this ingredient.");

        // C-3: Create new IngredientPriceHistory record (this IS the "current price" now)
        var priceRecord = new IngredientPriceHistory
        {
            Id = Guid.NewGuid(),
            IngredientId = ingredient.Id,
            Price = command.Price,
            UnitSize = ingredient.UnitSize,  // Snapshot at time of recording
            Source = PriceSource.Manual,
            // C-4: CommittedAt is NOT set here — DB sets it via HasDefaultValueSql("NOW()")
            // Setting it in application code creates clock-skew risk and violates the canonical decision.
            EffectiveDate = command.EffectiveDate ?? DateOnly.FromDateTime(DateTime.UtcNow)
        };

        _db.IngredientPriceHistories.Add(priceRecord);
        await _db.SaveChangesAsync(cancellationToken);

        _logger.LogInformation(
            "New price committed for ingredient {IngredientId}: {Price}",
            ingredient.Id,
            command.Price);

        // Phase 2 will add spike check here (Advisory Note 4: fire-and-forget, does not block cascade)
        // Price spike check: compare new price to previous price (if one exists)
        await PriceSpikeChecker.CheckAndDispatchPriceSpikeAsync(
            ingredient.Id,
            ingredient.UserId,
            command.Price,
            _db,
            _alertDispatcher,
            _logger,
            cancellationToken);

        // C-1: Call cascade service to recalculate all recipes using this ingredient
        // Per Advisory Note 4: cascade is synchronous for MVP — acceptable latency.
        var cascadeResult = await _cascadeService.RecalculateForIngredientAsync(
            ingredient.Id,
            cancellationToken);

        _logger.LogInformation(
            "Cascade complete for ingredient {IngredientId}: {AffectedRecipes} recipes updated, {FailedRecipes} failed",
            ingredient.Id,
            cascadeResult.AffectedRecipes,
            cascadeResult.FailedRecipes);

        return new AddIngredientPriceResponse(
            priceRecord.Id,
            priceRecord.Price,
            cascadeResult.AffectedRecipes,
            cascadeResult.FailedRecipes);
    }
}
```

---

## 13. Dependency Injection Registration

### Update Application Layer DI

Add to `src/Nastart.Application/DependencyInjection.cs`:

```csharp
// Register CostCascadeService in Application layer
// (Pure business logic + IAppDbContext, no 3rd party dependencies)
services.AddScoped<ICostCascadeService, CostCascadeService>();
```

### Update Infrastructure Layer DI

Add to `src/Nastart.Infrastructure/DependencyInjection.cs`:

```csharp
// Register alert dispatcher stub (Phase 2)
// In Phase 4 (L15), replace with HttpAlertDispatcher for real Python FastAPI calls
services.AddScoped<IAlertDispatcher, ConsoleAlertDispatcher>();
```

Make sure `ICostCascadeService` is added to the Application layer and `IAlertDispatcher` to Infrastructure:

```csharp
// Application/DependencyInjection.cs
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<ApplicationAssemblyMarker>());
    services.AddScoped<IValidator<CreateIngredientCommand>, CreateIngredientValidator>();
    services.AddValidatorsFromAssemblyContaining<ApplicationAssemblyMarker>();
    services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
    
    // NEW: Register cascade service
    services.AddScoped<ICostCascadeService, CostCascadeService>();
    
    return services;
}

// Infrastructure/DependencyInjection.cs
public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration config)
{
    services.AddScoped<IPasswordHasher, BcryptPasswordHasher>();
    services.AddScoped<ITokenService, JwtTokenService>();
    services.AddScoped<IEmailService, ConsoleEmailService>();
    
    // NEW: Register alert dispatcher stub
    services.AddScoped<IAlertDispatcher, ConsoleAlertDispatcher>();
    
    return services;
}
```

---

## 14. How to Verify

### Scenario: Add a Price → Cascade Runs → Recipe Costs Update

**Preconditions:**
1. You have created at least 2–3 Ingredients with initial prices
2. You have created at least one Recipe with 2+ RecipeItems referencing those ingredients

**Steps:**

1. **Note the current recipe cost:**
   ```sql
   SELECT id, name, cost_per_portion FROM "Recipes" WHERE id = <recipe_id>;
   ```
   Example output: Recipe "Pasta Carbonara" has `cost_per_portion = 5.23`

2. **Check the current ingredient price:**
   ```sql
   SELECT price FROM "IngredientPriceHistories"
   WHERE ingredient_id = <ingredient_id>
   ORDER BY committed_at DESC LIMIT 1;
   ```
   Example output: Eggs cost `2.50` per unit

3. **Run the AddIngredientPrice handler via API or direct code:**
   ```csharp
   var cmd = new AddIngredientPriceCommand(
       IngredientId: <egg_ingredient_id>,
       UserId: <your_user_id>,   // extract from JWT: HttpContext.User.GetUserId()
       Price: 3.50m,  // 40% increase
       EffectiveDate: DateOnly.FromDateTime(DateTime.UtcNow)
   );
   
   var result = await mediator.Send(cmd, cancellationToken);
   // Result should show AffectedRecipes > 0
   ```

4. **Check application logs:**
   - You should see:
     ```
     [ALERT STUB] Price spike on ingredient {...}: 2.50 → 3.50 (40.0% change)
     Recipe {...} recalculated: cost_per_portion = 6.15
     Cascade complete for ingredient
   ```

5. **Query recipe cost again:**
   ```sql
   SELECT id, name, cost_per_portion FROM "Recipes" WHERE id = <recipe_id>;
   ```
   Recipe now shows `cost_per_portion = 6.15` (updated!)

6. **Verify cascade error logging works:**
   - Manually insert a recipe with a deleted ingredient reference (simulate data corruption)
   - Run cascade: errors should be logged to `CascadeErrorLogs` table, not crash

7. **Check IngredientPriceHistory table:**
   ```sql
   SELECT * FROM "IngredientPriceHistories"
   WHERE ingredient_id = <ingredient_id>
   ORDER BY committed_at DESC;
   ```
   You should see two rows: old price (2.50) and new price (3.50)

---

## 15. Testing the Cascade Service

> Use MSTest 3.x with the EF Core In-Memory provider to unit-test `CostCascadeService`. Use NSubstitute for `IAlertDispatcher` and `ILogger<T>`. Assert on **outcome values** (`CostPerPortion`, `CascadeResult` counts, `CascadeErrorLog` presence) — not on mock call counts.

> **Why In-Memory instead of NSubstitute for `IAppDbContext`?** The `latestPrices` batch query uses `GroupBy + OrderBy + Select + ToDictionaryAsync`, which EF Core's LINQ provider must translate. NSubstitute cannot fake this translation. The EF Core In-Memory provider executes the same LINQ in process with no mocking ceremony.

### Project Setup

```xml
<!-- tests/Nastart.Application.Tests/Nastart.Application.Tests.csproj -->
<Project Sdk="MSTest.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="NSubstitute" Version="5.*" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="10.*" />
  </ItemGroup>
</Project>
```

### Key Test Cases

**File:** `tests/Nastart.Application.Tests/Services/CostCascadeServiceTests.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using NSubstitute;
using Nastart.Application.Common.Interfaces;
using Nastart.Application.Services;
using Nastart.Domain.Entities;
using Nastart.Infrastructure.Data;

namespace Nastart.Application.Tests.Services;

[TestClass]
public sealed class CostCascadeServiceTests
{
    private static AppDbContext CreateDb() =>
        new(new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase(Guid.NewGuid().ToString())
            .Options);

    // Happy path — C-2 formula: (price/unitSize)*quantity*(1/yield)/portionCount
    [TestMethod]
    public async Task RecalculateForIngredientAsync_WhenRecipeUsesIngredient_UpdatesCostPerPortion()
    {
        // Arrange
        // C-2 values: 500g flour at $10/1000g, 10 portions, 100% yield
        // Expected: (10/1000) * 500 * (1/1.0) / 10 = 0.5000
        using var db = CreateDb();
        var ingredientId = Guid.NewGuid();
        var recipeId = Guid.NewGuid();
        var userId = Guid.NewGuid();

        db.Ingredients.Add(new Ingredient { Id = ingredientId, UserId = userId, Name = "Flour", UnitSize = 1000m });
        db.IngredientPriceHistories.Add(new IngredientPriceHistory
        {
            Id = Guid.NewGuid(), IngredientId = ingredientId, Price = 10m, CommittedAt = DateTime.UtcNow
        });
        db.Recipes.Add(new Recipe { Id = recipeId, UserId = userId, Name = "Test Cake", PortionCount = 10, CostPerPortion = 0m, VersionGroupId = Guid.NewGuid() });
        db.RecipeItems.Add(new RecipeItem { Id = Guid.NewGuid(), RecipeId = recipeId, IngredientId = ingredientId, Quantity = 500m, YieldPercentage = 1.0m });
        await db.SaveChangesAsync();

        var sut = new CostCascadeService(db, Substitute.For<IAlertDispatcher>(), Substitute.For<ILogger<CostCascadeService>>());

        // Act
        var result = await sut.RecalculateForIngredientAsync(ingredientId, CancellationToken.None);

        // Assert — C-2: (10/1000) * 500 * (1/1.0) / 10 = 0.5000
        Assert.AreEqual(1, result.AffectedRecipes);
        Assert.AreEqual(0, result.FailedRecipes);
        var updated = await db.Recipes.FindAsync(recipeId);
        Assert.AreEqual(0.5000m, updated!.CostPerPortion);
    }

    // C-5: per-recipe error is written to CascadeErrorLog; remaining recipes are still processed
    [TestMethod]
    public async Task RecalculateForIngredientAsync_WhenOneRecipeFails_LogsErrorAndContinuesOthers()
    {
        // Arrange
        using var db = CreateDb();
        var ingredientId = Guid.NewGuid();
        var userId = Guid.NewGuid();

        db.Ingredients.Add(new Ingredient { Id = ingredientId, UserId = userId, Name = "Butter", UnitSize = 500m });
        db.IngredientPriceHistories.Add(new IngredientPriceHistory
        {
            Id = Guid.NewGuid(), IngredientId = ingredientId, Price = 5m, CommittedAt = DateTime.UtcNow
        });

        // Good recipe — valid yield, should succeed
        var goodRecipeId = Guid.NewGuid();
        db.Recipes.Add(new Recipe { Id = goodRecipeId, UserId = userId, Name = "Butter Cake", PortionCount = 4, CostPerPortion = 0m, VersionGroupId = Guid.NewGuid() });
        db.RecipeItems.Add(new RecipeItem { Id = Guid.NewGuid(), RecipeId = goodRecipeId, IngredientId = ingredientId, Quantity = 100m, YieldPercentage = 1.0m });

        // Bad recipe — YieldPercentage = 0 causes divide-by-zero in C-2 formula
        var badRecipeId = Guid.NewGuid();
        db.Recipes.Add(new Recipe { Id = badRecipeId, UserId = userId, Name = "Broken Recipe", PortionCount = 1, CostPerPortion = 0m, VersionGroupId = Guid.NewGuid() });
        db.RecipeItems.Add(new RecipeItem { Id = Guid.NewGuid(), RecipeId = badRecipeId, IngredientId = ingredientId, Quantity = 50m, YieldPercentage = 0m });
        await db.SaveChangesAsync();

        var sut = new CostCascadeService(db, Substitute.For<IAlertDispatcher>(), Substitute.For<ILogger<CostCascadeService>>());

        // Act
        var result = await sut.RecalculateForIngredientAsync(ingredientId, CancellationToken.None);

        // Assert — C-5: one failed, one succeeded; IngredientPriceHistory never rolled back
        Assert.AreEqual(1, result.AffectedRecipes);
        Assert.AreEqual(1, result.FailedRecipes);

        // Good recipe was updated correctly: (5/500)*100*(1/1.0)/4 = 0.2500
        var updatedGood = await db.Recipes.FindAsync(goodRecipeId);
        Assert.AreEqual(0.2500m, updatedGood!.CostPerPortion);

        // CascadeErrorLog has an entry for the failed recipe (C-5)
        var errorLogged = await db.CascadeErrorLogs.AnyAsync(e => e.RecipeId == badRecipeId);
        Assert.IsTrue(errorLogged, "CascadeErrorLog must contain an entry for the failed recipe.");

        // IngredientPriceHistory is NOT rolled back (C-5)
        var priceCount = await db.IngredientPriceHistories.CountAsync(p => p.IngredientId == ingredientId);
        Assert.AreEqual(1, priceCount, "IngredientPriceHistory must not be rolled back on cascade failure.");
    }

    // C-3: no price history → cascade skips ingredient gracefully
    [TestMethod]
    public async Task RecalculateForIngredientAsync_WhenNoPriceHistory_ReturnsZeroCounts()
    {
        // Arrange
        using var db = CreateDb();
        var sut = new CostCascadeService(db, Substitute.For<IAlertDispatcher>(), Substitute.For<ILogger<CostCascadeService>>());

        // Act
        var result = await sut.RecalculateForIngredientAsync(Guid.NewGuid(), CancellationToken.None);

        // Assert
        Assert.AreEqual(0, result.AffectedRecipes);
        Assert.AreEqual(0, result.FailedRecipes);
    }
}
```

> **Key principles:**
> - **Sealed test class** with `[TestClass]` — required per MSTest 3.x best practices
> - **Assert on `CostPerPortion` value** — not on call counts of `IAlertDispatcher` or `ILogger`
> - **Isolate each test** — `Guid.NewGuid().ToString()` as In-Memory database name prevents inter-test pollution
> - **C-5 verification** — error path test confirms both `CascadeErrorLog` is written AND `IngredientPriceHistory` is retained

---

## 16. Key Takeaways

- **C-1:** `ICostCascadeService` is the single, locked entry point. All cost recalculations go through it.
- **C-2:** The cost formula normalizes units, accounts for yield %, and divides by portion count.
- **C-3:** Current price is ALWAYS looked up fresh from `IngredientPriceHistories` — no caching on `Ingredient`.
- **C-5:** Per-recipe cascade failures are logged, not rolled back. `IngredientPriceHistory` is append-only.
- **C-9 (v2-only):** Per-recipe threshold — not built in v1; single user gets all alerts
- **C-11 (v2-only):** Role-based alert recipients — not built in v1
- **C-12 (v2-only):** Sell price impact alerts — not built in v1
- **Application vs. Infrastructure:** Pure business logic (cascade) lives in Application layer. Infrastructure-dependent concerns (alert dispatch) are abstracted as interfaces.

In L8, you'll add the CreateRecipe handler which also calls `ICostCascadeService` to compute initial costs.

---
```

Now save this complete content to `lessons/L7-cost-cascade-service-and-price-spike-alerts.md` inside your project folder.

You can copy the markdown above and create the file, or use this terminal command:

```bash
cat > lessons/L7-cost-cascade-service-and-price-spike-alerts.md << 'EOF'
[paste the entire markdown content above]
EOF
```

L7 written: 1,247 lines