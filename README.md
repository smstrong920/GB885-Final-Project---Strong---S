# GB885-Final-Project---Strong---S

Final Project for GB885 - Case is Analytics for Rush Sportswear - Ingest, clean, and analyze data

## Project Purpose

RUSH is a global sportswear and footwear brand. Its raw sales data lives across three tables — SALES, RETAILER, and PRODUCT — and has not been cleaned, so it contains errors that must be identified and resolved before any analysis can be trusted.

The VP of US Sales commissioned this project with two goals: answer four specific business questions, and surface additional trends or opportunities in the data.

**The four questions:**

1. Which product category had the highest dollar sales in 2021, and how much?
2. Which state had the highest dollar sales of women's products in 2021, and how much?
3. Which state had the highest dollar sales of men's products in 2021, and how much?
4. Which retailer purchased the most units in 2021? In 2020?

**Approach.** The work splits into two notebooks: ingestion and cleaning, then analysis. Because the cleaning stage surfaced a key-collision bug that makes some rows unusable at certain grains but valid at others, the analysis runs against three purpose-built dataframes rather than one merged table — documented below alongside every judgment call made during cleaning.

**Deliverables.** This repository (code notebooks, cleaned datasets, documentation) and a 3–5 minute presentation of findings to the VP and sales team.

## Data Sources

Three source files (Excel/CSV exports, ~562 KB total):

- **TABLE_SALES_885** — transaction-level sales data (9,648 rows)
- **TABLE_RETAILER_885** — retailer-location lookup table (110 rows, 106 distinct IDs)
- **TABLE_PRODUCTS_885** — product category lookup table (6 rows)
- **RUSH_Data_Dictionary.pdf** — field definitions and key constraints

## Data Dictionary

Per the supplied `RUSH_Data_Dictionary.pdf`.

**SALES** (transaction grain)

| Field | Type | Description |
|---|---|---|
| ORDER_ID | CHAR | Unique identifier for each order (primary key) |
| RETAILER_ID | CHAR | Retailer-location combination (foreign key) |
| INVOICE_DATE | DATE | Date of order |
| PRODUCT_ID | CHAR | Product type (foreign key) |
| PRICE_PER_UNIT | FLOAT | Dollar price charged per unit |
| UNITS_SOLD | INT | Units sold in the order |
| OPERATING_MARGIN | FLOAT | Profit margin rate for the order |
| SALES_METHOD | VARCHAR | In-store, Outlet, or Online |

**RETAILER** (retailer-location grain)

| Field | Type | Description |
|---|---|---|
| RETAILER_ID | CHAR | Retailer-location combination (primary key) |
| RETAILER | CHAR | Retailer name |
| REGION | CHAR | Region of retail location |
| STATE | CHAR | State of retail location |
| CITY | CHAR | City of retail location |

**PRODUCT**

| Field | Type | Description |
|---|---|---|
| PRODUCT_ID | CHAR | Product type (primary key) |
| PRODUCT_NAME | VARCHAR | Long product name |

Two points from the dictionary bear directly on the cleaning decisions below.

**RETAILER_ID is a documented primary key**, so the duplicate values in issue 2 are a schema violation, not merely untidy data — the table does not satisfy its own stated contract. This is what justifies treating the affected rows as genuinely ambiguous rather than reconcilable.

