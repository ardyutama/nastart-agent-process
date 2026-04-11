# Persona Research — Recipe Cost Management App

> Product concept: A dynamic recipe costing tool with vendor invoice scanning and Telegram bot integration.  
> Research date: April 1, 2026

---

## Personas

| # | File | Persona | Primary Focus |
|---|---|---|---|
| 1 | [01-fnb-owner.md](01-fnb-owner.md) | F&B Owner / Restaurateur | Margin protection, pricing decisions, on-the-go access |
| 2 | [02-head-chef.md](02-head-chef.md) | Head Chef / Kitchen Manager | Recipe management, yield, daily cost accuracy |
| 3 | [03-procurement.md](03-procurement.md) | Procurement / Purchasing Staff | Supplier price tracking, invoice processing, substitution |
| 4 | [04-cost-controller.md](04-cost-controller.md) | Accountant / F&B Cost Controller | COGS accuracy, audit trail, multi-outlet reporting |
| 5 | [05-home-baker.md](05-home-baker.md) | Home Baker / Small Food Entrepreneur | Mobile-first pricing, receipt scanning, informal market |

---

## Cross-Persona Themes

### Shared critical requirements (all personas agree)
- OCR/scanning must flag low-confidence items for manual review before committing — silent errors are worse than no automation.
- Telegram access must be role-based — not everyone should see the same data.
- Cost must update automatically when an invoice is scanned, rippling through all affected recipes.

### Shared feature gaps (raised by multiple personas)
- **Yield / waste factor support** — raised by Head Chef and Cost Controller; true cost-per-portion requires yield % at the ingredient level.
- **Unit normalization** — raised by Procurement; suppliers invoice in different units (kg, box, carcass); must normalize to a consistent cost.
- **Packaging and overhead cost add-ons** — raised by Home Baker; ingredients alone are an incomplete cost picture.
- **Export (CSV/PDF)** — raised by Cost Controller and Home Baker; needed for accountants and management reports.
- **Audit trail / record immutability** — raised by Cost Controller; backdatable edits without logs are a compliance risk.

### Key open architecture questions
1. How are Telegram user roles and permissions managed?
2. What happens to invoice records and recipe history if a user's subscription ends?
3. What is the OCR strategy for non-standard (handwritten, thermal, informal market) receipts?
4. Is multi-outlet / multi-concept support in scope for v1?
5. What is the minimum viable POS or accounting system integration?
