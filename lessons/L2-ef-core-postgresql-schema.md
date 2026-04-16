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

Before we write any entity code, we need a running database.

### 1a. Create the secrets file

The database password lives in a plain-text file that Docker mounts inside the container. **This file is never committed to version control.**

```bash
# Create the folder that will hold all local secret files
mkdir secrets
```

**File:** `secrets/db_password.txt`

```
dev_password
```

> The file contains only the password — no quotes, no newline at the end. PostgreSQL reads it via the `POSTGRES_PASSWORD_FILE` env var and uses the raw contents as the password.

Now protect it from accidental commits. Add these lines to your **`.gitignore`** at the project root (create the file if it doesn't exist):

```gitignore
# Local secrets — never commit these
secrets/

# .NET development overrides — contains connection strings with passwords
appsettings.Development.json
```

### 1b. Docker Compose

**File:** `docker-compose.yml`

```yaml
# docker-compose.yml
# Compose v2 spec — no "version:" field needed (deprecated and ignored by modern Docker)

services:
  postgres:
    image: postgres:18-alpine  # PostgreSQL 18 on lightweight Alpine Linux (~80 MB vs ~400 MB full)

    environment:
      POSTGRES_DB: recipe_cost_dev             # Database Docker will create on first startup
      POSTGRES_USER: dev                       # Username your app connects with
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password  # Reads password from the mounted secret file
                                                         # Never use POSTGRES_PASSWORD here — that puts
                                                         # the plaintext password in a file you may commit

    secrets:
      - db_password  # Mounts secrets/db_password.txt as a read-only file at /run/secrets/db_password
                     # Only this container can read it — it is never exposed as an environment variable

    ports:
      - "127.0.0.1:5432:5432" # host_ip:host_port:container_port
                               # 127.0.0.1 binds only to localhost — not exposed on your LAN/Wi-Fi
                               # Your .NET API on the host reaches it via Host=localhost;Port=5432

    volumes:
      - postgres_data:/var/lib/postgresql/data # Named volume: your database rows survive
                                               # "docker compose down"    -- data persists
                                               # "docker compose down -v" -- data is wiped

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dev -d recipe_cost_dev"]
      interval: 5s      # Check container health every 5 seconds
      timeout: 5s       # Fail the check if no response within 5 seconds
      retries: 5        # Mark unhealthy after 5 consecutive failures
      start_period: 10s # Give Postgres 10 seconds to initialise before checks begin

volumes:
  postgres_data: # Named volume definition — Docker manages the storage location
                 # Run "docker volume ls" to see it; "docker volume inspect postgres_data" for details

secrets:
  db_password:
    file: ./secrets/db_password.txt  # Compose reads this file from disk and mounts it inside the container
                                     # Change the password here without ever touching docker-compose.yml
```

> **What just happened?**
>
> **Docker secrets (`POSTGRES_PASSWORD_FILE`):** Instead of writing the password directly into `docker-compose.yml`, we put it in a separate file (`secrets/db_password.txt`) and tell Compose to mount it inside the container at `/run/secrets/db_password`. PostgreSQL's official image reads the `_FILE`-suffixed variable and loads the password from that path. The result: `docker-compose.yml` contains zero secrets and is safe to commit.
>
> **Detached mode (`-d`):** `docker compose up -d` starts the container in the background. Your terminal returns immediately — the container keeps running until you stop it.
>
> **Port mapping (`127.0.0.1:5432:5432`):** This tells Docker: forward traffic arriving at `localhost:5432` on your machine into port `5432` inside the container. The `127.0.0.1` prefix locks the binding to your local loopback only — the port is invisible to other devices on your network.
>
> **How the connection string finds the container:** Your .NET API runs directly on Windows (not in Docker). When EF Core reads `Host=localhost;Port=5432`, it connects to `localhost:5432` on the host machine. Docker intercepts that traffic and routes it into the Postgres container. From the API's perspective, Postgres is just another local process.
>
> **Named volumes:** `postgres_data` is a Docker-managed storage area that lives outside the container. When you run `docker compose down`, the container is removed but the volume — and all your data — survives. Only `docker compose down -v` destroys the volume.
>
> **Why no `depends_on` here?** `depends_on` links two services defined in the same `docker-compose.yml`. In v1, the .NET API runs directly on your host machine — it is not a Compose service — so there is nothing to wire up. When we containerise the full stack in a later phase, `depends_on` with `condition: service_healthy` will use the health check above to ensure Postgres is ready before the API starts.

```bash
# ── Start ──────────────────────────────────────────────────────────────────────

# Start the postgres container in detached mode (runs in the background, terminal returns immediately)
docker compose up -d

# Check container status — look for "healthy" under Status (appears after ~15 seconds)
docker compose ps

# Stream Postgres logs to your terminal — useful to confirm startup or debug connection errors
# Press Ctrl+C to stop following; the container keeps running
docker compose logs -f postgres

# ── Teardown ───────────────────────────────────────────────────────────────────

# Stop and remove the container — your data is SAFE (the named volume persists)
docker compose down

# Stop, remove the container AND wipe all database data — use when you want a clean slate
# Warning: this deletes every table, row, and migration history stored in the volume
docker compose down -v

# View all Docker volumes (confirm postgres_data still exists after "docker compose down")
docker volume ls
```

> **Windows users — Docker Desktop**
>
> - **Install Docker Desktop for Windows** from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop). Enable the WSL 2 backend when prompted — it is significantly faster than Hyper-V for Linux containers.
> - **Ensure Docker Desktop is running** before any `docker compose` command. Look for the whale icon in the system tray. If absent, launch Docker Desktop from the Start menu.
> - **Use PowerShell or Windows Terminal** — all `docker compose` commands work identically. Avoid Git Bash for interactive commands like `docker exec -it ... psql`, as terminal emulation can cause display issues.
> - **Named volumes are WSL 2-managed** — `postgres_data` lives inside the WSL 2 VM. You do not need to worry about Windows path separators.
> - **Port conflicts:** If port 5432 is already in use (e.g., a local PostgreSQL installation), stop the local service first, or change the host port to `"127.0.0.1:5433:5432"` and update your connection string `Port` to `5433`.

