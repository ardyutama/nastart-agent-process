# Lesson 5 — JWT Authentication & Authorization

> **What you'll learn:**
> - How JWT (JSON Web Token) authentication works in .NET 10
> - How to build Register and Login slices with password hashing
> - How to issue JWTs with user identity claims (userId, email)
> - How to protect endpoints with `RequireAuthorization()` for authenticated users
>
> **Out of scope for this lesson:**
> - **Telegram bot-secret authentication** — deferred to Phase 4 (L15). The bot calls .NET API endpoints using a shared `X-Bot-Secret` header, not JWT. That service-to-service auth pattern is introduced when the Python bot is built, not here.

> ⚠️ **v1 Solopreneur Amendments (April 9, 2026)**
>
> This lesson was originally written for multi-role, multi-outlet JWT auth. v1 is single-user with no roles. Apply these changes throughout:
>
> **JWT claims — v1 contains ONLY:**
> ```csharp
> new Claim(ClaimTypes.NameIdentifier, userId.ToString()),
> new Claim(ClaimTypes.Email, email)
> // NO outletId claim — no outlets in v1
> // NO role claim — no roles in v1
> ```
>
> **ITokenService signature:**
> ```csharp
> // v1
> public interface ITokenService
> {
>     string GenerateToken(Guid userId, string email);
> }
> ```
>
> **LoginResponse — simplified:**
> ```csharp
> // v1 — no outlet, no role returned
> public record LoginResponse(string Token);
> ```
>
> **LoginHandler — simplified:**
> - Remove `OutletUser` query — no outlet lookup after login
> - After email/password verification: `var token = _tokenService.GenerateToken(user.Id, user.Email);`
> - Return `new LoginResponse(token)`
>
> **Authorization policies — replace all role-based policies:**
> ```csharp
> // v1 — single policy, all authenticated users have the same access
> builder.Services.AddAuthorization(options =>
> {
>     options.FallbackPolicy = new AuthorizationPolicyBuilder()
>         .RequireAuthenticatedUser()
>         .Build();
> });
> // All protected endpoints: .RequireAuthorization()
> // Remove ALL: "OwnerOnly", "OwnerOrChef", "OwnerOrProcurement", "CanManageIngredients"
> ```
>
> **ClaimsPrincipalExtensions — keep only:**
> ```csharp
> public static Guid GetUserId(this ClaimsPrincipal user) { ... }
> public static string GetEmail(this ClaimsPrincipal user) { ... }
> // DELETE: GetOutletId(), GetRole()
> ```
>
> **Do NOT build:**
> - `CreateCompanyCommand` / company onboarding flow — no company in v1
> - Any "setup" endpoint (`POST /api/setup`) — remove entirely
> - The Role enum reference (`C-8`) — canonical decision C-8 is v2-only

---

## 1. JWT — How It Works

A JWT is a signed token that the API gives to the user after login. The user sends it with every request. The API verifies the signature and reads the claims inside.

```
LOGIN:
  Client → POST /api/auth/login { email, password }
  Server → validates credentials → creates JWT → returns token

EVERY REQUEST AFTER:
  Client → GET /api/ingredients  [Header: Authorization: Bearer eyJhbG...]
  Server → reads JWT → extracts userId, email → allows or denies
```

A JWT contains **claims** — key-value pairs about the user:

| Claim | Purpose |
|---|---|
| `sub` (subject) | The user's ID (userId) |
| `email` | The user's email address |
| `exp` | Expiration timestamp |

> **v1 note:** `role` and `outletId` claims do NOT exist in v1. There is one user with one identity. All protected endpoints use `RequireAuthorization()` — no policy name needed.

> ⚠️ **v2-only — do not build the Role enum in v1.** Single-user design has no roles. See amendment block above.

