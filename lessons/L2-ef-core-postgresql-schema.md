# Lesson 2 — EF Core, PostgreSQL & Phase 1 Schema

> **What you'll learn:**
> - What Entity Framework Core is and how code-first works
> - How to model 7 domain entities as C# classes (v1 solopreneur scope)
> - How to configure relationships, constraints, and indexes with Fluent API
> - How to run your first migration and create real database tables
> - Docker Compose for local PostgreSQL

> ⚠️ **v1 Solopreneur Amendments (April 9, 2026)**
>
> This lesson was originally written for an enterprise multi-tenant design. v1 is single-user. Apply these changes as you follow along:
>
> **Do NOT build these — v1 excludes them:**
> - `Company` entity and `CompanyConfiguration`
> - `Outlet` entity and `OutletConfiguration`
> - `OutletUser` join entity and `OutletUserConfiguration`
> - `Invitation` entity and `InvitationConfiguration`
> - `Supplier` entity and `SupplierConfiguration`
> - `Role` enum (`Owner`, `Chef`, `Procurement`, `Viewer`) — no roles in v1
> - `InvitationStatus` enum — no invitations in v1
>
> **Change these:**
> - `Ingredient`: replace `Guid OutletId` with `Guid UserId` (FK → `User.Id`). Update the unique index from `(OutletId, Name)` to `(UserId, Name)`.
> - `Recipe` (introduced in L7): replace `Guid OutletId` with `Guid UserId`. Remove `SellingPrice`. Remove `CostThresholdPercentage`. Add `decimal PackagingCost` (default 0) and `decimal TargetMargin` (default 0).
> - `DbSet` registrations: only register the 7 entities below. Remove all 5 enterprise entities.
>
> **v1 entity list (build these only):**
> `User`, `TelegramLink`, `Ingredient`, `IngredientPriceHistory`, `Unit`, `Category`, `Recipe`, `RecipeItem`, `CascadeErrorLog`
>
> **Keep as-is:**
> - `TelegramLinkStatus` enum — still needed for personal Telegram cost alerts
> - `PriceSource` enum (`Manual`, `InvoiceScan`) — canonical decision C-13 unchanged
> - All canonical decisions C-1 through C-5, C-13 — unchanged

---

## 1. Docker Compose — Local PostgreSQL

Before we write any entity code, we need a running database. Create this at the project root:

**File:** `docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: recipe_cost_dev
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

```bash
# Start PostgreSQL
docker compose up -d

# Verify it's running
docker compose ps
# Should show postgres service as "running"
```

### Connection string

**File:** `src/RecipeCost.API/appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=recipe_cost_dev;Username=dev;Password=dev_password"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

> **For production:** Override via environment variable `ConnectionStrings__DefaultConnection` or use `appsettings.Production.json`. Never commit production credentials.

---

## 2. Install NuGet Packages

```bash
# Infrastructure needs EF Core + Npgsql (PostgreSQL provider)
dotnet add src/RecipeCost.Infrastructure package Microsoft.EntityFrameworkCore
dotnet add src/RecipeCost.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL

# API needs the EF Core design-time tools for migrations
dotnet add src/RecipeCost.API package Microsoft.EntityFrameworkCore.Design

# Application needs EF Core for DbSet<T> reference in IAppDbContext
dotnet add src/RecipeCost.Application package Microsoft.EntityFrameworkCore
```

Install the EF Core CLI tool globally (if not already):

```bash
dotnet tool install --global dotnet-ef
```

---

## 3. Domain Enums

Before creating entities, define the enums they depend on.

> ⚠️ **v2-only — do not build in v1.** Roles are introduced when the product expands to multi-user teams.

**File:** `src/RecipeCost.Domain/Enums/Role.cs`

```csharp
namespace RecipeCost.Domain.Enums;

// Canonical Decision C-8: Exactly 4 roles — no Admin, no Manager, no Superuser
public enum Role
{
    Owner,
    Chef,
    Procurement,
    Viewer
}
```

**File:** `src/RecipeCost.Domain/Enums/TelegramLinkStatus.cs`