### Connection string

The connection string is split across two files. `appsettings.json` contains a safe-to-commit placeholder; `appsettings.Development.json` (gitignored) overrides it with the real password for local dev.

**File:** `src/Nastart.API/appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=recipe_cost_dev;Username=dev;Password=PLACEHOLDER"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

**File:** `src/Nastart.API/appsettings.Development.json` ← **gitignored, never committed**

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=recipe_cost_dev;Username=dev;Password=dev_password"
  }
}
```

> **How ASP.NET Core merges these:** At startup, .NET loads `appsettings.json` first, then `appsettings.Development.json` on top. The `Development` file overrides any matching key — so `DefaultConnection` gets the real password at runtime without ever existing in a committed file.
>
> The password in `appsettings.Development.json` must match what is in `secrets/db_password.txt`. Both files are gitignored. To change the password: update both files and restart the container with `docker compose down && docker compose up -d`.
>
> **For production:** Set the `ConnectionStrings__DefaultConnection` environment variable on the server (or use a secrets manager). `appsettings.Development.json` is never deployed.

---

## 2. Install NuGet Packages

```bash
# Infrastructure needs EF Core + Npgsql (PostgreSQL provider)
dotnet add src/Nastart.Infrastructure package Microsoft.EntityFrameworkCore
dotnet add src/Nastart.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL

# API needs the EF Core design-time tools for migrations
dotnet add src/Nastart.API package Microsoft.EntityFrameworkCore.Design

# Application needs EF Core for DbSet<T> reference in IAppDbContext
dotnet add src/Nastart.Application package Microsoft.EntityFrameworkCore
```

Install the EF Core CLI tool globally (if not already):

```bash
dotnet tool install --global dotnet-ef
```

---

## 3. Domain Enums

Before creating entities, define the enums they depend on.

> ⚠️ **v2-only — do not build in v1.** Roles are introduced when the product expands to multi-user teams.

**File:** `src/Nastart.Domain/Enums/Role.cs`

