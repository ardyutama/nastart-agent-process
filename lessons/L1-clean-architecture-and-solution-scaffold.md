# Lesson 1 — Clean Architecture, Dependency Injection & Solution Scaffold

> **What you'll learn:**
> - What Clean Architecture is and why it matters for a production .NET app
> - The 4-ring dependency rule and which code goes where
> - Dependency Injection — the pattern that powers everything in .NET
> - How to scaffold a multi-project .NET 10 solution from scratch

---

## 1. Clean Architecture — The 4 Rings

Clean Architecture organizes your code into layers (rings). The fundamental rule:

> **Dependencies point inward.** Outer rings depend on inner rings. Inner rings know NOTHING about outer rings.

```
┌─────────────────────────────────────────────┐
│  API  (outermost — HTTP, Program.cs, DI)    │
│  ┌─────────────────────────────────────────┐│
│  │  Infrastructure (EF Core, email, S3)    ││
│  │  ┌─────────────────────────────────────┐││
│  │  │  Application (handlers, DTOs, CQRS) │││
│  │  │  ┌─────────────────────────────────┐│││
│  │  │  │  Domain (entities, interfaces)  ││││
│  │  │  └─────────────────────────────────┘│││
│  │  └─────────────────────────────────────┘││
│  └─────────────────────────────────────────┘│
└─────────────────────────────────────────────┘
```

### What goes where

| Project | Contains | May Reference |
|---|---|---|
| `Nastart.Domain` | Entity classes, enums, value objects, domain interfaces (e.g. `IIngredientRepository`) | Nothing — this is the innermost ring |
| `Nastart.Application` | Feature slices (Commands, Queries, Handlers), DTOs, service interfaces (e.g. `ICostCascadeService`), validation rules | Domain only |
| `Nastart.Infrastructure` | EF Core `DbContext`, repository implementations, external service clients (email, file storage), service implementations | Domain + Application |
| `Nastart.API` | `Program.cs`, Minimal API endpoint definitions, middleware, DI wiring, configuration | Domain + Application + Infrastructure |

### Why this matters for your recipe costing app

Your `Domain` project defines `Ingredient`, `Recipe`, `User` etc. — pure C# classes with zero dependency on Entity Framework, HTTP, or any library. This means:

- You can test domain logic without a database
- You can swap PostgreSQL for another database by replacing `Infrastructure` only
- Your `Application` handlers contain business logic that works regardless of which framework serves HTTP requests

---

## 2. Dependency Injection — The Pattern Behind Everything

Every lesson from L2 onward uses Dependency Injection (DI). Let's understand it before we need it.

### The problem DI solves

Without DI, a class creates its own dependencies:

```csharp
// BAD — tightly coupled
public class IngredientService
{
    private readonly PostgresDb _db;

    public IngredientService()
    {
        // This class decides HOW to connect to the database
        _db = new PostgresDb("Host=localhost;Database=recipe...");
    }
}
```

Problems:
- Can't test without a real database
- Can't swap PostgresDb for a different implementation
- Connection string is buried inside the class

### DI flips control — the framework provides the dependency

```csharp
// GOOD — dependency injected via constructor
public class IngredientService
{
    private readonly IAppDbContext _db;

    // The class declares WHAT it needs, not HOW to create it
    public IngredientService(IAppDbContext db)
    {
        _db = db;
    }
}
```

Now `IngredientService` doesn't know or care if `_db` is PostgreSQL, SQLite, or a mock. It receives whatever the DI container gives it.

### How .NET's DI container works

In `Program.cs`, you register services into a container. When a class needs a dependency, .NET creates it automatically:

```csharp
var builder = WebApplication.CreateBuilder(args);

// REGISTERING services — tell the container "when someone asks for X, give them Y"
builder.Services.AddScoped<IAppDbContext, AppDbContext>();     // Per-request
builder.Services.AddSingleton<IPasswordHasher, BcryptHasher>(); // One instance for the app's lifetime
builder.Services.AddTransient<IEmailService, ConsoleEmailService>(); // New instance every time

var app = builder.Build();
```

### Three lifetimes you need to know

| Lifetime | When a new instance is created | Use for |
|---|---|---|
| `AddScoped` | Once per HTTP request | DbContext, handlers — stateful per request |
| `AddSingleton` | Once for the entire app | Configuration, caches — stateless and thread-safe |
| `AddTransient` | Every time it's requested | Lightweight, stateless helpers |

