# 06 — Fulfillment & Delivery Analysis Log
Olist E-Commerce Python Analytics

Documents what `06_fulfillment_and_delivery_analysis.ipynb` answers, section by section. Covers the logistics pipeline itself: where delivery time gets spent, how reliable it is, whether geography changes it, and whether delivery performance/estimates shifted over time.

---

## Foundational decisions
- Reuses the cleaned `orders`, `order_items`, `customers`, `sellers` tables; `purchaseMonth` rebuilt the same way as 04/05.
- Every "delivered" population check is restricted to `order_status == 'delivered'` before computing a rate or average — same population-matching discipline as 04/05, after the `deliveredOrders`/`purchaseMonth` ordering bug caught in 04.
- Any month used in a trend is checked against order volume first (`order_status` by month) before its rate/average is trusted — several months in this dataset have too little or zero volume to support a conclusion on their own.

---

## Section A — Delivery Timeline Breakdown
- Three legs measured: purchase→approval, approval→carrier handoff, carrier→customer delivery.
- **Data-quality check, carrier-before-approval**: 1,183 rows (~1.2%) show `order_delivered_carrier_date` earlier than `order_approved_at`. 1,174 of those (99.2%) are `delivered` orders — this is timestamp-ordering noise on otherwise-successful orders, not a sign of broken ones, and is not excluded from the leg calculations.
- **Data-quality check, missing dates**: 336 / 1,982 / 2,988 rows missing `order_approved_at` / `order_delivered_carrier_date` / `order_delivered_customer_date` respectively — the increasing count down the pipeline lines up with orders that simply didn't progress that far (confirmed against `order_status`: non-delivered statuses dominate each NaN group). One genuine small gap in each: 190 / 201 / 31 `delivered` orders are missing a timestamp despite completing — under 0.3% each, footnoted rather than excluded.
- **Q1-Q3** — leg durations, mean vs. median: all three legs are right-skewed, Leg 1 (approval) worst by far — median 20 minutes vs. mean 10h12min (~30x). Leg 2 median 1d19h vs. mean 2d19h. Leg 3 median 7d2h vs. mean 9d8h. Median is the honest "typical" figure for every leg, especially Leg 1.
- **Q4** — Leg 3 (carrier→customer) dominates total delivery time (median ~7 days vs. ~20 minutes for Leg 1 and ~1.8 days for Leg 2) and has the largest *absolute* variability (std ~8.8 days). Leg 1 has the highest *relative* variability — its std (~1 day) is ~75x its own median (20 minutes), confirming it's the most outlier-driven leg despite being shortest on average.

## Section B — On-Time vs. Late Delivery, Overall
- **Q5** — 91.86% of delivered orders are on-time, 8.11% late, 0.03% unknown.
- **Q6** — among late orders, median delay is 5 days, mean 8.87 days (~77% higher) — same right-skew pattern as everywhere else in this project. Checking the 10 most extreme late orders (150-188 days) surfaced a data artifact: several share a near-identical `order_delivered_customer_date` (multiple landing on 2017-09-19) — a batch data-entry event, not genuine individual lateness. Flagged as a data-quality note; these extreme rows should not be read as real 150+ day deliveries.
- **Q7** — monthly late-rate: 2016-09/10/12 show extreme rates (up to 100%) but are near-empty ramp-up months (as few as 1 delivered order) — noise, not signal. Among months with real volume, late-rate sits in a 3-8% band through most of 2017, but spikes to 14.3% (Nov 2017) and 16.0%/21.4% (Feb/Mar 2018) — all high-volume months (6,500-7,300 delivered orders), so these are genuine signals. Nov 2017 lines up with Black Friday-season demand; Feb/Mar 2018 has no obvious seasonal driver and would need a follow-up look to explain.
- **Population check** — `order_status` by month shows 2018-09/10 have **zero** delivered orders at all (not just few) and must be excluded outright from any delivered-only trend; 2016-09/12 have only 1 delivered order each.
- **Q8** — delivery-estimate accuracy: negative = delivered early. Excluding the zero/near-zero-volume months above, the early-delivery buffer shrinks from ~-19/-27 days in early 2017 to ~-6/-13 days by 2018. Checked against actual `delivery_days` over the same window: it did **not** increase — it dropped to 7-9 days by mid/late 2018 (from 11-15 days in 2017). So the shrinking buffer reflects Olist tightening its delivery **estimates** while real delivery performance was simultaneously improving, not deliveries getting slower and eating into a fixed buffer.