```csharp
namespace Nastart.Domain.Enums;

// Canonical Decision C-8: Exactly 4 roles — no Admin, no Manager, no Superuser
public enum Role
{
    Owner,       // Full access: costs, margins, food_cost_pct, supplier prices, all alerts
    Chef,        // cost_per_portion only — no margins, no food_cost_pct
    Procurement, // cost_per_portion + supplier prices — no food_cost_pct
    Viewer       // cost_per_portion only — read-only
}
```

> ⚠️ **v2-only:** Authorization policies, JWT role claims, and role-masked response DTOs are all v2 features. Do not build any of these in v1.

---

## 2. Install Packages

```bash
# API project needs JWT auth
dotnet add src/Nastart.API package Microsoft.AspNetCore.Authentication.JwtBearer

# Infrastructure needs password hashing
dotnet add src/Nastart.Infrastructure package BCrypt.Net-Next
```

---

## 3. JWT Configuration

Add JWT settings to `appsettings.json`:

**File:** `src/Nastart.API/appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database=recipe_cost_dev;Username=dev;Password=dev_password"
  },
  "Jwt": {
    "Issuer": "Nastart.API",
    "Audience": "Nastart.Client"
    // SecretKey is NOT stored here — it is loaded from user-secrets (dev) or environment variable (prod)
    // See setup instructions below
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

> ⚠️ **Security: Never store `Jwt:SecretKey` in `appsettings.json` or any committed file.**
>
> **Development — use .NET User Secrets:**
> ```bash
> # Run from the API project folder
> cd src/Nastart.API
> dotnet user-secrets init
> dotnet user-secrets set "Jwt:SecretKey" "your-dev-secret-min-32-chars-long-here"
> ```
>
> User secrets are stored in `%APPDATA%\Microsoft\UserSecrets\<guid>\secrets.json` (Windows) or `~/.microsoft/usersecrets/` (macOS/Linux). They are **never committed to git**.
>
> **Production — use environment variables:**
> ```
> Jwt__SecretKey=your-production-secret
> ```
> (Double underscore `__` maps to `:` in ASP.NET Core configuration hierarchy.)
>
> **Required minimum:** The secret must be **at least 32 characters** for HMAC-SHA256. Use a cryptographically random value, not a memorable phrase.

---

## 4. Register Slice — User Sign-Up

```bash
mkdir -p src/Nastart.Application/Features/Auth/Commands/Register
```

**File:** `src/Nastart.Application/Features/Auth/Commands/Register/RegisterCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Auth.Commands.Register;

public record RegisterCommand(
    string Name,
    string Email,
    string Password
) : IRequest<ErrorOr<RegisterResponse>>;
```

**File:** `src/Nastart.Application/Features/Auth/Commands/Register/RegisterResponse.cs`

```csharp
namespace Nastart.Application.Features.Auth.Commands.Register;

public record RegisterResponse(Guid UserId, string Message);
```

**File:** `src/Nastart.Application/Features/Auth/Commands/Register/RegisterCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Auth.Commands.Register;

public class RegisterCommandValidator : AbstractValidator<RegisterCommand>
{
    public RegisterCommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().MaximumLength(255);

        RuleFor(x => x.Email)
            .NotEmpty().EmailAddress().MaximumLength(255);

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8).WithMessage("Password must be at least 8 characters.");
    }
}
```

### Password Hashing Interface

Define in Application (interface), implement in Infrastructure:

**File:** `src/Nastart.Application/Common/Interfaces/IPasswordHasher.cs`

```csharp
namespace Nastart.Application.Common.Interfaces;

public interface IPasswordHasher
{
    string Hash(string password);
    bool Verify(string password, string hash);
}
```

**File:** `src/Nastart.Infrastructure/Services/BcryptPasswordHasher.cs`

```csharp
using Nastart.Application.Common.Interfaces;

namespace Nastart.Infrastructure.Services;

public class BcryptPasswordHasher : IPasswordHasher
{
    public string Hash(string password)
    {
        return BCrypt.Net.BCrypt.HashPassword(password);
    }

