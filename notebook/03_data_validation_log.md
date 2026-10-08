# 03 — Data Validation Log
Olist E-Commerce Python Analytics

Documents what `03_data_validation.ipynb` checks, table by table, and why each check exists. The guiding question throughout: did the cleaning do *exactly* what `02_data_cleaning_log.md` claims it did — nothing more, nothing less?

---

## Customers
- `customer_city.nunique()` compared raw vs. clean — expected to drop, reflecting the strip/lower normalization.
- Confirmed the digit-containing city (`quilometro 14 do mutum`) is still present unchanged — it's a real place name, not something cleaning should have touched.
- `customer_zip_code_prefix`: confirmed every value is now a zero-padded 5-character string, and reconstructing the raw int prefixes with the same zero-padding produces an exact match — catches any value that was mangled in the dtype conversion.
- Row count checked unchanged (99,441 both sides).

## Orders
- Row count checked unchanged.
- `validDateCheck` category counts checked to sum to the full row count (no rows uncategorized).
- For `Approved After Carrier Delivered` rows: confirmed `order_approved_at` and `order_delivered_carrier_date` are now null, and `order_delivered_customer_date` is untouched (still populated wherever raw had it).
- For `Carrier After Customer` rows: confirmed `order_delivered_carrier_date` and `order_delivered_customer_date` are now null, and `order_approved_at` is untouched.
- Confirmed zero rows are flagged by both violation categories simultaneously — verifies the `if/elif` branching in `orderCheck` isn't silently hiding a genuine double-violation.
- `delivery_days` / `delivery_delay_days`: checked for negative values (should be none) and confirmed nulls line up exactly with rows that have a null underlying date.
- `lateDelivery`: confirmed it's `Unknown` only where a date is actually missing, and never `Unknown` when both dates are present.

## Order Items
- Row count checked unchanged.
- `item_total_value` re-verified to equal `price + freight_value` for every row individually, not just on average.
- `priceFreightCheck` category counts cross-checked against the audit's original findings.
- The "cheap/bulky" story behind `Delivery Expensive` re-run against the cleaned, joined `products` table, rather than trusting the cleaning notebook's finding without re-confirming it downstream.

## Payments
- Row count checked unchanged.
- Confirmed zero remaining `payment_installments == 0`.
- `zero_value_flag` counts cross-checked against the audit (3 Broken Record, 6 Voucher Full Coverage).
- Confirmed the installments fix touched only the two known rows, and that `payment_value` / `payment_type` on those rows are otherwise unchanged from raw.

## Payments vs Order Items — Reconciliation
- Re-ran the per-order `item_total_value` sum vs. per-order `payment_value` sum comparison on the fully cleaned tables, to confirm the mismatch count is understood and stable (not something introduced by later cleaning steps in other tables).

## Reviews
- Quantified the exact increase in comment-column null counts from raw to clean (whitespace-to-NaN conversion), rather than assuming it's negligible.
- Confirmed `order_level_reviews` has exactly one row per unique `order_id` in the raw table — no order lost or duplicated in the collapse.
- Confirmed zero remaining nulls in `review_score_std`, and specifically that every single-review order shows `std == 0` (not just "not null" — the actual expected value).
- Re-ran the original audit sanity checks (`creation_date > answer_timestamp`, score out of 1–5 range) against the cleaned copy — both expected empty.

## Products
- Row count checked unchanged.
- Confirmed the renamed columns (`product_name_length`, `product_description_length`) exist and the old misspelled names don't.
- Confirmed `product_category_name_english` nulls equal `listing_metadata_missing` sum *exactly* — an off-by-even-one mismatch would signal the manual translation merge went wrong.
- Confirmed zero remaining `product_weight_g == 0`, and recomputed the category median independently to confirm the imputed value is plausible, not an outlier artifact.
- Confirmed `dimensions_missing` count is unchanged from the audit (2 rows) — nothing in cleaning should have touched these.

## Sellers
- Row count checked unchanged.
- Confirmed zero remaining digit-containing city names.
- Confirmed the recovered city name for the fixed row is a real, known city — not itself a stray value.
- `seller_zip_code_prefix`: same zero-padding reconstruction check as customers.

## Geolocation
The most heavily transformed table — validated as a chain, so each drop is accounted for individually rather than checked as one combined "smaller than before" number:
1. Raw rows → after `drop_duplicates()` — drop count checked.
2. → after Brazil bounding-box filter — drop count checked.
3. → after re-running the state-conflict filter on this same post-filter data, confirming zero rows now disagree with their own prefix's modal state.
4. → final unique-prefix count compared directly against `geo_lookup`'s row count.
- Confirmed the 5 lost prefixes are the *same* 5 identified in cleaning — nothing else is silently dropping additional prefixes beyond what was documented.
- Confirmed `geo_lookup` has zero duplicate prefixes.

## Cross-Table — Zip Prefix Joins After the Dtype Change
Three zip prefix columns changed from `int` to zero-padded string during cleaning — this is the one place a per-table fix could quietly break something downstream without any single-table check catching it.
- Customers → `geo_lookup` match rate compared under the raw int dtype vs. the cleaned string dtype — confirms the match rate didn't drop.
- Sellers → `geo_lookup` match rate — same comparison.

## Summary Block
Final cell runs a consolidated pass/fail check across the highest-value assertions from every table above (row counts, all fix-specific zero-counts, the translation-merge equality, geo_lookup uniqueness, the no-double-flag check), printed as a simple `PASS`/`FAIL` list — meant as a fast regression check if `02_data_cleaning.ipynb` is ever edited later.
