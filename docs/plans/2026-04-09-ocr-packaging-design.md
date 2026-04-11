# Design: OCR Receipt Handling + Packaging Cost

> Date: April 9, 2026  
> Status: Approved  
> Resolves: Open Questions 1 and 2 from `docs/ideas/recipe-cost-solopreneur.md`

---

## Problem 1 — OCR on Mixed Receipt Types

### Decision: Confidence-Scored Review Queue (Approach A)

All receipt types (supermarket and supplier invoice alike) go through the same pipeline. No special cases per vendor.

### How It Works

```
Receipt photo
     ↓
OCR text extraction  (AWS Textract / Tesseract)
     ↓
LLM name-matching pass
  → maps "WW BTTRMILK 2L"  →  Buttermilk (library match)
  → maps "SF PLN FLOUR 1KG" →  Plain Flour (library match)
  → assigns confidence score per line  (0.0 – 1.0)
     ↓
Review queue
  → ≥ 0.85  →  green  →  auto-staged (user can still edit)
  → 0.50–0.84 →  yellow →  flagged for review
  → < 0.50  →  red   →  unmatched, user resolves manually
     ↓
User confirms / edits / skips each line
     ↓
Confirmed items → IngredientPriceHistory INSERT → cascade
```

### Key Design Rules

- **No silent commits.** Every scan lands in the review queue first. Auto-accepted green lines are staged, not committed — the user sees them and confirms the batch.
- **Unmatched lines are fine.** A receipt with 10 items where only 6 match is still useful. The other 4 stay in the queue as "needs manual match" and the user can link them to ingredients or dismiss.
- **LLM task is narrow.** The LLM does one job: map a noisy receipt string to an ingredient name in the library. It does not parse prices, quantities, or units — the OCR layer extracts those fields structurally.
- **Threshold is configurable.** Confidence cutoff is an app-level setting, not hardcoded. Start at 0.85 for green; adjust based on real usage data.

### Architecture
- No new entities needed. Uses the existing review-queue pattern from `business-flows/04-invoice-scanning-review-commit.md`.
- Python FastAPI handles OCR + LLM matching. .NET API owns the confirm-and-cascade step.
- Receipt image stored in S3 with the review session record for audit trail.

---

## Problem 2 — Packaging Cost in Sell Price

### Decision: Flat Add-On Field Per Recipe (Approach A)

A single `packaging_cost` field (decimal, optional, default 0) on the `Recipe` entity. No packaging library. No packaging as a pseudo-ingredient.

### Sell Price Formula

$$\text{sell price} = \frac{\text{ingredient cost per portion} + \text{packaging cost}}{1 - \text{target margin}}$$

Where:
- `ingredient cost per portion` = existing cascade-calculated cost (no change to C-2 formula)
- `packaging_cost` = flat dollar amount per portion, entered manually by the user
- `target margin` = decimal (e.g. 0.40 for 40%)

### Example

| Field | Value |
|---|---|
| Ingredient cost per portion | $3.20 |
| Packaging cost | $0.45 |
| Target margin | 40% |
| **Recommended sell price** | **$6.08** |

Calculation: $(3.20 + 0.45) / (1 - 0.40) = 3.65 / 0.60 = \$6.08$

### Schema Change

```sql
ALTER TABLE Recipe ADD COLUMN packaging_cost DECIMAL(10,4) NOT NULL DEFAULT 0;
```

One column. No new tables.

### UI Behaviour

- `packaging_cost` field appears on the recipe builder form, below ingredient list.
- Label: **"Packaging cost per portion ($)"** with helper text: *"e.g. box, bag, label"*
- Field is optional — defaults to $0.00. No packaging cost is a valid state.
- Sell price and cost display update live as the user types.

---

## What This Does NOT Change

- The cascade service (`ICostCascadeService.RecalculateForIngredient`) is unchanged — it still recalculates ingredient cost only. Sell price is derived at read time in the API response, not stored.
- `IngredientPriceHistory` pattern is unchanged.
- The OCR review queue UX pattern is unchanged — this design just lowers the entry threshold to accept messier inputs.

---

## Not Doing

- **Per-vendor OCR templates** — high maintenance, breaks on store redesigns, not worth the complexity
- **Packaging as ingredients** — conceptually confusing, pollutes the ingredient library
- **Labour rate / hourly cost** — out of v1 scope; can be added as a second add-on field in v2 using the same pattern as `packaging_cost`
- **Overhead percentage** — opaque to user, erodes trust in the calculation