    public bool Verify(string password, string hash)
    {
        return BCrypt.Net.BCrypt.Verify(password, hash);
    }
}
```

Register in `DependencyInjection.cs` (Infrastructure):

```csharp
services.AddScoped<IPasswordHasher, BcryptPasswordHasher>();
```

### Register Handler

**File:** `src/Nastart.Application/Features/Auth/Commands/Register/RegisterHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using System.Security.Cryptography;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;

namespace Nastart.Application.Features.Auth.Commands.Register;

public class RegisterHandler : IRequestHandler<RegisterCommand, ErrorOr<RegisterResponse>>
{
    private readonly IAppDbContext _db;
    private readonly IPasswordHasher _hasher;
    private readonly IEmailService _email;

    public RegisterHandler(IAppDbContext db, IPasswordHasher hasher, IEmailService email)
    {
        _db = db;
        _hasher = hasher;
        _email = email;
    }

    public async Task<ErrorOr<RegisterResponse>> Handle(
        RegisterCommand command, CancellationToken ct)
    {
        // Flow 01 Step 2: email uniqueness check
        var emailTaken = await _db.Users
            .AnyAsync(u => u.Email == command.Email.ToLowerInvariant(), ct);

        if (emailTaken)
            return Error.Conflict("User.EmailTaken", "An account with this email already exists.");

        // Flow 01 Step 3: create user with isVerified=false, isActive=false
        var user = new User
        {
            Id = Guid.NewGuid(),
            Name = command.Name,
            Email = command.Email.ToLowerInvariant(),
            PasswordHash = _hasher.Hash(command.Password),
            IsVerified = false,
            IsActive = false
        };

        _db.Users.Add(user);
        await _db.SaveChangesAsync(ct);

        // Flow 01 Step 4: send verification email
        // ConsoleEmailService logs to console in dev — real SMTP in production
        // Security: use RandomNumberGenerator, not Guid.NewGuid() — GUIDs are not
        // cryptographically secure for use as security tokens.
        var verificationToken = Convert.ToHexString(RandomNumberGenerator.GetBytes(32));

        // In a real app, store the token hash for verification lookup
        // For Phase 1, we directly log it for manual testing
        await _email.SendAsync(
            user.Email,
            "Verify your Nastart account",
            $"Click here to verify: /api/auth/verify?token={verificationToken}&userId={user.Id}");

        return new RegisterResponse(user.Id, "Check your email to verify your account.");
    }
}
```

---

## 5. Login Slice — Issue JWT

```bash
mkdir -p src/Nastart.Application/Features/Auth/Commands/Login
```

### Token Generation Interface

**File:** `src/Nastart.Application/Common/Interfaces/ITokenService.cs`

```csharp
namespace Nastart.Application.Common.Interfaces;

public interface ITokenService
{
    string GenerateToken(Guid userId, string email);
}
```

**File:** `src/Nastart.Infrastructure/Services/JwtTokenService.cs`

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.Extensions.Configuration;
using Microsoft.IdentityModel.Tokens;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Infrastructure.Services;

public class JwtTokenService : ITokenService
{
    private readonly IConfiguration _config;

    public JwtTokenService(IConfiguration config)
    {
        _config = config;
    }

    public string GenerateToken(Guid userId, string email)
    {
        var claims = new List<Claim>
        {
            new(ClaimTypes.NameIdentifier, userId.ToString()),
            new(ClaimTypes.Email, email)
        };
        // Token expiry: set to 24 hours
        var expires = DateTime.UtcNow.AddHours(24);

        var secret = _config["Jwt:SecretKey"]
            ?? throw new InvalidOperationException(
                "Jwt:SecretKey is not configured. Set it via user-secrets (dev) or " +
                "Jwt__SecretKey environment variable (production).");

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: expires,
            signingCredentials: creds);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

Register in Infrastructure's `DependencyInjection.cs`:

```csharp
services.AddScoped<ITokenService, JwtTokenService>();
```

### Login Command

**File:** `src/Nastart.Application/Features/Auth/Commands/Login/LoginCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Auth.Commands.Login;