```csharp
namespace RecipeCost.Domain.Enums;

// Canonical Decision C-6: TelegramLink lifecycle states
public enum TelegramLinkStatus
{
    Pending,
    Confirmed,
    Unlinked
}
```

> ⚠️ **v2-only — do not build in v1.** Invitations do not exist in the single-user v1 product.

**File:** `src/RecipeCost.Domain/Enums/InvitationStatus.cs`

```csharp
namespace RecipeCost.Domain.Enums;

public enum InvitationStatus
{
    Pending,
    Used,
    Expired
}
```

**File:** `src/RecipeCost.Domain/Enums/PriceSource.cs`

```csharp
namespace RecipeCost.Domain.Enums;

// Canonical Decision C-13: Exactly these two values, case-sensitive
// EF Core will store these as strings: "Manual", "InvoiceScan"
public enum PriceSource
{
    Manual,
    InvoiceScan
}
```

**File:** `src/RecipeCost.Domain/Enums/InvoiceStatus.cs`

```csharp
namespace RecipeCost.Domain.Enums;

// Forward-declared for Phase 3 — Invoice entity references this in L2
// but invoice processing logic is built in L12–L14
public enum InvoiceStatus
{
    Processing,
    Review,
    Committed,
    Failed
}
```

---

## 4. Domain Entities — All 11 for Phase 1

Each entity inherits from `BaseEntity` (created in L1). All live in `src/RecipeCost.Domain/Entities/`.

### Company

> ⚠️ **v2-only — do not build in v1.** Company hierarchy is introduced when the product expands to multi-outlet teams.

**File:** `src/RecipeCost.Domain/Entities/Company.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;

public class Company : BaseEntity
{
    public string Name { get; set; } = string.Empty;
    public string Currency { get; set; } = "USD";
    public string Timezone { get; set; } = "UTC";

    // Navigation — a company has many outlets
    public ICollection<Outlet> Outlets { get; set; } = new List<Outlet>();
}
```

### Outlet

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/RecipeCost.Domain/Entities/Outlet.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;

public class Outlet : BaseEntity
{
    public string Name { get; set; } = string.Empty;

    // FK to Company
    public Guid CompanyId { get; set; }
    public Company Company { get; set; } = null!;

    // Navigation
    public ICollection<OutletUser> OutletUsers { get; set; } = new List<OutletUser>();
    public ICollection<Ingredient> Ingredients { get; set; } = new List<Ingredient>();
}
```

### User

**File:** `src/RecipeCost.Domain/Entities/User.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;

// Canonical Decision C-6: User has NO telegram_user_id column.
// All Telegram identity lives on TelegramLink only.
public class User : BaseEntity
{
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string PasswordHash { get; set; } = string.Empty;
    public bool IsVerified { get; set; }
    public bool IsActive { get; set; }

    // Navigation
    // v2-only: OutletUsers removed — no OutletUser entity in v1
    public ICollection<TelegramLink> TelegramLinks { get; set; } = new List<TelegramLink>();
    public ICollection<Ingredient> Ingredients { get; set; } = new List<Ingredient>();
}
```

### OutletUser (Join Table)

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/RecipeCost.Domain/Entities/OutletUser.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;
using RecipeCost.Domain.Enums;

// A user can belong to multiple outlets with DIFFERENT roles at each.
// The role is per-outlet, not per-user.
public class OutletUser : BaseEntity
{
    public Guid UserId { get; set; }
    public User User { get; set; } = null!;

    public Guid OutletId { get; set; }
    public Outlet Outlet { get; set; } = null!;

    // C-8: Exactly Owner, Chef, Procurement, Viewer
    public Role Role { get; set; }
}
```

### Invitation

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/RecipeCost.Domain/Entities/Invitation.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;
using RecipeCost.Domain.Enums;

public class Invitation : BaseEntity
{
    public string Email { get; set; } = string.Empty;
    public Role Role { get; set; }

    public Guid OutletId { get; set; }
    public Outlet Outlet { get; set; } = null!;

    // The owner who sent the invitation
    public Guid InvitedByUserId { get; set; }
    public User InvitedByUser { get; set; } = null!;