```csharp
namespace Nastart.Domain.Enums;

// Canonical Decision C-8: Exactly 4 roles — no Admin, no Manager, no Superuser
public enum Role
{
    Owner,
    Chef,
    Procurement,
    Viewer
}
```

**File:** `src/Nastart.Domain/Enums/TelegramLinkStatus.cs`

```csharp
namespace Nastart.Domain.Enums;

// Canonical Decision C-6: TelegramLink lifecycle states
public enum TelegramLinkStatus
{
    Pending,
    Confirmed,
    Unlinked
}
```

> ⚠️ **v2-only — do not build in v1.** Invitations do not exist in the single-user v1 product.

**File:** `src/Nastart.Domain/Enums/InvitationStatus.cs`

```csharp
namespace Nastart.Domain.Enums;

public enum InvitationStatus
{
    Pending,
    Used,
    Expired
}
```

**File:** `src/Nastart.Domain/Enums/PriceSource.cs`

```csharp
namespace Nastart.Domain.Enums;

// Canonical Decision C-13: Exactly these two values, case-sensitive
// EF Core will store these as strings: "Manual", "InvoiceScan"
public enum PriceSource
{
    Manual,
    InvoiceScan
}
```

**File:** `src/Nastart.Domain/Enums/InvoiceStatus.cs`

```csharp
namespace Nastart.Domain.Enums;

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

Each entity inherits from `BaseEntity` (created in L1). All live in `src/Nastart.Domain/Entities/`.

### Company

> ⚠️ **v2-only — do not build in v1.** Company hierarchy is introduced when the product expands to multi-outlet teams.

**File:** `src/Nastart.Domain/Entities/Company.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

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

**File:** `src/Nastart.Domain/Entities/Outlet.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

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

**File:** `src/Nastart.Domain/Entities/User.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

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

**File:** `src/Nastart.Domain/Entities/OutletUser.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;
using Nastart.Domain.Enums;

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

**File:** `src/Nastart.Domain/Entities/Invitation.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;
using Nastart.Domain.Enums;

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

**File:** `src/Nastart.Domain/Entities/TelegramLink.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;
using Nastart.Domain.Enums;

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

**File:** `src/Nastart.Domain/Entities/Unit.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

public class Unit : BaseEntity
{
    public string Name { get; set; } = string.Empty;       // e.g. "kilogram", "litre", "piece"
    public string Abbreviation { get; set; } = string.Empty; // e.g. "kg", "L", "pc"
}
```

> **Design decision — Unit table scope:**
> `Unit` has no `UserId`. This makes units a shared system catalogue (e.g., "kg", "L", "piece") seeded at startup and shared by all users. In v1 (single user), this is functionally equivalent to user-scoped.
> In v2, if users need custom units, add a nullable `UserId` column with a partial unique index: `(name) WHERE user_id IS NULL` for system units and `(user_id, name)` for user-defined units.
> **v1 action required:** Seed the standard unit catalogue in a migration or startup service.

### Category

**File:** `src/Nastart.Domain/Entities/Category.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

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

**File:** `src/Nastart.Domain/Entities/Supplier.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

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

**File:** `src/Nastart.Domain/Entities/Ingredient.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;

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

**File:** `src/Nastart.Domain/Entities/IngredientPriceHistory.cs`

```csharp
namespace Nastart.Domain.Entities;

using Nastart.Domain.Common;
using Nastart.Domain.Enums;

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

**File:** `src/Nastart.Infrastructure/Persistence/AppDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Common;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence;

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
    // Added in L7 — entities and configurations are defined in that lesson.
    // Comment these out until you reach L7, or EF Core will fail to build the model.
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

