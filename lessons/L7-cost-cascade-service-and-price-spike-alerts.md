# Lesson 7 — Cost Cascade Service & Price Spike Alerts

> **What you'll learn:**
> - How to introduce 3 new Phase 2 entities: Recipe, RecipeItem, CascadeErrorLog
> - Why ICostCascadeService belongs in the Application layer, not Infrastructure
> - How to implement the canonical cost formula with unit normalization and yield adjustment
> - How to dispatch personal price spike alerts to a single user while preserving sell price impact alerts as v2-only extension points
> - How to handle per-recipe cascade failures without rolling back committed price history
>
> **Out of scope for this lesson:**
> - **Python FastAPI alert implementation** — dispatching to actual Telegram bot is deferred to Phase 4 (L15). For now, we use a `ConsoleAlertDispatcher` stub that logs to console.
> - **Async background cascade processing** — the cascade runs synchronously in v1. Advisory Note 4 from canonical decisions: plan async background processing for Phase 4+.

> **v1 scope notes (already integrated)**
>
> This lesson is written for the **v1 single-user** scope. v2 artifacts (OutletId, CostThresholdPercentage, role-split alert payloads) are preserved as XML `<remarks>` and `// v2-only:` comments throughout so that future migration is straightforward. Nothing in this lesson needs to be "corrected" before use — the v1 design is the authoritative version.
>
> | v2 concept | Where preserved |
> |---|---|
> | `CostThresholdPercentage` (per-Recipe C-9) | `Recipe` XML `<remarks>` + EF config comment |
> | Role-split alert payloads (C-11, C-12) | `IAlertDispatcher` commented methods |
> | `SendCostThresholdAlertAsync` | `IAlertDispatcher` + `ConsoleAlertDispatcher` commented stubs |
> | `PriceSpikeThresholdPct` (per-Ingredient) | `IPriceSpikeChecker` remarks — v1 reads the per-ingredient threshold; `null` or `0` disables alerts |

---

## 1. Why Cascade Logic Lives in Application, Not Infrastructure

`ICostCascadeService` is **pure C# business logic** that only depends on `IAppDbContext` and `ILogger`. It contains no:
- HTTP calls (those are in `IAlertDispatcher`)
- 3rd-party library integrations (those belong in Infrastructure)
- Domain entity manipulation beyond simple property updates

**Layer placement:**

| Class / Interface | Layer | Reason |
|---|---|---|
| `ICostCascadeService` | Application — interface | Pure cost formula; no infrastructure deps |
| `CostCascadeService` | Application — implementation | LINQ + EF Core In-Memory testable; no HTTP |
| `IPriceSpikeChecker` | Application — interface | Spike detection logic; depends on IAppDbContext |
| `PriceSpikeChecker` | Application — implementation | Reads price history, calls IAlertDispatcher |
| `IAlertDispatcher` | Application — interface | Defines the notification contract |
| `ConsoleAlertDispatcher` | **Infrastructure** — implementation | v1 stub; Phase 4 replaces with `HttpAlertDispatcher` |

**SRP split — why `IPriceSpikeChecker` is separate from `ICostCascadeService`:**
Spike detection is NOT part of the cost formula (C-2). Separating it means:
- `CostCascadeService` can be tested without any alert dependency
- The spike threshold logic reads `Ingredient.PriceSpikeThresholdPct`; null or 0 disables alerts
- `AddIngredientPriceHandler` composes both concerns sequentially, not the cascade service itself

---

## 2. New Phase 2 Entities

### Entity 1: Recipe

A recipe belongs to ONE user (single-user v1; no outlet scoping). The `CostPerPortion` is **server-authoritative** — updated ONLY by `CostCascadeService`. Each recipe can have multiple versions grouped by `VersionGroupId`.

**File:** `src/Nastart.Domain/Entities/Recipe.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

/// <summary>
/// Represents a costing recipe belonging to the authenticated user.
/// </summary>
/// <remarks>
/// <para><b>v1:</b> Each recipe is scoped to a single <see cref="UserId"/> — no outlet scoping.
/// <c>CostPerPortion</c> is server-authoritative, updated only by <c>ICostCascadeService</c>,
/// and never accepted from the client.</para>
/// <para>Sell price is derived at read time:
/// <c>(CostPerPortion + PackagingCost) / (1 − TargetMargin)</c> — never stored.</para>
/// <para><b>v2 (planned — C-9):</b> A <c>CostThresholdPercentage</c> field (per-<c>Recipe</c>,
/// not per-<c>Outlet</c>) will enable recipe-level cost deviation alerts. Omitted in v1
/// because a single user receives all spike alerts unconditionally via
/// <c>IAlertDispatcher.SendPriceSpikeAlertAsync</c>.</para>
/// </remarks>
public sealed class Recipe : BaseEntity
{
    /// <summary>Display name of the recipe.</summary>
    public string Name { get; set; } = string.Empty;

    /// <summary>
    /// ID of the user who owns this recipe.
    /// v1: single authenticated user. v2: may be outlet-scoped.
    /// </summary>
    public Guid UserId { get; set; }

    /// <summary>Navigation property to the owning <see cref="User"/>.</summary>
    public User User { get; set; } = null!;

    /// <summary>Number of portions this recipe yields. Divisor in the C-2 cost formula.</summary>
    public int PortionCount { get; set; } = 1;

    /// <summary>
    /// Server-authoritative cost per portion (4 dp precision).
    /// Set only by <c>ICostCascadeService</c> — never accepted from the client.
    /// </summary>
    public decimal CostPerPortion { get; set; }

    /// <summary>
    /// Flat packaging cost added per portion (e.g. box + label).
    /// Included in the derived sell price: <c>(CostPerPortion + PackagingCost) / (1 − TargetMargin)</c>.
    /// </summary>
    public decimal PackagingCost { get; set; } = 0m;

    /// <summary>
    /// Target gross margin as a decimal (e.g. <c>0.40</c> = 40%).
    /// Sell price derived at read time — never stored.
    /// </summary>
    public decimal TargetMargin { get; set; } = 0m;

    // v2-only: CostThresholdPercentage (decimal) — C-9: per-recipe cost deviation threshold
    // v2-only: SellingPrice (decimal)             — replaced by TargetMargin + derived sell price in v1

    /// <summary>Groups all versions of the same recipe concept (e.g. Standard, Summer, Large Batch).</summary>
    public Guid VersionGroupId { get; set; }

    /// <summary>Sequential version number within the <see cref="VersionGroupId"/> group.</summary>
    public int VersionNumber { get; set; } = 1;

    /// <summary>Human-readable version label (e.g. "Standard", "Summer", "Large Batch").</summary>
    public string VersionLabel { get; set; } = "Standard";

    /// <summary>Ingredient line items that make up this recipe.</summary>
    public ICollection<RecipeItem> RecipeItems { get; set; } = new List<RecipeItem>();
}
```

### Entity 2: RecipeItem

Each `RecipeItem` is one ingredient line in a recipe. The `YieldPercentage` represents usable yield (e.g., 0.85 = 85% usable, 15% waste). Package size is not stored on `RecipeItem`; the cascade service uses the latest committed `IngredientPriceHistory.UnitSize` snapshot that belongs to the current price record.

**File:** `src/Nastart.Domain/Entities/RecipeItem.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

/// <summary>
/// Represents a single ingredient line in a <see cref="Recipe"/>.
/// </summary>
/// <remarks>
/// <para><b>C-2 cost formula per item:</b>
/// <c>item_cost = (price / latestPrice.UnitSize) × Quantity × (1 / YieldPercentage)</c></para>
/// <para>The package-size snapshot lives on <c>IngredientPriceHistory.UnitSize</c>, not here.
/// Each price record carries the package size that made that price meaningful.</para>
/// </remarks>
public sealed class RecipeItem : BaseEntity
{
    /// <summary>ID of the recipe this item belongs to.</summary>
    public Guid RecipeId { get; set; }

    /// <summary>Navigation property to the parent <see cref="Recipe"/>.</summary>
    public Recipe Recipe { get; set; } = null!;

    /// <summary>ID of the ingredient used in this line item.</summary>
    public Guid IngredientId { get; set; }

    /// <summary>Navigation property to the <see cref="Ingredient"/>.</summary>
    public Ingredient Ingredient { get; set; } = null!;

    /// <summary>
    /// Amount of the ingredient used per recipe batch (in the ingredient's own unit, e.g. kg, L).
    /// </summary>
    public decimal Quantity { get; set; }

    /// <summary>
    /// Usable yield as a decimal (e.g. <c>0.85</c> = 85% usable, 15% waste).
    /// Applied as the divisor <c>1 / YieldPercentage</c> in the C-2 formula.
    /// </summary>
    public decimal YieldPercentage { get; set; } = 1.0m;

}
```

### Entity 3: CascadeErrorLog

Per-recipe cascade failures are logged here. `IngredientPriceHistory` is NEVER rolled back when cascade fails — the error is recorded separately and processing continues.

