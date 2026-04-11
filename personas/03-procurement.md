# Persona 3 — Procurement / Purchasing Staff

> **Role summary**: The operator who handles vendor relationships, processes incoming invoices, tracks ingredient prices across multiple suppliers, and bridges the gap between purchasing costs and kitchen recipe data.

---

## User Stories

| # | User Story |
|---|---|
| 1 | As a purchasing staff, I want to log a new supplier invoice by photographing it so that ingredient costs update immediately without manual data entry. |
| 2 | As a purchasing staff, I want the system to flag when a scanned invoice price differs from the last recorded price by more than a set threshold so that I can catch price hikes before they silently erode margins. |
| 3 | As a purchasing staff, I want to compare the cost-per-dish across two or more suppliers for the same ingredient so that I can make sourcing decisions backed by actual data rather than memory or habit. |
| 4 | As a purchasing staff, I want to receive a Telegram message when the food cost % for a high-volume dish crosses a defined limit so that I can respond quickly — renegotiate, substitute, or alert the chef — without waiting for a weekly report. |
| 5 | As a purchasing staff, I want to search for an ingredient via Telegram and immediately see all suppliers that carry it, their last quoted price, and last purchase date so that I can make fast decisions when a supplier is out of stock. |
| 6 | As a purchasing staff, I want the system to automatically apply a scanned invoice to all recipes that share an ingredient so that every dish cost is recalculated in one step rather than me updating each recipe manually. |
| 7 | As a purchasing staff, I want to pull a monthly report of price movements per ingredient across all suppliers so that I can present data to management when negotiating volume discounts or switching vendors. |
| 8 | As a purchasing staff, I want to attach a scanned invoice to a specific purchase order so that the finance team has a full audit trail without me having to file physical copies. |
| 9 | As a purchasing staff, I want to query a recipe's full ingredient breakdown via Telegram while at the market so that I can decide on-the-spot whether a lower-priced substitute will keep the dish within budget. |
| 10 | As a purchasing staff, I want to set price ceilings per ingredient so that the system automatically notifies me via Telegram if any scanned invoice exceeds those ceilings, regardless of which staff member processes the delivery. |

---

## Use Cases

**1. Morning delivery processing**
Fresh produce supplier delivers at 6:30 AM with handwritten invoice → scan with the app → OCR reads line items (lettuce up RM 4.80/kg from RM 3.90, cherry tomatoes RM 11.00/kg, olive oil RM 22.50/bottle) → all affected recipes auto-updated → Telegram alert flags lettuce as a 23% increase above rolling average.

**2. Supplier negotiation prep**
Meeting with dry goods supplier next Tuesday → pull 6-month price history for flour, sugar, and cooking oil across 3 vendors → Vendor B is 8% cheaper on flour but not being used → bring data to the meeting and request price matching or switch vendors with management approval.

**3. Menu engineering support**
Chef proposes new grilled salmon dish → build the recipe costing from the chef's ingredient list → raw food cost comes to RM 19.40/portion → proposed selling price of RM 48 gives 40% food cost, above the 32% target → send breakdown to chef via Telegram → suggest reducing salmon portion from 180g to 150g → cost drops to RM 16.20, hitting the margin target.

**4. End-of-month reconciliation**
Run report comparing expected ingredient spend (from recipe costing) vs. actual spend (from scanned invoices) → persistent variance on beef usage → triggers a kitchen conversation about portioning accuracy.

**5. Emergency stock substitution**
Chicken supplier calls at 8 AM — cannot fulfill today's order → query Telegram: "chicken thigh boneless current prices all suppliers" → bot returns 3 alternatives with last invoice prices and delivery recency → call the most recent active supplier → order placed and purchase record updated before kitchen prep team starts at 9 AM.

---

## Pain Points Solved

1. **Spreadsheet price tracking is always behind** — one person, one update at a time, perpetually stale; scanning makes it real-time and shared across the team.
2. **Paper invoice filing is slow to retrieve** — scanning and attaching invoices to purchase orders makes any invoice instantly searchable.
3. **No real-time communication between procurement and kitchen/management** — a supplier price hike was invisible until the monthly P&L; Telegram closes this gap immediately.
4. **Recipe costing for new dishes was a separate manual spreadsheet exercise** — now integrated with live purchase prices.
5. **Comparing suppliers objectively was difficult** — different invoice formats, unit sizes, and delivery terms; normalized data makes true cost comparison possible for the first time.

---

## Concerns & Gaps

1. **OCR accuracy on handwritten or low-quality invoices** — small local suppliers still use handwritten slips and carbon copy paper that scan poorly; a misread quantity or unit price could silently corrupt recipe costs; needs an error detection and review queue before data is committed.
2. **Multi-unit and pack size handling** — one supplier sells chicken by kg, another by 10kg box, another by variable-weight carcass; system must normalize these into a consistent unit cost and flag ambiguous units for manual review; without this, recipe costing data is meaningless.
3. **Access control and data sensitivity** — purchase prices and supplier terms are commercially sensitive; Telegram bot access must be role-based so kitchen staff can see recipe costs without also seeing supplier pricing or invoice details.

---

## Day in the Life

> 7:00 AM. Check Telegram overnight summary — beef tenderloin up 6% from last invoice, two recipes flagged above 35% food cost threshold. Alert the chef before the morning briefing. 7:30 AM: two suppliers arrive. Scan both invoices — first one processes cleanly, second has a smudged line item the system flags for review. Correct the unit price manually, confirm. By 8:00 AM all affected recipes are updated and a PDF cost summary is ready to attach to the purchase order. Mid-morning: review next week's planned menu, check projected ingredient spend at current supplier prices, identify the weekend fish special is going to be expensive because snapper prices spiked. Message the chef a cost comparison — sea bass from Supplier B is RM 3.20/kg cheaper right now and holds the same margin. Chef agrees to the substitution. Leave at 2 PM. Every invoice logged, every recipe cost current. No spreadsheet opened once.
