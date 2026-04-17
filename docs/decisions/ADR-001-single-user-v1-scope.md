# ADR-001: Lock v1 to a Single-User Solopreneur Scope

## Status
Accepted

## Date
2026-04-17

## Context

The early planning material in this repository started from an enterprise-oriented F&B model with:

- `Company`, `Outlet`, and `OutletUser` hierarchy
- invitation-driven onboarding
- four user roles (`Owner`, `Chef`, `Procurement`, `Viewer`)
- role-filtered responses and alert payloads

On April 9, 2026, the product direction was refined toward a much narrower and more valuable v1: a single-user recipe costing app for home bakers and solo food entrepreneurs. That shift is reflected in `docs/ideas/recipe-cost-solopreneur.md`, `AGENTS.md`, and `.github/context/v1-constraints.md`, and it is partially applied across the lesson series through amendment blocks.

The problem was documentation drift. Some top-level files still read like the enterprise design is current, while the newer v1 constraints explicitly forbid that architecture in the first release. Without a recorded decision, future sessions and agents can reintroduce outlet, role, or invitation logic by mistake.

## Decision

For v1, the project is a single-user application.

- Build one authenticated user account with direct ownership of ingredients, recipes, categories, and Telegram linking.
- Do not build `Company`, `Outlet`, `OutletUser`, `Invitation`, or role-based authorization in v1.
- Scope v1 JWT claims to `userId` and `email` only.
- Use authenticated-user authorization only in v1; no named role policies.
- Model recipe pricing with `CostPerPortion`, `PackagingCost`, and `TargetMargin`, and derive sell price at read time instead of storing it.
- Treat Telegram as a personal alert channel tied to `TelegramLink -> UserId`, not as a role-aware team notification surface.
- Preserve older enterprise-oriented lessons and flow documents only as historical or v2 reference material unless they are explicitly updated to match v1.

## Alternatives Considered

### Keep the enterprise model in v1
- Pros: would preserve the original company/outlet/role design without extra documentation work
- Cons: adds a large amount of auth, schema, and authorization complexity before the core solopreneur use case is proven
- Rejected: it solves future team needs at the expense of the current single-user product goal

### Keep outlet and role abstractions in code, but hide them in the UI
- Pros: could make a later v2 migration feel smaller
- Cons: still expands the database, API, JWT, and authorization surface area; increases the chance of building the wrong thing now
- Rejected: hidden complexity is still complexity, and it would violate the explicit v1 constraints

### Rely on amendment notes only, without a formal ADR
- Pros: less up-front documentation
- Cons: amendment blocks are distributed across many files and do not create a single architectural record
- Rejected: the repo already showed drift between lessons, checkpoint material, and business-flow docs

## Consequences

- The current v1 authority order is `AGENTS.md`, `.github/context/v1-constraints.md`, this ADR, then the progress ledger in `docs/progress/`.
- Older files such as `CHECKPOINT.md` and some business-flow documents must be clearly labeled when they preserve historical enterprise context.
- Lessons can keep enterprise examples for teaching purposes only if the v1 path is explicit and unambiguous.
- Canonical decisions C-1 through C-7, C-13, and C-14 remain active for v1. C-8 through C-12 are preserved as v2-only reference decisions.
- Future expansion to multi-user teams requires a new ADR rather than silently reusing the older enterprise notes.