**File:** `src/Nastart.Domain/Entities/CascadeErrorLog.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

/// <summary>
/// Immutable audit record for per-recipe cascade failures.
/// </summary>
/// <remarks>
/// <para><b>C-5:</b> When <c>ICostCascadeService</c> fails to recalculate a recipe's cost,
/// the failure is recorded here and processing continues for remaining recipes.
/// <c>IngredientPriceHistory</c> is append-only and is <b>never</b> rolled back on cascade
/// failure.</para>
/// <para>No FK constraints are placed on <see cref="IngredientId"/> or <see cref="RecipeId"/> —
/// this is intentional (see <c>CascadeErrorLogConfiguration</c>). If an ingredient or recipe is
/// subsequently deleted, this record must persist to preserve the incident history.</para>
/// <para><c>BaseEntity.CreatedAt</c> (inherited) serves as the error timestamp.</para>
/// </remarks>
public sealed class CascadeErrorLog : BaseEntity
{
    /// <summary>ID of the ingredient whose price change triggered the failed cascade.</summary>
    public Guid IngredientId { get; set; }

    /// <summary>ID of the recipe whose cost recalculation failed.</summary>
    public Guid RecipeId { get; set; }

    /// <summary>Exception message captured at failure time. Max 2 000 characters.</summary>
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

Add these properties to `src/Nastart.Infrastructure/Persistence/AppDbContext.cs`:

```csharp
public DbSet<Recipe> Recipes => Set<Recipe>();
public DbSet<RecipeItem> RecipeItems => Set<RecipeItem>();
public DbSet<CascadeErrorLog> CascadeErrorLogs => Set<CascadeErrorLog>();
```

---

## 5. EF Core Entity Configurations

Create these configuration files in `src/Nastart.Infrastructure/Persistence/Configurations/`.

### RecipeConfiguration

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/RecipeConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

/// <summary>
/// EF Core Fluent API configuration for the <see cref="Recipe"/> entity.
/// </summary>
/// <remarks>
/// Monetary precision is set to 18,4 for cost fields and 5,4 for ratio fields
/// to match the C-2 formula's 4 dp accuracy.
/// v2-only fields are preserved as comments below to guide the v2 migration addition.
/// </remarks>
public sealed class RecipeConfiguration : IEntityTypeConfiguration<Recipe>
{
    /// <inheritdoc/>
    public void Configure(EntityTypeBuilder<Recipe> entity)
    {
        entity.HasKey(r => r.Id);

        entity.Property(r => r.Name)
            .IsRequired()
            .HasMaxLength(200);

        // Name must be unique per user
        entity.HasIndex(r => new { r.UserId, r.Name })
            .IsUnique();

        entity.Property(r => r.CostPerPortion)
            .HasPrecision(18, 4);

        // v2-only: SellingPrice              — decimal(18,4); sell price is derived at read time in v1
        // v2-only: CostThresholdPercentage   — decimal(5,4); C-9: per-recipe deviation threshold

        entity.Property(r => r.PackagingCost)
            .HasPrecision(10, 4)
            .HasDefaultValue(0m);

        entity.Property(r => r.TargetMargin)
            .HasPrecision(5, 4)
            .HasDefaultValue(0m);

        // P0: Index for fetching the latest version in a recipe group (dashboard, cascade).
        // Query pattern: WHERE version_group_id = @id ORDER BY version_number DESC LIMIT 1
        entity.HasIndex(r => new { r.VersionGroupId, r.VersionNumber })
            .IsDescending(false, true);  // VersionGroupId ASC, VersionNumber DESC

        // P0: Index for listing all recipe groups belonging to a user (recipe list screen).
        // Query pattern: WHERE user_id = @id GROUP BY version_group_id
        entity.HasIndex(r => new { r.UserId, r.VersionGroupId });

        // FK to user — cascade delete: deleting the user deletes all their recipes
        entity.HasOne(r => r.User)
            .WithMany()
            .HasForeignKey(r => r.UserId)
            .OnDelete(DeleteBehavior.Cascade);

        // One recipe to many RecipeItems — cascade delete: deleting a recipe deletes its items
        entity.HasMany(r => r.RecipeItems)
            .WithOne(ri => ri.Recipe)
            .HasForeignKey(ri => ri.RecipeId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

### RecipeItemConfiguration

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/RecipeItemConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

/// <summary>
/// EF Core Fluent API configuration for the <see cref="RecipeItem"/> entity.
/// </summary>
public sealed class RecipeItemConfiguration : IEntityTypeConfiguration<RecipeItem>
{
    /// <inheritdoc/>
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

        // P0: FK index — PostgreSQL does not auto-create indexes on FK columns
        entity.HasIndex(ri => ri.IngredientId);
    }
}
```

### CascadeErrorLogConfiguration

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/CascadeErrorLogConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

/// <summary>
/// EF Core Fluent API configuration for the <see cref="CascadeErrorLog"/> entity.
/// </summary>
/// <remarks>
/// <para><b>C-5:</b> No FK constraints are placed on <c>IngredientId</c> or <c>RecipeId</c>.
/// <c>CascadeErrorLog</c> is an immutable audit record — if an ingredient or recipe is later
/// deleted, this record must persist to preserve the incident history. Bare GUIDs retain the
/// original IDs without referential integrity enforcement.</para>
/// <para>Query performance is maintained via the indexes defined below.</para>
/// </remarks>
public sealed class CascadeErrorLogConfiguration : IEntityTypeConfiguration<CascadeErrorLog>
{
    /// <inheritdoc/>
    public void Configure(EntityTypeBuilder<CascadeErrorLog> entity)
    {
        entity.HasKey(c => c.Id);

        entity.Property(c => c.ErrorMessage)
            .HasMaxLength(2000)
            .IsRequired();

        // C-5: Intentionally NO FK constraints on IngredientId or RecipeId.
        // See XML docs on this class for full rationale.

        // Index for querying all errors for a specific ingredient
        entity.HasIndex(c => c.IngredientId);

        // Composite index: error history for a recipe, newest first (audit view)
        entity.HasIndex(c => new { c.RecipeId, c.CreatedAt })
            .IsDescending(false, true);  // RecipeId ASC, CreatedAt DESC
    }
}
```

---

## 6. Create Migration

Run these commands to create and apply the migration:

```bash
dotnet ef migrations add AddPhase2RecipeEntities --project src/Nastart.Infrastructure --startup-project src/Nastart.Api

dotnet ef database update --project src/Nastart.Infrastructure --startup-project src/Nastart.Api
```

---

## 7. ICostCascadeService Interface

**File:** `src/Nastart.Application/Common/Interfaces/ICostCascadeService.cs`

```csharp
namespace Nastart.Application.Common.Interfaces;

/// <summary>
/// Defines the single entry point for all recipe cost recalculations triggered by
/// ingredient price changes.
/// </summary>
/// <remarks>
/// <para><b>C-1:</b> This is the only code path that may update <c>Recipe.CostPerPortion</c>.
/// The service always fetches the current price internally — a price value is never accepted
/// as a parameter. Called by: <c>AddIngredientPriceHandler</c> (manual entry),
/// <c>InvoiceScanCommitHandler</c> (Phase 3), and any structural mutation handler
/// (quantity, yield, or portion-count changes on recipe items).</para>
/// <para><b>C-3:</b> Current price is derived at cascade time from
/// <c>IngredientPriceHistory ORDER BY committed_at DESC LIMIT 1</c> — never from a stored
/// field on <c>Ingredient</c>.</para>
/// <para><b>C-5:</b> Per-recipe cascade failures are written to <c>CascadeErrorLog</c> and
/// processing continues. <c>IngredientPriceHistory</c> is never rolled back on cascade
/// failure.</para>
/// </remarks>
public interface ICostCascadeService
{
    /// <summary>
    /// Recalculates <c>CostPerPortion</c> for all recipes that use the specified ingredient,
    /// using the ingredient's most recently committed price (C-3).
    /// </summary>
    /// <param name="ingredientId">The ID of the ingredient whose price has just changed.</param>
    /// <param name="cancellationToken">Propagates notification that the operation should be cancelled.</param>
    /// <returns>
    /// A <see cref="CascadeResult"/> containing counts of successfully updated and failed recipes.
    /// </returns>
    Task<CascadeResult> RecalculateForIngredientAsync(
        Guid ingredientId,
        CancellationToken cancellationToken);
}