    public string TokenHash { get; set; } = string.Empty;
    public DateTimeOffset ExpiresAt { get; set; }
    public InvitationStatus Status { get; set; } = InvitationStatus.Pending;
}
```

### TelegramLink

**File:** `src/RecipeCost.Domain/Entities/TelegramLink.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;
using RecipeCost.Domain.Enums;

// Canonical Decision C-6: Full schema
// C-14: Column is telegramUserId — not chatId, not TelegramChatId
public class TelegramLink : BaseEntity
{
    public Guid UserId { get; set; }
    public User User { get; set; } = null!;

    // SHA-256 hash of the one-time linking code — NEVER stores plaintext
    public string CodeHash { get; set; } = string.Empty;

    // 15-minute TTL from code generation
    public DateTimeOffset ExpiresAt { get; set; }

    public TelegramLinkStatus Status { get; set; } = TelegramLinkStatus.Pending;

    // C-14: This is Telegram's user_id (= chat_id for private chats)
    public long? TelegramUserId { get; set; }
    public string? TelegramUsername { get; set; }

    // Set when status transitions to Confirmed
    public DateTimeOffset? LinkedAt { get; set; }
}
```

### Unit

**File:** `src/RecipeCost.Domain/Entities/Unit.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;

public class Unit : BaseEntity
{
    public string Name { get; set; } = string.Empty;       // e.g. "kilogram", "litre", "piece"
    public string Abbreviation { get; set; } = string.Empty; // e.g. "kg", "L", "pc"
}
```

### Category

**File:** `src/RecipeCost.Domain/Entities/Category.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;

public class Category : BaseEntity
{
    public string Name { get; set; } = string.Empty; // e.g. "Protein", "Dairy", "Produce"

    // User-scoped — each user manages their own categories
    public Guid UserId { get; set; }
    public User User { get; set; } = null!;

    public ICollection<Ingredient> Ingredients { get; set; } = new List<Ingredient>();
}
```

### Supplier

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/RecipeCost.Domain/Entities/Supplier.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;

public class Supplier : BaseEntity
{
    public string Name { get; set; } = string.Empty;
    public string? ContactInfo { get; set; }

    // Supplier is outlet-scoped — different outlets may have different supplier lists
    public Guid OutletId { get; set; }
    public Outlet Outlet { get; set; } = null!;
}
```

### Ingredient

**File:** `src/RecipeCost.Domain/Entities/Ingredient.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;

// Canonical Decision C-3: There is NO current_price column.
// Current price is ALWAYS derived from the latest IngredientPriceHistory record.
public class Ingredient : BaseEntity
{
    public string Name { get; set; } = string.Empty;

    // User-scoped — an ingredient belongs to one user
    public Guid UserId { get; set; }
    public User User { get; set; } = null!;;

    public Guid? CategoryId { get; set; }
    public Category? Category { get; set; }

    public Guid UnitId { get; set; }
    public Unit Unit { get; set; } = null!;

    // The default purchase size for this ingredient (e.g., 1.0 for 1kg bag)
    public decimal UnitSize { get; set; }

    // Per-ingredient spike alert threshold.
    // Null = no spike alerts for this ingredient.
    // Non-null = alert fires when price change exceeds this % (e.g., 10 = 10%).
    public decimal? PriceSpikeThresholdPct { get; set; }

    // Navigation
    public ICollection<IngredientPriceHistory> PriceHistory { get; set; } = new List<IngredientPriceHistory>();
}
```

### IngredientPriceHistory

**File:** `src/RecipeCost.Domain/Entities/IngredientPriceHistory.cs`