public record LoginCommand(string Email, string Password) : IRequest<ErrorOr<LoginResponse>>;
```

**File:** `src/Nastart.Application/Features/Auth/Commands/Login/LoginResponse.cs`

```csharp
namespace Nastart.Application.Features.Auth.Commands.Login;

// v1: single user — token only, no outlet or role returned
public record LoginResponse(string Token);
```

**File:** `src/Nastart.Application/Features/Auth/Commands/Login/LoginCommandValidator.cs`

```csharp
using FluentValidation;

namespace Nastart.Application.Features.Auth.Commands.Login;

public class LoginCommandValidator : AbstractValidator<LoginCommand>
{
    public LoginCommandValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Password).NotEmpty();
    }
}
```

**File:** `src/Nastart.Application/Features/Auth/Commands/Login/LoginHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Auth.Commands.Login;

public class LoginHandler : IRequestHandler<LoginCommand, ErrorOr<LoginResponse>>
{
    private readonly IAppDbContext _db;
    private readonly IPasswordHasher _hasher;
    private readonly ITokenService _tokenService;

    public LoginHandler(IAppDbContext db, IPasswordHasher hasher, ITokenService tokenService)
    {
        _db = db;
        _hasher = hasher;
        _tokenService = tokenService;
    }

    public async Task<ErrorOr<LoginResponse>> Handle(
        LoginCommand command, CancellationToken ct)
    {
        // Find user by email
        var user = await _db.Users
            .FirstOrDefaultAsync(u => u.Email == command.Email.ToLowerInvariant(), ct);

        if (user is null)
            return Error.Unauthorized("Auth.InvalidCredentials", "Invalid email or password.");

        // Verify password
        if (!_hasher.Verify(command.Password, user.PasswordHash))
            return Error.Unauthorized("Auth.InvalidCredentials", "Invalid email or password.");

        // flow 01 step 7 guard: must be verified AND active before issuing JWT
        // Security: use IDENTICAL error codes and messages for both guards.
        // Revealing WHICH check failed (not verified vs deactivated) is a user
        // enumeration risk — attackers could probe account state via different error text.
        if (!user.IsVerified)
            return Error.Unauthorized("Auth.InvalidCredentials", "Invalid email or password.");

        if (!user.IsActive)
            return Error.Unauthorized("Auth.InvalidCredentials", "Invalid email or password.");

        var token = _tokenService.GenerateToken(user.Id, user.Email);
        return new LoginResponse(token);
    }
}
```

---

## 6. Auth Endpoints

**File:** `src/Nastart.API/Endpoints/AuthEndpoints.cs`

```csharp
using MediatR;
using Nastart.API.Extensions;
using Nastart.Application.Features.Auth.Commands.Login;
using Nastart.Application.Features.Auth.Commands.Register;

namespace Nastart.API.Endpoints;

public static class AuthEndpoints
{
    public static void MapAuthEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/auth")
            .WithTags("Authentication");

        // Public — no auth required
        group.MapPost("/register", async (RegisterCommand command, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.Send(command, ct);
            return result.ToCreatedResult($"/api/users/{result.Value?.UserId}");
        });

        group.MapPost("/login", async (LoginCommand command, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.Send(command, ct);
            return result.ToApiResult();
        });
    }
}
```

---

## 7. Wire JWT Authentication in Program.cs

**File:** `src/Nastart.API/Program.cs`

```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using Nastart.Application;
using Nastart.Infrastructure;
using Nastart.API.Endpoints;
using Nastart.API.Middleware;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration, builder.Environment);
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// JWT Authentication setup
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidAudience = builder.Configuration["Jwt:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(
                    builder.Configuration["Jwt:SecretKey"]
                    ?? throw new InvalidOperationException(
                        "Jwt:SecretKey is not configured. Set via user-secrets or environment variable.")))
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseExceptionHandler();