/// <summary>
/// Immutable result returned by <see cref="ICostCascadeService.RecalculateForIngredientAsync"/>.
/// </summary>
/// <param name="AffectedRecipes">Recipes whose <c>CostPerPortion</c> was successfully updated.</param>
/// <param name="FailedRecipes">
/// Recipes whose recalculation failed. Each failure is persisted to <c>CascadeErrorLog</c>
/// (C-5); processing continued for all remaining recipes.
/// </param>
public record CascadeResult(int AffectedRecipes, int FailedRecipes);
```

> **TDD first:** Before implementing the code in Sections 8-13, add the tests from Section 15, run the targeted failing cases, and then use the snippets below as the smallest change that makes them pass.

---

## 8. CostCascadeService Implementation

**File:** `src/Nastart.Application/Services/CostCascadeService.cs`

This is the heart of the cost engine. Key improvements from baseline:
- **Primary constructor syntax** (C# 12) — explicit field storage removed
- **`sealed`** — no inheritance needed; aids JIT devirtualization
- **`IAlertDispatcher` removed** — alert dispatch is now `IPriceSpikeChecker`'s concern (SRP)
- **`ConfigureAwait(false)`** on every async DB call
- **`RecalculateRecipeAsync`** — renamed from the erroneous `RecalculateNastartAsync`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;

namespace Nastart.Application.Services;

/// <summary>
/// Application-layer implementation of <see cref="ICostCascadeService"/>.
/// Recalculates <c>CostPerPortion</c> for every recipe affected by an ingredient price change.
/// </summary>
/// <remarks>
/// Contains pure C# business logic only — dependencies are limited to
/// <see cref="IAppDbContext"/> and <see cref="ILogger{TCategoryName}"/>.
/// No HTTP calls, no 3rd-party libraries, no infrastructure concerns.
/// See canonical decisions C-1, C-2, C-3, C-5.
/// </remarks>
public sealed class CostCascadeService(
    IAppDbContext db,
    ILogger<CostCascadeService> logger) : ICostCascadeService
{
    /// <inheritdoc/>
    public async Task<CascadeResult> RecalculateForIngredientAsync(
        Guid ingredientId,
        CancellationToken cancellationToken)
    {
        // C-3: Current price is ALWAYS the most recent committed record.
        // AsNoTracking — read-only; this record is never modified.
        var currentPriceRecord = await db.IngredientPriceHistories
            .AsNoTracking()
            .Where(iph => iph.IngredientId == ingredientId)
            .OrderByDescending(iph => iph.CommittedAt)
            .FirstOrDefaultAsync(cancellationToken)
            .ConfigureAwait(false);

        if (currentPriceRecord is null)
        {
            logger.LogWarning(
                "Cascade skipped for ingredient {IngredientId}: no price history found",
                ingredientId);
            return new CascadeResult(0, 0);
        }

        // Fetch all distinct recipe IDs that reference this ingredient
        var affectedRecipeIds = await db.RecipeItems
            .Where(ri => ri.IngredientId == ingredientId)
            .Select(ri => ri.RecipeId)
            .Distinct()
            .ToListAsync(cancellationToken)
            .ConfigureAwait(false);

        if (affectedRecipeIds.Count == 0)
        {
            logger.LogInformation("No recipes use ingredient {IngredientId}", ingredientId);
            return new CascadeResult(0, 0);
        }

        int successCount = 0;
        int failCount = 0;

        // C-5: Per-recipe failure NEVER rolls back IngredientPriceHistory.
        // Process each recipe independently; log and continue on failure.
        foreach (var recipeId in affectedRecipeIds)
        {
            try
            {
                await RecalculateRecipeAsync(recipeId, cancellationToken)
                    .ConfigureAwait(false);
                successCount++;
            }
            catch (Exception ex)
            {
                failCount++;
                logger.LogError(
                    ex,
                    "Cascade failed for recipe {RecipeId} (triggered by ingredient {IngredientId})",
                    recipeId,
                    ingredientId);

                // C-5 sample: attempt to append an audit record and continue.
                // If this DbContext is already faulted, use a fresh context or dedicated
                // audit writer in production-hardening so the log write has its own boundary.
                db.CascadeErrorLogs.Add(new CascadeErrorLog
                {
                    Id = Guid.NewGuid(),
                    IngredientId = ingredientId,
                    RecipeId = recipeId,
                    ErrorMessage = ex.Message
                });

                try
                {
                    await db.SaveChangesAsync(cancellationToken)
                        .ConfigureAwait(false);
                }
                catch (Exception logEx)
                {
                    logger.LogError(
                        logEx,
                        "Failed to write CascadeErrorLog for recipe {RecipeId}",
                        recipeId);
                }
            }
        }

        return new CascadeResult(successCount, failCount);
    }

    // Recalculates CostPerPortion for a single recipe using the C-2 formula.
    // Isolated into a private method so the caller can catch per-recipe exceptions
    // without aborting the entire cascade run.
    private async Task RecalculateRecipeAsync(Guid recipeId, CancellationToken cancellationToken)
    {
        // Load the recipe and its items. Package size comes from the latest
        // IngredientPriceHistory row for each ingredient, not from RecipeItem.
        var recipe = await db.Recipes
            .Include(r => r.RecipeItems)
            .FirstOrDefaultAsync(r => r.Id == recipeId, cancellationToken)
            .ConfigureAwait(false)
            ?? throw new InvalidOperationException(
                $"Recipe {recipeId} not found during cascade");

        // Batch-load the latest price for every ingredient in this recipe — avoids N+1.
        // A single query fetches the most-recent price per ingredient ID.
        var ingredientIds = recipe.RecipeItems
            .Select(ri => ri.IngredientId)
            .Distinct()
            .ToList();

        var latestPrices = await db.IngredientPriceHistories
            .Where(iph => ingredientIds.Contains(iph.IngredientId))
            .GroupBy(iph => iph.IngredientId)
            .Select(g => g.OrderByDescending(iph => iph.CommittedAt)
                .Select(iph => new { iph.IngredientId, iph.Price, iph.UnitSize })
                .First())
            .ToDictionaryAsync(x => x.IngredientId, x => x, cancellationToken)
            .ConfigureAwait(false);

        decimal totalCost = 0m;

        // C-2 formula: item_cost = (price / latestPrice.UnitSize) × Quantity × (1 / YieldPercentage)
        // Every recipe item must have a latest price row. Missing price data is treated as a
        // recipe-level failure so the service never persists a partial CostPerPortion.
        // The price and unit-size snapshot come from the same latest price-history row.
        // Example (100/10) × 2 × (1/0.85) = 23.5294 (rounded to 4dp):
        //   price = 100, latestPrice.UnitSize = 10 kg, Quantity = 2 kg, YieldPercentage = 0.85
        foreach (var item in recipe.RecipeItems)
        {
            if (!latestPrices.TryGetValue(item.IngredientId, out var latestPrice)
                || latestPrice is null)
            {
                throw new InvalidOperationException(
                    $"Ingredient {item.IngredientId} in recipe {recipeId} has no price history.");
            }

            if (latestPrice.UnitSize <= 0m)
                throw new InvalidOperationException(
                    $"Ingredient {item.IngredientId} has invalid unit size in latest price history.");

            var itemCost = (latestPrice.Price / latestPrice.UnitSize)
                * item.Quantity
                * (1m / item.YieldPercentage);

            totalCost += itemCost;
        }

        // C-2: cost_per_portion = SUM(item costs) / portion_count
        // Guard: PortionCount = 0 → clamp to 0 rather than throw DivideByZeroException
        recipe.CostPerPortion = recipe.PortionCount > 0
            ? Math.Round(totalCost / recipe.PortionCount, 4)
            : 0m;

        await db.SaveChangesAsync(cancellationToken)
            .ConfigureAwait(false);

        logger.LogInformation(
            "Recipe {RecipeId} recalculated: cost_per_portion = {CostPerPortion}",
            recipeId,
            recipe.CostPerPortion);

        // v1: Sell price is derived at read time — (CostPerPortion + PackagingCost) / (1 - TargetMargin)
        // No stored cost threshold in v1; spike alerts are dispatched by IPriceSpikeChecker
        // in AddIngredientPriceHandler before cascade is called.

        // P1 — HANDLER RULE: Call cascade on ALL cost-affecting mutations, not just price changes:
        //   - RecipeItem.Quantity changes     - RecipeItem.YieldPercentage changes
        //   - Recipe.PortionCount changes     - A RecipeItem is added or removed
        //
        // In each structural mutation handler:
        //   foreach (var iId in recipe.RecipeItems.Select(ri => ri.IngredientId).Distinct())
        //       await _cascadeService.RecalculateForIngredientAsync(iId, ct);  // C-1: no price param
    }
}
```

> **Durability note:** This sample keeps the recipe recalculation and `CascadeErrorLog` write on the same `IAppDbContext` to keep the lesson focused. That means the audit insert is still best-effort if the current context or database connection is already faulted. If you need stronger durability, persist the error log through a fresh context or dedicated audit writer.

---

## 9. Price Spike Detection — `IPriceSpikeChecker` + `PriceSpikeChecker`

The original design used a `static` helper class — non-injectable and non-mockable. Refactored to an interface + injectable service so it can be tested in isolation and injected into the handler.

**File:** `src/Nastart.Application/Common/Interfaces/IPriceSpikeChecker.cs`

