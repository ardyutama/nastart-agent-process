# Implementation Plan: Versioned Business Flow Documentation

## Overview

This plan implements the approved documentation design that keeps `business-flows/` as the canonical root, introduces an active `v1/` track, preserves enterprise material in `v2-reference/`, and removes the current ambiguity created by the separate `flows/` folder.

## Architecture Decisions

- `business-flows/` remains the single canonical flow root.
- `business-flows/v1/` is the only implementation-ready flow track for the current project.
- `business-flows/v2-reference/` preserves historical enterprise flows and is not current implementation authority.
- `AGENTS.md` and `.github/context/v1-constraints.md` remain the top-level authority above all flow documents.
- The migration should prefer clean v1 rewrites over trimmed enterprise copies.

## Task List

### Phase 1: Routing And Structure Foundation

## Task 1: Create versioned flow directories

**Description:**
Create the `business-flows/v1/` and `business-flows/v2-reference/` directories so the canonical flow root has explicit version boundaries before any content migration begins.

**Acceptance criteria:**
- [ ] `business-flows/v1/` exists
- [ ] `business-flows/v2-reference/` exists
- [ ] root-level numbered flow files are not moved yet in this task

**Verification:**
- [ ] Directory listing confirms the two new subfolders
- [ ] Root router files still remain discoverable after the directory creation
- [ ] Markdown diagnostics remain clean

**Dependencies:** None

**Files likely touched:**
- `business-flows/`

**Estimated scope:** Small

## Task 2: Move enterprise flow files into the reference track

**Description:**
Move the current enterprise-oriented numbered flow files into `business-flows/v2-reference/` so the canonical root stops mixing active and historical flow documents.

**Acceptance criteria:**
- [ ] the current enterprise flow files live under `business-flows/v2-reference/`
- [ ] the root `business-flows/` directory keeps only router and canonical-decision documents
- [ ] moved files preserve their existing filenames and content for traceability

**Verification:**
- [ ] Directory listing shows the numbered enterprise files under `business-flows/v2-reference/`
- [ ] No numbered enterprise flow files remain directly under `business-flows/`
- [ ] Markdown diagnostics remain clean after the move

**Dependencies:** Task 1

**Files likely touched:**
- `business-flows/v2-reference/`
- `business-flows/`

**Estimated scope:** Medium

## Task 3: Add historical banners to the reference files

**Description:**
Add a consistent top-level banner to each file in `business-flows/v2-reference/` so the reference track cannot be mistaken for active v1 implementation guidance.

**Acceptance criteria:**
- [ ] every file in `business-flows/v2-reference/` has a historical-reference banner near the top
- [ ] the banner states the track is not current implementation authority
- [ ] banner wording is consistent across all reference files

**Verification:**
- [ ] Search confirms the banner appears in every `business-flows/v2-reference/` file
- [ ] Read-through confirms the banner is visible before any enterprise flow steps
- [ ] Markdown diagnostics remain clean

**Dependencies:** Task 2

**Files likely touched:**
- `business-flows/v2-reference/01-auth-company-telegram-linking.md`
- `business-flows/v2-reference/02-ingredient-price-management.md`
- `business-flows/v2-reference/03-recipe-builder-costing-engine.md`
- `business-flows/v2-reference/04-invoice-scanning-review-commit.md`
- `business-flows/v2-reference/05-telegram-bot-flows.md`

**Estimated scope:** Medium

## Task 4: Rework the flow router documents

**Description:**
Update the root routing documents so they explain the split, default to v1, and clearly mark the enterprise track as historical reference.

**Acceptance criteria:**
- [ ] `business-flows/00-index.md` explains the new structure and authority order
- [ ] `business-flows/00-canonical-decisions.md` keeps explicit applicability notes for v1 vs enterprise
- [ ] routing language points readers to `business-flows/v1/` by default

**Verification:**
- [ ] Read-through confirms one-screen routing clarity
- [ ] Search results show `v2-reference` is described as historical reference
- [ ] No references describe enterprise flows as current v1 guidance

**Dependencies:** Tasks 2-3

**Files likely touched:**
- `business-flows/00-index.md`
- `business-flows/00-canonical-decisions.md`

**Estimated scope:** Medium

### Checkpoint: Foundation