// Authentication middleware must come before Authorization
app.UseAuthentication();
app.UseAuthorization();

// Public endpoints
app.MapGet("/health", () => Results.Ok(new { status = "healthy" }));
app.MapAuthEndpoints();

// Protected endpoints — require authentication
// app.MapOutletEndpoints(); // v2-only — remove this line; no outlets in v1
app.MapIngredientEndpoints();

app.Run();
```

---

## 8. Protect Endpoints with Authorization

Update existing endpoints to require authentication. Also add the helper to extract claims from the JWT:

**File:** `src/Nastart.API/Extensions/ClaimsPrincipalExtensions.cs`

```csharp
using System.Security.Claims;

namespace Nastart.API.Extensions;

public static class ClaimsPrincipalExtensions
{
    public static Guid GetUserId(this ClaimsPrincipal principal)
    {
        var value = principal.FindFirstValue(ClaimTypes.NameIdentifier)
            ?? throw new UnauthorizedAccessException("userId claim missing from token.");
        return Guid.Parse(value);
    }

    public static string GetEmail(this ClaimsPrincipal user)
    {
        return user.FindFirstValue(ClaimTypes.Email)
            ?? throw new InvalidOperationException("Email claim not found.");
    }

    // v2-only: GetOutletId() — removed in v1, no outlet context in JWT
    // v2-only: GetRole() — removed in v1, no role claim in JWT
}
```

> ⚠️ **v2-only — do not build in v1.** The `OutletEndpoints.cs` code below is for reference only in Phase 2+. Outlets do not exist in the single-user v1 design. Do not create this file in v1.

**File:** `src/Nastart.API/Endpoints/OutletEndpoints.cs` (v2-only)

```csharp
using System.Security.Claims;
using MediatR;
using Nastart.API.Extensions;
using Nastart.Application.Features.Outlets.Commands.CreateOutlet;
using Nastart.Application.Features.Outlets.Queries.GetOutlets;

namespace Nastart.API.Endpoints;

public static class OutletEndpoints
{
    public static void MapOutletEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/outlets")
            .WithTags("Outlets")
            .RequireAuthorization(); // All outlet endpoints require a valid JWT

        group.MapGet("/", async (Guid companyId, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.Send(new GetOutletsQuery(companyId), ct);
            return Results.Ok(result);
        });

        group.MapPost("/", async (CreateOutletCommand command, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.Send(command, ct);
            return Results.Created($"/api/outlets/{result.Id}", result);
        });
    }
}
```

Update `IngredientEndpoints.cs` — **v1 uses a flat route, not outlet-scoped:**

```csharp
// v1: ingredients belong to the authenticated user — no outletId in route
var group = app.MapGroup("/api/ingredients")
    .WithTags("Ingredients")
    .RequireAuthorization(); // Require valid JWT
```

---

## 9. Role-Based Endpoint Protection

Some endpoints are restricted to specific roles. Use authorization policies:

Add to `Program.cs` (after `AddAuthorization()`):

```csharp
// v1: fallback policy — all authenticated users have the same access
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
// All protected endpoints use: .RequireAuthorization()
// v2: named role policies (OwnerOnly, OwnerOrChef, etc.) will be added when multi-user support is introduced
```

> ⚠️ **v2-only:** named policies (OwnerOnly, OwnerOrChef, etc.) are introduced in v2 when multi-role access is needed. All v1 protected endpoints use `.RequireAuthorization()` with no policy name — the FallbackPolicy above handles them.

> ⚠️ **v2-only — do not build in v1.** Named policy example:

```csharp
// v2-only example: only Owner and Procurement can update prices
group.MapPost("/price", async (...) => { ... })
    .RequireAuthorization("OwnerOrProcurement");
