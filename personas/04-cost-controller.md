# Persona 4 — Accountant / F&B Cost Controller

> **Role summary**: The financial guardian of the operation — responsible for COGS accuracy, food cost % reporting, audit trails, and ensuring that management decisions are grounded in reliable, up-to-date cost data across single or multiple outlets.

---

## User Stories

| # | User Story |
|---|---|
| 1 | As a cost controller, I want recipe costs to automatically recalculate when a supplier invoice is scanned so that food cost percentages reflect actual purchase prices without manual intervention. |
| 2 | As a cost controller, I want to flag any ingredient whose cost has moved more than 10% from its last recorded price so that I can investigate supplier pricing anomalies before they distort my monthly COGS report. |
| 3 | As a cost controller, I want to query the Telegram bot for the current theoretical food cost of any dish by name so that I can answer a chef's or GM's question on the spot without opening a desktop application. |
| 4 | As a cost controller, I want scanned invoices to be stored with a reference number, supplier name, date, and full line-item breakdown so that I have a complete audit trail when reconciling against accounts payable. |
| 5 | As a cost controller, I want to model a "what-if" scenario by temporarily adjusting an ingredient price and seeing the ripple effect across all recipes that use it so that I can advise management before committing to a new supplier contract. |
| 6 | As a cost controller, I want the system to alert me via Telegram when a scanned invoice contains an item not yet mapped to any recipe so that unmapped ingredients don't quietly inflate raw material costs without attribution. |
| 7 | As a cost controller, I want to assign yield percentages to ingredients at the recipe level (e.g., 70% yield on fresh fish after trimming) so that cost-per-portion reflects actual usable quantity, not purchase weight. |
| 8 | As a cost controller, I want to export a menu-level food cost summary as CSV or PDF at month-end so that I can attach it to the management accounts pack without reformatting data manually. |
| 9 | As a cost controller, I want to tag recipes by outlet or concept (e.g., "Bistro Menu", "Banquet") so that I can run food cost analysis at a business-unit level and not just across the entire group. |
| 10 | As a cost controller, I want the system to track historical ingredient price changes with timestamps so that I can produce a price variance report showing which inputs drove food cost increases in any given period. |

---

## Use Cases

**1. Monthly COGS close**
Last working day of the month. 40+ supplier invoices in PDF and photo format received from the receiving team → batch-upload through the scanning tool → system extracts line items, matches to existing ingredients, updates the price master → run recipe cost refresh → pull food cost % report by category. What previously took two full days of manual keying now takes 2–3 hours.

**2. Spot price spike investigation**
Protein supplier increases chicken breast price mid-month without formal notice → next invoice scan triggers an automatic Telegram alert → query bot with ingredient name → get old price, new price, list of 8 affected recipes and their revised margins → forward directly to executive chef and F&B director in the same Telegram thread for a decision.

**3. New dish approval**
Chef proposes a new pasta special before a seasonal launch → build the recipe in the system with ingredient quantities → system pulls current purchase prices and yields, calculates cost-per-portion immediately → lands at 38% food cost, above the 32% target → test alternative protein and smaller portion size in what-if view → cost drops to an acceptable level → share the approved costing sheet with the chef before the dish goes on the menu.

**4. Multi-outlet consolidated reporting**
Group owns three outlets with different supplier relationships → maintain separate price lists per outlet → at month-end run a consolidated food cost report → Outlet 2 is running 5 points above budget → drill down to ingredient level → find an unrecorded supplier price increase that was missed because receiving staff didn't flag it → invoice scan log confirms the document was received but not processed → address with outlet manager using documented evidence.

**5. Audit and compliance support**
Internal auditor asks for the cost basis used for a specific dish during Q3 → retrieve recipe version history showing ingredient prices in force at that time, alongside the scanned invoice documents that justified those prices → audit query resolved in under 15 minutes with no manual reconstruction.

---

## Pain Points Solved

1. **Recipe costs updated once a quarter at best** — rekeying supplier prices is too time-consuming; real-time invoice scanning eliminates this lag entirely.
2. **Price changes reach cost controller weeks after they happen** — Telegram alerts collapse that gap to hours.
3. **Spreadsheet recipe costing is fragile** — formula errors, version conflicts, and copy-paste mistakes produce incorrect food cost % that go undetected until month-end; a purpose-built system with ingredient linking removes this risk.
4. **Yield and waste factors almost never captured correctly** — spreadsheet-based models require discipline to update; embedding yield at ingredient-recipe level forces the right calculation every time.
5. **Multi-outlet consolidation requires significant manual aggregation** — outlet-tagged recipes and outlet-specific price lists produce consolidated reporting in seconds.

---

## Concerns & Gaps

1. **OCR accuracy and error handling** — OCR on supplier invoices is inconsistent, especially with handwritten delivery orders or low-quality photos taken in a poorly lit storeroom; needs a human-in-the-loop correction workflow; low-confidence line items must be surfaced for manual review, not silently committed; an automation that misreads a price is worse than no automation.
2. **Audit trail integrity and data lock** — processed invoices and historical recipe costs must be immutable records; any retroactive edit must generate a logged change entry; if a supplier price can be backdated without a log, the entire audit trail is compromised; this must be clarified with the finance director before adoption.
3. **Telegram as a business-critical channel** — Telegram is a consumer messaging app; using it for live cost data raises questions about data residency, access control when a staff member leaves the company, and whether this channel meets IT security policy; a bot that can return live margin data to anyone who knows the chat handle is a genuine commercial sensitivity risk.

---

## Day in the Life

> Morning starts with Telegram notifications from overnight. Two alerts: seafood delivery invoice scanned — 12% sea bass price increase — and three dry goods invoice line items that couldn't be auto-matched to existing ingredients. Acknowledge the sea bass alert, pull up affected recipes (sea bass risotto at 41%, grilled fillet special at 44% food cost), message the F&B director with the numbers before the morning briefing. The unmatched dry goods items take about ten minutes — two are existing ingredients with slightly different naming, the third is a new SKU that gets added to the master list. By 9:30 AM, everything that would have sat in an email inbox until Thursday is resolved. At midday the chef asks whether a proposed lamb shank dish will work financially. Build the recipe on phone using the bot in under five minutes, send cost-per-portion and suggested sell price to hit 30% food cost before the description is finished. Month-end close this cycle is the first one not spent rebuilding prices from a stack of paper invoices — that time goes into actual variance analysis instead.