- [ ] The canonical root is unambiguous
- [ ] Agents can discover the v1 track from `business-flows/00-index.md`
- [ ] Enterprise docs are preserved but not presented as active guidance

### Phase 2: v1 Flow Rewrites

## Task 5: Rewrite v1 auth and linking flow

**Description:**
Create the v1-native auth and Telegram-linking flow document with single-user terminology, JWT rules, and no company, outlet, invitation, or role concepts.

**Acceptance criteria:**
- [ ] `business-flows/v1/01-auth-telegram-linking.md` exists
- [ ] the flow matches single-user auth rules from `.github/context/v1-constraints.md`
- [ ] the flow keeps Telegram linking mechanics that still apply to v1
- [ ] banned enterprise concepts are absent

**Verification:**
- [ ] Search confirms no `Company|Outlet|Invitation|Role` terms in the file
- [ ] Route shapes match v1 route conventions
- [ ] Markdown diagnostics remain clean

**Dependencies:** Task 4

**Files likely touched:**
- `business-flows/v1/01-auth-telegram-linking.md`

**Estimated scope:** Medium

## Task 6: Rewrite v1 ingredient management flow

**Description:**
Create the v1 ingredient flow using user-scoped ownership, append-only price history, and the cascade rules that remain valid in the single-user build.

**Acceptance criteria:**
- [ ] `business-flows/v1/02-ingredient-price-management.md` exists and is user-scoped
- [ ] route patterns match the v1 constraints
- [ ] ingredient flows preserve append-only price history and cascade trigger rules
- [ ] no outlet-scoped or role-based logic remains

**Verification:**
- [ ] Search confirms no `Outlet|Role|Invitation|Supplier` terms in the file
- [ ] Price-history behavior matches `.github/context/v1-constraints.md`
- [ ] Markdown diagnostics remain clean

**Dependencies:** Task 4

**Files likely touched:**
- `business-flows/v1/02-ingredient-price-management.md`

**Estimated scope:** Medium

## Task 7: Rewrite v1 recipe costing flow

**Description:**
Create the v1 recipe costing flow using user-scoped recipes, derived sell price rules, `PackagingCost`, `TargetMargin`, and unified response semantics.

**Acceptance criteria:**
- [ ] `business-flows/v1/03-recipe-builder-costing-engine.md` exists
- [ ] sell-price behavior is described as derived, not stored
- [ ] the flow uses `PackagingCost` and `TargetMargin`
- [ ] no role-split DTOs, payload masking, or outlet-scoped logic remains

**Verification:**
- [ ] Search confirms no `Outlet|Role|CostThresholdPercentage|SellingPrice` terms in the file
- [ ] Derived-sell-price behavior matches `.github/context/v1-constraints.md`
- [ ] Markdown diagnostics remain clean

**Dependencies:** Task 4

**Files likely touched:**
- `business-flows/v1/03-recipe-builder-costing-engine.md`

**Estimated scope:** Medium

### Checkpoint: Core Recipe Costing Flows

- [ ] Auth, ingredient, and recipe v1 flows all exist
- [ ] The highest-risk single-user scope changes are implemented early
- [ ] No core v1 flow depends on enterprise-only concepts

### Phase 3: Remaining v1 Flows And Context Cleanup

## Task 8: Rewrite v1 invoice scanning flow

**Description:**
Create the v1 invoice-review and price-commit flow as a single-user document that preserves OCR review, append-only price commits, and cascade behavior without enterprise authorization wrappers.

**Acceptance criteria:**
- [ ] `business-flows/v1/04-invoice-scanning-review-commit.md` exists
- [ ] invoice commit behavior preserves append-only price history plus cascade rules
- [ ] the flow does not use company or outlet authorization language
- [ ] the flow stays aligned with the OCR packaging design where applicable

**Verification:**
- [ ] Search confirms no `Company|Outlet|Role|Invitation` terms in the file
- [ ] Search confirms `InvoiceScan` and cascade behavior are described consistently with current constraints
- [ ] Markdown diagnostics remain clean

**Dependencies:** Tasks 4, 6

**Files likely touched:**
- `business-flows/v1/04-invoice-scanning-review-commit.md`

**Estimated scope:** Medium

## Task 9: Rewrite v1 Telegram bot flow