```csharp
namespace Nastart.Application.Common.Interfaces;

/// <summary>
/// Detects ingredient price spikes and dispatches alerts to the authenticated user.
/// </summary>
/// <remarks>
/// Called by <c>AddIngredientPriceHandler</c> immediately after a new
/// <c>IngredientPriceHistory</c> record is committed, before the cascade runs.
/// The alert does not block cascade execution. v1 uses <c>Ingredient.PriceSpikeThresholdPct</c>;
/// null or 0 disables spike alerts for that ingredient.
/// </remarks>
public interface IPriceSpikeChecker
{
    /// <summary>
    /// Compares <paramref name="newPrice"/> to the previous committed price and dispatches
    /// a spike alert via <see cref="IAlertDispatcher"/> if the change exceeds the threshold.
    /// </summary>
    /// <param name="ingredientId">ID of the ingredient whose price just changed.</param>
    /// <param name="userId">ID of the owning user (v1: single authenticated user).</param>
    /// <param name="newPrice">The price that was just committed.</param>
    /// <param name="cancellationToken">Propagates notification that the operation should be cancelled.</param>
    Task CheckAndDispatchAsync(
        Guid ingredientId,
        Guid userId,
        decimal newPrice,
        CancellationToken cancellationToken);
}
```

**File:** `src/Nastart.Application/Services/PriceSpikeChecker.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Services;

/// <summary>
/// Injectable implementation of <see cref="IPriceSpikeChecker"/>.
/// </summary>
/// <remarks>
/// Registered in the Application DI layer alongside <c>ICostCascadeService</c>.
/// </remarks>
public sealed class PriceSpikeChecker(
    IAppDbContext db,
    IAlertDispatcher alertDispatcher,
    ILogger<PriceSpikeChecker> logger) : IPriceSpikeChecker
{
    /// <inheritdoc/>
    public async Task CheckAndDispatchAsync(
        Guid ingredientId,
        Guid userId,
        decimal newPrice,
        CancellationToken cancellationToken)
    {
        var thresholdPct = await db.Ingredients
            .AsNoTracking()
            .Where(i => i.Id == ingredientId && i.UserId == userId)
            .Select(i => i.PriceSpikeThresholdPct)
            .FirstOrDefaultAsync(cancellationToken)
            .ConfigureAwait(false);

        if (thresholdPct is null or <= 0m)
            return;

        // Fetch the previous price (2nd most recent CommittedAt).
        // Skip(1) skips the record we just committed, which is now the most recent.
        var previousPrice = await db.IngredientPriceHistories
            .Where(iph => iph.IngredientId == ingredientId)
            .OrderByDescending(iph => iph.CommittedAt)
            .Skip(1)
            .Select(iph => (decimal?)iph.Price)
            .FirstOrDefaultAsync(cancellationToken)
            .ConfigureAwait(false);

        if (previousPrice is null)
        {
            // First price entry for this ingredient — no baseline to compare against
            logger.LogInformation(
                "Price spike check skipped for ingredient {IngredientId}: first price entry",
                ingredientId);
            return;
        }

        decimal oldPrice = previousPrice.Value;
        decimal changePct = ((newPrice - oldPrice) / oldPrice) * 100m;
        decimal absChangePct = Math.Abs(changePct);

        if (absChangePct <= thresholdPct.Value)
            return;

        logger.LogWarning(
            "Price spike detected on ingredient {IngredientId}: {OldPrice} → {NewPrice} "
            + "({ChangePct:F1}% change, threshold = {Threshold}%)",
            ingredientId,
            oldPrice,
            newPrice,
            changePct,
            thresholdPct.Value);

        // Await the alert with a try-catch so an unexpected dispatcher failure does not
        // propagate to the caller and does not roll back the committed price record (C-5).
        try
        {
            await alertDispatcher.SendPriceSpikeAlertAsync(
                    userId,
                    ingredientId,
                    oldPrice,
                    newPrice,
                    cancellationToken)
                .ConfigureAwait(false);
        }
        catch (Exception ex)
        {
            logger.LogError(
                ex,
                "Failed to dispatch price spike alert for ingredient {IngredientId}",
                ingredientId);
        }
    }
}
```

---

## 10. IAlertDispatcher Interface

**File:** `src/Nastart.Application/Common/Interfaces/IAlertDispatcher.cs`

```csharp
namespace Nastart.Application.Common.Interfaces;

/// <summary>
/// Dispatches operational alerts to the authenticated user via the configured notification channel.
/// </summary>
/// <remarks>
/// <para><b>v1:</b> All alerts target the single authenticated user directly — no outlet scoping,
/// no role-split payloads. The Phase 2 stub (<c>ConsoleAlertDispatcher</c>) logs to console.
/// In Phase 4 (L15), replaced with <c>HttpAlertDispatcher</c> that POSTs to the Python FastAPI
/// alert service. The interface contract is unchanged so no calling code requires modification.</para>
/// <para>v2-only extension points are preserved as commented methods below.</para>
/// </remarks>
public interface IAlertDispatcher
{
    /// <summary>
    /// Dispatches a price spike alert when an ingredient's price changes beyond the configured threshold.
    /// </summary>
    /// <param name="userId">ID of the owning user (v1: single authenticated user).</param>
    /// <param name="ingredientId">ID of the ingredient whose price spiked.</param>
    /// <param name="oldPrice">The most recently committed price before the spike.</param>
    /// <param name="newPrice">The newly committed price that triggered the spike.</param>
    /// <param name="cancellationToken">Propagates notification that the operation should be cancelled.</param>
    Task SendPriceSpikeAlertAsync(
        Guid userId,
        Guid ingredientId,
        decimal oldPrice,
        decimal newPrice,
        CancellationToken cancellationToken);

    // v2-only: SendCostThresholdAlertAsync
    //   Notifies when a recipe's recalculated cost deviates beyond a per-recipe threshold (C-9).
    //   No stored threshold in v1 — single user receives all spike alerts unconditionally.
    //   Personal Telegram dispatch added in Phase 4 (L15).
    //
    // Task SendCostThresholdAlertAsync(
    //     Guid userId, Guid recipeId,
    //     decimal oldCost, decimal newCost,
    //     CancellationToken cancellationToken);

    // v2-only: SendSellPriceImpactAlertAsync
    //   Notifies when a cost change materially shifts the derived sell price (C-12). Not built in v1.
    //
    // Task SendSellPriceImpactAlertAsync(
    //     Guid userId, Guid recipeId,
    //     decimal oldSellPrice, decimal newSellPrice,
    //     CancellationToken cancellationToken);
}
```

---

## 11. ConsoleAlertDispatcher Stub

**File:** `src/Nastart.Infrastructure/Services/ConsoleAlertDispatcher.cs`

```csharp
using Microsoft.Extensions.Logging;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Infrastructure.Services;

/// <summary>
/// Phase 2 stub implementation of <see cref="IAlertDispatcher"/> that writes alerts to the
/// application log instead of dispatching to an external service.
/// </summary>
/// <remarks>
/// <para>Replaced in Phase 4 (L15) with <c>HttpAlertDispatcher</c>, which POSTs to the Python
/// FastAPI alert service → Telegram. The <see cref="IAlertDispatcher"/> contract is unchanged,
/// so no calling code requires modification at swap time.</para>
/// <para><c>sealed</c>: this stub is not intended to be subclassed.</para>
/// </remarks>
public sealed class ConsoleAlertDispatcher(
    ILogger<ConsoleAlertDispatcher> logger) : IAlertDispatcher
{
    /// <inheritdoc/>
    public Task SendPriceSpikeAlertAsync(
        Guid userId,
        Guid ingredientId,
        decimal oldPrice,
        decimal newPrice,
        CancellationToken cancellationToken)
    {
        logger.LogWarning(
            "[ALERT STUB — Phase 2] Price spike on ingredient {IngredientId} for user {UserId}: "
            + "{OldPrice:C} → {NewPrice:C}. "
            + "In Phase 4 (L15), this dispatches to Python FastAPI → Telegram.",
            ingredientId,
            userId,
            oldPrice,
            newPrice);

        return Task.CompletedTask;
    }

    // v2-only — not implemented in v1:
    // public Task SendCostThresholdAlertAsync(...) { ... }
    // public Task SendSellPriceImpactAlertAsync(...) { ... }
}
```

---

## 12. Updated AddIngredientPrice Handler

Now we wire the cascade and spike check into the handler that commits a new ingredient price. This replaces the L6 handler that had the TODO comment.

For the .NET 10 version below, system time comes from `TimeProvider` instead of direct `DateTime.UtcNow` reads so validation and timestamp creation stay deterministic in tests.

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

/// <summary>
/// Appends a new price history record for the specified ingredient.
/// </summary>
/// <remarks>
/// <para>Append-only (C-5): the current price is always derived from the most recent
/// <c>CommittedAt</c> entry (C-3). Never updates existing records.</para>
/// <para>Triggers: (1) price spike check via <c>IPriceSpikeChecker</c>; (2) cascade
/// recalculation of all recipes using this ingredient via <c>ICostCascadeService</c> (C-1).</para>
/// </remarks>
/// <param name="IngredientId">The ingredient to price.</param>
/// <param name="UserId">
/// The authenticated user's ID, extracted from JWT claims only — never accepted from the
/// request body. Used to verify ownership (OWASP A01).
/// </param>
/// <param name="Price">The new price per package unit. Must be greater than zero.</param>
/// <param name="EffectiveDate">
/// Business-effective date of this price (C-4). Defaults to today (UTC) if omitted.
/// Cannot be in the future.
/// </param>
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