```

---

## 10. Role-Based Response Stripping (C-10)

> ⚠️ **v2-only — do not build in v1.** This pattern applies when multi-user roles are introduced in v2. In v1 (single user), there is one response DTO with full access — no role-based stripping needed.

This is critical: **non-Owner roles must NEVER receive `food_cost_pct` or `gross_margin` in any API response.** The fields must be structurally absent — not null, not 0, not present at all.

The pattern: create separate response DTOs per role visibility level.

**File:** `src/Nastart.Application/Common/RoleVisibility/NastartResponse.cs`

```csharp
namespace Nastart.Application.Common.RoleVisibility;

// C-10: Role-masked responses — absent fields are NOT present in JSON

// Owner sees everything
public record NastartOwnerResponse(
    Guid RecipeId,
    string RecipeName,
    decimal CostPerPortion,
    decimal FoodCostPct,      // Only Owner sees this
    decimal GrossMargin,      // Only Owner sees this
    int PortionCount);

// Chef, Procurement, Viewer — financial metrics stripped
public record NastartStandardResponse(
    Guid RecipeId,
    string RecipeName,
    decimal CostPerPortion,
    int PortionCount);
// food_cost_pct and gross_margin are not present — not null, not zero, ABSENT
```

**How to use in a handler:**

```csharp
// In a recipe query handler (built in Phase 2), the pattern is:
public async Task<ErrorOr<object>> Handle(GetRecipeQuery query, CancellationToken ct)
{
    var recipe = await _db.Recipes.FindAsync(query.RecipeId, ct);
    if (recipe is null)
        return Error.NotFound("Recipe.NotFound", "Recipe not found.");

    // query.Role comes from the JWT claims, passed through the endpoint
    if (query.Role == "Owner")
    {
        return new NastartOwnerResponse(
            recipe.Id, recipe.Name, recipe.CostPerPortion,
            recipe.FoodCostPct, recipe.GrossMargin, recipe.PortionCount);
    }

    // Non-owner: food_cost_pct and gross_margin fields do not exist in this DTO
    return new NastartStandardResponse(
        recipe.Id, recipe.Name, recipe.CostPerPortion, recipe.PortionCount);
}
```

> This pattern is established now. It will be used in Phase 2 (recipe queries) and Phase 4 (Telegram alert payloads). The key rule: **never return a single DTO with nullable financial fields — use two separate DTOs.**

---

## 11. Email Verification — Simplified for Phase 1

Full email verification with token storage is a future improvement. For Phase 1, a simple verify endpoint that activates the user:

```bash
mkdir -p src/Nastart.Application/Features/Auth/Commands/VerifyEmail
```

**File:** `src/Nastart.Application/Features/Auth/Commands/VerifyEmail/VerifyEmailCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Auth.Commands.VerifyEmail;

public record VerifyEmailCommand(Guid UserId) : IRequest<ErrorOr<VerifyEmailResponse>>;
```

**File:** `src/Nastart.Application/Features/Auth/Commands/VerifyEmail/VerifyEmailResponse.cs`

```csharp
namespace Nastart.Application.Features.Auth.Commands.VerifyEmail;

public record VerifyEmailResponse(string Message);
```

**File:** `src/Nastart.Application/Features/Auth/Commands/VerifyEmail/VerifyEmailHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Microsoft.EntityFrameworkCore;
using Nastart.Application.Common.Interfaces;

namespace Nastart.Application.Features.Auth.Commands.VerifyEmail;

public class VerifyEmailHandler : IRequestHandler<VerifyEmailCommand, ErrorOr<VerifyEmailResponse>>
{
    private readonly IAppDbContext _db;

    public VerifyEmailHandler(IAppDbContext db)
    {
        _db = db;
    }