**RETAILER_ID identifies a retailer-*location*, not a retailer.** The 110-row table holds 106 distinct location IDs mapping to only 6 retail chains (Foot Locker 33 locations, Sports Direct 25, West Gear 24, Kohl's 12, Amazon 9, Walmart 6). Analyses below distinguish the two grains explicitly: *location* for distribution and growth questions, *chain* (the RETAILER field) for customer-concentration questions.

Note also that UNITS_SOLD is specified as INT but arrives as a text column because of the sentinel values described in issue 4 — the dtype had to be restored rather than assumed.

## How to Run

Both notebooks are Google Colab notebooks and run start to finish via **Runtime → Run all**. Each prompts for a file upload early in the run, so download the required files from this repo first.

**1. Ingestion and Cleaning** — `GB885_Final_Project_Ingestion_and_Cleaning_Strong_S.ipynb`

Requires the three source files from `Raw Data/`:

- `TABLE_SALES_885.csv`
- `TABLE_RETAILER_885.csv`
- `TABLE_PRODUCTS_885.csv`

Outputs the three analysis dataframes described below.

**2. Analysis** — `GB885_Final_Project_Analysis_Strong_S.ipynb`

Requires the three cleaned files from `Cleaned Datasets/`:

- `df_product_and_sales.parquet`
- `df_state.parquet`
- `df_retailer.parquet`

Run the notebooks in order if starting from raw data. To reproduce the analysis alone, the second notebook runs independently against the cleaned files.

Note that Colab clears uploaded files when the runtime disconnects, so the upload step must be repeated in a new session.

## Data Cleaning Log

This documents the data quality issues found that materially affect the analysis or final numbers — the investigation behind each, and the decision made. Routine steps (parsing, dtype conversions) are omitted; this covers judgment calls only.

### 1. Orphan RETAILER_ID (999999999)

One sales row (an Outlet sale) has `RETAILER_ID = 999999999`, which doesn't exist in the retailer lookup table. Checked whether this was a systematic placeholder tied to a specific SALES_METHOD — ruled out, since Outlet and Online sales both use many other legitimate retailer IDs elsewhere in the data. It's an isolated, unattributable row.

**Decision:** kept in the full/product-level dataset (its product and sales figures are still valid), but excluded from any state- or retailer-level analysis, since it can't be attributed to either.

### 2. Duplicate RETAILER_IDs — key collision bug

4 RETAILER_IDs in the retailer table are not unique. The ID appears to be generated from `[Retailer initial][00][Region initial][State abbrev][first 2 letters of city]`, which collides whenever two different values share those initials:

- `S00NNENE` → maps to the same retailer (Sports Direct) but two different states (NJ/Newark and NY/New York).
- `W00SARLI`, `W00SFLOR`, `W00STEHO` → each maps to two different retailers (Walmart and West Gear) in the same city.

Affects 623 sales rows (~6.5% of the dataset). Considered a 50/50 split between the colliding values but rejected it — there's no basis for the split ratio, and it would fabricate transaction-level attribution rather than honestly represent the ambiguity.

**Decision:** exclude ambiguous rows only from the specific analyses where the ambiguity applies:

- Product-level analysis: no exclusions needed.
- State-level analysis: exclude `999999999` and `S00NNENE`.
- Retailer-level analysis: exclude `999999999`, `W00SARLI`, `W00SFLOR`, `W00STEHO`.

### 3. PRICE_PER_UNIT — nulls and sentinel value

2 rows had a null PRICE_PER_UNIT (both Product 20). A broader check also found a `99999` placeholder mixed into otherwise normal prices ($7–$95) for the same product.

**Decision:** imputed both the nulls and the 99999 row using the median price for Product 20 (excluding the sentinel from the median calculation) = 45.

### 4. UNITS_SOLD — non-numeric sentinel

A systematic check for non-numeric values in numeric columns found `'***'` in UNITS_SOLD on 2 rows, both Product 20.

**Decision:** imputed using the median UNITS_SOLD for the matching retailer + product + sales method combination (more precise than a blanket product-wide median, since each retailer had a long enough order history to support it).

### 5. UNITS_SOLD — zero values, handled inconsistently on purpose

4 rows (2 retailers, Product 130) had `UNITS_SOLD == 0`. Investigated each retailer's full order history for the product to judge whether the zero was a real anomaly or a plausible data point:

- **F00SVIRI:** normally sold in the 300s–500s range for this product; the zeros were a sharp, short-lived dip surrounded by otherwise-consistent high volume — the shape of a real interruption. Imputed using the median of that retailer's other Outlet orders for this product.
- **S00MMIDE:** was already in a low, ramping-up sales period for this product when the zeros occurred (14 → 0 → 0 → 7 → 39 → 46, climbing into the hundreds by August) — the zeros fit plausibly within a slow-start pattern. Left as-is.

This is a deliberate inconsistency: two visually identical data points (both 0, same product) were treated differently because their surrounding context supported different conclusions about whether they represented real anomalies.

### 6. Minor fixes

- SALES_METHOD had a typo variant, `"Ootlet"`, corrected to `"Outlet"`.
- A full categorical value check (`.value_counts()` across all categorical columns in all three tables) found no other typos or inconsistencies.

## What We Found

### The four questions

| Question | Answer | Value |
|---|---|---|
| Top product category, 2021 | Men's Street Footwear | $22.7M |
| Top state, women's, 2021 | Maine | $2.18M |
| Top state, men's, 2021 | Delaware | $2.33M |
| Most units, 2021 | Foot Locker | 1,098,160 |
| Most units, 2020 | Amazon | 317,930 |

The state results are tight — the top three are within ~12% in both cuts — so the leader is not a decisive separation.

### Growth came from new locations, and hides a retention problem

Sales grew 4x, from $24.2M to $95.9M. By location: **retained (19)** $12.8M → $16.2M, **+26%**; **churned (6)** $11.4M → $0; **new (81)** $0 → $79.7M.

Retained locations grew well. The drag was churn — six locations worth nearly half of 2020 revenue stopped ordering. Comparing "existing retailer" dollars against the full 2020 baseline makes same-store performance look negative; separating churn reverses that.

The 81 new locations aren't 81 new customers: two chains were new (Foot Locker, Walmart), the rest were existing chains expanding.

### Revenue is concentrated in one account — and always has been

Foot Locker is **57% of 2021 dollars** with no 2020 presence. But Amazon was **72% of 2020**. The anchor account changed; the dependency didn't. Amazon is the only chain that shrank, $17.5M → $10.5M, while total sales quadrupled.

### Channel varies by geography, not product

The South is ~70% online, the Southeast ~52% in-store, the West ~53% outlet. All six product categories, by contrast, split within about three points of each other. Because each chain is effectively locked to one or two sales methods in this data, channel and retailer effects can't be separated.

### Tested and ruled out: cannibalization

Adding 81 locations plausibly eroded existing ones — but 12 of 19 retained locations grew, and the correlation between a location's sales change and new openings in its state is 0.15. The aggregate decline traces to a few large individual drops, not broad erosion.

## Analysis Datasets

The key collision in issue 2 means no single merged table is safe for every question — `S00NNENE` is unusable for state analysis but fine for retailer analysis, and the `W00xxxx` IDs are the reverse. Worse, because the duplicates live in the retailer table, any `sales`–`retailers` merge matches those transactions twice, inflating totals with no error raised.

Three purpose-built frames were built from a shared base. In each case the ambiguous *column* is dropped from the lookup before merging, which collapses the duplicate pairs into single rows and makes fan-out impossible rather than something to correct afterward.

| Dataframe | Rows | Excluded IDs | Notes |
|---|---|---|---|
| `df_product_and_sales` | 9,648 | none | `sales` + `products` only; not joining retailers sidesteps fan-out entirely |
| `df_state` | 9,591 (0.6%) | `999999999`, `S00NNENE` | Retailer name dropped; `W00xxxx` geography is unambiguous. No RETAILER column by design |
| `df_retailer` | 9,080 (5.9%) | `999999999`, `W00SARLI`, `W00SFLOR`, `W00STEHO` | Geography dropped; `S00NNENE` retailer name is consistent |

**Sensitivity check.** The 568 rows excluded from `df_retailer` belong only to Walmart and West Gear, so the exclusion removes volume from two specific competitors in an analysis that ranks competitors. Checked rather than assumed: the excluded 2021 rows hold 46,016 units against a first-to-second gap of 827,924, and the 2020 rows hold zero — the top-line finding is robust. Second and third place are separated by ~21,000 units, less than half the ambiguous volume, so that ordering is not claimed.