**Rule of thumb for this project:** DbContext and handlers are `Scoped`. Most services are `Scoped`. Only use `Singleton` when you're sure the service is thread-safe and stateless.

### Why this matters for the next lessons

- **L2** uses `builder.Services.AddDbContext<AppDbContext>(...)` — that's DI registration
- **L3** uses `builder.Services.AddMediatR(...)` — DI auto-discovers all handlers
- **L4** uses `builder.Services.AddValidatorsFromAssembly(...)` — DI auto-discovers validators
- **L5** uses `builder.Services.AddAuthentication(...)` — DI registers JWT middleware

Every `Add*()` call is registering a service into the DI container. Now you know what that means.

---

## 3. Scaffold the Solution

### Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download) installed
- A terminal (VS Code integrated terminal works)
- PostgreSQL (we'll set this up with Docker in L2)

### Create the solution and projects

```bash
# Create solution folder
mkdir nastart && cd nastart

# Create the solution file
dotnet new sln -n Nastart

# Create the 4 projects
dotnet new classlib -n Nastart.Domain -o src/Nastart.Domain --framework net10.0
dotnet new classlib -n Nastart.Application -o src/Nastart.Application --framework net10.0
dotnet new classlib -n Nastart.Infrastructure -o src/Nastart.Infrastructure --framework net10.0
dotnet new webapi -n Nastart.API -o src/Nastart.API --framework net10.0

# Add all projects to the solution
dotnet sln add src/Nastart.Domain
dotnet sln add src/Nastart.Application
dotnet sln add src/Nastart.Infrastructure
dotnet sln add src/Nastart.API

# Create the test project (TDD is a project requirement — tests live here)
dotnet new xunit -n Nastart.Tests -o tests/Nastart.Tests --framework net10.0
dotnet sln add tests/Nastart.Tests

# Test project references Application (most unit tests hit handlers and services)
dotnet add tests/Nastart.Tests reference src/Nastart.Application
dotnet add tests/Nastart.Tests reference src/Nastart.Domain

# Add test helper packages
dotnet add tests/Nastart.Tests package AwesomeAssertions
dotnet add tests/Nastart.Tests package NSubstitute
```

> **Test framework — xUnit vs MSTest.Sdk:** This project uses **xUnit**. For new .NET projects, the current best-practice recommendation is **MSTest.Sdk** (`<Sdk Name="MSTest.Sdk">`): it requires fewer package references, supports sealed test classes for performance, and provides richer built-in assertions (`Assert.ThrowsExactly`, `Assert.HasCount`, `Assert.ContainsSingle`) without an additional library. xUnit is kept here because it pairs naturally with AwesomeAssertions and NSubstitute, which are already part of this setup. If you prefer MSTest, replace `dotnet new xunit` with `dotnet new mstest`, swap `[Fact]` for `[TestMethod]`/`[TestClass]`, and remove the AwesomeAssertions package in favour of MSTest's native assertions. Both frameworks have full `dotnet test`, VS Test Explorer, and GitHub Actions support.
> **AwesomeAssertions:** `result.Should().Be(expected)` — readable assertion syntax (used with xUnit in this project). A free, MIT-licensed fork of FluentAssertions with an identical API.
> **NSubstitute:** `Substitute.For<IAppDbContext>()` — creates mock implementations of interfaces for unit testing without a real database.

### Set up project references (the dependency rule)

This is where Clean Architecture is enforced. References ONLY point inward:

```bash
# Application depends on Domain
dotnet add src/Nastart.Application reference src/Nastart.Domain

# Infrastructure depends on Domain + Application
dotnet add src/Nastart.Infrastructure reference src/Nastart.Domain
dotnet add src/Nastart.Infrastructure reference src/Nastart.Application

# API depends on all (it's the outermost ring — it wires everything)
dotnet add src/Nastart.API reference src/Nastart.Domain
dotnet add src/Nastart.API reference src/Nastart.Application
dotnet add src/Nastart.API reference src/Nastart.Infrastructure
```

**What you must NEVER do:** Add a reference from Domain to Application, Infrastructure, or API. Domain references nothing.

### Verify the structure

```
nastart/
├── Nastart.sln
├── src/
│   ├── Nastart.Domain/            ← innermost ring
│   │   └── Nastart.Domain.csproj
│   ├── Nastart.Application/       ← business logic ring
│   │   └── Nastart.Application.csproj
│   ├── Nastart.Infrastructure/    ← data access ring
│   │   └── Nastart.Infrastructure.csproj
│   └── Nastart.API/               ← outermost ring
│       ├── Nastart.API.csproj
│       └── Program.cs
└── tests/
    └── Nastart.Tests/             ← unit tests (xUnit + AwesomeAssertions + NSubstitute)
        └── Nastart.Tests.csproj
```

### Clean up template files

Delete the auto-generated `Class1.cs` from the class library projects and the template weather forecast code from API:

```bash
rm src/Nastart.Domain/Class1.cs
rm src/Nastart.Application/Class1.cs
rm src/Nastart.Infrastructure/Class1.cs
```

In `src/Nastart.API/Program.cs`, replace the template content with a clean starting point:

```csharp
var builder = WebApplication.CreateBuilder(args);

// Services will be registered here in future lessons

var app = builder.Build();

// Health check — verifies the API is running
app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));

app.Run();
```

### Verify it builds and runs

```bash
# Build entire solution
dotnet build

# Run the API project
dotnet run --project src/Nastart.API

# In another terminal, test the health endpoint
curl http://localhost:5000/health
# Expected: {"status":"healthy"}
```

---

## 3.5. Pin the SDK and Share Build Settings Across All Projects

### Create global.json to pin the .NET SDK version

**File:** `global.json` (project root, next to `Nastart.sln`)

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestMinor"
  }
}
```

Without `global.json`, a developer with .NET 9 installed will scaffold `net9.0` by default, causing subtle compatibility issues. Pin the version explicitly.

> **Update the version number** when a newer .NET 10 SDK is released (e.g., 10.0.200, 10.0.300). Use `dotnet --version` to check your installed SDK and keep this file in sync.

### Create Directory.Build.props to share settings across all 4 projects

**File:** `Directory.Build.props` (project root, next to `Nastart.sln`)

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <LangVersion>latest</LangVersion>
  </PropertyGroup>
</Project>
```