**File:** `src/Nastart.Application/Common/Interfaces/IAppDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Nastart.Domain.Entities;

namespace Nastart.Application.Common.Interfaces;

public interface IAppDbContext
{
    DbSet<User> Users { get; }
    DbSet<TelegramLink> TelegramLinks { get; }
    DbSet<Ingredient> Ingredients { get; }
    DbSet<IngredientPriceHistory> IngredientPriceHistories { get; }
    DbSet<Unit> Units { get; }
    DbSet<Category> Categories { get; }
    // Added in L7 — comment these out until you reach L7.
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
> `FromSqlInterpolated` binds parameters automatically. `FromSqlRaw` with string interpolation is a SQL injection risk.
> Always select specific columns — `SELECT *` defeats covering indexes:
> ```csharp
> // Safe — specific columns selected, parameters bound by EF Core
> var results = await _db.Ingredients
>     .FromSqlInterpolated($"SELECT id, name, unit_size FROM ingredients WHERE user_id = {userId}")
>     .AsNoTracking()
>     .ToListAsync(ct);
>
> // NEVER — SQL injection risk
> // .FromSqlRaw($"SELECT * FROM ingredients WHERE user_id = '{userId}'")
> ```
>
> **6. Always guard list queries with `Take()` — never return unbounded results**
> For a single user with hundreds of ingredients, an unbounded `ToListAsync()` is safe today but
> will degrade as data grows. Add a maximum page size to every list endpoint:
> ```csharp
> // Bad — returns every row for this user unconditionally
> var all = await _db.Ingredients
>     .Where(i => i.UserId == userId)
>     .AsNoTracking()
>     .ToListAsync(ct);
>
> // Good — cursor-based pagination (preferred) or at minimum a safety cap
> var page = await _db.Ingredients
>     .Where(i => i.UserId == userId && i.CreatedAt < cursor)
>     .OrderByDescending(i => i.CreatedAt)
>     .Take(50)   // ← cap prevents runaway queries
>     .AsNoTracking()
>     .ToListAsync(ct);
> ```
>
> **7. Fetching current price in a list — use LINQ projection, not a loop**
> C-3 says current price is always `ORDER BY committed_at DESC LIMIT 1` from `ingredient_price_histories` — never stored on `Ingredient`. A naive foreach loop fires one query per ingredient (N+1). Use a LINQ projection instead — EF Core / Npgsql translates it into a single SQL statement using the `(ingredient_id, committed_at DESC)` index:
> ```csharp
> // BAD — N+1: 500 ingredients = 500 separate price queries
> var ingredients = await _db.Ingredients.Where(i => i.UserId == userId).ToListAsync(ct);
> foreach (var i in ingredients)
>     i.CurrentPrice = await _db.IngredientPriceHistories   // ← one query per row!
>         .Where(p => p.IngredientId == i.Id)
>         .OrderByDescending(p => p.CommittedAt)
>         .Select(p => p.Price)
>         .FirstOrDefaultAsync(ct);
>
> // GOOD — single SQL with indexed LIMIT 1 subselect per row (no schema change, C-3 preserved)
> var ingredients = await _db.Ingredients
>     .AsNoTracking()
>     .Where(i => i.UserId == userId)
>     .Select(i => new IngredientListDto
>     {
>         Id           = i.Id,
>         Name         = i.Name,
>         UnitSize     = i.UnitSize,
>         CurrentPrice = i.PriceHistory  // ← EF generates: ORDER BY committed_at DESC LIMIT 1
>             .OrderByDescending(p => p.CommittedAt)
>             .Select(p => (decimal?)p.Price)
>             .FirstOrDefault(),
>         CommittedAt  = i.PriceHistory
>             .OrderByDescending(p => p.CommittedAt)
>             .Select(p => (DateTimeOffset?)p.CommittedAt)
>             .FirstOrDefault()
>     })
>     .ToListAsync(ct);
> // The composite index ix_ingredient_price_history_ingredient_committed covers this exactly.
> // C-3 is preserved: current_price is never stored on Ingredient.
> ```

---

## 7. Fluent API Configurations

EF Core's Fluent API lets you configure columns, constraints, and indexes separately from the entity class. Create one configuration file per entity in Infrastructure:

```bash
mkdir -p src/Nastart.Infrastructure/Persistence/Configurations
```

### Key configurations (not every entity — just the ones with important constraints)

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/UserConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasIndex(u => u.Email).IsUnique()
            .HasDatabaseName("users_email_idx");
        builder.Property(u => u.Email).HasMaxLength(255).IsRequired();
        builder.Property(u => u.Name).HasMaxLength(255).IsRequired();
        builder.Property(u => u.PasswordHash).HasMaxLength(255).IsRequired();
    }
}
```

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/OutletUserConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

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

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/TelegramLinkConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Infrastructure.Persistence.Configurations;