```csharp
namespace RecipeCost.Domain.Entities;

using RecipeCost.Domain.Common;
using RecipeCost.Domain.Enums;

// This is an APPEND-ONLY table. Records are NEVER updated or deleted.
// Canonical Decision C-5: Cascade errors never roll back price history.
public class IngredientPriceHistory : BaseEntity
{
    public Guid IngredientId { get; set; }
    public Ingredient Ingredient { get; set; } = null!;

    public decimal Price { get; set; }

    // Snapshot of ingredient's unit_size at the time of recording
    // Allows historical cost calculation even if unit_size changes later
    public decimal UnitSize { get; set; }

    // C-13: Exactly 'Manual' or 'InvoiceScan' (case-sensitive via enum string conversion)
    public PriceSource Source { get; set; }

    // C-4: System timestamp, set on insert. Non-editable. Used for ordering.
    // Current price is: ORDER BY CommittedAt DESC LIMIT 1
    // DateTimeOffset required: Npgsql maps timestamptz → DateTimeOffset (not DateTime)
    public DateTimeOffset CommittedAt { get; set; }

    // C-4: User-editable. Business effective date.
    // Defaults to invoice date (if from scan) or today (if manual).
    public DateOnly EffectiveDate { get; set; }

    // Nullable FK — null for Manual entries, populated for InvoiceScan entries
    // Links back to the specific invoice line item that created this price
    // Phase 3 only: this FK is forward-declared here for schema completeness.
    // The InvoiceLineItem entity and invoice scanning logic are built in L12–L14.
    public Guid? InvoiceLineItemId { get; set; }
}
```

---

## 5. Entity Framework Core — What It Does

EF Core is an **Object-Relational Mapper (ORM)**. It maps your C# classes to database tables:

```
C# Class          →    PostgreSQL Table
─────────────────       ──────────────────
Ingredient.cs     →    ingredients
  └─ Id (Guid)    →      id (uuid PK)
  └─ Name         →      name (text)
  └─ UserId       →      user_id (uuid FK)
```

**Code-first** means you define classes first, then EF Core generates SQL migration scripts to create the tables. You never write `CREATE TABLE` manually.

---

## 6. The DbContext

The `AppDbContext` is your gateway to the database. It lives in Infrastructure:

> ⚠️ **v2-only — omit enterprise DbSets above.** AppDbContext registers only the 9 entities below.

**File:** `src/RecipeCost.Infrastructure/Persistence/AppDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using RecipeCost.Application.Common.Interfaces;
using RecipeCost.Domain.Common;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Infrastructure.Persistence;

public class AppDbContext : DbContext, IAppDbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    // Each DbSet = one database table
    public DbSet<User> Users => Set<User>();
    public DbSet<TelegramLink> TelegramLinks => Set<TelegramLink>();
    public DbSet<Ingredient> Ingredients => Set<Ingredient>();
    public DbSet<IngredientPriceHistory> IngredientPriceHistories => Set<IngredientPriceHistory>();
    public DbSet<Unit> Units => Set<Unit>();
    public DbSet<Category> Categories => Set<Category>();
    public DbSet<Recipe> Recipes => Set<Recipe>();
    public DbSet<RecipeItem> RecipeItems => Set<RecipeItem>();
    public DbSet<CascadeErrorLog> CascadeErrorLogs => Set<CascadeErrorLog>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply all entity configurations from this assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }

    // Automatically set CreatedAt/UpdatedAt on save
    public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        foreach (var entry in ChangeTracker.Entries<BaseEntity>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = DateTimeOffset.UtcNow;
                    break;
                case EntityState.Modified:
                    entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;
                    break;
            }
        }

        return await base.SaveChangesAsync(cancellationToken);
    }
}
```

Now update `IAppDbContext` to include the DbSet properties:

> ⚠️ **v2-only — omit enterprise DbSets above.** AppDbContext registers only the 9 entities below.

