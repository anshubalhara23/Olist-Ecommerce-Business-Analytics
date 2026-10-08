# 02 — Data Cleaning Log
Olist E-Commerce Python Analytics

Documents the cleaning actions taken in `02_data_cleaning.ipynb`, table by table, against the findings from `01_raw_data_audit.md`. Convention: raw files are copied once into `data/cleaned/` as working copies, modified in place, then saved at the end. No table goes through a generic one-size-fits-all loop — each fix is a specific, justified decision.

---

## Customers
- `customer_city`: stripped and lowercased to collapse casing/whitespace duplicates.
- `customer_zip_code_prefix`: converted from `int64` to a zero-padded 5-character string (see "Zip Code Normalization" below).
- No rows dropped, no other changes — this is the cleanest table in the dataset.

## Orders
- All 5 date columns converted to `datetime`, with an assertion that conversion introduces no new nulls.
- **`lateDelivery` flag** — labels each order `Late Delivery` / `On Time Delivery` / `Unknown`. This is a business outcome, never "corrected."
- **`validDateCheck` flag** — labels genuine date-order violations:
  - `Approved After Carrier Delivered`: gap > 2-day cutoff (chosen from the gap distribution — median ~17hrs, 75th pct ~1 day — to isolate the ~176 genuine errors from ~1,359 rows of normal logging noise).
  - `Carrier After Customer`: always an error, no cutoff needed (23 rows).
  - Verified no row is flagged by both conditions simultaneously (the `if/elif` branching doesn't hide a double-violation).
- **Flag + null**: for rows in either violation category, the specific contradictory date fields are nulled (not the whole row, not silently dropped) so downstream aggregates can't be corrupted by impossible dates. The flag column stays for auditability.
- `lateDelivery` recomputed *after* nulling, so the 23 `Carrier After Customer` rows correctly fall to `Unknown` post-fix.
- Derived `delivery_days` and `delivery_delay_days` — NaN wherever the underlying dates are null, by design.
- **Known, documented, unfixed anomaly:** the 1 `shipped` order with no items/carrier date/customer date is kept in order-count totals but naturally excluded from delivery/item metrics.

## Order Items
- `shipping_limit_date` converted to `datetime`.
- **`priceFreightCheck` flag** — labels `Delivery Expensive` (freight > price) and `Free Delivery` (freight == 0), each verified against real patterns before being treated as legitimate business behavior rather than assumed:
  - `Free Delivery`: confirmed concentrated in 9 sellers, 96% from the top 4 — a seller promotion pattern.
  - `Delivery Expensive`: checked against product category and price distribution to confirm it aligns with cheap/bulky items rather than being random noise.
- `item_total_value = price + freight_value` — deliberately not called "profit" or "revenue" since Olist provides no COGS data.

## Payments
- **`zero_value_flag`** — separates the two different $0 patterns found in the audit: `Broken Record` (3 `not_defined` rows — genuinely broken) vs. `Voucher Full Coverage` (6 `voucher` rows — legitimate).
- `payment_installments == 0` corrected to `1` — both affected rows are real credit-card payments with nonzero values, so 0 has no valid meaning and the fix is unambiguous (flag-and-fix, not flag-and-leave).

## Payments vs Order Items — Cross-Table Reconciliation *(new since the raw audit)*
- Per-order `item_total_value` sum compared against per-order `payment_value` sum.
- This is the one check spanning two independently-collected tables rather than validating a table against itself — catches a different, more serious class of error than any single-table check can.

## Reviews
- `review_creation_date` and `review_answer_timestamp` converted to `datetime`.
- Whitespace-only comment strings normalized to true `NaN` (they passed `isna()` as non-null but carried no real content).
- Re-ran the audit's sanity checks (`creation_date > answer_timestamp`, `review_score` out of 1–5 range) against the cleaned copy — both expected empty.
- **`slow_response` flag** — `Slow Response` where the answer gap exceeds a 10-day cutoff (chosen from the actual gap distribution: median ~1.7 days, 75th pct ~3 days — captures only the genuine long tail, ~3% of reviews).
- **`order_level_reviews`** built to resolve the many-to-many `review_id`↔`order_id` relationship (551 orders with multiple reviews; 814 review_ids spanning multiple orders) into one row per order. Keeps `review_score_std` alongside the mean, so a volatile order (scores 1,5 → avg 3) is distinguishable from a consistently mediocre one (scores 3,3 → avg 3).
- `review_score_std` nulls (present for every single-review order — the majority) filled with `0`, since "only one review, no variance" is a real value, not an unknown.
- **Always join `order_level_reviews` for order-level analysis — never the raw `order_reviews` table directly.**

## Products
- Renamed `product_name_lenght` → `product_name_length`, `product_description_lenght` → `product_description_length` (fixing the source data's own spelling).
- Two missingness patterns kept as separate flags, not merged into one "broken" label:
  - `listing_metadata_missing` (610 rows) — no category/title/description/photos, but dimensions intact — still usable for shipping/freight analysis.
  - `dimensions_missing` (2 rows) — no weight/size — these actually break freight calculations.
- `product_weight_g == 0` (4 rows, all `cama_mesa_banho`) imputed with that category's median, computed from valid nonzero weights only so the broken rows don't pollute their own fix.
- Manually mapped the 2 untranslated categories (`pc_gamer` → `gaming_pc`, `portateis_cozinha_e_preparadores_de_alimentos` → `kitchen_appliances_and_food_preparers`) into `category_translation` before merging, rather than allowing silent nulls.
- Verified `product_category_name_english` null count equals `listing_metadata_missing` exactly — confirms the merge introduced no new nulls.

## Sellers
- **Digit-in-city fix** *(new since the raw audit)*: the row with `seller_city = "04482255"` had its real city recovered via that seller's own zip prefix, looked up directly against raw geolocation data (since `geo_lookup` doesn't exist yet at this point in the notebook).
- No other changes — audit confirmed no nulls, duplicates, blank strings, or zip range issues, and city text already collapses cleanly under strip+lower.
- `seller_zip_code_prefix` converted to zero-padded string (see below).

## Geolocation
- Confirmed the 8,556 city-name conflicts per prefix are pure accent/encoding noise — parked as a future enhancement (accent-stripping), resolved for now via modal aggregation.
- Dropped 261,831 full-row duplicates.
- Filtered to Brazil's actual bounding box (lat -33.75 to 5.27, lng -73.99 to -34.79) — drops 8 rows with impossible coordinates.
- **State-conflict check** *(new since the raw audit)*: checked how many zip prefixes span more than one distinct state post-filter, then kept only rows matching each prefix's modal state before averaging — because averaging lat/lng across genuinely different states would produce a meaningless centroid, unlike the city-name case where a mode-picked label is harmless.
- Collapsed to one row per zip prefix (`geo_lookup`): mean lat/lng, modal city/state.
- Result: 19,010 prefixes — 5 fewer than raw's 19,015. Confirmed real-world impact: these 5 prefixes lost every row to the bounding-box filter, affecting 1 customer and 0 sellers — documented as negligible.

## Zip Code Prefix Normalization *(new since the raw audit)*
- `customer_zip_code_prefix`, `seller_zip_code_prefix`, and `geolocation_zip_code_prefix` all loaded as `int64`, silently dropping leading zeros on what are categorical codes, not real numbers.
- Converted all three to zero-padded 5-character strings **together**, in the same step — doing this to only one side of a future join would break matching between an int column and a zero-padded string column.

---

## Raw-vs-Clean Reconciliation (closing checkpoint)

| Table | Raw rows | Clean rows | Note |
|---|---|---|---|
| customers | 99,441 | 99,441 | unchanged |
| orders | 99,441 | 99,441 | unchanged (only dates nulled) |
| order_items | 112,650 | 112,650 | unchanged |
| order_payments | 103,886 | 103,886 | unchanged |
| order_reviews | 99,224 | 99,224 | unchanged |
| products | 32,951 | 32,951 | unchanged |
| sellers | 3,095 | 3,095 | unchanged |
| geolocation | 1,000,163 | 782,331 (pre-aggregation) → 19,010 (`geo_lookup`) | large intentional drop: duplicates + bounding box + state conflicts |

Confirmed at save time:
- `payment_installments == 0` → 0 remaining.
- `product_weight_g == 0` → 0 remaining.
- `seller_city` digit-containing rows → 0 remaining.
- Contradictory order dates → only the untouched field in each anomaly group remains populated.

## Outputs saved to `data/cleaned/`
`olist_customers_dataset.csv`, `olist_orders_dataset.csv`, `olist_order_items_dataset.csv`, `olist_order_payments_dataset.csv`, `olist_order_reviews_dataset.csv`, `order_level_reviews.csv`, `olist_products_dataset.csv`, `product_category_name_translation.csv`, `olist_sellers_dataset.csv`, `geo_lookup.csv`.
