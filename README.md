# Amazon Product Dataset — Data Cleaning & Exploratory Analysis

## Overview

This project cleans and explores a dataset of **1,465 Amazon.in product listings** spanning pricing, ratings, reviews, and product metadata across categories like Electronics, Computers & Accessories, and Home & Kitchen. The raw export came directly from Amazon's site and was not analysis-ready — prices were stored as currency strings, ratings contained a non-numeric error value, and review counts had missing entries. This document covers the cleaning process, what the data reveals once usable, and recommendations for the business based on those findings.

## Dataset

| | |
|---|---|
| **Rows** | 1,465 |
| **Columns** | 16 |
| **Source** | Amazon.in product listings (scraped export) |
| **Key fields** | `product_id`, `product_name`, `category`, `discounted_price`, `actual_price`, `discount_percentage`, `rating`, `rating_count`, `about_product`, review fields |

## Data Cleaning

### 1. Price columns (`discounted_price`, `actual_price`)
Both columns were stored as text with currency formatting, e.g. `"₹1,099"`. These were split on character position to strip the `₹` symbol and thousands separator, then cast to `float` so they could be used in calculations and comparisons.

### 2. Rating data type
`rating` was stored as text and converted to `float` to enable aggregation and statistical analysis.

### 3. Rating data error
One row contained a malformed rating value (`|`) instead of a number. The affected `product_id` was looked up directly on Amazon.in, and the value was corrected to **3.9** based on the live listing — rather than dropping the row or imputing a generic average, which would have weakened the dataset's accuracy on a real, identifiable product.

### 4. Missing values (`rating_count`)
Two rows had a missing `rating_count`. These were imputed with `0`, since the records lacked a numeric value entirely (cleaner, on inspection, are simply unreviewed listings rather than data entry errors).

### 5. Duplicate check
A full-row duplicate check returned **zero duplicates**. However, this is worth a caveat for downstream analysis: **114 distinct `product_id` values appear more than once** (206 rows total), each instance carrying a different review/reviewer. Treating every row as a unique product without grouping by `product_id` would overweight heavily-reviewed products in any per-product analysis (see Findings).

## Findings

With the data cleaned, a few patterns stand out:

- **Discounting is aggressive and standard practice.** The average discount across the catalog is **~48%**, with the median actual price at ₹1,650 dropping to a median discounted price of ₹799 — implying most listings are priced roughly half off from their "actual" price as a baseline strategy, not an occasional promotion.
- **Deeper discounts correlate with slightly lower ratings, not higher.** The correlation between `discount_percentage` and `rating` is **−0.15** — a weak but negative relationship. Heavily discounted items are not, on average, the best-reviewed ones, which cuts against the assumption that bigger discounts drive better customer satisfaction.
- **Category quality varies meaningfully.** Among the three dominant categories (Electronics, Computers & Accessories, Home & Kitchen — together ~98% of listings), **Computers & Accessories** has the highest average rating (4.15), followed by Electronics (4.08) and Home & Kitchen (4.04). Niche categories like Office Products (4.31 avg, though only 31 listings) outperform on rating but have far less catalog depth.
- **Review volume is extremely concentrated.** Amazon Basics HDMI cables and boAt earphones dominate `rating_count`, with the top listings exceeding **400,000 ratings** — multiple orders of magnitude above the dataset average of ~18,270. A small number of high-volume staple products are carrying most of the catalog's social proof.
- **Repeated `product_id`s skew naive per-row analysis.** Because 114 products are duplicated across rows (each tied to a different review), any aggregate statistic computed at the row level — including the ones above — is implicitly weighted toward products with more reviews in this extract. This should be corrected (see Recommendations).

## Recommendations

1. **De-duplicate to the product level before drawing business conclusions.** Aggregate review-level rows up to one row per `product_id` (e.g., taking review count as a sum and rating as a weighted average) before computing category- or catalog-wide statistics. As-is, popular products are overrepresented in any row-level average.
2. **Investigate the discount–rating relationship rather than assume deep discounts help.** Since heavier discounts don't correlate with better ratings, the business should test whether steep markdowns are being used to move lower-performing inventory — and validate that "actual price" baselines used to calculate the discount are accurate and not inflated to make the markdown look larger than it is.
3. **Double-check `actual_price` integrity given outlier discounts.** Discounts as high as 90%+ on accessory items are worth a manual audit; if "actual price" is not a genuine historical or MSRP price, the displayed discount percentage may be a misleading rather than a real promotional signal.
4. **Invest further in growing review volume for mid-tier products.** With review counts this concentrated in a handful of bestsellers, most listings are likely under-trusted by new buyers. Campaigns encouraging reviews on lower-`rating_count` (but well-rated) products could improve conversion without needing further discounting.
5. **Use category-level rating differences to prioritize catalog quality control.** Home & Kitchen, despite having a similar listing count to Electronics, has the lowest average rating among the three major categories — a candidate area for a deeper quality or returns-reason audit.
6. **Standardize data capture at the source.** The `₹` formatting, comma-separated thousands, percentage strings, and the malformed rating value all point to inconsistent export/scraping logic. Fixing this upstream (or adding validation rules) would reduce the need for this kind of manual cleanup on future extracts.

## Files

- `amazon_dataset.csv` — original raw dataset
- `amazon_data_cleaning.csv` — cleaned dataset with corrected data types and values