    public async Task<ErrorOr<VerifyEmailResponse>> Handle(
        VerifyEmailCommand command, CancellationToken ct)
    {
        var user = await _db.Users
            .FirstOrDefaultAsync(u => u.Id == command.UserId, ct);

        if (user is null)
            return Error.NotFound("User.NotFound", "User not found.");

        if (user.IsVerified)
            return Error.Conflict("User.AlreadyVerified", "Email is already verified.");

        // Flow 01 Step 6: set isVerified=true, isActive=true
        user.IsVerified = true;
        user.IsActive = true;

        await _db.SaveChangesAsync(ct);

        return new VerifyEmailResponse("Email verified. You can now log in.");
    }
}
```

Add to `AuthEndpoints.cs`:

```csharp
// Simplified verification — in production, validate actual token hash
group.MapPost("/verify", async (VerifyEmailCommand command, ISender sender, CancellationToken ct) =>
{
    var result = await sender.Send(command, ct);
    return result.ToApiResult();
});
```

---

## 12. Company + Outlet Onboarding (Post-Registration)

> ⚠️ **v2-only — do not build in v1.** No company or outlet onboarding in single-user design.

After a new user's first login, they have no outlet yet. They need to create a Company and first Outlet. Build the Company creation slice:

```bash
mkdir -p src/Nastart.Application/Features/Companies/Commands/CreateCompany
```

**File:** `src/Nastart.Application/Features/Companies/Commands/CreateCompany/CreateCompanyCommand.cs`

```csharp
using ErrorOr;
using MediatR;

namespace Nastart.Application.Features.Companies.Commands.CreateCompany;

public record CreateCompanyCommand(
    Guid UserId,
    string Name,
    string Currency,
    string Timezone
) : IRequest<ErrorOr<CreateCompanyResponse>>;
```

**File:** `src/Nastart.Application/Features/Companies/Commands/CreateCompany/CreateCompanyResponse.cs`

```csharp
namespace Nastart.Application.Features.Companies.Commands.CreateCompany;

public record CreateCompanyResponse(Guid CompanyId, Guid OutletId, string Token);
```

**File:** `src/Nastart.Application/Features/Companies/Commands/CreateCompany/CreateCompanyHandler.cs`

```csharp
using ErrorOr;
using MediatR;
using Nastart.Application.Common.Interfaces;
using Nastart.Domain.Entities;
using Nastart.Domain.Enums;

namespace Nastart.Application.Features.Companies.Commands.CreateCompany;

public class CreateCompanyHandler
    : IRequestHandler<CreateCompanyCommand, ErrorOr<CreateCompanyResponse>>
{
    private readonly IAppDbContext _db;
    private readonly ITokenService _tokenService;

    public CreateCompanyHandler(IAppDbContext db, ITokenService tokenService)
    {
        _db = db;
        _tokenService = tokenService;
    }

    public async Task<ErrorOr<CreateCompanyResponse>> Handle(
        CreateCompanyCommand command, CancellationToken ct)
    {
        // Flow 01 Steps 10-12: create Company → create first Outlet → create OutletUser(Owner)
        var company = new Company
        {
            Id = Guid.NewGuid(),
            Name = command.Name,
            Currency = command.Currency,
            Timezone = command.Timezone
        };

        var outlet = new Outlet
        {
            Id = Guid.NewGuid(),
            Name = $"{command.Name} - Main", // Default first outlet name
            CompanyId = company.Id
        };

        // The creating user becomes Owner of the first outlet
        var outletUser = new OutletUser
        {
            Id = Guid.NewGuid(),
            UserId = command.UserId,
            OutletId = outlet.Id,
            Role = Role.Owner
        };

        _db.Companies.Add(company);
        _db.Outlets.Add(outlet);
        _db.OutletUsers.Add(outletUser);
        await _db.SaveChangesAsync(ct);

        // Flow 01 Step 13: re-issue JWT with outlet scope
        // v2-only — this uses the v2 signature (userId, outletId, role); v1 signature is GenerateToken(userId, email)
        var token = _tokenService.GenerateToken(command.UserId, outlet.Id, Role.Owner);

        return new CreateCompanyResponse(company.Id, outlet.Id, token);
    }
}
```

Endpoint:

> ⚠️ **v2-only — do not build in v1.** No company or outlet onboarding in single-user design.

```csharp
// In a new CompanyEndpoints.cs or added to AuthEndpoints
group.MapPost("/setup", async (CreateCompanyRequest request, ClaimsPrincipal user, ISender sender, CancellationToken ct) =>
{
    var userId = user.GetUserId();
    var command = new CreateCompanyCommand(userId, request.Name, request.Currency, request.Timezone);
    var result = await sender.Send(command, ct);
    return result.ToApiResult();
}).RequireAuthorization(); // Must be logged in (even without outlet)
```

---

## Checkpoint — Full Registration → Login Flow

Test the complete flow. The API should be running with all endpoints wired:

```bash
dotnet run --project src/Nastart.API
```

```bash
# Step 1: Register
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name": "John Owner", "email": "john@test.com", "password": "password123"}'
# Expected: 201 {"userId":"...", "message":"Check your email to verify your account."}
# Check console output — you should see the verification email logged

