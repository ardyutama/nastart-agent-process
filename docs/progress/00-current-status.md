# Current Status

> Last updated: April 17, 2026  
> Purpose: track lesson progress and record how the repository shifted from the earlier enterprise planning model to the current single-user v1 scope.

---

## Documentation Authority

Read these in order when you need the current v1 truth:

1. `AGENTS.md`
2. `.github/context/v1-constraints.md`
3. `docs/decisions/ADR-001-single-user-v1-scope.md`
4. This file
5. `lessons/L1` through `lessons/L8`

`CHECKPOINT.md` and parts of `business-flows/` remain useful as historical context, but they are no longer the primary implementation source for v1 unless they explicitly match the files above.

---

## Repository Snapshot

- The product direction is locked to a single-user solopreneur app: one user, one kitchen, many ingredients, many recipes.
- Lessons exist through `L8`, covering foundation through recipe costing.
- The April 9, 2026 amendment blocks across `L2` to `L8` capture the v1 pivot away from the earlier enterprise design.
- This repository is a process and documentation workspace. It does not yet contain the generated application code, solution files, or runnable test projects described by the lessons.

---

## Lesson Progress

| Lesson | Focus | Status | Notes |
|---|---|---|---|
| `L1` | Clean Architecture scaffold | Current | Foundation material is aligned with v1 and stays mostly architecture-agnostic. |
| `L2` | EF Core schema | Use with caveats | The amendment block is correct, but later summary sections still mention excluded enums and entities from the older enterprise model. |
| `L3` | CQRS + MediatR | Use as pattern only | The structural guidance is useful, but the teaching example still revolves around outlet slices that v1 does not build. |
| `L4` | Validation + ErrorOr | Use with caveats | The validation pattern is still correct; treat the `CreateIngredient` example as transitional and prefer the production-ready ingredient slice in `L6`. |
| `L5` | JWT auth | Use with caveats | The v1 amendment is correct, but the body still preserves v2 role and outlet examples as reference material. |
| `L6` | Ingredient CRUD + price history | Current with cleanup pending | This is the authoritative v1 ingredient slice, but the file footer still contains draft/export residue that should be cleaned later. |
| `L7` | Cascade + price spike alerts | Current with cleanup pending | The lesson is aligned to the v1 cascade model, but it still contains some enterprise residue and export text that should be cleaned. |
| `L8` | Recipe builder + costing | Needs summary cleanup | The recipe workflow is current, but the completion summary still mentions role-split responses and v2-only decisions. |

---

## Revision Log

| Date | Revision | Why it changed |
|---|---|---|
| 2026-04-09 | Narrowed v1 from enterprise team software to a single-user solopreneur product | The home baker and solo entrepreneur use case became the primary product target. |
| 2026-04-09 | Replaced `Company`/`Outlet` ownership with direct `UserId` ownership on v1 entities | The single-user scope removed the need for hierarchy and outlet scoping. |
| 2026-04-09 | Moved roles, invitations, and outlet-scoped authorization to v2 reference only | Those features add complexity without solving a v1 user problem. |
| 2026-04-09 | Added `PackagingCost` and `TargetMargin`; kept sell price derived at read time | Solopreneur pricing depends on packaging overhead and target margin, but sell price should remain server-derived. |
| 2026-04-09 | Simplified Telegram alerts to one linked user instead of role-based recipients | Telegram is a personal alert channel in v1, not a team collaboration surface. |
| 2026-04-17 | Added an ADR and progress ledger; marked older checkpoint and flow files as historical where needed | The repo needed a clear source-of-truth after multiple lesson amendments and architecture revisions. |

---

## Documentation Update Status

| Area | Status | Notes |
|---|---|---|
| `README.md` | Updated | Now points readers to the authoritative v1 docs and current status files. |
| `CHECKPOINT.md` | Updated with historical framing | Preserved as a snapshot, but now clearly defers to the current v1 authority files. |
| `docs/decisions/` | Started | ADR-001 records the single-user v1 scope decision and its consequences. |
| `docs/progress/` | Started | This file is the current lesson and documentation ledger. |
| `business-flows/00-index.md` | Updated | Now explains which flows are current for v1 and which remain historical. |
| `business-flows/01-auth-company-telegram-linking.md` | Updated | Marked as historical enterprise reference for auth and onboarding. |
| `business-flows/05-telegram-bot-flows.md` | Updated with scope warning | Marked so only `/link` mechanics remain actionable for v1; deeper rewrite is still pending. |

---

## Remaining Cleanup

- Rewrite the `L3` example path around `GetIngredients` instead of `GetOutlets`.
- Separate strict v1 implementation guidance from v2 reference blocks in `L5`.
- Remove the authoring residue from the end of `L6`.
- Clean the remaining enterprise and export residue from `L7`.
- Refresh the `L8` completion summary so it only lists v1-applicable behavior.
- Rewrite the historical Telegram and cross-flow sections into explicit v1 and v2 appendices instead of relying on scope notes alone.