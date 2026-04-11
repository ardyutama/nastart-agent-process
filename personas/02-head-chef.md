# Persona 2 — Head Chef / Kitchen Manager

> **Role summary**: The person who builds, maintains, and costs every recipe daily — bridging kitchen operations with financial targets, and needing cost data accessible during fast-paced service.

---

## User Stories

| # | User Story |
|---|---|
| 1 | As a head chef, I want to see the real-time cost-per-portion of every dish so that I can make informed decisions about pricing and margins without waiting for month-end accountant numbers. |
| 2 | As a head chef, I want to scan an incoming supplier invoice so that ingredient costs update automatically across all recipes, reflecting what I actually paid today — not last month's prices. |
| 3 | As a head chef, I want to receive a Telegram notification when a key ingredient (e.g., beef tenderloin, saffron) increases by more than 10% so that I can immediately consider substitutions or adjust menu pricing before it damages margins. |
| 4 | As a head chef, I want to create recipe variations (e.g., winter vs. summer version of a dish) and compare their costs side by side so that I can plan seasonal menus with confidence about profitability. |
| 5 | As a head chef, I want to query recipe costs via Telegram while on the floor or in a supplier meeting so that I don't need to step away to a computer to answer a question about food cost percentage. |
| 6 | As a head chef, I want to set a target food cost % per dish and have the system flag any recipe that exceeds it so that I stay aligned with ownership targets without manually auditing every recipe weekly. |
| 7 | As a head chef, I want to log yield adjustments on ingredients (e.g., whole salmon yields 60% usable flesh after butchering) so that the cost calculation reflects true usable cost, not just purchase price per kilo. |
| 8 | As a head chef, I want to duplicate an existing recipe and modify it for a private event menu so that I can cost special-event dishes quickly without re-entering every ingredient from scratch. |
| 9 | As a head chef, I want to see a history of price changes for each ingredient over time so that I can identify which suppliers are consistently raising prices and negotiate better terms or find alternatives. |
| 10 | As a head chef, I want sous chefs to be able to look up recipe costs via Telegram without access to the full back-end system so that they have the information they need during prep without compromising financial data. |

---

## Use Cases

**1. Invoice arrives from meat supplier**
Delivery arrives Tuesday morning with changed prices on short rib and chicken thighs → photograph invoice with phone → system parses line items, matches to existing ingredients, updates costs → all affected recipes auto-recalculated → accurate cost-per-plate by the time service starts.

**2. Menu engineering before a seasonal refresh**
Planning the spring menu → build draft recipes in the system against current prices → compare projected food cost % against house target (28%) → two new dishes come in above 35% → adjust portion sizes and swap one ingredient before the menu ever prints.

**3. Mid-service pricing question from FOH**
FOH manager asks whether comping a table's main courses will take a loss → at the pass, message Telegram bot "cost duck confit" → cost-per-portion returned in seconds → confirm the comp is acceptable → back to plating.

**4. Catering quote for a private event**
Client wants four-course dinner for 80 guests → enter recipes → system calculates total ingredient cost for all 80 covers → add labour buffer → defensible, accurate quote ready in under an hour.

**5. Tracking a problematic supplier**
Dry goods supplier has raised spice prices 5–8% with each invoice — too small to notice individually → ingredient price history view shows 30% cumulative increase over 12 months → bring data to purchasing meeting to negotiate or switch vendors.

---

## Pain Points Solved

1. **Recipe costing spreadsheets go stale immediately** — live cost data replaces historical snapshots that stop being trusted after two weeks.
2. **Gap between theoretical and actual cost** — scanning real invoices closes the gap; costing against actual purchase data, not standard rates set months ago.
3. **No computer access during service or supplier visits** — Telegram access travels with the chef; no separate app login or laptop required.
4. **Cost communication to ownership is reactive** — proactively present margin impact analysis before management asks.
5. **Scaling recipes for events is error-prone** — system handles scaling and recalculates costs at event volume automatically.

---

## Concerns & Gaps

1. **Invoice parsing accuracy** — supplier SKU codes may not match recipe ingredient names; needs a review/confirmation step before any price update is committed, not a fully automatic write; a false update (e.g., saffron price applied to sea salt) would corrupt every recipe using that ingredient silently.
2. **Telegram access control** — prep cooks must not see margin percentages or wholesale prices; needs clearly defined role tiers.
3. **Integration with existing POS and ordering systems** — a standalone product without POS connection becomes a third isolated system; minimum viable: CSV/PDF export; ideal: POS sales integration for automatic food cost variance tracking.

---

## Day in the Life

> 7:00 AM. Two deliveries waiting — produce vendor and meat distributor. Photograph both invoices before opening the boxes. Walk the walk-in doing morning inventory check while the system processes. By the time the walk-in check is done, a Telegram message confirms five ingredient prices have changed. Two current menu items — braised lamb shoulder and spring pea risotto — have crossed the food cost threshold. Before the sous chefs arrive, pull up both recipes, adjust the lamb portion by 10g, note the pea substitution for staff meal. At 10:00 AM, the owner calls asking the all-in cost of the new tasting menu debuting next week. Two minutes, Telegram, seven-course sequence at one cover. Read her the numbers directly. No spreadsheet, no guesswork. At 2:00 PM during family meal, sous chef asks about a pan-seared halibut special given good prices on fresh fish this morning. Log the special as a quick recipe, attach halibut at today's invoice price, confirm the margin is strong enough. By service start, the number is real.