**Description:**
Create the v1 Telegram bot flow as a single-user document that preserves linking and alert mechanics without role-based visibility or enterprise recipient routing.

**Acceptance criteria:**
- [ ] `business-flows/v1/05-telegram-bot-flows.md` exists
- [ ] Telegram flows use the single linked user rather than role-based recipients
- [ ] the flow references personal alerts and personal lookup behavior only
- [ ] the flow does not reintroduce outlet or role visibility matrices

**Verification:**
- [ ] Search confirms no `Role|Outlet|Owner|Chef|Procurement|Viewer` terms in the file
- [ ] Search confirms the recipient model is a single linked user
- [ ] Markdown diagnostics remain clean

**Dependencies:** Tasks 4, 5, 7, 8

**Files likely touched:**
- `business-flows/v1/05-telegram-bot-flows.md`

**Estimated scope:** Medium

## Task 10: Update repository context and agent routing

**Description:**
Update the surrounding context documents so the new flow structure is discoverable and the old layout is no longer presented as active.

**Acceptance criteria:**
- [ ] `AGENTS.md` references the versioned `business-flows/` structure where needed
- [ ] `.github/context/project-map.md` describes the new canonical flow layout
- [ ] references to the old `flows/` folder are removed or explicitly marked retired

**Verification:**
- [ ] Search confirms the new `business-flows/v1/` path is discoverable from context docs
- [ ] Search confirms no context doc presents `flows/` as active guidance
- [ ] Markdown diagnostics remain clean

**Dependencies:** Tasks 4-9

**Files likely touched:**
- `AGENTS.md`
- `.github/context/project-map.md`

**Estimated scope:** Medium

### Checkpoint: v1 Flow Set Complete

- [ ] All five v1 flow files exist
- [ ] Each v1 flow aligns with `.github/context/v1-constraints.md`
- [ ] Context documents route readers to the versioned structure

### Phase 4: Legacy Retirement And Final Review

## Task 11: Retire legacy flows and run consistency review

**Description:**
Remove or archive the old `flows/` folder and run a repository-wide documentation consistency pass to verify that agents will land on the correct v1 track.

**Acceptance criteria:**
- [ ] `flows/` no longer acts as an active competing root
- [ ] cross-links do not target retired flow paths
- [ ] the repo presents one default flow path for current implementation work

**Verification:**
- [ ] Search for `flows/` references shows only archival or migration notes
- [ ] Search for banned enterprise terms inside `business-flows/v1/` returns no matches
- [ ] Final doc review confirms authority order remains intact

**Dependencies:** Task 10

**Files likely touched:**
- `flows/`
- any markdown files with stale links

**Estimated scope:** Medium

### Checkpoint: Complete

- [ ] `business-flows/v1/` is the default implementation track
- [ ] `business-flows/v2-reference/` is clearly historical
- [ ] `flows/` no longer creates discovery ambiguity
- [ ] Context documents consistently point to the versioned structure

## Parallelization Opportunities

- Tasks 5, 6, and 7 can run in parallel after Task 4 because they create independent v1 core flow documents.
- Task 8 can run after Task 6 if the ingredient flow becomes the main dependency for invoice price commits.
- Task 9 should remain after Tasks 5, 7, and 8 because it references auth, recipe, and invoice behavior.
- Tasks 10 and 11 should stay sequential because retiring legacy paths before context updates would create broken guidance.

## Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| v1 rewrites inherit enterprise language by copy-paste | High | Use v1 constraints as the rewrite checklist for every file |
| Broken cross-links after path changes | High | Reserve a final repository-wide link and path review task |
| Canonical decisions become split-brain between tracks | Medium | Keep `00-canonical-decisions.md` at the root with explicit applicability notes |
| Legacy `flows/` remains discoverable and causes drift | High | Retire or archive it only after all references are updated |

## Open Questions

- Should `v2-reference/` remain permanently historical, or should a future implementation-ready enterprise track use a separate `v2/` folder later?

## Human Review Gate

Before implementation begins, confirm:

- [ ] The design doc is accepted as the migration baseline
- [ ] The task order looks correct
- [ ] The repository owner agrees that `flows/` should be retired after migration
- [ ] No task is larger than a single focused documentation session