**File:** `src/RecipeCost.Application/Common/Interfaces/IAppDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Application.Common.Interfaces;

public interface IAppDbContext
{
    DbSet<User> Users { get; }
    DbSet<TelegramLink> TelegramLinks { get; }
    DbSet<Ingredient> Ingredients { get; }
    DbSet<IngredientPriceHistory> IngredientPriceHistories { get; }
    DbSet<Unit> Units { get; }
    DbSet<Category> Categories { get; }
    DbSet<Recipe> Recipes { get; }
    DbSet<RecipeItem> RecipeItems { get; }
    DbSet<CascadeErrorLog> CascadeErrorLogs { get; }

    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

> **EF Core Query Best Practices**
>
> Apply these patterns consistently across all handlers:
>
> **1. `AsNoTracking()` for read-only queries**
> EF Core tracks every loaded entity by default. Disable it for reads where you aren't saving changes — lower memory use and faster queries:
> ```csharp
> // Read-only handler — skip tracking
> var ingredients = await _db.Ingredients
>     .AsNoTracking()
>     .Where(i => i.UserId == userId)
>     .ToListAsync(ct);
>
> // Only omit AsNoTracking() when you intend to modify the entity
> var ingredient = await _db.Ingredients
>     .FirstOrDefaultAsync(i => i.Id == id, ct); // tracked — you'll call SaveChangesAsync
> ingredient.Name = command.Name;
> await _db.SaveChangesAsync(ct);
> ```
>
> **2. Use `Any()` for existence checks — never `Count()`**
> `Count()` scans all matching rows. `Any()` stops at the first match:
> ```csharp
> // Bad — scans entire table
> if (await _db.Ingredients.CountAsync(i => i.UserId == userId && i.Name == name, ct) > 0)
>
> // Good — stops at first match
> if (await _db.Ingredients.AnyAsync(i => i.UserId == userId && i.Name == name, ct))
> ```
>
> **3. Filter before materializing — `.Where()` before `.ToList()`**
> Calling `.ToListAsync()` before filtering loads the whole table into memory:
> ```csharp
> // Bad — loads all rows, filters in C#
> var all = await _db.Ingredients.ToListAsync(ct);
> var mine = all.Where(i => i.UserId == userId).ToList();
>
> // Good — WHERE runs in SQL
> var mine = await _db.Ingredients
>     .Where(i => i.UserId == userId)
>     .ToListAsync(ct);
> ```
>
> **4. Prevent N+1 — use `.Include()` or projection**
> Accessing a navigation property in a loop triggers one SQL query per row:
> ```csharp
> // N+1 — one extra query per ingredient
> var ingredients = await _db.Ingredients.ToListAsync(ct);
> foreach (var ing in ingredients)
>     Console.WriteLine(ing.PriceHistory.Count); // lazy-load query per row!
>
> // Eager load — single JOIN query
> var ingredients = await _db.Ingredients
>     .Include(i => i.PriceHistory)
>     .AsNoTracking()
>     .ToListAsync(ct);
>
> // Projection — best: only fetches columns you need
> var summaries = await _db.Ingredients
>     .Select(i => new { i.Id, i.Name, LatestPrice = i.PriceHistory
>         .OrderByDescending(p => p.CommittedAt)
>         .Select(p => p.Price)
>         .FirstOrDefault() })
>     .AsNoTracking()
>     .ToListAsync(ct);
> ```
>
> **5. Raw SQL: `FromSqlInterpolated` only — never `FromSqlRaw` with user input**
> `FromSqlInterpolated` binds parameters automatically. `FromSqlRaw` with string interpolation is a SQL injection risk:
> ```csharp
> // Safe — parameters bound by EF Core
> var results = await _db.Ingredients
>     .FromSqlInterpolated($"SELECT * FROM ingredients WHERE user_id = {userId}")
>     .AsNoTracking()
>     .ToListAsync(ct);
>
> // NEVER — SQL injection risk
> // .FromSqlRaw($"SELECT * FROM ingredients WHERE user_id = '{userId}'")
> ```

---

## 7. Fluent API Configurations

EF Core's Fluent API lets you configure columns, constraints, and indexes separately from the entity class. Create one configuration file per entity in Infrastructure:

```bash
mkdir -p src/RecipeCost.Infrastructure/Persistence/Configurations
```

### Key configurations (not every entity — just the ones with important constraints)

**File:** `src/RecipeCost.Infrastructure/Persistence/Configurations/UserConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Infrastructure.Persistence.Configurations;

