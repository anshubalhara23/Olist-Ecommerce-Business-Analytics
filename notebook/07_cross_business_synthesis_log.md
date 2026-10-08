# 07 — Cross-Business Synthesis Log
Olist E-Commerce Python Analytics

Documents what `07_cross_business_synthesis.ipynb` answers. Unlike 04-06, every question here deliberately combines two or more of those notebooks' lenses — the point of 07 is to connect sales, customer/seller, and delivery signals that none of the earlier notebooks could answer alone.

---

## Foundational decisions
- Each question rebuilds whichever base tables it needs from the cleaned CSVs directly (07 is its own notebook/kernel, same convention as 05→06) rather than inheriting variables from an earlier notebook.
- "First order" is defined at month granularity (`purchaseMonth`), using `.transform('min')` to broadcast each customer's earliest month back onto every one of their order rows, then filtering to rows where `purchaseMonth == firstPurchase`. 1,322 customers placed more than one order within their own first calendar month; these were deduplicated (`drop_duplicates('customer_unique_id', keep='first')`) before being treated as "the first order."

---

## A. Delivery → Customer behavior
**A.1 — Does a late first delivery predict lower repeat-buying?**
Repeat-customer rate by first-order `lateDelivery` status:
- Late Delivery: 2.62%
- On Time Delivery: 3.13%
- Unknown: 4.23%

Customers whose first order was late are about 16% relatively less likely to become repeat customers than those whose first order was on-time (2.62% vs. 3.13%) — consistent with 05's delivery-outcome-drives-reviews finding, now extended to retention, not just review scores.

The `Unknown` group's higher rate (4.23%) is a different story, not a contradiction: checking `order_status` for this group shows it's dominated by non-delivered statuses (`shipped`, `unavailable`, `canceled`, `invoiced`, `processing` — only ~1% are actually `delivered`). The likely explanation: a customer whose first order never arrived (canceled/unavailable) isn't demonstrating loyalty by ordering again — they're more likely just placing a second order to get the item they originally wanted. This "repeat" count may be measuring a failed-order retry, not genuine return behavior, and should be read with that caveat rather than taken at face value.

## B. Delivery → Revenue exposure
**B.2 — What % of total revenue sits in the seller-sparse, slow-delivery states?**
Seller-sparse states (TO, AL, RR, AP — no local seller presence, per the raw audit) account for **1.16%** of total revenue.

The ~2x delivery-time penalty found in 06 (22 vs. 12 days) is real and operationally severe per-order, but it sits in a financially small corner of the business. This isn't a case where an operational problem is quietly costing meaningful revenue — it's a severe but narrow issue affecting a small number of customers, not a material drag on the top line.

## C. Category → Delivery
**C.3 — Do freight-expensive items (freight > price) also show worse delivery outcomes?**
Late-delivery rate for freight-expensive items: 8.92% vs. 8.11% overall baseline — close, not a meaningful difference. These items are **not** meaningfully more likely to arrive late.

`Unknown`-status rate for freight-expensive items: 2.96% vs. 0.03% overall baseline — nearly **100x higher**. Freight-expensive items are far more likely to end up non-delivered (canceled/unavailable/stuck) than the general population. Interpretation: a "Delivery Expensive" item is disproportionately a cheap, low-margin item where the economics barely work — more likely to get deprioritized or canceled than actually delivered late. The real connection between freight-expensive categories and fulfillment problems is about **non-completion**, not lateness.

## D. Consolidated seller scorecard
**D.4 — Are high-revenue sellers also reliable, or are revenue and reliability separate?**
Built one combined table per seller (`sellerTenurePerf`): `totalRev`, `avgReviewScore`, `avgDelay`, `tenureTier`, by merging the pieces already computed separately in 05.

First pass with loose thresholds (top-50 revenue, review > 2.5, delay ≤ 0) found all 50 top-revenue sellers passing all three bars — but these bars were clearing 90%+ of *all* sellers, so the result was meaningless (too lenient to discriminate anything).

Tightened to `review > 4` and `delay ≤ median seller's delay`: of the top 50 revenue sellers, **16 pass all three bars**, **34 do not**. Breaking the 34 down by which bar they actually fail:
- 12 fail both review and delay
- 18 fail delay only (review already above 4)
- 4 fail review only (delay already better than median)

Delay drives most of the "weak" flag (30 of 34 fail on delay, vs. 16 of 34 on review) — meaning most of these high-revenue "weak" sellers already have solid customer satisfaction; they're just not quite as fast as the median seller, even though nearly all of them are still delivering early in absolute terms (most average -9 to -12 days). This is a softer risk than the loose-threshold pass initially suggested, and softer than it sounds from "34 of 50 are weak" alone.

## E. Consolidated state scorecard
**Not yet implemented.** Planned: one table per state combining revenue, customer count, repeat rate (05) alongside delivery time and seller-sparsity (06), to find which states look healthy on revenue alone but are quietly underserved operationally.

## F. Timeline overlay
**Not yet implemented.** Planned: overlay revenue growth, late-delivery rate, and new-customer-acquisition rate on one monthly timeline, to check whether the delivery-performance spikes found in 06 (Nov 2017, Feb/Mar 2018) coincide with any dip in revenue or acquisition.

---

## Summary takeaways
1. Delivery outcome affects more than satisfaction scores — it measurably affects whether a customer returns at all (A.1), extending 05's review-score finding into actual retention behavior.
2. The seller-sparse delivery problem (06) is operationally severe but financially small (B.2) — worth fixing for customer experience, not a revenue priority.
3. Freight-expensive items have a non-completion problem, not a lateness problem (C.3) — a subtly different fix (why are these orders getting canceled?) than what "freight-expensive" alone would suggest.
4. High revenue and high reliability mostly go together among top sellers, but a meaningful third (34 of 50) don't fully clear a reasonably strict bar — and for most of them, the gap is a delay-speed issue, not a satisfaction issue (D.4).
5. Threshold choice changes the story entirely — the same sellers looked "all reliable" under loose bars and "largely weak" under strict ones. Any segmentation claim needs its thresholds stated explicitly, not just the resulting counts.
