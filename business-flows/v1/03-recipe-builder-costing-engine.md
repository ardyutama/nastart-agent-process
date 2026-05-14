# Flow 03 - Recipe Builder & Costing Engine

> **Track:** `business-flows/v1/`
> **Authority:** Active implementation guidance for the current single-user v1 build. Read this file with `AGENTS.md`, `.github/context/v1-constraints.md`, and `business-flows/00-canonical-decisions.md`.
>
> **Service owner:** Nastart.Api
> **Entities:** Recipe, RecipeItem, Ingredient, IngredientPriceHistory
>
> **v1 scope:** Recipes belong directly to the authenticated user. Cost is server-authoritative. Sell price is derived at read time from `CostPerPortion`, `PackagingCost`, and `TargetMargin`.

---

## Active Rules For This Flow

- `Recipe.UserId` is the ownership boundary; there is no outlet scope
- `CostPerPortion` is computed by the server and never accepted from the client
- `PackagingCost` and `TargetMargin` are stored inputs in v1
- `DerivedSellPrice` and `FoodCostPct` are computed in read handlers
- `RecipeResponse` is unified for the single user; there are no role-split DTOs
- Recipe versioning remains active through `VersionGroupId`, `VersionNumber`, and `VersionLabel`

---

## Shared Cost Formula

```text
cost_per_portion = SUM(
  (current_price / unit_size) * quantity * (1 / yield_percentage)
) / portion_count
```

**Derived sell price formula:**

```text
derived_sell_price = (CostPerPortion + PackagingCost) / (1 - TargetMargin)
```

If `TargetMargin >= 1`, derived sell price is `null`.

---

## Flow 1: Create A New Recipe

**Actor:** Authenticated User
**Entry point:** Recipe creation screen in Vue.js

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Opens new recipe form | Vue.js | - | empty form | - | - |
| 2 | User | Enters recipe header and items | Vue.js | name, portionCount, packagingCost, targetMargin, versionLabel, recipeItems[] | draft recipe | portionCount must be >= 1 | Client-side validation |
| 3 | Nastart.Api | Resolves `UserId` and validates request | Nastart.Api | JWT + request body | User.id | Invalid input -> 400 | Reject |
| 4 | Nastart.Api | Validates ingredient ownership for all recipe items | PostgreSQL | ingredientIds[], User.id | validated ingredients | Any ingredient not owned by user -> 403 or 404 | Reject |
| 5 | Nastart.Api | Creates Recipe record | PostgreSQL | UserId, name, portionCount, packagingCost, targetMargin, versionGroupId=new UUID, versionNumber=1, versionLabel | Recipe.id | Duplicate name for same user can be rejected by rule if enforced | DB error -> 500 |
| 6 | Nastart.Api | Creates RecipeItem rows | PostgreSQL | recipeId, ingredientId, quantity, yieldPercentage | RecipeItem[] | - | DB error -> 500 |
| 7 | Nastart.Api | Calculates server-authoritative recipe cost | Application service + PostgreSQL | recipeId and ingredient data | CostPerPortion | Missing ingredient price can yield partial or null handling per business rule | Handler error -> 500 |
| 8 | Nastart.Api | Stores `CostPerPortion` on Recipe | PostgreSQL | recipeId, costPerPortion | updated Recipe | - | DB error -> 500 |
| 9 | Nastart.Api | Returns created recipe response | Nastart.Api | Recipe | unified `RecipeResponse` | - | - |
| 10 | Vue.js | Shows the saved recipe with derived sell price | Vue.js | RecipeResponse | recipe detail | - | - |

**Result:** The first version of the recipe is stored with authoritative cost and version metadata.

---

## Flow 2: Read Recipes With Unified v1 Response

**Actor:** Authenticated User
**Entry point:** Recipe list page or recipe detail page

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Requests recipe list or recipe detail | Vue.js | JWT, optional recipeId | request | - | Unauthenticated -> login |
| 2 | Nastart.Api | Resolves `UserId` from claims | Nastart.Api | JWT | User.id | Invalid token -> 401 | Reject |
| 3 | Nastart.Api | Queries user-owned recipes | PostgreSQL | User.id, optional recipeId | Recipe rows | Missing recipe -> 404 | - |
| 4 | Nastart.Api | Computes `DerivedSellPrice` and `FoodCostPct` in the handler | Nastart.Api | CostPerPortion, PackagingCost, TargetMargin | computed values | `TargetMargin >= 1` -> derived sell price null | - |
| 5 | Nastart.Api | Returns unified response DTOs | Nastart.Api | Recipe + computed values | `RecipeResponse` | - | - |
| 6 | Vue.js | Renders list or detail view | Vue.js | `RecipeResponse` | recipe cards/detail | - | Empty state if no recipes exist |

**Unified v1 response fields:**
- `Id`
- `Name`
- `PortionCount`
- `CostPerPortion`
- `PackagingCost`
- `TargetMargin`
- `DerivedSellPrice`
- `FoodCostPct`
- `VersionNumber`
- `VersionLabel`
- `VersionGroupId`