/// <summary>
/// Returned after successfully committing a new ingredient price and running the cascade.
/// </summary>
/// <param name="PriceHistoryId">ID of the newly created <c>IngredientPriceHistory</c> record.</param>
/// <param name="Price">The committed price value.</param>
/// <param name="AffectedRecipes">
/// Number of recipes whose <c>CostPerPortion</c> was successfully recalculated.
/// </param>
/// <param name="FailedRecipes">
/// Number of recipes where cascade recalculation failed; each failure is persisted to
/// <c>CascadeErrorLog</c> (C-5) without rolling back the committed price.
/// </param>
public record AddIngredientPriceResponse(
    Guid PriceHistoryId,
    decimal Price,
    int AffectedRecipes,
    int FailedRecipes
);
```

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

public class AddIngredientPriceCommandValidator : AbstractValidator<AddIngredientPriceCommand>
{
    public AddIngredientPriceCommandValidator(TimeProvider timeProvider)
    {
        RuleFor(x => x.UserId)
            .NotEmpty().WithMessage("UserId is required.");

        RuleFor(x => x.IngredientId)
            .NotEmpty().WithMessage("IngredientId is required.");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than zero.");

        RuleFor(x => x.EffectiveDate)
            .Must(effectiveDate => effectiveDate is null
                || effectiveDate.Value <= DateOnly.FromDateTime(timeProvider.GetUtcNow().UtcDateTime))
            .WithMessage("Effective date cannot be in the future.");
    }
}
```

**File:** `src/Nastart.Application/Features/Ingredients/Commands/AddIngredientPrice/AddIngredientPriceHandler.cs`

This is the handler that was waiting for L7. It now:
1. Verifies ingredient exists and belongs to this user (OWASP A01)
2. Commits new `IngredientPriceHistory` (C-3: the signal for "new price"; C-4: `CommittedAt` written in the handler as the system timestamp, with the DB default kept as a safety net)
3. Checks for price spike via `IPriceSpikeChecker` (non-fatal — alert failure never rolls back the price)
4. Calls cascade service to recalculate all affected recipes (C-1)

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Application.Features.Ingredients.Commands.AddIngredientPrice;

