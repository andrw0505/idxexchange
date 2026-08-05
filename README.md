# CRMLS Listing & Sold Data Pipeline

Cleaning, validation, and feature engineering pipeline for California Regional MLS (CRMLS) residential real estate data, enriched with FRED mortgage-rate data. Built during an internship at IDX Exchange.

The pipeline turns ~1.5M raw MLS rows across two feeds (active listings and closed sales) into two analysis-ready datasets with market metrics, quality flags, and a macro rate join.

---

## What this does

| Stage | Operation | Why it matters |
|---|---|---|
| 1. Ingest | Concatenate monthly `CRMLSListing{YYYYMM}.csv` and `CRMLSSold{YYYYMM}.csv` files (2024–2026) into unified frames | MLS exports arrive one file per month; analysis needs a single panel |
| 2. Scope filter | Keep `PropertyType == 'Residential'` | Raw feed mixes 8 property types (land, commercial lease, business opportunity, etc.) that aren't comparable |
| 3. Completeness prune | Drop columns with ≥90% nulls | 15 of 86 columns in the sold feed are effectively empty and only add noise |
| 4. Macro enrichment | Pull 30-year fixed mortgage rate (`MORTGAGE30US`) from FRED, resample weekly→monthly mean, left-join on `year_month` | Lets you condition price and time-on-market behavior on the rate environment |
| 5. Logical validation | Flag rows where `ListingContractDate > CloseDate` or `PurchaseContractDate > CloseDate` | Impossible timelines indicate data-entry errors, not real transactions |
| 6. Geographic validation | Coerce lat/long to numeric; require lat ∈ [32.5, 42.0] and long ∈ [−124.5, −114.0] | California bounding box — catches null islands, swapped coordinates, and out-of-state records |
| 7. Feature engineering | `price_ratio`, `ppsf`, `listing_duration`, `close_duration`, `yrmo` | Derived market metrics, described below |
| 8. Outlier flagging | Tukey IQR fences (Q1 − 1.5·IQR, Q3 + 1.5·IQR) on `ClosePrice`, `LivingArea`, `DaysOnMarket` | Raw max values include a $989.5M close price and a 17M sqft "living area" |

### Derived metrics

| Field | Formula | Interpretation |
|---|---|---|
| `price_ratio` | `ClosePrice / OriginalListPrice` | >1 = sold above ask (competitive market); <1 = price concession |
| `ppsf` | `ClosePrice / LivingArea` | Size-normalized price; the standard unit for cross-market comparison |
| `listing_duration` | `PurchaseContractDate − ListingContractDate` | Time to find a buyer (demand-side signal) |
| `close_duration` | `CloseDate − PurchaseContractDate` | Escrow/financing time (credit-conditions signal) |
| `yrmo` | `YYYY * 100 + MM` | Integer period key for time-series grouping |

Splitting total time-to-close into `listing_duration` and `close_duration` separates two different economic forces: how fast buyers show up, versus how fast financing clears.

---

## Data attrition

Every filter is logged with before/after row counts. Headline numbers from the current run:

**Sold feed**
```
Raw                              612,097 rows × 86 cols
→ Residential only              411,523   (−200,574)
→ ≥90%-null columns dropped              (−15 cols → 71)
→ Timeline + coordinate valid   395,266   (−16,257)
→ Positive values + IQR fences  ~338,000  (−56,858)
```

**Listing feed**
```
Raw                              887,636 rows × 86 cols
→ Residential only              563,856   (−323,780)
→ Timeline + coordinate valid   482,953   (−80,903)
```

The listing feed loses proportionally far more rows to coordinate validation (14% vs. 4%). Worth noting: active listings are entered by agents in real time, while sold records have been through a closing process that forces correction.

---

## Repository contents

```
idx_deliverable.ipynb            Main pipeline: ingest → clean → enrich → validate → feature-engineer
idx_deliverable_listings.ipynb   Weeks 1–2 exploratory pass on the listing feed only
.gitignore                       Excludes *.csv and *.pdf
```

### Data is not in this repository

CRMLS data is licensed and cannot be redistributed, so all CSVs are gitignored. To run the notebooks you need the monthly exports placed in the working directory:

```
CRMLSListing202401.csv, CRMLSListing202402.csv, ...
CRMLSSold202401.csv,    CRMLSSold202402.csv,    ...
```

Missing months are skipped silently (`FileNotFoundError` is caught), so a partial dataset will run without error but produce a shorter panel.

### Outputs written

| File | Stage |
|---|---|
| `cleaned_listings.csv` / `cleaned_sold.csv` | After residential filter + column prune |
| `enriched_listing.csv` / `enriched_sold.csv` | After FRED mortgage-rate join |
| `enriched_listing_v3.csv` / `enriched_sold_v3.csv` | After timeline + coordinate validation |
| `completed_listing.csv` / `completed_sold.csv` | Final: features + outliers removed |

---

## Requirements

```bash
pip install pandas numpy matplotlib
```

Python 3.9+. The FRED pull is a live HTTP request to `fred.stlouisfed.org` — no API key required, but the notebook needs network access at that cell.

---

## How to run

1. Place the monthly CRMLS CSVs in the same directory as the notebook.
2. Open `idx_deliverable.ipynb` and run cells top to bottom.

Cells are order-dependent and pass state in memory through a versioned chain (`enriched_sold_v3` → `v4` → `v5` → `v6` → `v7`). Running out of order will raise `NameError` or silently operate on a stale frame.

---

## Known limitations

- **Listing-side price metrics are sparse.** `price_ratio` and `ppsf` both depend on `ClosePrice`, which is null for the majority of active listings (280,180 of 887,636 raw rows have a `CloseDate`). These fields are only meaningful for the subset of listing records that have already closed.
- **The IQR outlier rule is symmetric and unconditional.** A single fence is applied across the whole state. A $3M Palo Alto sale and a $3M Bakersfield sale are treated identically, so the filter removes legitimate high-end transactions in expensive submarkets. Grouping the fences by `CountyOrParish` or `MLSAreaMajor` would be more defensible.
- **`DaysOnMarket` contains negative values** (min = −288) that are removed by the `> 0` filter rather than investigated. The negative sign likely reflects a listing re-entry or backdating convention in the source system.
- **The mortgage-rate join is a monthly average.** A loan locked mid-month at a different rate than the monthly mean will be mismeasured. This is an acceptable approximation for trend analysis, not for loan-level attribution.
- **No deduplication on `ListingKey`.** A property that is listed, withdrawn, and relisted may appear multiple times across monthly files.