**What this does:**
- `Nullable>enable` — all 4 projects get nullable reference types enabled. A string property that isn't marked `string?` is guaranteed non-null at compile time.
- `ImplicitUsings>enable` — `System`, `System.Collections.Generic`, `System.Linq` etc. are available everywhere without explicit `using` statements.
- `TreatWarningsAsErrors>true` — warnings become errors. Catches null dereference risks at build time, not runtime.
- `LangVersion>latest` — always target the latest C# language features for the installed SDK.

> **Important:** Because you have `Directory.Build.props` at the root, you can remove `<TargetFramework>net10.0</TargetFramework>` from each individual `.csproj` file — it is inherited automatically. The `dotnet new` commands we ran will include it in each `.csproj`; you can simplify those files once the solution is building.

---

## 4. Folder Structure Convention (for all future lessons)

Set up these folders now — they'll be populated in the coming lessons:

### Domain project

```bash
mkdir -p src/Nastart.Domain/Entities
mkdir -p src/Nastart.Domain/Enums
mkdir -p src/Nastart.Domain/Common
```

- `Entities/` — your C# entity classes (`Ingredient.cs`, `User.cs`, etc.)
- `Enums/` — `TelegramLinkStatus`, `PriceSource`, `InvoiceStatus` enums (no `Role` enum in v1 — that is v2-only)
- `Common/` — base entity class shared by all entities

### Application project

```bash
mkdir -p src/Nastart.Application/Common/Interfaces
mkdir -p src/Nastart.Application/Common/Behaviors
```

- `Common/Interfaces/` — service interfaces like `IAppDbContext`
- `Common/Behaviors/` — MediatR pipeline behaviors (validation, logging)
- Feature folders will be added as we build slices in L3+

### Infrastructure project

```bash
mkdir -p src/Nastart.Infrastructure/Persistence
mkdir -p src/Nastart.Infrastructure/Services
```

- `Persistence/` — `AppDbContext`, entity configurations, migrations
- `Services/` — implementations of interfaces defined in Application

### API project

```bash
mkdir -p src/Nastart.API/Endpoints
mkdir -p src/Nastart.API/Extensions
```

- `Endpoints/` — Minimal API endpoint groups
- `Extensions/` — DI registration extension methods (keeps Program.cs clean)

---

## 5. The Base Entity

Every entity in our database shares common fields. Create a base class:

**File:** `src/Nastart.Domain/Common/BaseEntity.cs`

