# Design: Versioned Business Flow Documentation

Date: 2026-05-14
Status: Approved

## Overview

The repository currently mixes three different ideas across two flow doc families: draft vs validated, v1 vs enterprise, and active vs historical. That makes discovery unsafe for both humans and agents. The approved direction is to keep `business-flows/` as the single canonical root and split it into an active v1 track plus a historical enterprise reference track.

## Problem Statement

How might we make the flow documentation easy for agents to discover and safe to follow for the current v1 build, while still preserving enterprise flow history for future reference?

## Goals

- Provide one canonical home for flow documentation.
- Make the v1 implementation path obvious by default.
- Preserve enterprise flow documents without letting them masquerade as current implementation authority.
- Align discovery and authority order with `AGENTS.md` and `.github/context/v1-constraints.md`.
- Remove the current ambiguity created by the separate `flows/` and `business-flows/` roots.

## Non-Goals

- Reintroduce enterprise concepts into the active v1 flow set.
- Keep two competing active flow roots.
- Treat the enterprise track as implementation-ready for the current project state.
- Change product rules defined in `AGENTS.md` or `.github/context/v1-constraints.md`.

## Proposed Structure

The canonical documentation root remains `business-flows/`.

```text
business-flows/
├── 00-index.md
├── 00-canonical-decisions.md
├── v1/
│   ├── 01-auth-telegram-linking.md
│   ├── 02-ingredient-price-management.md
│   ├── 03-recipe-builder-costing-engine.md
│   ├── 04-invoice-scanning-review-commit.md
│   └── 05-telegram-bot-flows.md
└── v2-reference/
    ├── 01-auth-company-telegram-linking.md
    ├── 02-ingredient-price-management.md
    ├── 03-recipe-builder-costing-engine.md
    ├── 04-invoice-scanning-review-commit.md
    └── 05-telegram-bot-flows.md
```

## Authority Model

Authority order must be explicit and repeated wherever agents enter the docs:

1. `AGENTS.md`
2. `.github/context/v1-constraints.md`
3. `business-flows/v1/`
4. Supporting lessons and other context docs
5. `business-flows/v2-reference/` for historical enterprise context only

`business-flows/00-index.md` should be the routing page that explains this order in one screen.

## Track Rules

### v1 Track

`business-flows/v1/` is the only implementation-ready flow track for the current project.

Rules:
- Use v1-native terminology only.
- Do not include `Company`, `Outlet`, `OutletUser`, `Invitation`, or `Role`.
- Do not include role-based JWT claims, outlet-scoped routes, or role-based response masking.
- Use user-scoped flows that match `.github/context/v1-constraints.md`.
- Prefer clean rewrites over edited-down enterprise documents.

### v2 Reference Track

`business-flows/v2-reference/` preserves enterprise architecture history.

Rules:
- Every file must carry a top-level banner stating that it is historical enterprise reference.
- Files are readable for future planning but are not current implementation authority.
- Enterprise concepts may remain here, but they must not be cross-linked as active v1 guidance.

## Canonical Decisions Placement

`business-flows/00-canonical-decisions.md` remains at the root for now because several decisions still span both tracks, including:
- `ICostCascadeService.RecalculateForIngredient(ingredientId)`
- current price lookup by latest `committed_at`
- `TelegramLink.codeHash` hashing rules

The file should retain explicit applicability notes so readers know which decisions apply to v1 and which remain enterprise-only.

## Migration Plan

1. Create `business-flows/v1/` with clean v1-native flow files.
2. Move the current enterprise-oriented flow files into `business-flows/v2-reference/`.
3. Update `business-flows/00-index.md` to route readers to the correct track and explain authority order.
4. Update `.github/context/project-map.md` so it reflects the versioned flow structure.
5. Update `AGENTS.md` only where flow-loading guidance needs to reference the new structure.
6. Remove or archive the old `flows/` folder after all references are updated.

## Validation Rules

### v1 Validation

Every file in `business-flows/v1/` must be checked against `.github/context/v1-constraints.md` to ensure it excludes:
- `Company`
- `Outlet`
- `OutletUser`
- `Invitation`
- `Role`
- role-based JWT claims
- outlet-scoped routes
- role-split DTOs or payloads

### v2 Reference Validation

Every file in `business-flows/v2-reference/` must:
- include a clear historical-reference banner
- avoid presenting itself as current implementation guidance
- preserve enterprise context without conflicting with v1 routing pages

### Repository-Level Validation

Success means a new agent can inspect the repository and reach the v1 flow set without relying on tribal knowledge.

Checks:
- `business-flows/00-index.md` routes to v1 by default
- `.github/context/project-map.md` no longer presents `flows/` as a parallel active family
- `AGENTS.md` still points top-level authority to `AGENTS.md` plus v1 constraints first
- cross-links do not send readers to retired paths

## Risks And Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| v1 files become trimmed copies of enterprise docs | High | Rewrite v1 docs cleanly instead of editing enterprise text in place |
| Agents continue opening retired paths | High | Remove or archive `flows/` after updating all references |
| Root-level canonical decisions become ambiguous | Medium | Keep applicability notes explicit in `00-canonical-decisions.md` |
| Enterprise reference is mistaken for future implementation authority | Medium | Use `v2-reference` naming and a mandatory historical banner |

## Completion Criteria

The migration is complete when:
- the active flow path is `business-flows/v1/`
- the enterprise history path is `business-flows/v2-reference/`
- `flows/` no longer competes as a discoverable active root
- top-level routing docs consistently point to the versioned structure
- a repo review finds no conflicting references to the retired layout

## Not Doing

- Keeping both `flows/` and `business-flows/` as active discovery paths
- Labeling enterprise docs as plain `v2/` without clarifying their historical status
- Mixing v1 and enterprise instructions in the same flow file without explicit boundaries
- Changing product or architecture rules beyond documentation routing and structure

## Open Questions

- If the enterprise track later becomes implementation-ready, should `v2-reference/` be renamed at that time or should a separate future `v2/` track be introduced?