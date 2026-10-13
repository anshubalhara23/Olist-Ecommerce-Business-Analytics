# Olist E-Commerce Analytics — Project Summary
Full pipeline overview: raw audit → cleaning/validation → sales & revenue → customer & seller intelligence → fulfillment & delivery → cross-business synthesis → visualization (matplotlib/seaborn).

This file is the top-level map of the project. Each notebook has its own detailed `_log.md` with exact numbers, decisions, and caveats — this file ties them together and tells the story in order.

---

## Dataset
[Olist Brazilian E-Commerce Public Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) (Kaggle) — ~100k orders, 2016–2018. Download and place the raw CSVs in `data/raw/` before running notebooks 01-03 (raw data isn't committed to this repo — see `.gitignore`).

---

## 01–03 — Raw Audit, Cleaning, Validation

- **Raw audit**: surfaced the structural facts every later notebook relies on — `customer_id` is a per-order surrogate key (`customer_unique_id` is the real person); some states (TO, AL, RR, AP) have no local seller presence at all ("seller-sparse states"); a handful of `not_defined`/`Broken Record` payment rows exist at ₹0 value.
- **Cleaning**: produced the cleaned CSVs every downstream notebook loads (`orders`, `order_items`, `customers`, `sellers`, `products`, `payments`, etc.). Handled missing timestamps, categorical cleanup, and known data-quality rows (e.g. the 3 broken payment records).
- **Validation**: confirmed row counts and join integrity before anything downstream trusted a merge (e.g. `order_items` row count checked against the merged base table in 05/06 before building on top of it).

---

## 04 — Sales & Revenue Analysis
**Scope**: Overall Sales, Sales Over Time, Payments, Category Performance.

- Revenue defined as `item_total_value` (price + freight) throughout; gross (all orders) and net (delivered only) always reported as a labeled pair, never blended.
- Monthly trends built on `purchaseMonth`; 2016-09/10/12 and 2018-09/10 flagged as near-empty ramp-up/tail-off months excluded from growth narratives.
- Credit card dominates payment revenue (~79%) and is the only payment type where installments vary meaningfully (r≈0.38 with value).
- Category revenue is sharply Pareto-concentrated: top 3 categories = 25.2% of revenue, top 20 = 83.9%.
- Freight-burden categories split into two distinct causes: genuine weight (furniture/appliances) vs. thin margin on cheap items (flowers, home-comfort goods) — different fixes needed for each.
- Every "average price/value" figure checked against its median — right-skew is the norm, not the exception, across this entire dataset.

→ Full detail: `04_sales_and_revenue_analysis_log.md`

---

## 05 — Customer & Seller Intelligence
**Scope**: Seller Performance, Customer Value, plus extension questions on customer acquisition trend and customer geography.

- Seller revenue concentration is steeper than category concentration (04).
- Delivery outcome — not seller size — is the strongest driver of review score found in this notebook; non-delivery itself is an even stronger driver than mere lateness (sellers with zero delivered orders average review score 1.78 vs. 1.0 median).
- Repeat purchasing is not Olist's revenue engine at any point in its history: one-time buyers drive order volume in essentially every month (order-to-new-customer ratio sits at 1.0–1.04 nearly throughout), not just in the lifetime aggregate (97% one-time buyers).
- Customer value per person is inversely related to market size: SP (biggest, most urban) has the *lowest* average revenue per customer; small remote states have the highest, driven by both freight and item price.
- Geography predicts spend per customer but **not** loyalty — repeat rate shows no real state-level pattern.

→ Full detail: `05_customer_and_seller_intelligence_log.md`

---

## 06 — Fulfillment & Delivery Analysis
**Scope**: Delivery Timeline Breakdown, On-Time vs. Late, Geographic Patterns, Delivery Trends.

- Carrier→customer leg dominates total delivery time by a wide margin — any speed-up effort should target that leg first.
- On-time delivery is the norm (91.86%); two statistically real late-rate spikes found (Nov 2017, Feb/Mar 2018) — genuine, not small-sample noise, since both are high-volume months.
- Delivery estimates tightened over time *while* actual delivery performance simultaneously improved — two independent positive trends, not one masking the other.
- Geography produces the largest effect sizes in the whole project: same-state delivery ~2x faster than cross-state; seller-sparse customer states take ~2x longer (22.04 vs. 11.93 days) than the rest of the country.
- Repeated lesson: check real order volume before trusting any monthly/categorical average — several dramatic-looking numbers here (2016-09's "100% late rate", 2018-09/10's zero delivered orders) were near-zero-sample artifacts.

→ Full detail: `06_fulfillment_and_delivery_analysis_log.md`

---

## 07 — Cross-Business Synthesis
**Scope**: deliberately combines 2+ of notebooks 04/05/06's lenses per question — connecting sales, customer/seller, and delivery signals none of the earlier notebooks could answer alone.

- **A.1**: A late first delivery measurably lowers repeat-buying (2.62% vs. 3.13% repeat rate) — extends 05's delivery→review finding into actual retention behavior.
- **B.2**: The seller-sparse delivery penalty (06) is operationally severe but financially small — only 1.16% of total revenue sits in those states.
- **C.3**: Freight-expensive items have a *non-completion* problem (~100x higher Unknown/non-delivered rate), not a lateness problem — a different fix than "freight-expensive" alone would suggest.
- **D.4**: Among top-50-revenue sellers, 16/50 pass a strict reliability bar (review>4, delay≤median) and 34/50 don't — but most of the 34 fail only on delay speed, not satisfaction; the "weak seller" story is softer than the raw 34/50 split suggests, and the threshold choice itself changes the story entirely (loose bars passed all 50).

→ Full detail: `07_cross_business_synthesis_log.md`

---

## 08 — Key Insights Visualization (matplotlib + seaborn)
**Scope**: one notebook, 8 charts, each visualizing one finding already established in 04–07 rather than re-deriving new analysis.

| # | Source | Chart type | Finding visualized |
|---|--------|-----------|---------------------|
| 1 | 04 | Line chart | Monthly sales trend |
| 2 | 04 | Pareto chart (bar + cumulative line, dual axis) | Category revenue concentration |
| 3 | 05 | Heatmap | RFM Recency × Monetary cross-tab (near-uniform → independence finding) |
| 4 | 05 | Lorenz-curve-style cumulative concentration | Seller revenue concentration |
| 5 | 06 | Annotated line chart | Monthly late-delivery rate, with Nov 2017 / Feb-Mar 2018 spikes called out |
| 6 | 06 | Grouped bar (mean + median) | Same-state vs. cross-state delivery time |
| 7 | 07 | Bar chart | Repeat rate by first-order delivery status |
| 8 | 07 | Scatter plot with threshold lines | Seller scorecard — review vs. delay, top-50 revenue sellers highlighted |

Charts 3, 4, 6, 7, 8 read small pre-aggregated summary tables exported as CSVs from 05/06/07 (`../data/summary/*.csv`). Charts 1 and 2 recompute fresh from the cleaned base tables.

→ Full detail: `08_key_insights_visualization_log.md`

---

## How the story connects, end to end

1. Sales are healthy and growing, but revenue is Pareto-concentrated at both the category level (04) and the seller level (05) — a small number of categories/sellers carry most of the business.
2. Repeat buying is structurally absent — Olist runs on continuous new-customer acquisition, not loyalty, at every point in its history (05), and delivery outcome is one of the few levers that measurably moves whether someone *ever* returns (07, A.1).
3. Delivery performance is strong and improving, but has one real structural weak point: seller-sparse states, where delivery takes ~2x as long (06) — and cross-checking against revenue (07, B.2) shows this is a customer-experience problem worth fixing, not a revenue-at-risk problem.
4. Freight-expensive items looked like a lateness risk but are actually a non-completion risk (07, C.3) — the fix is about why orders get canceled, not about speeding up shipping.
5. High-revenue sellers are mostly — but not uniformly — reliable, and "reliability" itself is threshold-dependent (07, D.4): the same sellers can look fine or weak depending on where the bar is set, a caution that applies to any segmentation claim in this project.