# Step 2: Verify email (simplified — use the userId from step 1)
curl -X POST http://localhost:5000/api/auth/verify \
  -H "Content-Type: application/json" \
  -d '{"userId": "<userId-from-step-1>"}'
# Expected: 200 {"message":"Email verified. You can now log in."}

# Step 3: Login
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "john@test.com", "password": "password123"}'
# Expected: 200 {"token":"eyJhbG..."}
# v1: token only — single user, no outlet or role in response

# Step 4: Verify ingredient endpoint requires auth (protected route)
curl http://localhost:5000/api/ingredients
# Expected: 401 Unauthorized (no token provided)

# Step 5: Login and use the JWT
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "your_password"}'
# Expected: 200 OK with {"token":"eyJh..."}

# Step 6: Use the token to access ingredients
TOKEN="eyJh..."  # paste the token from step 5
curl http://localhost:5000/api/ingredients \
  -H "Authorization: Bearer $TOKEN"
# Expected: 200 OK with [] (empty list)

# Step 7: Test login guard — try logging in before verification
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name": "Unverified User", "email": "unverified@test.com", "password": "password123"}'

curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "unverified@test.com", "password": "password123"}'
# Expected: 401 "Invalid email or password." (same message for all auth failures — prevents account enumeration)
```

---

## Phase 1 Complete — What You've Built

Across L1–L5, you now have:

| Component | Status |
|---|---|
| 4-project Clean Architecture solution | ✅ |
| 11 domain entities + 5 enums | ✅ |
| EF Core + PostgreSQL with migration | ✅ |
| MediatR + CQRS vertical slices | ✅ |
| FluentValidation pipeline behavior | ✅ |
| ErrorOr result pattern | ✅ |
| JWT authentication with custom claims | ✅ |
| Authorization (FallbackPolicy — authenticated user) | ✅ |
| Register → Verify → Login flow | ✅ |
| Role-based authorization policies | v2-only |
| Role-masked response pattern (C-10) | v2-only |
| Company + Outlet onboarding | v2-only |
| Docker Compose for local PostgreSQL | ✅ |

### Deferred to later phases

| Feature | Phase | Lesson |
|---|---|---|
| Ingredient full CRUD with cascade | Phase 2 | L6–L8 |
| Recipe builder + costing engine | Phase 2 | L9–L11 |
| Invoice upload + OCR + review queue | Phase 3 | L12–L14 |
| Telegram linking + bot-secret auth | Phase 4 | L15–L17 |
| IFileStorageService | Phase 3 | L12 |
| Docker Compose (full 5-service) | Phase 5 | L18 |