```csharp
namespace Nastart.Domain.Common;

public abstract class BaseEntity
{
    // init: ID is set at construction time and immutable afterward.
    // Guid.CreateVersion7() generates time-ordered UUIDs \u2014 no B-tree page splits on INSERT.
    public Guid Id { get; init; } = Guid.CreateVersion7();
    // public set: AppDbContext.ApplyAuditTimestamps() (in Nastart.Infrastructure) assigns these.
    // Cross-assembly write requires public access. Handlers must never assign these directly —
    // that convention is enforced by code review, not by the language, in v1.
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? UpdatedAt { get; set; }
}
```

All entities will inherit from this. EF Core will map `Id` as the primary key, and `CreatedAt`/`UpdatedAt` will be set automatically (configured in L2's DbContext).

> **Why `Guid.CreateVersion7()`:** UUID v7 is time-ordered — new rows always sort to the end of the database B-tree index, eliminating the random page splits that UUID v4 causes on every insert. This is a built-in .NET 9+ API (no packages needed). The ID is generated in C# at instantiation time, so it's available before the entity is saved to the database.

---

## 6. Your First Unit Test — Verifying the Test Setup

Before writing handlers in L3+, verify the test infrastructure works. Create one smoke test to confirm xUnit, NSubstitute, and AwesomeAssertions are correctly wired together.

```bash
mkdir tests/Nastart.Tests/Smoke
```

**File:** `tests/Nastart.Tests/Smoke/InfrastructureSmokeTest.cs`

```csharp
using AwesomeAssertions;
using NSubstitute;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Tests.Smoke;

public class InfrastructureSmokeTest
{
    [Fact]
    public void TestInfrastructure_ShouldWork()
    {
        // Arrange — NSubstitute creates a mock of IAppDbContext
        var db = Substitute.For<IAppDbContext>();

        // Assert — AwesomeAssertions verifies the mock is not null
        db.Should().NotBeNull(
            "NSubstitute and AwesomeAssertions packages are installed correctly");
    }
}
```

```bash
dotnet test tests/Nastart.Tests
# Expected output includes: "1 passed"
# Test name: Nastart.Tests.Smoke.InfrastructureSmokeTest.TestInfrastructure_ShouldWork
```

> **What this proves:** xUnit discovered the test, NSubstitute created a mock of `IAppDbContext`, and AwesomeAssertions made an assertion without crashing. TDD infrastructure is ready.

> **Test-first rule from L3 onward:** Every feature handler gets a failing test written BEFORE the handler code. This smoke test confirms the harness works so you can follow that rule from the first slice.

---

## 7. The IAppDbContext Interface

This interface lives in Application (not Infrastructure) so that handlers can depend on it without knowing about EF Core:

**File:** `src/Nastart.Application/Common/Interfaces/IAppDbContext.cs`

```csharp
using Microsoft.EntityFrameworkCore;
using Nastart.Domain.Entities;

namespace Nastart.Application.Common.Interfaces;

// Handlers depend on this interface — they never see AppDbContext directly.
// This keeps the Application layer free of Infrastructure concerns.
public interface IAppDbContext
{
    // These DbSet properties will grow as we add entities in L2
    // DbSet<Ingredient> Ingredients { get; }
    // DbSet<User> Users { get; }

    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

> **Note:** We'll uncomment and add `DbSet<>` properties as entities are created in L2. The interface is defined now so you see where it lives in the architecture.

> **Important:** Yes, `IAppDbContext` references `Microsoft.EntityFrameworkCore` for the `DbSet<T>` type. This is a pragmatic decision — a pure clean architecture would define its own abstraction, but for a real production app, leaking just the `DbSet<T>` type into Application is an industry-accepted tradeoff. The alternative (repository-per-entity) adds significant boilerplate with little benefit for this project's scale.

---

## Checkpoint — Verify Before Moving to L2

Run these checks:

```bash
# 1. Solution builds with zero errors
dotnet build

# 2. API starts and responds
dotnet run --project src/Nastart.API
# curl http://localhost:5000/health → {"status":"healthy"}

# 3. Verify project references are correct
dotnet list src/Nastart.Domain/Nastart.Domain.csproj reference
# Should show: (empty — Domain references nothing)

dotnet list src/Nastart.Application/Nastart.Application.csproj reference
# Should show: Nastart.Domain

dotnet list src/Nastart.Infrastructure/Nastart.Infrastructure.csproj reference
# Should show: Nastart.Domain, Nastart.Application

dotnet list src/Nastart.API/Nastart.API.csproj reference
# Should show: Nastart.Domain, Nastart.Application, Nastart.Infrastructure
```

If all 3 pass, your Clean Architecture foundation is solid. Move to L2.