public class AddIngredientPriceHandler(
    IAppDbContext db,
    ICostCascadeService cascadeService,
    IPriceSpikeChecker priceSpikeChecker,
    TimeProvider timeProvider,
    ILogger<AddIngredientPriceHandler> logger)
    : IRequestHandler<AddIngredientPriceCommand, ErrorOr<AddIngredientPriceResponse>>
{
    public async Task<ErrorOr<AddIngredientPriceResponse>> Handle(
        AddIngredientPriceCommand command,
        CancellationToken cancellationToken)
    {
        // OWASP A01: Verify the ingredient exists AND belongs to this user in one query.
        // AsNoTracking: read-only ownership check; ingredient is never modified here.
        var ingredient = await db.Ingredients
            .AsNoTracking()
            .FirstOrDefaultAsync(
                i => i.Id == command.IngredientId && i.UserId == command.UserId,
                cancellationToken)
            .ConfigureAwait(false);

        if (ingredient is null)
            return Error.Forbidden("Ingredient.AccessDenied", "You do not have access to this ingredient.");

        var utcNow = timeProvider.GetUtcNow();

        // C-5: Append-only — never update or delete existing price records.
        // C-13: Source = PriceSource.Manual serialised as 'Manual' (case-sensitive)
        //        via .HasConversion<string>() in IngredientPriceHistoryConfiguration (L2).
        // C-4: Set CommittedAt from TimeProvider in the handler; HasDefaultValueSql("NOW()") is a DB safety net.
        // UnitSize snapshot: captures the ingredient's package unit size at this price entry's
        //   commit time. Each IngredientPriceHistory row carries its own UnitSize so that
        //   historical C-2 calculations remain accurate even if the ingredient's unit size changes.
        var priceRecord = new IngredientPriceHistory
        {
            Id = Guid.NewGuid(),
            IngredientId = command.IngredientId,
            Price = command.Price,
            UnitSize = ingredient.UnitSize,    // Snapshot at commit time
            Source = PriceSource.Manual,       // C-13: exactly 'Manual'
            CommittedAt = utcNow,
            EffectiveDate = command.EffectiveDate ?? DateOnly.FromDateTime(utcNow.UtcDateTime)
        };

        db.IngredientPriceHistories.Add(priceRecord);
        await db.SaveChangesAsync(cancellationToken).ConfigureAwait(false);

        logger.LogInformation(
            "New price committed for ingredient {IngredientId}: {Price}",
            ingredient.Id,
            command.Price);

        // Spike check: non-fatal. A dispatcher failure must not roll back the committed price (C-5).
        await priceSpikeChecker.CheckAndDispatchAsync(
                ingredient.Id,
                ingredient.UserId,
                command.Price,
                cancellationToken)
            .ConfigureAwait(false);

        // C-1: Recalculate CostPerPortion on every recipe using this ingredient.
        // Synchronous for v1 MVP — async background processing deferred to Phase 4.
        var cascadeResult = await cascadeService.RecalculateForIngredientAsync(
                ingredient.Id,
                cancellationToken)
            .ConfigureAwait(false);

        logger.LogInformation(
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

```csharp
// Application/DependencyInjection.cs
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    var assembly = typeof(DependencyInjection).Assembly;

    services.AddMediatR(cfg =>
    {
        cfg.RegisterServicesFromAssembly(assembly);

        // L4: keep the validation pipeline behavior wired for every handler
        cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));
    });

    // L4: keep scanning validators from the Application assembly
    services.AddValidatorsFromAssembly(assembly);

    // Phase 2: cascade + spike checker (Application layer — no infrastructure deps)
    services.AddScoped<ICostCascadeService, CostCascadeService>();
    services.AddScoped<IPriceSpikeChecker, PriceSpikeChecker>();

    return services;
}

// Infrastructure/DependencyInjection.cs
public static IServiceCollection AddInfrastructure(
    this IServiceCollection services,
    IConfiguration configuration,
    IHostEnvironment environment)
{
    var connectionString = configuration.GetConnectionString("DefaultConnection")
        ?? throw new InvalidOperationException(
            "Connection string 'DefaultConnection' is missing. " +
            "Add it to appsettings.Development.json (gitignored — never commit).");

    services.AddDbContext<AppDbContext>(options =>
        options.UseNpgsql(
            connectionString,
            npgsqlOptions => npgsqlOptions.MigrationsAssembly(
                typeof(AppDbContext).Assembly.GetName().Name))
        .UseSnakeCaseNamingConvention());

    // L2: handlers depend on IAppDbContext, not AppDbContext directly
    services.AddScoped<IAppDbContext>(provider =>
        provider.GetRequiredService<AppDbContext>());

    // L5: auth and email services
    services.AddScoped<IPasswordHasher, BcryptPasswordHasher>();
    services.AddScoped<ITokenService, JwtTokenService>();

    if (environment.IsDevelopment())
        services.AddScoped<IEmailService, ConsoleEmailService>();
    else
        throw new InvalidOperationException(
            "No production email provider is configured. " +
            "Register a real IEmailService implementation for non-development environments.");

    // Phase 2 stub — replace with HttpAlertDispatcher in Phase 4 (L15)
    services.AddScoped<IAlertDispatcher, ConsoleAlertDispatcher>();

    return services;
}
```

> Reuse the same `AddInfrastructure(builder.Configuration, builder.Environment)` pattern from L2, L3, and L5 so later lessons do not silently change the method signature.

---

## 14. How to Verify

### Scenario: Add a Price → Cascade Runs → Recipe Costs Update

**Preconditions:**
1. You have created at least 2–3 Ingredients with initial prices
2. You have created at least one Recipe with 2+ RecipeItems referencing those ingredients

**Steps:**

1. **Note the current recipe cost:**
   ```sql
    SELECT id, name, cost_per_portion FROM recipes WHERE id = '<recipe_id>';
   ```
   Example output: Recipe "Pasta Carbonara" has `cost_per_portion = 5.23`

2. **Check the current ingredient price:**
   ```sql
    SELECT price FROM ingredient_price_histories
    WHERE ingredient_id = '<ingredient_id>'
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
     [ALERT STUB — Phase 2] Price spike on ingredient {...}: 2.50 → 3.50 (40.0% change, threshold = 10%)
     Recipe {...} recalculated: cost_per_portion = 6.15
     Cascade complete for ingredient {...}: 1 recipes updated, 0 failed
   ```sql
    SELECT id, name, cost_per_portion FROM recipes WHERE id = '<recipe_id>';
   ```
   Recipe now shows `cost_per_portion = 6.15` (updated!)

6. **Verify cascade error logging works:**
    - In a non-production test database, seed an invalid latest price-history row with `unit_size = 0` for one ingredient, or a recipe item with `YieldPercentage = 0`
    - Run cascade: errors should be logged to `CascadeErrorLogs` table, not crash

7. **Check IngredientPriceHistory table:**
   ```sql
    SELECT * FROM ingredient_price_histories
    WHERE ingredient_id = '<ingredient_id>'
   ORDER BY committed_at DESC;
   ```
   You should see two rows: old price (2.50) and new price (3.50)

---

## 15. Testing the Cascade Service

> Follow the repo's TDD loop for every behavior in this lesson: write the smallest failing test first, run that targeted test to confirm the failure, implement the minimum fix, rerun the targeted test, then run the broader test project as verification.
>
> Use MSTest v4 with PostgreSQL Testcontainers for any test that exercises `AppDbContext`, EF Core query translation, migrations, defaults, precision, or relational constraints. For new .NET 10 MSTest projects, prefer `MSTest.Sdk` running on Microsoft.Testing.Platform (MTP) instead of the older VSTest-first package stack. Keep pure unit tests for arithmetic-only logic. Use NSubstitute only for external ports such as `IAlertDispatcher` and `ILogger<T>`. Assert on **outcome values** (`CostPerPortion`, `CascadeResult` counts, `CascadeErrorLog` presence) — not on mock call counts.
>
> **Why Testcontainers instead of EF Core InMemory?** The `latestPrices` batch query uses `GroupBy + OrderBy + Select + ToDictionaryAsync`, and `PriceSpikeChecker` relies on provider-accurate ordering via `Skip(1)`. EF Core InMemory cannot validate PostgreSQL SQL translation, migrations, defaults, or relational behavior. A PostgreSQL 18 Testcontainer executes the real Npgsql provider and makes these tests portable across machines with a local container runtime.
>
> **Prerequisite:** Docker Desktop, Rancher Desktop, or Podman with a Docker-compatible socket must be available before `dotnet test`. If the container runtime is unavailable, report that blocker instead of silently falling back to EF Core InMemory.

### Project Setup

Pin the repo's chosen .NET 10 SDK, the MSTest SDK version, and the native .NET 10 test runner at the repo root:

```json
{
    "sdk": {
        "version": "10.0.100"
    },
    "msbuild-sdks": {
        "MSTest.Sdk": "4.1.0"
    },
    "test": {
        "runner": "Microsoft.Testing.Platform"
    }
}
```

```xml
<!-- tests/Nastart.Application.Tests/Nastart.Application.Tests.csproj -->
<Project Sdk="MSTest.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Npgsql" Version="10.0.2" />
    <PackageReference Include="NSubstitute" Version="5.*" />
    <PackageReference Include="Testcontainers.PostgreSql" Version="4.12.0" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\src\Nastart.Application\Nastart.Application.csproj" />
    <ProjectReference Include="..\..\src\Nastart.Infrastructure\Nastart.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

> `MSTest.Sdk` uses the MSTest runner on Microsoft.Testing.Platform by default. For a new .NET 10 MSTest project, do not add `Microsoft.NET.Test.Sdk` unless you intentionally need VSTest compatibility.
>
> With the `global.json` runner above, use the native .NET 10 CLI form:
>
> ```bash
> dotnet test --project tests/Nastart.Application.Tests/Nastart.Application.Tests.csproj
> ```

### Shared PostgreSQL Fixture

**File:** `tests/Nastart.Application.Tests/Infrastructure/PostgreSqlTestDatabase.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Npgsql;
using Nastart.Infrastructure.Persistence;
using Testcontainers.PostgreSql;

namespace Nastart.Application.Tests.Infrastructure;

/// <summary>
/// Starts one PostgreSQL container per test class and creates a fresh database per test.
/// Each test database is migrated before use so EF Core runs against the real provider.
/// </summary>
public sealed class PostgreSqlTestDatabase : IAsyncDisposable
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:18-alpine")
        .WithDatabase("postgres")
        .WithUsername("postgres")
        .WithPassword("postgres")
        .Build();

    public Task InitializeAsync() => _container.StartAsync();

    public async Task<AppDbContext> CreateDbContextAsync()
    {
        var databaseName = $"test_{Guid.NewGuid():N}";

        await using (var adminConnection = new NpgsqlConnection(_container.GetConnectionString()))
        {
            await adminConnection.OpenAsync();

            await using var createDatabase = adminConnection.CreateCommand();
            createDatabase.CommandText = $"CREATE DATABASE \"{databaseName}\";";
            await createDatabase.ExecuteNonQueryAsync();
        }

        var connectionStringBuilder = new NpgsqlConnectionStringBuilder(_container.GetConnectionString())
        {
            Database = databaseName
        };

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(
                connectionStringBuilder.ConnectionString,
                npgsql => npgsql.MigrationsAssembly(typeof(AppDbContext).Assembly.GetName().Name))
            .UseSnakeCaseNamingConvention()
            .Options;

        var db = new AppDbContext(options);
        await db.Database.MigrateAsync();
        return db;
    }

    public ValueTask DisposeAsync() => _container.DisposeAsync();
}
```

### CostCascadeServiceTests

**File:** `tests/Nastart.Application.Tests/Services/CostCascadeServiceTests.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using NSubstitute;
using Nastart.Application.Services;
using Nastart.Application.Tests.Infrastructure;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Application.Tests.Services;

[TestClass]
public sealed class CostCascadeServiceTests
{
    private static readonly PostgreSqlTestDatabase Database = new();

    [ClassInitialize]
    public static Task ClassInitialize(TestContext _) => Database.InitializeAsync();

    [ClassCleanup]
    public static Task ClassCleanup() => Database.DisposeAsync().AsTask();

    // ─────────────────────────────────────────────────────────────────────────
    // C-2 formula: cost_per_portion = SUM((price / priceHistory.UnitSize) * qty * (1 / yield)) / portions
    // Parameterized to cover multiple formula scenarios in one method.
    // ─────────────────────────────────────────────────────────────────────────
    //   Scenario 1: (10/1000)*500*(1/1.0)/10  = 0.5000
    //   Scenario 2: (5/500)*100*(1/1.0)/4     = 0.2500
    //   Scenario 3: (100/10)*2*(1/0.85)/1     = 23.5294
    public static IEnumerable<object[]> C2FormulaScenarios =>
    [
        // price, unitSize,  qty,   yield,  portions, expectedCostPerPortion
        [10m,    1000m,     500m,   1.0m,   10,       0.5000m],
        [5m,     500m,      100m,   1.0m,   4,        0.2500m],
        [100m,   10m,       2m,     0.85m,  1,        23.5294m],
    ];

    [DataTestMethod]
    [DynamicData(nameof(C2FormulaScenarios))]
    public async Task RecalculateForIngredientAsync_C2FormulaScenarios_ComputesCorrectCostPerPortion(
        decimal price, decimal unitSize, decimal quantity, decimal yieldPct, int portionCount, decimal expectedCost)
    {
        // Arrange
        await using var db = await Database.CreateDbContextAsync();
        var ingredientId = Guid.NewGuid();
        var recipeId     = Guid.NewGuid();
        var userId       = Guid.NewGuid();

        db.Ingredients.Add(new Ingredient
        {
            Id = ingredientId, UserId = userId, Name = "Test Ingredient", UnitSize = unitSize
        });
        db.IngredientPriceHistories.Add(new IngredientPriceHistory
        {
            Id = Guid.NewGuid(), IngredientId = ingredientId,
            Price = price, UnitSize = unitSize, CommittedAt = DateTimeOffset.UtcNow,
            Source = PriceSource.Manual,
            EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
        });
        db.Recipes.Add(new Recipe
        {
            Id = recipeId, UserId = userId, Name = "Formula Test Recipe",
            PortionCount = portionCount, CostPerPortion = 0m,
            VersionGroupId = Guid.NewGuid()
        });
        db.RecipeItems.Add(new RecipeItem
        {
            Id = Guid.NewGuid(), RecipeId = recipeId, IngredientId = ingredientId,
            Quantity = quantity, YieldPercentage = yieldPct
        });
        await db.SaveChangesAsync();

        var sut = new CostCascadeService(db, Substitute.For<ILogger<CostCascadeService>>());

        // Act
        var result = await sut.RecalculateForIngredientAsync(ingredientId, CancellationToken.None);

        // Assert
        Assert.AreEqual(1, result.AffectedRecipes);
        Assert.AreEqual(0, result.FailedRecipes);
        var updated = await db.Recipes.FindAsync(recipeId);
        Assert.AreEqual(
            expectedCost,
            updated!.CostPerPortion,
            $"C-2 mismatch: ({price}/{unitSize})*{quantity}*(1/{yieldPct})/{portionCount}");
    }

    // ─────────────────────────────────────────────────────────────────────────
    // C-2: multi-ingredient recipe — ALL item costs are summed before dividing by portionCount
    //   Flour:  (10/1000)*500*(1/1.0) = 5.0
    //   Butter: (20/250)*100*(1/1.0)  = 8.0
    //   Total = 13.0 / 4 portions     = 3.2500
    // ─────────────────────────────────────────────────────────────────────────
    [TestMethod]
    public async Task RecalculateForIngredientAsync_WhenMultipleIngredientsInRecipe_SumsAllItemCosts()
    {
        // Arrange
        await using var db = await Database.CreateDbContextAsync();
        var flourId  = Guid.NewGuid();
        var butterId = Guid.NewGuid();
        var recipeId = Guid.NewGuid();
        var userId   = Guid.NewGuid();

        db.Ingredients.AddRange(
            new Ingredient { Id = flourId,  UserId = userId, Name = "Flour",  UnitSize = 1000m },
            new Ingredient { Id = butterId, UserId = userId, Name = "Butter", UnitSize = 250m  }
        );
        db.IngredientPriceHistories.AddRange(
            new IngredientPriceHistory
            {
                Id = Guid.NewGuid(), IngredientId = flourId,
                Price = 10m, UnitSize = 1000m, CommittedAt = DateTimeOffset.UtcNow,
                Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
            },
            new IngredientPriceHistory
            {
                Id = Guid.NewGuid(), IngredientId = butterId,
                Price = 20m, UnitSize = 250m, CommittedAt = DateTimeOffset.UtcNow,
                Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
            }
        );
        db.Recipes.Add(new Recipe
        {
            Id = recipeId, UserId = userId, Name = "Butter Cake",
            PortionCount = 4, CostPerPortion = 0m, VersionGroupId = Guid.NewGuid()
        });
        db.RecipeItems.AddRange(
            new RecipeItem
            {
                Id = Guid.NewGuid(), RecipeId = recipeId, IngredientId = flourId,
                Quantity = 500m, YieldPercentage = 1.0m
            },
            new RecipeItem
            {
                Id = Guid.NewGuid(), RecipeId = recipeId, IngredientId = butterId,
                Quantity = 100m, YieldPercentage = 1.0m
            }
        );
        await db.SaveChangesAsync();

        var sut = new CostCascadeService(db, Substitute.For<ILogger<CostCascadeService>>());

        // Act — cascade triggered for flour; butter's price is batch-loaded in the same pass
        var result = await sut.RecalculateForIngredientAsync(flourId, CancellationToken.None);

        // Assert — 5.0 (flour) + 8.0 (butter) = 13.0 / 4 = 3.2500
        Assert.AreEqual(1, result.AffectedRecipes);
        Assert.AreEqual(0, result.FailedRecipes);
        var updated = await db.Recipes.FindAsync(recipeId);
        Assert.AreEqual(3.2500m, updated!.CostPerPortion);
    }

    // ─────────────────────────────────────────────────────────────────────────
    // Guard: PortionCount = 0 → CostPerPortion clamped to 0, no DivideByZeroException
    // ─────────────────────────────────────────────────────────────────────────
    [TestMethod]
    public async Task RecalculateForIngredientAsync_WhenPortionCountIsZero_SetsCostToZero()
    {
        // Arrange
        await using var db = await Database.CreateDbContextAsync();
        var ingredientId = Guid.NewGuid();
        var recipeId     = Guid.NewGuid();
        var userId       = Guid.NewGuid();

        db.Ingredients.Add(new Ingredient
        {
            Id = ingredientId, UserId = userId, Name = "Flour", UnitSize = 1000m
        });
        db.IngredientPriceHistories.Add(new IngredientPriceHistory
        {
            Id = Guid.NewGuid(), IngredientId = ingredientId,
            Price = 10m, UnitSize = 1000m, CommittedAt = DateTimeOffset.UtcNow,
            Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
        });
        db.Recipes.Add(new Recipe
        {
            Id = recipeId, UserId = userId, Name = "Zero Portion Recipe",
            PortionCount = 0,                    // guard under test
            CostPerPortion = 99m,                // pre-set to non-zero to prove it is overwritten
            VersionGroupId = Guid.NewGuid()
        });
        db.RecipeItems.Add(new RecipeItem
        {
            Id = Guid.NewGuid(), RecipeId = recipeId, IngredientId = ingredientId,
            Quantity = 500m, YieldPercentage = 1.0m
        });
        await db.SaveChangesAsync();

        var sut = new CostCascadeService(db, Substitute.For<ILogger<CostCascadeService>>());

        // Act
        var result = await sut.RecalculateForIngredientAsync(ingredientId, CancellationToken.None);

        // Assert
        Assert.AreEqual(1, result.AffectedRecipes);
        Assert.AreEqual(0, result.FailedRecipes);
        var updated = await db.Recipes.FindAsync(recipeId);
        Assert.AreEqual(0m, updated!.CostPerPortion,
            "PortionCount=0 must clamp CostPerPortion to zero rather than throw DivideByZeroException");
    }

    // ─────────────────────────────────────────────────────────────────────────
    // C-3: no price history → cascade skips the ingredient gracefully
    // ─────────────────────────────────────────────────────────────────────────
    [TestMethod]
    public async Task RecalculateForIngredientAsync_WhenNoPriceHistory_ReturnsZeroCounts()
    {
        // Arrange
        await using var db = await Database.CreateDbContextAsync();
        var sut = new CostCascadeService(db, Substitute.For<ILogger<CostCascadeService>>());

        // Act
        var result = await sut.RecalculateForIngredientAsync(Guid.NewGuid(), CancellationToken.None);

        // Assert
        Assert.AreEqual(0, result.AffectedRecipes);
        Assert.AreEqual(0, result.FailedRecipes);
    }

    // ─────────────────────────────────────────────────────────────────────────
    // C-5: a per-recipe failure writes CascadeErrorLog and does NOT roll back
    //      IngredientPriceHistory; remaining recipes are still processed
    // ─────────────────────────────────────────────────────────────────────────
    [TestMethod]
    public async Task RecalculateForIngredientAsync_WhenOneRecipeFails_LogsErrorAndContinuesOthers()
    {
        // Arrange
        await using var db = await Database.CreateDbContextAsync();
        var ingredientId = Guid.NewGuid();
        var userId       = Guid.NewGuid();

        db.Ingredients.Add(new Ingredient
        {
            Id = ingredientId, UserId = userId, Name = "Butter", UnitSize = 500m
        });
        db.IngredientPriceHistories.Add(new IngredientPriceHistory
        {
            Id = Guid.NewGuid(), IngredientId = ingredientId,
            Price = 5m, UnitSize = 500m, CommittedAt = DateTimeOffset.UtcNow,
            Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
        });

        // Good recipe — (5/500)*100*(1/1.0)/4 = 0.2500
        var goodRecipeId = Guid.NewGuid();
        db.Recipes.Add(new Recipe
        {
            Id = goodRecipeId, UserId = userId, Name = "Butter Cake",
            PortionCount = 4, CostPerPortion = 0m, VersionGroupId = Guid.NewGuid()
        });
        db.RecipeItems.Add(new RecipeItem
        {
            Id = Guid.NewGuid(), RecipeId = goodRecipeId, IngredientId = ingredientId,
            Quantity = 100m, YieldPercentage = 1.0m
        });

        // Bad recipe — YieldPercentage = 0 causes 1m/0m → DivideByZeroException in C-2 formula
        var badRecipeId = Guid.NewGuid();
        db.Recipes.Add(new Recipe
        {
            Id = badRecipeId, UserId = userId, Name = "Broken Recipe",
            PortionCount = 1, CostPerPortion = 0m, VersionGroupId = Guid.NewGuid()
        });
        db.RecipeItems.Add(new RecipeItem
        {
            Id = Guid.NewGuid(), RecipeId = badRecipeId, IngredientId = ingredientId,
            Quantity = 50m, YieldPercentage = 0m     // zero yield → exception in C-2
        });
        await db.SaveChangesAsync();

        var sut = new CostCascadeService(db, Substitute.For<ILogger<CostCascadeService>>());

        // Act
        var result = await sut.RecalculateForIngredientAsync(ingredientId, CancellationToken.None);

        // Assert — C-5: counts
        Assert.AreEqual(1, result.AffectedRecipes, "One recipe should succeed");
        Assert.AreEqual(1, result.FailedRecipes,   "One recipe should fail");

        var updatedGood = await db.Recipes.FindAsync(goodRecipeId);
        Assert.AreEqual(0.2500m, updatedGood!.CostPerPortion,
            "The successful recipe must be recalculated correctly");

        // C-5: error is written to CascadeErrorLog
        var errorLogged = await db.CascadeErrorLogs.AnyAsync(e => e.RecipeId == badRecipeId);
        Assert.IsTrue(errorLogged, "C-5: CascadeErrorLog must contain an entry for the failed recipe.");

        // C-5: IngredientPriceHistory is append-only — never rolled back on cascade failure
        var priceCount = await db.IngredientPriceHistories
            .CountAsync(p => p.IngredientId == ingredientId);
        Assert.AreEqual(1, priceCount,
            "C-5: IngredientPriceHistory must not be rolled back on cascade failure.");
    }
}
```

### PriceSpikeCheckerTests

**File:** `tests/Nastart.Application.Tests/Services/PriceSpikeCheckerTests.cs`

```csharp
using Microsoft.Extensions.Logging;
using NSubstitute;
using Nastart.Application.Common.Interfaces;
using Nastart.Application.Services;
using Nastart.Application.Tests.Infrastructure;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Application.Tests.Services;

[TestClass]
public sealed class PriceSpikeCheckerTests
{
    private static readonly PostgreSqlTestDatabase Database = new();

    [ClassInitialize]
    public static Task ClassInitialize(TestContext _) => Database.InitializeAsync();

    [ClassCleanup]
    public static Task ClassCleanup() => Database.DisposeAsync().AsTask();

    // ─────────────────────────────────────────────────────────────────────────
    // First price entry — Skip(1) yields no previous record → no spike possible
    // ─────────────────────────────────────────────────────────────────────────
    [TestMethod]
    public async Task CheckAndDispatch_WhenFirstPriceEntry_DoesNotDispatchAlert()
    {
        // Arrange
        await using var db = await Database.CreateDbContextAsync();
        var ingredientId = Guid.NewGuid();
        var userId       = Guid.NewGuid();

        db.Ingredients.Add(new Ingredient
            { Id = ingredientId, UserId = userId, Name = "Sugar", UnitSize = 1000m });
        // Exactly 1 record — Skip(1) returns nothing
        db.IngredientPriceHistories.Add(new IngredientPriceHistory
        {
            Id = Guid.NewGuid(), IngredientId = ingredientId,
            Price = 5m, UnitSize = 1000m, CommittedAt = DateTimeOffset.UtcNow,
            Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
        });
        await db.SaveChangesAsync();

        var alertDispatcher = Substitute.For<IAlertDispatcher>();
        var sut = new PriceSpikeChecker(
            db, alertDispatcher, Substitute.For<ILogger<PriceSpikeChecker>>());

        // Act
        await sut.CheckAndDispatchAsync(ingredientId, userId, newPrice: 5m, CancellationToken.None);

        // Assert — no previous price means no comparison is possible
        await alertDispatcher.DidNotReceive().SendPriceSpikeAlertAsync(
            Arg.Any<Guid>(), Arg.Any<Guid>(),
            Arg.Any<decimal>(), Arg.Any<decimal>(), Arg.Any<CancellationToken>());
    }

    // ─────────────────────────────────────────────────────────────────────────
    // Price change is within the threshold → no alert
    // old=10, new=11 → 10% change = threshold → silent (threshold is exclusive: absChangePct <= 10)
    // ─────────────────────────────────────────────────────────────────────────
    [TestMethod]
    public async Task CheckAndDispatch_WhenPriceChangeBelowOrAtThreshold_DoesNotDispatchAlert()
    {
        // Arrange
        await using var db = await Database.CreateDbContextAsync();
        var ingredientId = Guid.NewGuid();
        var userId       = Guid.NewGuid();

        db.Ingredients.Add(new Ingredient
            { Id = ingredientId, UserId = userId, Name = "Eggs", UnitSize = 12m, PriceSpikeThresholdPct = 10m });
        db.IngredientPriceHistories.AddRange(
            new IngredientPriceHistory
            {
                Id = Guid.NewGuid(), IngredientId = ingredientId,
                Price = 10m, UnitSize = 12m, CommittedAt = DateTimeOffset.UtcNow.AddMinutes(-10),  // previous
                Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
            },
            new IngredientPriceHistory
            {
                Id = Guid.NewGuid(), IngredientId = ingredientId,
                Price = 11m, UnitSize = 12m, CommittedAt = DateTimeOffset.UtcNow,                  // just committed
                Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
            }
        );
        await db.SaveChangesAsync();

        var alertDispatcher = Substitute.For<IAlertDispatcher>();
        var sut = new PriceSpikeChecker(
            db, alertDispatcher, Substitute.For<ILogger<PriceSpikeChecker>>());

        // Act
        await sut.CheckAndDispatchAsync(ingredientId, userId, newPrice: 11m, CancellationToken.None);

        // Assert — 10% change ≤ 10% threshold: no alert
        await alertDispatcher.DidNotReceive().SendPriceSpikeAlertAsync(
            Arg.Any<Guid>(), Arg.Any<Guid>(),
            Arg.Any<decimal>(), Arg.Any<decimal>(), Arg.Any<CancellationToken>());
    }

    // ─────────────────────────────────────────────────────────────────────────
    // Price change exceeds threshold → alert dispatched with correct identity and prices
    // old=10, new=15 → 50% change > 10% threshold → alert fires
    // ─────────────────────────────────────────────────────────────────────────
    [TestMethod]
    public async Task CheckAndDispatch_WhenPriceChangeExceedsThreshold_DispatchesAlertWithCorrectValues()
    {
        // Arrange
        await using var db = await Database.CreateDbContextAsync();
        var ingredientId = Guid.NewGuid();
        var userId       = Guid.NewGuid();

        db.Ingredients.Add(new Ingredient
            { Id = ingredientId, UserId = userId, Name = "Butter", UnitSize = 500m, PriceSpikeThresholdPct = 10m });
        db.IngredientPriceHistories.AddRange(
            new IngredientPriceHistory
            {
                Id = Guid.NewGuid(), IngredientId = ingredientId,
                Price = 10m, UnitSize = 500m, CommittedAt = DateTimeOffset.UtcNow.AddMinutes(-10),  // previous
                Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
            },
            new IngredientPriceHistory
            {
                Id = Guid.NewGuid(), IngredientId = ingredientId,
                Price = 15m, UnitSize = 500m, CommittedAt = DateTimeOffset.UtcNow,                  // just committed
                Source = PriceSource.Manual, EffectiveDate = DateOnly.FromDateTime(DateTime.UtcNow)
            }
        );
        await db.SaveChangesAsync();

        var alertDispatcher = Substitute.For<IAlertDispatcher>();
        var sut = new PriceSpikeChecker(
            db, alertDispatcher, Substitute.For<ILogger<PriceSpikeChecker>>());

        // Act
        await sut.CheckAndDispatchAsync(ingredientId, userId, newPrice: 15m, CancellationToken.None);

        // Assert — 50% change > 10% threshold → alert dispatched with correct old/new prices
        await alertDispatcher.Received(1).SendPriceSpikeAlertAsync(
            userId,
            ingredientId,
            10m,  // oldPrice
            15m,  // newPrice
            Arg.Any<CancellationToken>());
    }
}
```

> **Key principles:**
> - **Sealed test classes** with `[TestClass]` — used here for consistency; MSTest v4 does not require sealing
> - **`MSTest.Sdk` + MTP** — current MSTest-first setup for new .NET 10 projects; if the repo opts into MTP in `global.json`, prefer `dotnet test --project ...` over older positional/VSTest examples
> - **`CostCascadeService` constructor** takes only `db + logger` (no `IAlertDispatcher` — moved to `PriceSpikeChecker`)
> - **`IngredientPriceHistory.UnitSize`** populated in every test so the price and package-size snapshot stay together
> - **`[DynamicData]`** for the parameterized C-2 formula scenarios — covers three formula configurations in one test method
> - **Assert on outcome values** — `CostPerPortion`, `CascadeResult` counts, `CascadeErrorLog` presence; NOT mock call counts (except `PriceSpikeCheckerTests` which is specifically testing dispatch behaviour)

---

## 16. Key Takeaways

- **C-1:** `ICostCascadeService.RecalculateForIngredientAsync(...)` is the single, locked entry point. All cost recalculations go through it. Price is never passed as a parameter — the service reads it internally (C-3).
- **C-2:** The cost formula uses `IngredientPriceHistory.UnitSize`, not live `Ingredient.UnitSize`. Formula: `SUM((price / latestPrice.UnitSize) × qty × (1/yield)) / portionCount`.
- **C-3:** Current price is ALWAYS looked up fresh from `IngredientPriceHistories ORDER BY committed_at DESC` — never from a cached field on `Ingredient`.
- **C-5:** Per-recipe cascade failures are logged to `CascadeErrorLog` and skipped. `IngredientPriceHistory` is append-only and never rolled back.
- **SRP — `IPriceSpikeChecker` is separate from `ICostCascadeService`:** Spike detection is not part of the cost formula. Separating them lets you test each independently and swap the threshold logic in v2 without touching the cascade.
- **Fire-and-forget is dangerous:** The original static `PriceSpikeChecker` used `_ = alertDispatcher.SendPriceSpikeAlertAsync(...)`. Replaced with awaited try-catch so dispatcher failures are logged rather than silently swallowed.
- **Primary constructor syntax:** All service classes (`CostCascadeService`, `PriceSpikeChecker`, `ConsoleAlertDispatcher`) use C# 12 primary constructors — no boilerplate field assignments.
- **`TimeProvider` for system timestamps:** Validators and handlers use the .NET system clock abstraction rather than direct `DateTime.UtcNow` reads, which keeps business rules deterministic in tests.
- **`ConfigureAwait(false)`** on every async DB call — prevents deadlocks in non-ASP.NET contexts and is consistent with the dotnet-best-practices skill.
- **v2 preserved:** `CostThresholdPercentage` (C-9), role-split alerts (C-11), sell price impact alerts (C-12) are preserved as XML `<remarks>` and `// v2-only:` comments. Nothing was deleted.

In L8, you'll add the `CreateRecipe` handler which also calls `ICostCascadeService` to compute initial costs after a recipe is saved.

---
```

Now save this complete content to `lessons/L7-cost-cascade-service-and-price-spike-alerts.md` inside your project folder.

Create `lessons/L7-cost-cascade-service-and-price-spike-alerts.md` in VS Code and paste the complete markdown above. This content is long enough that editor review is safer than terminal redirection.

L7 written: 1,247 lines