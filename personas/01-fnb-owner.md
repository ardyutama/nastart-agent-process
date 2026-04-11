# Persona 1 — F&B Owner / Restaurateur

> **Role summary**: The business decision-maker who needs accurate food cost data to protect margins, set menu prices, and make on-the-spot calls without waiting for a report.

---

## User Stories

| # | User Story |
|---|---|
| 1 | As an F&B owner, I want to see the real-time cost of every dish so that I know which items are bleeding margin and can adjust pricing or portion sizes before my next service period. |
| 2 | As an F&B owner, I want to scan supplier invoices with my phone so that ingredient prices update automatically across all affected recipes instead of me updating spreadsheets by hand. |
| 3 | As an F&B owner, I want to receive a Telegram alert when a key ingredient's cost spikes beyond a set threshold so that I can decide on the spot — while at the market or on the road — whether to swap it, suspend the dish, or absorb the cost. |
| 4 | As an F&B owner, I want to compare the cost of a recipe across two different suppliers so that I can make a confident sourcing decision without needing separate spreadsheets. |
| 5 | As an F&B owner, I want to query the Telegram bot mid-service (e.g., "What's the food cost % on the black pepper crab tonight?") so that I can respond to a large-order discount request without leaving the floor. |
| 6 | As an F&B owner, I want to input a target food cost % and have the system back-calculate a recommended selling price so that my menu pricing is grounded in numbers, not guesswork. |
| 7 | As an F&B owner, I want the system to flag recipes where actual food cost has drifted more than 5% from baseline so that I can investigate whether the issue is supplier pricing, portion drift, or kitchen wastage. |
| 8 | As an F&B owner, I want to store multiple recipe versions (e.g., premium vs. budget) so that I can model what a seasonal or promotional menu would cost before committing to printing it. |
| 9 | As an F&B owner, I want a monthly summary pushed to my Telegram showing which dishes had the highest and lowest cost variance so that I can focus cost-control conversations with my chef on specific items with data, not feelings. |
| 10 | As an F&B owner, I want role-based access so that I can delegate invoice scanning to a sous chef without exposing my full margin structure to staff. |

---

## Use Cases

**1. Weekly supplier invoice processing**
Scan invoice at loading bay → system auto-updates all affected recipe costs → receive Telegram alert if any dish crosses food cost threshold → decide on pricing or menu adjustment before service.

**2. Pre-launch menu costing for a new outlet**
Build and cost 24 new dishes → model ingredient substitutions in real time (e.g., swap imported butter for local brand) → generate cost cards for all dishes → cut multi-day spreadsheet work down to an afternoon.

**3. Mid-service discount decision**
Corporate client calls at 6:45 PM asking for 15% discount on a 40-pax dinner → query Telegram bot → get food cost % back in seconds → counter-offer confidently based on real margin data, without stepping off the floor.

**4. Identifying a margin problem proactively**
Cash position tighter than expected despite strong revenue → query bot for "dishes with highest cost variance this month" → find portion drift, an unlogged recipe change, and a brand switch across 3 items → address with chef before month-end.

**5. Responding to a sudden supply disruption**
Supplier cannot fulfill chicken order → search by ingredient → get list of all 7 affected dishes → swap each to an alternative ingredient → see updated costs instantly → brief floor staff before dinner rush.

---

## Pain Points Solved

1. **Stale spreadsheet costing** — recipe costs tied to actual invoices update automatically; no more maintenance burden.
2. **Delayed cost awareness** — catch margin erosion within days, not at month-end review.
3. **Data inaccessible when it matters** — Telegram puts cost data in pocket during service, supplier visits, and market runs.
4. **Manual invoice transcription** — eliminates 3–4 hours/week of admin data entry.
5. **Gut-feel menu pricing** — proper costing surfaces bad-margin bestsellers on day one.

---

## Concerns & Gaps

1. **OCR accuracy on messy/handwritten invoices** — wet market traders and small local distributors issue handwritten or low-quality thermal invoices; needs flagging of uncertain items for manual review; a wrong price slipping through silently is worse than no automation.
2. **Telegram security** — food cost percentages and supplier pricing are commercially sensitive; needs role-based bot access, clarity on data residency, and secure chat history handling.
3. **Staff adoption friction** — scanning only works if done at delivery time, not batched later; needs mobile-first loading bay UX and an audit trail for processed vs. missed invoices.

---

## Day in the Life

> 8:15 AM at the second outlet. Produce delivery arrives. Scan the 12-line invoice in 40 seconds while receiving staff counts stock. System flags: "Red snapper — price increased 18% from last recorded. Confirm?" Confirm → steamed snapper dish jumps from 29% to 34% food cost. Swap to a substitute local variety in the recipe → cost drops to 31% → send a note to the chef via Telegram before leaving the loading bay. By 9 AM, in a coffee meeting with a catering prospect. They ask for a rough per-head price for a 60-pax set. Message the bot, ask for cost breakdown on Set Menu B at 60 covers, quote confidently before the client finishes describing the event. Wednesday evening: replace three spreadsheet tabs with one Telegram bot summary query. The full loop — delivery to decision — collapses from days to minutes.