## Section C — Geographic Fulfillment Patterns
- **Q9** — same-state vs. cross-state delivery: same-state orders deliver in **7.46 days** on average vs. **14.59 days** for cross-state — nearly twice as fast. On the estimate-buffer side, cross-state orders average `delivery_delay_days` of -12.91 (12.91 days early) vs. same-state's -10.46 — counterintuitive at first, but consistent with Olist's estimate already accounting for distance: cross-state orders get a longer promised window, so even though they take longer in absolute terms, they still land with a bigger early-buffer than the faster same-state orders.
- **Q10** — worst seller-state→customer-state pairs by average delay (pairs with ≥30 orders only, to avoid small-sample noise): every pair in the top 15 is still negative (early on average) — "worst" here means *closest to being late*, not actually late. PR→AL (-4.33 days, n=43) has the thinnest buffer; SP→SP, the largest pair by far (n=35,425, the bulk of all orders), sits mid-pack at -10.22. No pair in this list shows genuine average lateness.
- **Q11** — seller-sparse customer states (TO, AL, RR, AP — no local seller presence, per the raw audit) average **22.04 days** delivery time vs. **11.93 days** for all other states — nearly double. This is the single largest effect size found in this notebook; worth highlighting as the headline geographic finding.

## Section D — Order Status / Non-Fulfillment
**Not yet implemented.** Planned:
- Q12. What % of all orders fall into each `order_status`?
- Q13. For non-delivered orders, is there a pattern in how long they sit before being marked canceled/unavailable?
- Q14. Does freight cost or product category correlate with a higher cancellation/unavailability rate?

## Section E — Delivery Trends Over Time
- **Q15** — full-range monthly `delivery_days`: ramp-up months (2016-09/10/12) are noisy outliers as established in Section B; among real-volume months, delivery time holds mostly in a 10-16 day band through 2017 and early 2018 (with the same Nov 2017-Mar 2018 bump seen in the late-rate trend, 14.7-16.5 days), then drops sharply to 7-9 days by Jun-Aug 2018 — confirms the Q8 finding that delivery performance genuinely improved in the back half of the dataset.
- **Q16** — calendar-month seasonality (grouping by month number across all years, ignoring year): a visible pattern — delivery time rises through Nov-Mar (14.7-15.7 days) and falls through Jun-Aug (8.6-9.7 days), with Apr/May/Sep/Oct as a transition band (10.9-11.9 days). This lines up with Brazilian summer (Dec-Mar, including peak holiday shopping) being the slower period and Brazilian winter (Jun-Aug) being the faster one. **Caveat**: the dataset only spans ~2 full years, so each calendar month's "seasonal" average is really a *volume-weighted* blend of just 1-2 actual years (e.g. October blends 265 orders from the 2016 ramp-up, averaging 19.2 days, with 4,478 orders from 2017 at 11.4 days — a small but real upward pull on October's figure from the ramp-up period). Treat this as an exploratory, directionally-interesting pattern rather than a confirmed multi-year seasonal effect.

---

## Summary takeaways
1. The carrier→customer leg (Leg 3) dominates total delivery time by a wide margin — any effort to speed up delivery should focus there first, not on approval or carrier-handoff.
2. On-time delivery is the norm (91.86%), and the two statistically real late-rate spikes (Nov 2017, Feb/Mar 2018) occurred in high-volume months, not small-sample noise — worth investigating further if pursued.
3. Olist's delivery estimates got tighter over time while actual delivery performance simultaneously improved — two independent positive trends, not one masking a problem in the other.
4. Geography has the largest effect sizes found in this notebook: same-state delivery is ~2x faster than cross-state, and seller-sparse customer states take ~2x longer than the rest of the country — both substantially larger effects than anything found by seller size, payment type, or category in notebooks 04/05.
5. Always check real order volume before trusting a monthly or categorical average — several of this notebook's most dramatic-looking numbers (2016-09's "100% late rate", 2018-09/10's missing delivered orders) turned out to be artifacts of near-zero sample size, not genuine signal.