public class TelegramLinkConfiguration : IEntityTypeConfiguration<TelegramLink>
{
    public void Configure(EntityTypeBuilder<TelegramLink> builder)
    {
        // C-6: SHA-256 hash is always exactly 64 hex characters
        builder.Property(t => t.CodeHash).HasMaxLength(64).IsRequired();

        // C-14: Column name is telegram_user_id.
        // Note: HasColumnName() is no longer needed once UseSnakeCaseNamingConvention() is active —
        // EF will generate snake_case automatically. Kept here as an explicit documentation anchor
        // for the canonical decision; it is a no-op in practice when the convention is applied.
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
        // HasDatabaseName() ensures predictable snake_case index names in migration diffs.
        builder.HasIndex(t => t.CodeHash).IsUnique()
            .HasDatabaseName("telegram_links_code_hash_idx");
        builder.HasIndex(t => new { t.UserId, t.Status })
            .HasDatabaseName("telegram_links_user_id_status_idx");

        builder.HasOne(t => t.User)
            .WithMany(u => u.TelegramLinks)
            .HasForeignKey(t => t.UserId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/IngredientConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

public class IngredientConfiguration : IEntityTypeConfiguration<Ingredient>
{
    public void Configure(EntityTypeBuilder<Ingredient> builder)
    {
        // Ingredient name is unique within a user's scope
        builder.HasIndex(i => new { i.UserId, i.Name }).IsUnique()
            .HasDatabaseName("ingredients_user_id_name_idx");

        builder.Property(i => i.Name).HasMaxLength(255).IsRequired();
        builder.Property(i => i.UnitSize).HasPrecision(10, 4);
        builder.Property(i => i.PriceSpikeThresholdPct).HasPrecision(5, 2);

        // P1: CHECK constraints enforce data integrity at the database layer.
        // FluentValidation enforces these at the API boundary — the DB check is a
        // defence-in-depth guard against direct SQL inserts or migration seed errors.
        builder.HasCheckConstraint("ck_ingredient_unit_size_positive", "unit_size > 0");

        builder.HasOne(i => i.User)
            .WithMany(u => u.Ingredients)
            .HasForeignKey(i => i.UserId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasOne(i => i.Category)
            .WithMany(c => c.Ingredients)
            .HasForeignKey(i => i.CategoryId)
            .OnDelete(DeleteBehavior.SetNull);

        // P1: PostgreSQL does NOT auto-create indexes on FK columns.
        // Without this, every JOIN or filter on category_id does a full table scan.
        builder.HasIndex(i => i.CategoryId)
            .HasDatabaseName("ingredients_category_id_idx");

        builder.HasOne(i => i.Unit)
            .WithMany()
            .HasForeignKey(i => i.UnitId)
            .OnDelete(DeleteBehavior.Restrict);

        // P1: Index on unit_id — same reason as category_id above.
        builder.HasIndex(i => i.UnitId)
            .HasDatabaseName("ingredients_unit_id_idx");
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

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/IngredientPriceHistoryConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

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

        // ⚠️ Defer to L12 — InvoiceLineItem entity is built in Phase 3 (L12–L14).
        // Un-comment this block when you add the InvoiceLineItem entity in L12.
        // C-5: When an InvoiceLineItem is deleted, its price records are preserved (the FK is nulled).
        //
        // builder.HasOne<InvoiceLineItem>()
        //     .WithMany()
        //     .HasForeignKey(p => p.InvoiceLineItemId)
        //     .OnDelete(DeleteBehavior.SetNull)
        //     .IsRequired(false);

        // P1: Partial index on the nullable FK — skips the majority of NULL rows,
        // so Phase 3 invoice lookups remain fast without bloating the index.
        builder.HasIndex(p => p.InvoiceLineItemId)
            .HasFilter("invoice_line_item_id IS NOT NULL")
            .HasDatabaseName("ingredient_price_histories_invoice_line_item_id_idx");

        // P1: CHECK constraint — price must be positive.
        // Defense-in-depth: FluentValidation also enforces this at the API boundary.
        builder.HasCheckConstraint("ck_ingredient_price_history_price_positive", "price > 0");
        builder.HasCheckConstraint("ck_ingredient_price_history_unit_size_positive", "unit_size > 0");
    }
}
```

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/CategoryConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

public class CategoryConfiguration : IEntityTypeConfiguration<Category>
{
    public void Configure(EntityTypeBuilder<Category> builder)
    {
        builder.Property(c => c.Name).HasMaxLength(255).IsRequired();

        // Category name is unique within a user's scope
        builder.HasIndex(c => new { c.UserId, c.Name }).IsUnique()
            .HasDatabaseName("categories_user_id_name_idx");

        builder.HasOne(c => c.User)
            .WithMany()
            .HasForeignKey(c => c.UserId)
            .OnDelete(DeleteBehavior.Cascade);
    }
}
```

> ⚠️ **v2-only — do not build in v1.**

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/InvitationConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

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

**File:** `src/Nastart.Infrastructure/Persistence/Configurations/SupplierConfiguration.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nastart.Domain.Entities;

namespace Nastart.Infrastructure.Persistence.Configurations;

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

**File:** `src/Nastart.Infrastructure/DependencyInjection.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Nastart.Application.Common.Interfaces;
using Nastart.Infrastructure.Persistence;

namespace Nastart.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructure(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<AppDbContext>(options =>
            options.UseNpgsql(
                configuration.GetConnectionString("DefaultConnection"),
                npgsqlOptions => npgsqlOptions.MigrationsAssembly(typeof(AppDbContext).Assembly.FullName)
            )
            // P0: Maps C# PascalCase to PostgreSQL snake_case for all table and column names.
            // Without this, EF generates quoted "PascalCase" names that break raw SQL and
            // psql introspection. Must be set BEFORE the first migration is generated.
            .UseSnakeCaseNamingConvention()
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

**File:** `src/Nastart.API/Program.cs`

```csharp
using Nastart.Infrastructure;

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
  --project src/Nastart.Infrastructure \
  --startup-project src/Nastart.API \
  --output-dir Persistence/Migrations

# Apply the migration to the database
dotnet ef database update \
  --project src/Nastart.Infrastructure \
  --startup-project src/Nastart.API
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

**File:** `src/Nastart.Application/Common/Interfaces/IEmailService.cs`

```csharp
namespace Nastart.Application.Common.Interfaces;

public interface IEmailService
{
    Task SendAsync(string to, string subject, string body);
}
```

**File:** `src/Nastart.Infrastructure/Services/ConsoleEmailService.cs`

```csharp
using Microsoft.Extensions.Logging;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Infrastructure.Services;

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
# 1. Docker Postgres is running and HEALTHY
#    Wait for Status to show "healthy" — this confirms pg_isready passed and the DB accepts connections
#    If it shows "starting", wait a few more seconds and re-run
docker compose ps

# 2. Solution builds
dotnet build

# 3. Migration applied without errors
#    If this fails with "connection refused", Postgres isn't ready yet — wait for "healthy" first
dotnet ef database update \
  --project src/Nastart.Infrastructure \
  --startup-project src/Nastart.API

# 4. Tables exist in PostgreSQL
#    Opens a psql shell inside the running container and runs "\dt" to list tables
docker exec -it $(docker compose ps -q postgres) psql -U dev -d recipe_cost_dev -c "\dt"
# Should show 9 tables (plus __EFMigrationsHistory)

# 5. API runs
dotnet run --project src/Nastart.API
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
| Recipe, RecipeItem | Phase 2 | L7–L8 |
| CascadeErrorLog | Phase 2 | L7 |
| Invoice, InvoiceLineItem, ReviewQueueItem | Phase 3 | L12 |