public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasIndex(u => u.Email).IsUnique();
        builder.Property(u => u.Email).HasMaxLength(255).IsRequired();
        builder.Property(u => u.Name).HasMaxLength(255).IsRequired();
        builder.Property(u => u.PasswordHash).HasMaxLength(255).IsRequired();
    }
}
```

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/RecipeCost.Infrastructure/Persistence/Configurations/OutletUserConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Infrastructure.Persistence.Configurations;

public class OutletUserConfiguration : IEntityTypeConfiguration<OutletUser>
{
    public void Configure(EntityTypeBuilder<OutletUser> builder)
    {
        // A user can have only one role per outlet
        builder.HasIndex(ou => new { ou.UserId, ou.OutletId }).IsUnique();

        // Store enum as string for readability in DB
        builder.Property(ou => ou.Role).HasConversion<string>().HasMaxLength(20);

        builder.HasOne(ou => ou.User)
            .WithMany(u => u.OutletUsers)
            .HasForeignKey(ou => ou.UserId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasOne(ou => ou.Outlet)
            .WithMany(o => o.OutletUsers)
            .HasForeignKey(ou => ou.OutletId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

**File:** `src/RecipeCost.Infrastructure/Persistence/Configurations/TelegramLinkConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using RecipeCost.Domain.Entities;
using RecipeCost.Domain.Enums;

namespace RecipeCost.Infrastructure.Persistence.Configurations;

