# 05 — Customer & Seller Intelligence Log
Olist E-Commerce Python Analytics

Documents what `05_customer_and_seller_intelligence.ipynb` answers, section by section. Covers Seller Performance and Customer Value (moved out of the original combined notebook), plus four extension questions written specifically for this standalone notebook — these are what make 05 more than a copy-paste of the old sections.

---

## Foundational decisions
- Reloads cleaned tables independently rather than inheriting from 04, since it now stands on its own; rebuilds `purchaseMonth` and `orderOverTime` itself.
- Same revenue/gross-net conventions as 04: `item_total_value` (price + freight) throughout.
- `customer_unique_id` (not `customer_id`) used wherever a real person, not a per-order surrogate key, is the unit of analysis.

---

## Section A — Seller Performance
- Base table anchored on `order_items`, with `sellers` and `products` left-joined on — confirmed row count matches `order_items` exactly (112,650) before building anything on top of it.
- **Q1** — top sellers by distinct order count (`nunique`, not item-row count) vs. by revenue; ~15 of the top 20 overlap between the two rankings.
- **Q2** — seller revenue concentration at multiple cutoffs (top 5/10/30/50/100).
- **Q3** — free-delivery orders (`freight_value == 0`) average ₹42 *less* in item value than normal orders — free delivery is associated with lower-value orders here, not a premium-order perk.
- **Q4** — seller-level average `delivery_delay_days` (confirmed sign convention: negative = delivered early, the norm for ~97% of sellers) vs. average review score: r ≈ −0.22 among sellers with delivered orders. Separately, sellers with **no** delivered orders at all (`NaN` delay) show a much more severe pattern — mean review score 1.78, median 1.0 — meaning non-delivery itself, not just lateness, is a stronger driver of bad reviews than the correlation alone suggests.
- **Q5** — seller size (total revenue) correlates with neither review score (r ≈ 0.02) nor delivery delay (r ≈ 0.003) — scale is essentially a non-factor for both outcomes.
- **Q6/Q7** — average order value and category count per seller, combined into one profile table alongside total revenue; top-10 sellers show two distinct business models (narrow high-ticket specialists vs. broad high-volume generalists).
- **Q9** — mean vs. median order value per seller, same reliability check as categories; some top sellers' "average" is nearly double their median, flagging those figures as outlier-driven.
- **Q10** — monthly revenue trend for the top 10 sellers by revenue; reveals that two of the top 10 (`53243585a1d6dc2643021fd1853d8905`, `7e93a43ef30c4f03f38b393420bc753a`) peaked early and have since declined sharply, while others (`1025f0e2d44...`, `955fee9216a...`) are still growing — "top 10 by total revenue" does not mean "top 10 currently."

### Extension — Seller additions
- **Q_new3 — Seller tenure**: each seller's first-sale month (`min(purchaseMonth)` per `seller_id`) split into tertiles (`Oldest`/`Mid`/`Newest`) via `pd.qcut` on the Period's integer ordinal (`.astype('int64')` — `qcut` needs a sortable numeric, and casting a Period to `str` breaks chronological order). Tertile split landed reasonably balanced: Oldest 1,128 / Mid 944 / Newest 1,023 sellers. **Status: tiers built and confirmed balanced; the performance comparison (avg revenue/review score/delay by tier) has been coded but not yet run/confirmed — pending next pass.**
- **Q_new4 — Seller geography**: revenue/delivery performance by `seller_state`, including whether sellers in seller-sparse states (no local seller presence, per the raw audit: TO, AL, RR, AP) see different customer outcomes. **Status: not yet implemented.**

---

## Section B — Customer Value
- **Q1** — repeat customers (`customer_unique_id` ordering ≥2 times, 2,997 people) generate 5.82% of total revenue; one-time customers (93,099 people) generate 94.18%. Average revenue per repeat customer (₹307.66) is nearly double the one-time average (₹160.28) — repeat customers ARE individually more valuable, but there are too few of them to move total revenue. Consistent with Olist's structure as a multi-seller marketplace rather than a single-brand destination.
- **Q2** — RFM-style segmentation, simplified to Recency × Monetary (Frequency dropped as a separate axis, since it would just reproduce the Q1 repeat/one-time split — 97% of customers ordered exactly once). Tertile cross-tab is **near-uniform across all 9 cells** — Recency and Monetary behave as independent variables. This is itself a finding: it confirms, from a different angle, that RFM's usual "loyal customer" signal doesn't apply to a customer base this dominated by one-time buyers.

### Extension — Customer additions
- **Q_new1 — Customer acquisition trend**: each customer's first-order month (`min(purchaseMonth)` per `customer_unique_id`), counted per month, compared against total order volume that month. The ratio of total orders to new-customer count sits in a tight **1.0–1.04 band for nearly the entire 2016-12 to 2018-08 period** — almost every order each month comes from a customer making their first-ever purchase that month, not a returning one. (2018-09/10 show much higher ratios — 3.2 and 4.0 — but both are the same near-empty tail-off months flagged in 04's Sales-Over-Time caveat, with single-digit order/new-customer counts; noise, not signal.) Confirms, from a monthly operational angle, the lifetime 97%-one-time-buyer finding from Section B: Olist runs on continuous new acquisition at every point in its history, not a growing repeat-customer base.
- **Q_new2 (Part 1) — Customer geography, revenue**: revenue, customer count, average revenue/price/freight per `customer_state`. SP dominates total revenue and customer count by a wide margin (biggest, most urban state) but has the **lowest** average revenue per customer (₹146.93) of all 27 states. Small, remote states (PB, AC, AP, RO, RR) show the highest average revenue per customer (₹240–272) — driven by BOTH higher average freight (remote states run ₹32–43 vs. SP's ₹15, consistent with longer shipping distances to seller-sparse states per the raw audit) AND higher average item price, so it isn't a freight-only effect — customers in remote states appear to buy noticeably pricier items too, not just pay more to ship similarly-priced ones.
- **Q_new2 (Part 2) — Customer geography, repeat rate**: repeat-customer rate (≥2 orders) by `customer_state`. No real geographic pattern — the high-volume states (SP, RJ, MG, RS, PR, each with thousands of customers) all sit in a tight 3.0–3.4% band, consistent with each other. The apparent extremes (AC at 5.19%, AP at 1.49%) are both small, seller-sparse states with under 80 unique customers — a swing of 2–3 repeat customers moves the rate by several points, so these are sample-size noise, not a real signal. Unlike Part 1 (where remoteness clearly tracked average revenue), repeat-buying behavior doesn't appear to be geography-driven at all.

---

## Summary takeaways
1. Seller revenue is sharply concentrated — steeper per-seller than category concentration is per-category (04, Q6).
2. Delivery outcome (not seller size) is the strongest driver of review score found in this notebook — and non-delivery is a stronger driver than mere lateness.
3. Repeat purchasing is not Olist's revenue engine, at any point in its history — one-time, first-time buyers drive order volume every single month, not just in the lifetime aggregate.
4. Customer value per person is inversely related to market size: the biggest, most urban state (SP) has the lowest average revenue per customer; small, remote states have the highest, driven by both freight and item price.
5. Geography predicts spend per customer but not loyalty (repeat rate) — those are two different behaviors and shouldn't be assumed to move together.
6. Seller tenure and seller geography (Q_new3 performance comparison, Q_new4) remain open — see Extension sections above for exact status.
