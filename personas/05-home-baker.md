# Persona 5 — Casual Home Baker / Small Food Entrepreneur

> **Role summary**: A small-scale food seller — operating via Instagram, WhatsApp orders, or a market stall — who runs the entire business from a smartphone and needs a lightweight, always-accessible tool to price products accurately as ingredient costs change.

---

## User Stories

| # | User Story |
|---|---|
| 1 | As a home baker, I want to enter a recipe once and have the app calculate cost-per-batch from actual ingredient purchase prices so that I stop guessing my margins and know whether I'm making a profit. |
| 2 | As a home baker, I want to scan my grocery receipt after every shopping trip so that ingredient prices update automatically without me having to type in every line item manually. |
| 3 | As a home baker, I want to receive a Telegram message with my updated recipe cost when flour or butter prices change so that I can adjust my selling price before my next market day. |
| 4 | As a home baker, I want to group recipes into batch sizes (e.g., a box of 12 cupcakes) and see the total cost and suggested retail price so that I can quote customers quickly via WhatsApp without doing mental math. |
| 5 | As a home baker, I want to save multiple versions of the same recipe (e.g., standard vs. premium ingredients) and compare their cost difference so that I can offer a budget and a luxury tier to different customers. |
| 6 | As a home baker, I want to ask the Telegram bot "how much does my carrot cake cost to make right now?" and get an instant accurate answer so that I can respond to a customer inquiry on the spot while I'm busy in the kitchen. |
| 7 | As a home baker, I want the app to flag when the cost of a recipe has increased by more than 10% since I last set my price so that I'm not quietly selling at a loss without realising it. |
| 8 | As a home baker, I want to log shared bulk ingredients (e.g., a 2kg bag of flour or a 500ml bottle of vanilla extract) and have the system automatically split the cost proportionally across recipes so that my costing reflects real usage, not whole-unit prices. |
| 9 | As a home baker, I want to export a simple weekly cost summary so that I can show my partner or an accountant a basic breakdown of what I spent vs. what I need to earn. |
| 10 | As a home baker, I want to add non-ingredient costs like packaging, labels, and gas/electricity as a fixed amount or percentage add-on per product so that my final price reflects real overheads, not just raw ingredients. |

---

## Use Cases

**1. Market day prep (Friday night before Saturday market)**
Message Telegram bot: "show me costs for this week's menu" → returns lemon drizzle loaf, chocolate chip cookies per dozen, sourdough loaf with current costs → notice sourdough jumped from R26 to R31 → pull up the recipe, find that bread flour updated from last receipt scan → adjust market price from R65 to R75 before printing price tags.

**2. Post-shopping receipt scan**
Return from Woolworths with R850 of ingredients → open app, tap "Scan Receipt", photograph the slip → system reads off butter, cream cheese, eggs, castor sugar, vanilla extract → matches to saved ingredients, updates unit costs → two cake recipes show a small cost increase in the dashboard. No manual entry needed.

**3. Customer quote on WhatsApp**
Client asks for 4 dozen custom sugar cookies for a baby shower → standing at the kitchen counter → open bot, type "cost for 48 sugar cookies" → bot responds with ingredient cost (R94.80) plus saved packaging add-on (R18.00) → total base cost R112.80 → quote R280 with confidence that it's profitable.

**4. Price change alert**
Tuesday afternoon: Telegram notification — "Heads up — your Buttercream Birthday Cake recipe cost has increased by 14% since your last price update. Current cost: R58.20. Your current selling price: R150." → haven't changed the price in 3 months → butter and icing sugar both quietly went up → update Instagram price list that evening before the next batch of orders.

**5. New recipe viability check**
Want to add an Earl Grey shortbread to the range → create new recipe in the app, add ingredients (250g butter, 100g icing sugar, 300g cake flour, 2 tbsp loose leaf tea), enter batch size of 30 biscuits → app returns: cost per biscuit R2.10, cost per bag of 10 biscuits R21.00 → add R3 packaging cost per bag → land on a selling price of R45. Done in under 3 minutes.

---

## Pain Points Solved

1. **Flying blind on pricing** — replaces gut-feel estimates and competitor copying with real ingredient-based numbers.
2. **Margins shrink invisibly** — ingredient prices change every few weeks; selling prices stay fixed for months; automated alerts make price changes immediately visible before they cause loss.
3. **Manual receipt data entry is too tedious** — a folder full of unlogged grocery slips; scanning removes the barrier entirely; photographing a slip takes 10 seconds.
4. **No tool fits this scale** — proper accounting software (Xero, QuickBooks) is overkill and expensive; spreadsheets break when prices change; this product fills exactly the gap between "nothing" and "too much."
5. **Constant app context-switching** — running a business from a phone means baking, packing, and customer messaging all happen in the same moment; Telegram bot delivers cost data where the user already is.

---

## Concerns & Gaps

1. **Receipt scanning accuracy on informal receipts** — South African and Southeast Asian till slips vary widely in format; thermal prints scan poorly; some list generic names like "DAIRY PROD 500G" instead of "Clover Butter Unsalted"; system must handle informal market and spaza shop receipts, not only clean supermarket formats; if it only works cleanly for maybe 60% of shopping sources, it's a significant gap.
2. **Labour and packaging costs not in the core feature set** — ingredients alone are incomplete; without the ability to add packaging cost per unit and a labour rate per hour, the margin figure shown will be misleadingly optimistic and could lead to worse pricing decisions than before.
3. **Data privacy and recipe ownership** — recipes are the only real business asset; before uploading everything to an app, need clarity on: who can access the data, whether it is stored on a third-party server, what happens if the product shuts down, and whether there is a full data export function.

---

## Day in the Life

> Thursday afternoon. Finished a big bake — 4 dozen vanilla cupcakes and 2 loaves of seeded sourdough for a Friday delivery. While the cupcakes cool, pull out yesterday's receipt and scan it through the app. It picks up most items immediately, flags one line it couldn't read, asks for manual confirmation — takes about 40 seconds total. Check Telegram: an alert has come in — "Cream cheese price increased by 12% — affects cheesecake slice cost by R3.80 per unit." Make a quick note to review the cheesecake price before the weekend. Later, while packing orders, a customer on WhatsApp asks about a custom birthday cake for next week. Message the Telegram bot, ask for the cost on the layered chocolate fudge cake at double quantity, get a number back in seconds. Add 30% margin, quote R420, she confirms immediately. No calculator, no spreadsheet, no laptop. Just a phone — the same one already in hand.
