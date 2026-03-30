# BT5153 Group Project

## Dataset
- Amazon Electronics reviews: 20,994,353 reviews, 786,868 products
- Source: `Electronics.json.gz` (JSON Lines format, gzip compressed)
- Metadata not used — `asin` is sufficient as product identifier for analysis

## Goals
- Analyze top complaints per product
- Analyze sentiment distribution
- Extract “Top 3 reasons for 1-star reviews” per product
- Compare good vs bad products

## Data Cleaning Pipeline

### Step 1 — Stream and filter during load (`amazon_project.ipynb`)
Read `Electronics.json.gz` in chunks of 50,000 rows, writing directly to `electronics_cleaned.csv` to avoid loading 3GB+ into memory. Progress is saved to `progress.txt` so the job can resume if interrupted.

Fields kept: `asin`, `overall`, `reviewText`, `summary`, `verified`
Fields dropped: `reviewTime`, `unixReviewTime`, `reviewerID`, `reviewerName`, `style`, `vote`

Rows dropped during load:
- Missing `reviewText`
- `reviewText` with fewer than 20 words

Result: ~12.3M reviews retained (down from 22M)

### Step 2 — Filter by review count (`amazon_project.ipynb`)
Distribution of reviews per product (721,250 products total):
- Median: 3 reviews
- 75th percentile: 10 reviews
- Mean: 26, std: 163 — heavily right-skewed

**Threshold: keep only products with >= 200 reviews** (~11,000 products, top ~2% of all products). Products below this threshold lack sufficient signal for complaint extraction and sentiment analysis.

### Step 3 — Cap popular products (`amazon_project.ipynb`)
3,442 products have more than 500 reviews. To prevent popular products from dominating the dataset, randomly sample max 500 reviews per product (seed=42 for reproducibility).

**Output: `electronics_filtered.csv`** (~11K products, ~3.94M reviews total)

### Step 4 — Data sharing
File is too large for GitHub (100MB limit). Share via Google Drive. Teammates can mount in Colab:
```python
from google.colab import drive
drive.mount('/content/drive')
```

## Loading the filtered data
```python
import polars as pl
df = pl.scan_csv('electronics_filtered.csv', schema_overrides={'asin': pl.Utf8})
# filter as needed, e.g.:
negative = df.filter(pl.col('overall') <= 2).collect()
```
