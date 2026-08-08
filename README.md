# GB885-Final-Project---Strong---S
Final Project for GB885 - Case is Analytics for Rush Sportswear - Ingest, clean, and analyze data
Data Sources

Three source files (Excel/CSV exports, ~562 KB total):

TABLE_SALES_885 — transaction-level sales data (~9,648 rows)
TABLE_RETAILER_885 — retailer/location lookup table (110 rows)
TABLE_PRODUCTS_885 — product category lookup table (6 rows)
Data Cleaning Log

This documents the data quality issues found that materially affect the analysis or final numbers — the investigation behind each, and the decision made. Routine steps (parsing, dtype conversions) are omitted; this covers judgment calls only.

1. Orphan RETAILER_ID (999999999)

One sales row (an Outlet sale) has RETAILER_ID = 999999999, which doesn't exist in the retailer lookup table. Checked whether this was a systematic placeholder tied to a specific SALES_METHOD — ruled out, since Outlet and Online sales both use many other legitimate retailer IDs elsewhere in the data. It's an isolated, unattributable row.

Decision: kept in the full/product-level dataset (its product and sales figures are still valid), but excluded from any state- or retailer-level analysis, since it can't be attributed to either.

2. Duplicate RETAILER_IDs — key collision bug

4 RETAILER_IDs in the retailer table are not unique. The ID appears to be generated from [Retailer initial][00][Region initial][State abbrev][first 2 letters of city], which collides whenever two different values share those initials:

S00NNENE → maps to the same retailer (Sports Direct) but two different states (NJ/Newark and NY/New York).
W00SARLI, W00SFLOR, W00STEHO → each maps to two different retailers (Walmart and West Gear) in the same city.

Affects 623 sales rows (~6.5% of the dataset). Considered a 50/50 split between the colliding values but rejected it — there's no basis for the split ratio, and it would fabricate transaction-level attribution rather than honestly represent the ambiguity.

Decision: exclude ambiguous rows only from the specific analyses where the ambiguity applies:

Product-level analysis: no exclusions needed.
State-level analysis: exclude 999999999 and S00NNENE.
Retailer-level analysis: exclude 999999999, W00SARLI, W00SFLOR, W00STEHO.
3. PRICE_PER_UNIT — nulls and sentinel value

2 rows had a null PRICE_PER_UNIT (both Product 20). A broader check also found a 99999 placeholder mixed into otherwise normal prices ($7–$95) for the same product.

Decision: imputed both the nulls and the 99999 row using the median price for Product 20 (excluding the sentinel from the median calculation) = 45.

4. UNITS_SOLD — non-numeric sentinel

A systematic check for non-numeric values in numeric columns found '***' in UNITS_SOLD on 2 rows, both Product 20.

Decision: imputed using the median UNITS_SOLD for the matching retailer + product

sales method combination (more precise than a blanket product-wide median, since each retailer had a long enough order history to support it).
5. UNITS_SOLD — zero values, handled inconsistently on purpose

4 rows (2 retailers, Product 130) had UNITS_SOLD == 0. Investigated each retailer's full order history for the product to judge whether the zero was a real anomaly or a plausible data point:

F00SVIRI: normally sold in the 300s–500s range for this product; the zeros were a sharp, short-lived dip surrounded by otherwise-consistent high volume — the shape of a real interruption. Imputed using the median of that retailer's other Outlet orders for this product.
S00MMIDE: was already in a low, ramping-up sales period for this product when the zeros occurred (14 → 0 → 0 → 7 → 39 → 46, climbing into the hundreds by August) — the zeros fit plausibly within a slow-start pattern. Left as-is.

This is a deliberate inconsistency: two visually identical data points (both 0, same product) were treated differently because their surrounding context supported different conclusions about whether they represented real anomalies.

6. Minor fixes
SALES_METHOD had a typo variant, "Ootlet", corrected to "Outlet".
A full categorical value check (.value_counts() across all categorical columns in all three tables) found no other typos or inconsistencies.
