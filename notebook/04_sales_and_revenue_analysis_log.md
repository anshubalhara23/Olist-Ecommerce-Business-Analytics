# 04 — Sales & Revenue Analysis Log
Olist E-Commerce Python Analytics

Documents what `04_sales_and_revenue_analysis.ipynb` answers, section by section, and the key decisions that shape every number in it. Covers Overall Sales, Sales Over Time, Payments, and Category Performance. Seller Performance and Customer Value now live in `05_customer_and_seller_intelligence_log.md`.

---

## Foundational decisions
- **Revenue definition**: `item_total_value` (price + freight) is the primary "revenue" figure throughout, since it's what actually changes hands per item and is already a cleaned column. `price`-only is used specifically where freight would distort the question (category product-revenue).
- **Gross vs. net**: reported as two separate, labeled figures rather than one blended number — gross = all orders regardless of status; net = `delivered` orders only. This keeps the cancellation/non-fulfillment gap visible instead of hiding it inside a single total.
- **Category Performance is gross-only**: deliberately, since category revenue measures demand ("what are people trying to buy"), not fulfillment.

---

## Section 1 — Overall Sales
- Total orders, gross item/freight/item+freight sums, and three gross averages (order/item/freight value) — all computed over every order regardless of status.
- Net equivalents of the same figures, restricted to `order_status == 'delivered'`, with numerator and denominator drawn from the same (delivered-only) population throughout — avoids the trap of dividing a filtered revenue figure by an unfiltered order count.
- Unique customers counted via `customer_unique_id`, per the raw-audit note that `customer_id` is a per-order surrogate key.

## Section 2 — Sales Over Time
- All monthly grouping uses `purchaseMonth` (derived from `order_purchase_timestamp`) — when the customer ordered, not when it shipped or delivered.
- Gross and net monthly series computed in parallel for order count, item sales, freight, and item+freight.
- Net monthly average order value divides by the order-level delivered count, not an item-level row count (which would double-count multi-item orders).
- Month-over-month % change computed on both order count and net revenue, reindexed across the full 2016-09–2018-10 range so months with zero orders show as 0% growth-from-zero rather than being silently dropped.
- **Known caveat**: 2016-09/10/12 and 2018-09/10 are near-empty ramp-up/tail-off months; `pct_change()` in those months produces large or undefined values that reflect a near-zero base, not real volatility — excluded from any growth narrative. (The same ramp-up/tail-off pattern later explained an inflated order-to-new-customer ratio found in notebook 05's acquisition-trend question.)
- **Ordering bug, fixed**: `deliveredOrders`/`deliveredItems` were originally derived in Section 1, *before* `purchaseMonth` existed on `orders`. Filtering takes a snapshot of the source frame at that moment, so the filtered copies never picked up the column added afterward — any later `.groupby('purchaseMonth')` on them raised `KeyError: 'purchaseMonth'`. Fix: re-derive `deliveredOrders`/`deliveredItems` immediately after `orders['purchaseMonth']` is assigned, in Section 2, rather than relying on the Section 1 copies.

## Section 3 — Payments
- **Q1/Q2** — revenue split and average value by `payment_type`: credit card is ~79% of revenue (over-indexed relative to its ~74% share of transactions) and has the highest average value (₹163.32).
- **Q3** — installments vs. payment value, computed at the row level *within credit_card only* (76,795 rows) — the only payment type where installments genuinely vary. r ≈ 0.38, a real but moderate positive relationship.
- **Q4** — installments by payment type confirms this is a credit-card-specific feature; every other type sits at exactly 1.0 installments on average.
- **Q5** — payment-vs-item reconciliation, built as a merged DataFrame (not a raw Series `!=` comparison, which misaligns on index). Of 8,827 raw mismatches: 776 are NaN-driven (the 1 no-payment order, 775 no-item orders), and 375 are genuine value discrepancies (>₹0.01), mostly small overpayments (mean ₹7.66, median ₹3.23). Mismatch *rate* (not raw count) by month shows the real pattern: elevated in the platform's earliest, lowest-volume months (2016-10, 2017-01 — both >1%), settling into a stable 0.2%–0.6% band thereafter; the raw-count spike in Nov 2017 and Aug 2018 is a volume artifact, not a rate increase.
- **Q6** — credit card's revenue share by month is stable throughout (~75%–81%, excluding near-empty edge months) — no growing or shrinking trend.
- **Q7** — confirmed the 3 `not_defined`/`Broken Record` rows (₹0 value) are the same 3 identified in cleaning; excluded from any meaningful revenue-share conclusion.

## Section 4 — Category Performance
- Base table anchored on `order_items` (one row per sale) with `products` left-joined on — never the reverse, which would inflate category counts with catalog products that were never sold.
- **Q1** — top categories by item count vs. by gross revenue; rankings diverge (e.g. `cama_mesa_banho` leads on count, `beleza_saude`/`relogios_presentes` lead on revenue).
- **Q2** — average item price per category (revenue ÷ that category's own item count).
- **Q3** — freight ratio per category computed as freight ÷ *price* (not ÷ `item_total_value`, which already includes freight and would cap the ratio under 1.0, hiding categories where freight genuinely exceeds price). No category averages above 1.0 — freight never exceeds price on a category-wide basis, even though individual flagged items do.
- **Q4** — revenue share of `Delivery Expensive` items (freight > price): ~1.09% of total revenue despite being ~3.7% of transactions — confirms these are low-value items, not high-value ones. Rate (not raw count) used to find which categories concentrate this pattern, since raw count is dominated by high-volume categories regardless of true severity. `casa_conforto_2` leads both the freight-ratio and the rate ranking — a consistent, well-supported finding.
- **Q5** — products with `listing_metadata_missing == True` (no category/title/description/photos) still generated ~1.31% of revenue across 1,603 sales from 610 distinct products — a real, non-trivial data-quality finding.
- **Q6** — revenue concentration: top 3/5/10/20 categories = 25.2% / 39.2% / 62.3% / 83.9% of total revenue — a steep Pareto curve.
- **Q7** — unique sellers per category.
- **Q8** — monthly revenue trend for the top 10 categories.
- **Q9** — average weight vs. average freight per category: the highest-absolute-freight categories are furniture/appliances (genuinely heavy, expected). The highest-freight-*ratio* categories (`casa_conforto_2`, `flores`, etc.) are comparatively light — their freight burden is a pricing-structure issue (cheap item, ordinary shipping fee), not a physical-bulk one.
- **Q10** — mean vs. median price per category, to flag which "average price" figures are reliable. Every one of the top 10 categories shows mean meaningfully above median (18%–86% gap) — right-skewed pricing is the norm, not the exception, so median (or both figures together) is the more honest "typical price" to report.

---

## Summary takeaways
1. Gross and net revenue should always be reported as a labeled pair, never blended.
2. Category revenue is sharply concentrated (Pareto-shaped) — see `05_customer_and_seller_intelligence_log.md` for the equivalent, steeper pattern on the seller side.
3. Freight burden has two distinct causes depending on the category: genuine physical weight (furniture/appliances) vs. a thin margin on cheap items (flowers, diapers, home-comfort goods) — these call for different fixes (logistics vs. pricing).
4. Payment mismatches and credit-card revenue share are both stable once normalized by volume — raw-count spikes in both cases are volume artifacts, not real rate changes.
5. Always re-derive a filtered table (like `deliveredOrders`) *after* every column it needs downstream has been added to the source — a filter takes a snapshot, it doesn't stay linked to later changes in the source frame.