public class TelegramLinkConfiguration : IEntityTypeConfiguration<TelegramLink>
{
    public void Configure(EntityTypeBuilder<TelegramLink> builder)
    {
        // C-6: SHA-256 hash is always exactly 64 hex characters
        builder.Property(t => t.CodeHash).HasMaxLength(64).IsRequired();

        // C-14: Column name is telegramUserId
        builder.Property(t => t.TelegramUserId).HasColumnName("telegram_user_id");

        builder.Property(t => t.TelegramUsername).HasMaxLength(255);

        // C-6: Store enum as lowercase strings — 'pending' | 'confirmed' | 'unlinked'
        // Default HasConversion<string>() would store PascalCase ("Pending") which
        // violates the canonical decision contract. This converter normalizes to lowercase.
        builder.Property(t => t.Status)
            .HasConversion(
                v => v.ToString().ToLowerInvariant(),
                v => (TelegramLinkStatus)Enum.Parse(typeof(TelegramLinkStatus), v, ignoreCase: true))
            .HasMaxLength(20);

        // Index for querying links by user and status (used in linking + deactivation flows)
        builder.HasIndex(t => t.CodeHash).IsUnique();
        builder.HasIndex(t => new { t.UserId, t.Status });

        builder.HasOne(t => t.User)
            .WithMany(u => u.TelegramLinks)
            .HasForeignKey(t => t.UserId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

**File:** `src/RecipeCost.Infrastructure/Persistence/Configurations/IngredientConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Infrastructure.Persistence.Configurations;

public class IngredientConfiguration : IEntityTypeConfiguration<Ingredient>
{
    public void Configure(EntityTypeBuilder<Ingredient> builder)
    {
        // Ingredient name is unique within a user's scope
        builder.HasIndex(i => new { i.UserId, i.Name }).IsUnique();

        builder.Property(i => i.Name).HasMaxLength(255).IsRequired();
        builder.Property(i => i.UnitSize).HasPrecision(10, 4);
        builder.Property(i => i.PriceSpikeThresholdPct).HasPrecision(5, 2);

        builder.HasOne(i => i.User)
            .WithMany(u => u.Ingredients)
            .HasForeignKey(i => i.UserId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasOne(i => i.Category)
            .WithMany(c => c.Ingredients)
            .HasForeignKey(i => i.CategoryId)
            .OnDelete(DeleteBehavior.SetNull);

        builder.HasOne(i => i.Unit)
            .WithMany()
            .HasForeignKey(i => i.UnitId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

> **`HasQueryFilter` for user-scoped entities:** Because every `Ingredient` query must include a `UserId` filter, EF Core's `HasQueryFilter` can enforce this at the data-access layer automatically, acting as a safety net on top of handler-level filtering:
> ```csharp
> // In IngredientConfiguration.Configure() — optional belt-and-suspenders guard
> // Requires the current userId to be injected into AppDbContext (e.g. via ICurrentUserService)
> // builder.HasQueryFilter(i => i.UserId == _currentUserId);
> ```
> In v1 (single user), handlers already apply the `UserId` filter explicitly, so this is optional. Use `IgnoreQueryFilters()` on specific queries that deliberately bypass the filter (e.g. admin diagnostics).

**File:** `src/RecipeCost.Infrastructure/Persistence/Configurations/IngredientPriceHistoryConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Infrastructure.Persistence.Configurations;

public class IngredientPriceHistoryConfiguration : IEntityTypeConfiguration<IngredientPriceHistory>
{
    public void Configure(EntityTypeBuilder<IngredientPriceHistory> builder)
    {
        builder.Property(p => p.Price).HasPrecision(10, 4);
        builder.Property(p => p.UnitSize).HasPrecision(10, 4);

        // C-13: Store enum as exact string — "Manual" or "InvoiceScan"
        builder.Property(p => p.Source).HasConversion<string>().HasMaxLength(20);

        // C-4: System timestamp, set on insert. Used for current price ordering.
        builder.Property(p => p.CommittedAt).HasDefaultValueSql("NOW()");

        // Performance: composite index for "get the latest price" pattern
        // Per Flow 02: SELECT ... ORDER BY committed_at DESC LIMIT 1
        builder.HasIndex(p => new { p.IngredientId, p.CommittedAt })
            .IsDescending(false, true)
            .HasDatabaseName("ix_ingredient_price_history_ingredient_committed");

        builder.HasOne(p => p.Ingredient)
            .WithMany(i => i.PriceHistory)
            .HasForeignKey(p => p.IngredientId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

**File:** `src/RecipeCost.Infrastructure/Persistence/Configurations/CategoryConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Infrastructure.Persistence.Configurations;

public class CategoryConfiguration : IEntityTypeConfiguration<Category>
{
    public void Configure(EntityTypeBuilder<Category> builder)
    {
        builder.Property(c => c.Name).HasMaxLength(255).IsRequired();

        // Category name is unique within a user's scope
        builder.HasIndex(c => new { c.UserId, c.Name }).IsUnique();

        builder.HasOne(c => c.User)
            .WithMany()
            .HasForeignKey(c => c.UserId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/RecipeCost.Infrastructure/Persistence/Configurations/InvitationConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Infrastructure.Persistence.Configurations;

public class InvitationConfiguration : IEntityTypeConfiguration<Invitation>
{
    public void Configure(EntityTypeBuilder<Invitation> builder)
    {
        builder.Property(i => i.Email).HasMaxLength(255).IsRequired();
        builder.Property(i => i.TokenHash).HasMaxLength(64).IsRequired();
        builder.Property(i => i.Role).HasConversion<string>().HasMaxLength(20);
        builder.Property(i => i.Status).HasConversion<string>().HasMaxLength(20);
    }
}
```

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/RecipeCost.Infrastructure/Persistence/Configurations/SupplierConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using RecipeCost.Domain.Entities;

namespace RecipeCost.Infrastructure.Persistence.Configurations;

public class SupplierConfiguration : IEntityTypeConfiguration<Supplier>
{
    public void Configure(EntityTypeBuilder<Supplier> builder)
    {
        builder.Property(s => s.Name).HasMaxLength(255).IsRequired();
        builder.Property(s => s.ContactInfo).HasMaxLength(500);

        builder.HasOne(s => s.Outlet)
            .WithMany()
            .HasForeignKey(s => s.OutletId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

---

## 8. Register DbContext in DI

Wire EF Core into the DI container. Create a clean extension method:

**File:** `src/RecipeCost.Infrastructure/DependencyInjection.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using RecipeCost.Application.Common.Interfaces;
using RecipeCost.Infrastructure.Persistence;

namespace RecipeCost.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(
                configuration.GetConnectionString("DefaultConnection"),
                npgsqlOptions => npgsqlOptions.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)
            )
        );

        // Register AppDbContext as IAppDbContext for DI
        // Handlers ask for IAppDbContext — they receive AppDbContext
        services.AddScoped<IAppDbContext>(provider =>
            provider.GetRequiredService<AppDbContext>());

        return services;
    }
}
```

Update `Program.cs` to call it:

**File:** `src/RecipeCost.API/Program.cs`

```csharp
using RecipeCost.Infrastructure;

var builder = WebApplication.CreateBuilder(args);

// Register infrastructure services (EF Core, DbContext)
builder.Services.AddInfrastructure(builder.Configuration);

var app = builder.Build();

app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));

app.Run();
```

---

## 9. Create and Run the Migration

```bash
# Create the initial migration
dotnet ef migrations add CreatePhase1Schema \
  --project src/RecipeCost.Infrastructure \
  --startup-project src/RecipeCost.API \
  --output-dir Persistence/Migrations

# Apply the migration to the database
dotnet ef database update \
  --project src/RecipeCost.Infrastructure \
  --startup-project src/RecipeCost.API
```

> **What just happened:** EF Core compared your entity classes + Fluent API configuration against the current database (empty). It generated a C# migration file that creates all 11 tables with columns, primary keys, foreign keys, indexes, and constraints. Then `database update` executed the generated SQL against your PostgreSQL.

### Verify the tables exist

```bash
# Connect to PostgreSQL and list tables
docker exec -it $(docker compose ps -q postgres) psql -U dev -d recipe_cost_dev -c "\dt"
```

You should see tables for: users, telegram_links, ingredients, ingredient_price_histories, units, categories, recipes, recipe_items, cascade_error_logs.

---

## 10. Console Email Service (Dev Only)

For L5's email verification, we need an email service that logs to console instead of sending real emails:

**File:** `src/RecipeCost.Application/Common/Interfaces/IEmailService.cs`

```csharp
namespace RecipeCost.Application.Common.Interfaces;

public interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
}
```

**File:** `src/RecipeCost.Infrastructure/Services/ConsoleEmailService.cs`

```csharp
using Microsoft.Extensions.Logging;
using RecipeCost.Application.Common.Interfaces;

namespace RecipeCost.Infrastructure.Services;

// Development-only email service — logs email content to console.
// Replace with a real SMTP/SendGrid implementation for production.
public class ConsoleEmailService : IEmailService
{
    private readonly ILogger<ConsoleEmailService> _logger;

    public ConsoleEmailService(ILogger<ConsoleEmailService> logger)
    {
        _logger = logger;
    }

    public Task SendAsync(string to, string subject, string body)
    {
        _logger.LogInformation(
            "== EMAIL ==\nTo: {To}\nSubject: {Subject}\nBody:\n{Body}\n== END EMAIL ==",
            to, subject, body);
        return Task.CompletedTask;
    }
}
```

Register it in `DependencyInjection.cs`:

```csharp
// Add inside AddInfrastructure method, after AddDbContext
services.AddScoped<IEmailService, ConsoleEmailService>();
```

---

## Checkpoint — Verify Before Moving to L3

```bash
# 1. Docker Postgres is running
docker compose ps

# 2. Solution builds
dotnet build

# 3. Migration applied without errors
dotnet ef database update \
  --project src/RecipeCost.Infrastructure \
  --startup-project src/RecipeCost.API

# 4. Tables exist in PostgreSQL
docker exec -it $(docker compose ps -q postgres) psql -U dev -d recipe_cost_dev -c "\dt"
# Should show 9 tables (plus __EFMigrationsHistory)

# 5. API runs
dotnet run --project src/RecipeCost.API
# curl http://localhost:5000/health → {"status":"healthy"}
```

### What we built in this lesson

- 5 enums (Role, TelegramLinkStatus, InvitationStatus, PriceSource, InvoiceStatus)
- 9 entity classes in Domain/
- 6 Fluent API configurations with constraints, indexes, and relationships
- AppDbContext with automatic CreatedAt/UpdatedAt
- IAppDbContext interface in Application
- First migration creating all Phase 1 tables
- ConsoleEmailService for dev email logging
- docker-compose.yml for local PostgreSQL

### Entities deferred to later phases

| Entity | Phase | Lesson |
|---|---|---|
| Recipe, RecipeItem | Phase 2 | L9 |
| CascadeErrorLog | Phase 2 | L8 |
| Invoice, InvoiceLineItem, ReviewQueueItem | Phase 3 | L12 |