---

## Flow 3: Auto-Cascade Recipe Cost Recalculation

**Trigger:** A new `IngredientPriceHistory` row has been committed for an ingredient.
**Interface:** `ICostCascadeService.RecalculateForIngredient(ingredientId)`

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | Nastart.Api | Fetches latest price for the changed ingredient | PostgreSQL | ingredientId | current price | No price history -> stop | Log warning |
| 2 | Nastart.Api | Finds all RecipeItems using that ingredient | PostgreSQL | ingredientId | recipe ids + recipe item rows | No matching recipes -> stop | - |
| 3 | Nastart.Api | Loads full ingredient-price context for each affected recipe | PostgreSQL | recipe ids | recipe cost inputs | - | Query failure -> log and skip recipe |
| 4 | Nastart.Api | Recomputes `CostPerPortion` with the canonical formula | Application service | recipe items, prices, portionCount | new costPerPortion | Per-recipe calculation failure -> log `CascadeErrorLog` | Continue |
| 5 | Nastart.Api | Updates each affected Recipe | PostgreSQL | recipeId, new costPerPortion | updated Recipe | - | DB error -> log and continue |
| 6 | Nastart.Api | Leaves derived sell price for read time | Nastart.Api | PackagingCost, TargetMargin remain stored | no DB write for sell price | - | - |

**Failure rule:** A cascade failure for one recipe never rolls back the committed ingredient price history.

---

## Flow 4: Create A New Version Of A Recipe

**Actor:** Authenticated User
**Entry point:** Recipe detail page -> duplicate/new version action

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Starts a new recipe version from an existing recipe | Vue.js | sourceRecipeId, optional newVersionLabel | - | - | - |
| 2 | Nastart.Api | Loads source recipe and checks ownership | PostgreSQL | sourceRecipeId, User.id | source Recipe + RecipeItems | Not found or unauthorized -> 404 or 403 | Reject |
| 3 | Nastart.Api | Creates new Recipe row in same version group | PostgreSQL | copied fields, VersionGroupId=source.VersionGroupId, VersionNumber=source.VersionNumber+1, new VersionLabel | new Recipe.id | - | DB error -> 500 |
| 4 | Nastart.Api | Copies RecipeItems to the new version | PostgreSQL | newRecipeId, source recipe items | new RecipeItems | - | DB error -> 500 |
| 5 | Nastart.Api | Recalculates `CostPerPortion` for the new version | Application service | new recipe context | authoritative cost | - | Calculation failure -> 500 |
| 6 | Nastart.Api | Returns the new recipe version | Nastart.Api | Recipe + computed fields | `RecipeResponse` | - | - |
| 7 | Vue.js | Displays version comparison or opens the new version | Vue.js | `RecipeResponse` | updated UI | - | - |

**Versioning rule:** Each version is an independent recipe record that stays live-costed as ingredient prices change.

---

## Flow 5: Update Recipe Items Or Metadata

**Actor:** Authenticated User
**Entry point:** Recipe edit screen

| Step | Actor | Action | Component | Data IN | Data OUT | Decision? | Error Case |
|---|---|---|---|---|---|---|---|
| 1 | User | Edits recipe header or item list | Vue.js | recipeId, updated fields, updated items | draft changes | - | Client-side validation |
| 2 | Nastart.Api | Loads recipe and checks ownership | PostgreSQL | recipeId, User.id | Recipe | Not found or unauthorized -> 404 or 403 | Reject |
| 3 | Nastart.Api | Validates updated ingredient ownership | PostgreSQL | ingredientIds[], User.id | validated items | Invalid ingredient ownership -> 403 | Reject |
| 4 | Nastart.Api | Persists updated metadata and recipe items | PostgreSQL | updated recipe, updated items | updated rows | - | DB error -> 500 |
| 5 | Nastart.Api | Recomputes `CostPerPortion` | Application service | updated recipe context | authoritative cost | - | Calculation failure -> 500 |
| 6 | Nastart.Api | Returns updated unified response | Nastart.Api | updated Recipe | `RecipeResponse` | - | - |
| 7 | Vue.js | Refreshes recipe detail | Vue.js | `RecipeResponse` | updated UI | - | - |

---

## Route Summary

| Capability | Route pattern | Auth |
|---|---|---|
| List recipes | `GET /api/recipes` | `.RequireAuthorization()` |
| Get recipe by id | `GET /api/recipes/{id}` | `.RequireAuthorization()` |
| Create recipe | `POST /api/recipes` | `.RequireAuthorization()` |
| Update recipe | `PUT /api/recipes/{id}` | `.RequireAuthorization()` |
| Create new version | `POST /api/recipes/{id}/versions` | `.RequireAuthorization()` |

---

## Not In This v1 Flow

- `SellingPrice` stored on `Recipe`
- `CostThresholdPercentage` on `Recipe`
- Role-based response stripping
- Outlet-scoped recipe routes
- Team-routed recipe cost alerts