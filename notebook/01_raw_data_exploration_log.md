# 01 — Raw Data Audit
Olist E-Commerce Python Analytics

Findings from `01_raw_data_exploration.ipynb`, documented per table before any cleaning decisions are made. Each section lists what was checked, what was found, and whether it's flagged as an issue or a legitimate business pattern.

---

## Customers
- 99,441 rows, 5 columns, zero nulls, zero full-row duplicates.
- `customer_id` (99,441 unique) is a per-order surrogate key; `customer_unique_id` (96,096 unique) identifies the real person. **Always use `customer_unique_id` for repeat-customer/RFM analysis.**
- `customer_state`: 27 unique values, all valid Brazilian state codes.
- `customer_city`: 4,119 unique values before normalization; whitespace/casing checked, one city legitimately contains digits (`quilometro 14 do mutum` — a real rural place name, not an error).
- `customer_zip_code_prefix`: loaded as `int64`, range 1,001–99,990. **Issue:** this silently drops leading zeros on real 5-digit prefixes (e.g. `01001` → `1001`). Flagged for cleaning.

## Orders
- 99,441 rows, 8 columns.
- Nulls: `order_approved_at` (160), `order_delivered_carrier_date` (1,783), `order_delivered_customer_date` (2,965) — all explained by non-delivered order statuses (canceled, unavailable, created, etc.), except 8 `delivered` orders missing a customer delivery date.
- `order_status` counts: `delivered` 96,478; `shipped` 1,107; `canceled` 625; `unavailable` 609; `invoiced` 314; `processing` 301; `created` 5; `approved` 2. The last two are rare categories worth bucketing carefully in downstream analysis.
- Date-order violations found:
  - `order_approved_at > order_delivered_carrier_date`: 1,359 rows. Gap distribution: median ~17hrs, 75th pct ~1 day, max 171 days. A flat cutoff is needed to separate normal system-logging noise from genuine errors — the max-171-day case is extreme enough to warrant a manual look.
  - `order_delivered_carrier_date > order_delivered_customer_date`: 23 rows (physically impossible — always an error).
  - `order_delivered_customer_date > order_estimated_delivery_date`: 7,827 rows — this is a real business outcome (late delivery), not a data error.

## Order Items
- 112,650 rows, 7 columns, zero nulls, zero duplicates.
- 98,666 unique orders, 32,951 unique products, up to 21 items on a single order.
- `price` min 0.85, max 6,735; `freight_value` min 0, max 409.68.
- `freight_value == 0`: 383 rows, concentrated in just 9 sellers (96% from the top 4) — looks like a seller-level free-shipping pattern rather than random missing data.
- `freight_value > price`: 4,124 rows (~3.7% of items) — worth checking whether this is concentrated in specific categories/price ranges before treating it as a settled pattern.

## Order Payments
- 103,886 rows, 5 columns, zero nulls, zero duplicates.
- `payment_type` counts: credit_card 76,795; boleto 19,784; voucher 5,775; debit_card 1,529; **not_defined 3**.
- `payment_value == 0`: 9 rows total — 3 are `not_defined` (genuinely broken: unknown type AND no value), 6 are `voucher` (legitimate: voucher fully covered the order). These need separate treatment, not one blanket flag.
- `payment_installments == 0`: 2 rows, both real `credit_card` payments with nonzero values — a clear data-entry error, correct value obvious (1).
- Payment total (`payment_value.sum()`) reconciles with itself trivially; **not yet checked against `order_items` totals** — this cross-table check is the one still missing at audit stage.

## Reviews
- 99,224 rows, 7 columns.
- `review_comment_title` missing in 87,656 rows, `review_comment_message` missing in 58,247 — expected, since comments are optional.
- `review_score` distribution: 5★ (57,328), 4★ (19,142), 1★ (11,424), 3★ (8,179), 2★ (3,151).
- Many-to-many relationship with orders: 551 orders have 2–3 reviews each; conversely, some `review_id`s are attached to 2–3 *different* `order_id`s with identical creation/answer timestamps — worth understanding whether this is a genuine one-review-many-orders design (bundled checkout) or an export artifact, before deciding how to collapse to one row per order.

## Products
- 32,951 rows, 9 columns.
- `product_category_name`, `product_name_lenght`, `product_description_lenght`, `product_photos_qty` are all null together on the exact same 610 rows (correlated missingness) — dimensions (weight/length/height/width) are still present on these rows.
- 2 additional rows missing dimensions entirely.
- `product_weight_g == 0`: 4 rows, all in `cama_mesa_banho`, with real dimensions present — physically impossible, flagged for imputation.
- Column names `product_name_lenght` / `product_description_lenght` are misspelled in the source data.

## Category Translation
- 71 rows, no nulls, no duplicates, 71 unique category names mapping to 71 unique English names.
- 2 categories present in `products` are missing from this table entirely: `pc_gamer` and `portateis_cozinha_e_preparadores_de_alimentos` (13 products affected) — need manual translation.

## Sellers
- 3,095 rows, 4 columns, zero nulls, zero duplicates, zero blank strings.
- 23 unique seller states (customers span 27 — sellers are absent from `TO`, `AL`, `RR`, `AP`, a real business fact, not a data error).
- **Issue found:** one row has `seller_city` = `"04482255"` — a zip code sitting in the city field, not a real city name.
- Zip prefix range 1,001–99,730 — same leading-zero truncation issue as customers.

## Geolocation
- 1,000,163 rows, 5 columns, zero nulls.
- 19,015 unique zip prefixes; row count per prefix ranges 1–1,146.
- **261,831 full-row duplicates** — genuinely redundant, safe to drop.
- Lat/lng contain values far outside Brazil's actual bounding box (e.g. lat 45.07, lng 121.11) — 8 rows affected; a generic ±90/±180 range check would miss these entirely, so a Brazil-specific bounding box is needed.
- 8,556 of 19,015 prefixes have more than one distinct `geolocation_city` value — investigated and confirmed to be pure accent/encoding noise (`sao paulo` vs `são paulo`), not real geographic conflicts.
- **Not yet checked at audit stage:** whether any prefixes have more than one distinct `geolocation_state` — this matters more than the city conflict, since any lat/lng aggregation by prefix will need to avoid averaging across genuinely different states.
- 8 customer zip prefixes and 7 seller zip prefixes have no matching row in geolocation at all.

## Cross-table checks
- Customers ↔ Orders: fully consistent both directions (0 mismatches).
- Order Items ↔ Orders: 775 orders have no items — all explained by non-delivered statuses, except **1 `shipped` order with no items, no carrier date, no customer date** — a genuine incomplete record.
- Order Items ↔ Products / Sellers: fully consistent (0 mismatches).
- Payments ↔ Orders: 0 payments without a matching order; **1 order has no matching payment record** — needs inspection before deciding how to handle.
- Reviews ↔ Orders: fully consistent (0 mismatches).

---

## Open items carried into cleaning
1. Zip code leading-zero truncation (customers, sellers, geolocation).
2. Seller row with a zip code in the city field.
3. Geolocation state-conflict check (not yet run).
4. `freight_value > price` pattern not yet verified against category/price.
5. Payments-vs-order_items cross-table reconciliation not yet run.
6. The 1 order missing a payment record — needs inspection.
7. The 171-day order-approval outlier — needs a manual look.
