# Yelp Data Warehouse — Team 5

A full end-to-end data warehousing project built on the **Yelp Academic Dataset**, completed as part of a BI & Data Warehousing course at Santa Clara University (Fall 2024). The project covers every stage of the data engineering lifecycle: raw data ingestion, JSON-to-CSV conversion, data quality validation, dimensional modeling, cloud loading into Snowflake, OLAP analysis, and sentiment analysis.

---

## Dataset Source

The data was downloaded from the official [Yelp Open Dataset](https://business.yelp.com/data/resources/open-dataset/). The full download is a 4.35 GB compressed TAR file (8.65 GB uncompressed) containing 5 JSON files and documentation.

**Dataset scale at download time:**
- 6,990,280 reviews
- 150,346 businesses
- 11 metropolitan areas
- 200,100 photos

The 5 JSON source files used in this project:

| File | Description |
|---|---|
| `yelp_academic_dataset_business.json` | Business listings with attributes, hours, location |
| `yelp_academic_dataset_review.json` | User reviews with star ratings and text |
| `yelp_academic_dataset_user.json` | Yelp user profiles and social graph |
| `yelp_academic_dataset_checkin.json` | Timestamped checkin events per business |
| `yelp_academic_dataset_tip.json` | Short tips left by users on businesses |

---

## Raw Data Format (as seen in source files)

**Business** — each record is a flat JSON with nested `attributes` and `hours` dicts:
```
{"business_id":"mWMc6_wTdE0EUBKIGXDVfA","name":"Perkiomen Valley Brewery","address":"101 Walnut St",
"state":"PA","stars":4.5,"review_count":13,"is_open":1,
"attributes":{"RestaurantsTakeOut":"True","BusinessParking":"{'garage':None,'street':None,'lot':True,'valet':False}"},
"hours":{"Wednesday":"14:0-22:0","Friday":"12:0-22:0","Saturday":"12:0-22:0"}}
```

**Review** — contains review text, star rating, and reaction counts:
```
{"review_id":"KU_O5udG6zpxOg-VcAEodg","user_id":"mh_-eMZ6K5RLWhZyISBhwA",
"business_id":"XQfwVwDr-v0ZS3_CbbE5Xw","stars":3.0,"useful":0,"funny":0,"cool":0,
"text":"If you decide to eat here, just be aware...","date":"2018-07-07 22:09:11"}
```

**User** — contains social graph data (friends list as comma-separated IDs), elite years, and compliment counts:
```
{"user_id":"hA5lMy-EnncsH4JoR-hFGQ","name":"Karen","review_count":79,
"yelping_since":"2007-01-05","elite":"","friends":"PBK4q9KEEBHhFvSXCUirIw, 3FWPpM7KU1gXeOM_ZbYMbA, ..."}
```

**Checkin** — all timestamps stored as a single comma-separated string per business:
```
{"business_id":"---kPU91CF4Lq2-WlRu9Lw",
"date":"2020-03-13 21:10:56, 2020-06-02 22:18:06, 2020-07-24 22:42:27, ..."}
```

**Tip** — short customer tips with a compliment count:
```
{"user_id":"AGNUgVwnZUey3gcPCJ76iw","business_id":"3uLgwr0qeCNMjKenHJwPGQ",
"text":"Avengers time with the ladies.","date":"2012-05-18","compliment_count":0}
```

---

## Project Architecture

```
Yelp JSON Files (8.65 GB)
        │
        ▼
[Part 1: ETL — Extract & Transform]   Google Colab + Python/Pandas
        │
        ├── JSON → CSV conversion (custom parser, handles nested attributes)
        ├── Row & column validation
        ├── Null handling & feature engineering
        ├── Fact/dimension modeling (star schema)
        ├── Merge: review × business × user
        ├── Tip & checkin aggregation + join
        ├── Date filtering (2020–2022 reviews only)
        └── Export: 7 Parquet files (~1.2M rows total)
                │
                ▼
[Part 2: Load]   Snowflake via SnowSQL CLI
        │
        ├── Warehouse: BIDW (XSMALL, auto-suspend 60s)
        ├── Database: YELP_DW
        ├── Schema: RAW
        ├── Stage: STG_REVIEW_FACT_FINAL
        ├── File format: FF_PARQUET
        ├── INFER_SCHEMA → CREATE TABLE REVIEW_FACT_FINAL
        └── COPY INTO → 1,204,378 rows loaded, 0 errors
                │
                ▼
[Part 3: Analytics]   Snowflake SQL (OLAP) + Python (Sentiment)
        │
        ├── 7 OLAP queries on REVIEW_FACT_FINAL
        └── VADER sentiment analysis on 100K-row sample
```

---

## Row Counts

| Table | Raw Rows | Distinct |
|---|---|---|
| business | 150,346 | 150,346 |
| review (full dataset) | 1,204,411 | 1,204,411 |
| user | 1,987,897 | 1,987,897 |
| checkin | 13,356,875 | 13,353,332 |
| tip | 908,915 | 908,848 |
| **review_fact_final (2020–2022, loaded to Snowflake)** | **1,204,378** | — |

---

## Data Model

### Fact Table: `review`
- **Grain:** one row per review event
- **Primary key:** `review_id`
- **Foreign keys:** `business_id`, `user_id`
- **Numeric measures:** `stars`, `useful`, `funny`, `cool`

### Dimension Tables
- `business` — PK: `business_id` (25 columns including parsed parking and hours)
- `user` — PK: `user_id` (includes compliment counts, elite history, friend count)
- `checkin` — FK: `business_id` (all timestamps stored as pipe-separated string per business)
- `tip` — FKs: `user_id`, `business_id`

The final loaded table `REVIEW_FACT_FINAL` is a **denormalized wide table** combining all of the above — the fact table merged with its dimensions, plus aggregated tip text and checkin timestamps as additional columns.

---

## Part 1: ETL — Extract and Transform

### Step 1: JSON to CSV Conversion

The raw Yelp files are JSONL format (one JSON object per line). A custom Python parser was written to flatten the nested structures:

- **Business:** The `attributes` dict was flattened to extract `RestaurantsTakeOut` and `BusinessParking`. The parking value itself was a stringified nested dict (e.g., `"{'garage': None, 'lot': True, 'valet': False}"`). A custom `parse_parking` function handled the mixed string/dict format and extracted five boolean parking columns: `garage`, `street`, `validated`, `lot`, `valet`. The `hours` dict was unpacked into 7 separate `hours_monday` … `hours_sunday` columns.
- **Review, User, Checkin, Tip:** Converted row-by-row using a streaming JSON reader to avoid loading multi-GB files into memory.

### Step 2: Validation

Two rounds of validation were run after conversion:

**Validation 1 — Row count check:** Compared JSONL record count against CSV row count to confirm no rows were dropped during conversion.

**Validation 2 — Column schema check:** Verified that each CSV had the exact expected column headers against a predefined schema, with configurable `max_errors` error reporting.

### Step 3: Data Loading and Date Filtering

The review table was filtered to **2020–2022 only** before further processing:
```python
filtered_review = review[(review['date'] >= '2020-01-01') & (review['date'] <= '2022-12-31')]
```
This scoped the review dataset to ~1.2M rows for the rest of the pipeline.

All five CSVs were then loaded into Pandas DataFrames.

### Step 4: Dataset Profiling

`.info()` and `.describe()` were run across all five tables to understand dtypes, nulls, and numeric distributions. The `describe()` was focused on user engagement metrics: `review_count`, `useful`, `funny`, `cool`, `fans`, `average_stars`, and all 12 `compliment_*` columns.

### Step 5: Data Quality — Null Handling

**User table nulls:**
- `name`: filled with `"anonymous"`
- `elite`: filled with `False`
- `friends`: the raw comma-separated friend ID string was too wide to store and analyze. It was replaced with a derived `count_of_friends` column (`.str.count('|') + 1`), and the original `friends` column was dropped.

**Business table nulls:**
- Boolean columns (`restaurants_takeout`, `parking_garage`, `parking_street`, `parking_validated`, `parking_lot`, `parking_valet`): cast to boolean then mapped to `"True"` / `"False"` / `"Unknown"` for nulls.
- Hour columns (`hours_monday` through `hours_sunday`): nulls left as-is, treated as `"Closed"` in downstream analytics.

**Tip table nulls:**
- Rows with null `text` were dropped entirely.

### Step 6: Building the Fact Table

The fact table was built by joining three sources:
```python
review_with_business = review.merge(business, on='business_id', how='inner')
review_fact = review_with_business.merge(user, on='user_id', how='inner')
```
After joining, `business`, `user`, and intermediate DataFrames were deleted and `gc.collect()` was called to free RAM — the joined table was large enough to cause memory pressure in Colab.

### Step 7: Column Renaming and Reordering

Ambiguous joined columns were renamed for clarity:

| Original | Renamed |
|---|---|
| `stars_x` | `review_star` |
| `user_id` | `reviewer_id` |
| `date` | `review_date` |
| `useful_x` | `is_review_useful` |
| `funny_x` | `is_review_funny` |
| `cool_x` | `is_review_cool` |
| `name_x` | `business_name` |
| `stars_y` | `avg_stars_for_business` |
| `review_count_x` | `no_of_reviews_for_business` |

Columns were reordered into logical groups: identifiers → review-level fields → business attributes → reviewer attributes.

### Step 8: Elite Member Flag

The `elite` column in the user data was a comma-separated string of years the reviewer held Yelp Elite status (e.g., `"2019,2020,2021"`). This was cleaned (whitespace stripped, a `"20,20"` → `"2020"` typo corrected via regex) and converted into a binary flag `is_reviewer_elite_member` indicating whether the reviewer was elite **in the year they wrote the review**.

---

## Part 1: Exploratory Data Analysis (EDA)

All EDA was performed on the merged `review_fact` table in Python/Pandas.

### 1. Distribution of Open vs. Closed Businesses
A pie chart showing the split of reviews belonging to currently open vs. closed businesses. The majority of reviews in the 2020–2022 window are for businesses that are still open.

### 2. Elite vs. Non-Elite Reviewers
A donut chart showing the share of reviews written by Yelp Elite members vs. regular users. Elite members are a minority of users but account for a disproportionately large share of reviews.

### 3. Review Star Distribution
Reviews skew heavily bimodal: 5-star reviews are by far the most common, followed by 1-star reviews. Middle ratings (2–3 stars) are the least common — a pattern typical of review platforms where people are most motivated to write when they feel strongly. The breakdown was further split by elite vs. non-elite reviewer status to compare rating behavior across the two groups.

### 4. Geographic Analysis — States and Cities with Above-Average Reviews
Filtered to businesses with above-average review counts AND individual reviews rated higher than the business's own average. The top-performing state was identified and used to drill down to a city-level breakdown within that state.

### 5. Weekend Business Hours vs. Star Ratings
Businesses were classified as "Closed on Weekends" or "Open on Weekends" using the `hours_saturday` and `hours_sunday` columns. Average ratings and review counts were compared across both groups.

### 6. Review Volume vs. Perfect Score Credibility
A business insight comparison to illustrate the difference between surface-level and credible quality signals:
- Businesses with ≤ 5 reviews and a perfect 5.0 average: achievable with almost no data, not meaningful
- Businesses with ≥ 1,000 reviews and ≥ 4.5 average: far harder to sustain, a much more credible quality signal

### 7. Parking Availability vs. Average Star Rating
All five parking types (garage, street, lot, valet, validated) were normalized to boolean and collapsed into a single `has_parking` flag. A boxplot comparison showed:
- **Without parking:** mean ≈ 3.54, median = 3.5, wide spread with more low-end ratings
- **With parking:** mean ≈ 3.90, median = 4.0, tighter distribution, fewer bad ratings
- **Interpretation:** Parking doesn't just raise the average — it reduces bad experiences. The mean improvement is ~0.36 stars.

### 8. Takeout Availability vs. Average Star Rating
The same analysis was repeated for `restaurants_takeout`:
- **Without takeout:** mean ≈ 3.70, median = 4.0
- **With takeout:** mean ≈ 3.85, median = 4.0
- **Difference:** ~0.15 stars

**Combined comparison:**

| Feature | Mean Difference |
|---|---|
| Parking | ~0.36 stars |
| Takeout | ~0.15 stars |

Parking is a stronger differentiator. Takeout is a secondary convenience signal. Together they likely reinforce each other.

### 9. Top 20 Most Popular Businesses
Ranked by total review count after deduplicating on `business_id`, surfacing the highest-engagement businesses in the 2020–2022 window.

---

## Part 2: Final Dataset Assembly and Export

### Tip Aggregation
The tip dataset was cleaned (newlines stripped, empty text dropped) and grouped by `(business_id, user_id)` with multiple tips per pair concatenated using ` | ` as a separator into a single `tip_text` string.

### Checkin Aggregation
Checkin timestamps were grouped by `business_id` with all timestamps concatenated using ` | ` into a single `checkin_datetime` string per business.

### Final Join
The review fact table was extended with:
1. Aggregated tip data joined on `(business_id, reviewer_id)` — left join
2. Aggregated checkin data joined on `business_id` — left join

This produced `review_fact_final`: a single denormalized wide table containing review data, business attributes, reviewer attributes, any tips the reviewer left on that business, and all checkin timestamps for that business.

### Export to Parquet
The ~1.2M row final table was exported in **7 chunked Parquet files** of 200,000 rows each to Google Drive:

| File | Size |
|---|---|
| review_fact_final_0000.parquet | ~1.00 GB |
| review_fact_final_0001.parquet | ~1.46 GB |
| review_fact_final_0002.parquet | ~0.92 GB |
| review_fact_final_0003.parquet | ~1.02 GB |
| review_fact_final_0004.parquet | ~1.31 GB |
| review_fact_final_0005.parquet | ~1.11 GB |
| review_fact_final_0006.parquet | ~22 MB (final partial chunk) |

---

## Part 3: Data Warehouse — Snowflake

### Why Snowflake
1. Strong SQL and ELT support matching the project's transformation flow
2. Handles both structured and semi-structured data natively
3. Low maintenance overhead — no infrastructure to manage
4. SnowSQL CLI enabled running SQL and loading files directly from the command line

### Snowflake Setup
```sql
CREATE WAREHOUSE IF NOT EXISTS BIDW WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
CREATE DATABASE IF NOT EXISTS YELP_DW;
CREATE SCHEMA IF NOT EXISTS YELP_DW.RAW;
CREATE OR REPLACE STAGE STG_REVIEW_FACT_FINAL;
CREATE OR REPLACE FILE FORMAT FF_PARQUET TYPE = PARQUET;
```

### Loading the Data
The 7 Parquet files were staged and loaded using SnowSQL:
```sql
PUT file:////Users/.../bidw_parquet_files/*.parquet @STG_REVIEW_FACT_FINAL AUTO_COMPRESS=TRUE;

CREATE OR REPLACE TABLE REVIEW_FACT_FINAL
  USING TEMPLATE (SELECT ARRAY_AGG(OBJECT_CONSTRUCT(*)) FROM TABLE(
    INFER_SCHEMA(LOCATION => '@STG_REVIEW_FACT_FINAL', FILE_FORMAT => 'FF_PARQUET')));

COPY INTO REVIEW_FACT_FINAL FROM @STG_REVIEW_FACT_FINAL
  FILE_FORMAT = (FORMAT_NAME = 'FF_PARQUET')
  MATCH_BY_COLUMN_NAME = CASE_INSENSITIVE;
```
**Result:** 1,204,378 rows loaded, 0 errors across all 7 files.

---

## OLAP Queries

All 7 queries run on `REVIEW_FACT_FINAL` in Snowflake. A normalizing view was created first:
```sql
CREATE OR REPLACE VIEW VW_REVIEW_FACT_FINAL AS
SELECT *, TRY_TO_TIMESTAMP_NTZ(TO_VARCHAR("review_date"))::DATE AS review_dt
FROM "REVIEW_FACT_FINAL";
```

---

### Query 1: Businesses with Significant Recent Rating Drops

**Business question:** Which high-traffic businesses have seen their ratings decline meaningfully in the last 3 months compared to the 3 months before that?

**What it does:** Computes a monthly average rating per business, compares the last 3 months vs. the prior 3 months, filters to businesses with a minimum number of reviews and check-ins (to ensure signal strength), and returns those where the rating dropped by 0.3+ stars.

**Sample output:**

| Business | City | State | Checkins | Reviews (3Y) | Stars Last 3M | Stars Prev 3M | Drop |
|---|---|---|---|---|---|---|---|
| Royal House | New Orleans | LA | 28,927 | 443 | 3.32 | 4.12 | -0.80 |
| Puckett's Grocery & Restaurant | Nashville | TN | 4,691 | 345 | 2.97 | 3.64 | -0.67 |
| Mambo's | New Orleans | LA | 1,979 | 1,008 | 3.83 | 4.18 | -0.31 |

**Insight:** Royal House has the highest check-in volume in the list, making its rating drop especially high-visibility. Mambo's has the highest review volume, giving its drop the most statistical credibility. These are businesses already earning traffic that are at risk of losing customers if the trend continues.

---

### Query 2: Top Businesses Outperforming Their City Average

**Business question:** Which businesses are standout quality leaders relative to their local competition — not just nationally — and have the volume to back it up?

**What it does:** Calculates each business's average rating, review count, unique reviewer count, and check-in count. Computes a weighted average star rating per city as a local baseline. Measures how far above baseline each business sits (`STARS_VS_CITY`). Filters to businesses with high reviews and check-ins, then ranks by check-in popularity within their city.

**Sample output:**

| Business | City | State | Avg Stars | Reviews | Unique Reviewers | Checkins | City Avg | Stars vs City | Demand Rank |
|---|---|---|---|---|---|---|---|---|---|
| Arario Midtown | Reno | NV | 4.79 | 240 | 233 | 870 | 3.91 | +0.89 | 14 |
| Mazzaro's Italian Market | Saint Petersburg | FL | 4.65 | 303 | 299 | 2,624 | 3.98 | +0.67 | 2 |
| Reading Terminal Market | Philadelphia | PA | 4.71 | 501 | 499 | 18,615 | 4.08 | +0.63 | 2 |

**Insight:** Every business shown has 200+ reviews and 500+ check-ins — high-confidence winners, not small-sample outliers. Arario Midtown (Reno) is the single largest outperformer relative to its city baseline at +0.89 stars above average.

---

### Query 3: Businesses with the Highest Rate of Off-Hours Check-ins

**Business question:** Which businesses are receiving check-ins predominantly outside their listed operating hours — a signal of either stale hours data or genuinely unusual traffic patterns?

**What it does:** Parses each business's full check-in timestamp list and posted operating hours per day of the week. Labels each check-in as inside or outside posted hours (treating missing or "Closed" entries as outside). Summarizes total check-ins, off-hours check-ins, off-hours rate, peak off-hours day, and peak off-hours hour per business.

**Sample output:**

| Business | City | State | Total Checkins | Outside Checkins | Outside Rate | Peak Day | Peak Hour |
|---|---|---|---|---|---|---|---|
| Philadelphia International Airport | Philadelphia | PA | 52,144 | 52,144 | 1.00 | Saturday | 21 |
| Café Du Monde | New Orleans | LA | 40,109 | 40,109 | 1.00 | Saturday | 18 |
| Louis Armstrong New Orleans Airport | Kenner | LA | 37,562 | 37,562 | 1.00 | Saturday | 21 |
| Tampa International Airport | Tampa | FL | 37,518 | 37,518 | 1.00 | Saturday | 20 |
| Nashville International Airport | Nashville | TN | 31,168 | 31,168 | 1.00 | Saturday | 22 |

**Insight:** All top-20 results have a 100% off-hours rate. Airports and major transit hubs dominate because they operate 24/7 but their Yelp hours listings are not configured for round-the-clock operation. These are the strongest candidates for a data-quality review or a special business-type handling rule.

---

### Query 4: Lowest-Rated Business Categories by City

**Business question:** Which city + category combinations are consistently underperforming on Yelp?

**What it does:** Explodes the comma-separated `business_categories` column using `LATERAL SPLIT_TO_TABLE` so each review is counted under every category the business belongs to. Aggregates total businesses, total reviews, and weighted average stars at the `city + state + category` level. Returns results sorted ascending by weighted average rating.

**Sample output (bottom of the rating distribution):**

| City | State | Category | Businesses | Reviews | Weighted Avg Stars |
|---|---|---|---|---|---|
| Tucson | AZ | Real Estate | 261 | 1,268 | 2.20 |
| Nashville | TN | Fast Food | 234 | 1,981 | 2.43 |
| Tampa | FL | Car Dealers | 90 | 1,593 | 2.51 |
| Reno | NV | Hotels | 70 | 2,577 | 2.58 |
| Tampa | FL | Fast Food | 293 | 3,173 | 2.64 |

**Insight:** Real estate, fast food, car dealers, and hotels consistently appear at the bottom across cities — categories where customer expectations tend to be hardest to meet and complaint-driven reviews are most common.

---

### Query 5: Elite Member Review Concentration by City and Category

**Business question:** In which city + category combinations do elite Yelp members write a disproportionately high share of reviews — and are those categories rated higher as a result?

**What it does:** Explodes categories per review, flags elite vs. non-elite reviewers, and computes total reviews, elite review count, elite share, and average star rating at the `city + state + category` level, sorted by highest elite share.

**Sample output (highest elite share):**

| City | State | Category | Total Reviews | Elite Reviews | Elite Share | Avg Stars |
|---|---|---|---|---|---|---|
| Indianapolis | IN | Arts & Entertainment | 2,659 | 1,353 | 50.9% | 4.23 |
| Indianapolis | IN | Local Flavor | 1,037 | 499 | 48.1% | 4.47 |
| Indianapolis | IN | Breweries | 1,842 | 874 | 47.4% | 4.26 |
| Indianapolis | IN | Gastropubs | 1,254 | 573 | 45.7% | 4.23 |
| New Orleans | LA | Gift Shops | 1,324 | 539 | 40.7% | 4.13 |

**Insight:** Indianapolis dominates the top of this list across arts, breweries, gastropubs, and specialty food. These categories also carry higher average ratings, suggesting elite reviewers either gravitate toward higher-quality establishments or their engaged reviewing style tends to reflect better experiences.

---

### Query 6: Elite vs. Non-Elite Rating Gap by Category

**Business question:** Across all categories, where is the gap between how elite and non-elite reviewers rate businesses the widest?

**What it does:** Explodes categories, splits all reviews into elite and non-elite groups, and calculates the average star rating each group gives per category. Computes `ELITE_MINUS_NON_ELITE` as the gap, sorted descending.

**Sample output (largest gap):**

| Category | Total Reviews | Elite Reviews | Non-Elite Reviews | Avg Stars Elite | Avg Stars Non-Elite | Gap |
|---|---|---|---|---|---|---|
| Hospitals | 2,247 | 214 | 2,033 | 4.00 | 2.12 | +1.88 |
| Airlines | 788 | 189 | 599 | 3.77 | 1.99 | +1.77 |
| Banks & Credit Unions | 1,726 | 120 | 1,606 | 3.48 | 1.98 | +1.51 |
| Parking | 1,102 | 194 | 908 | 3.87 | 2.38 | +1.49 |
| Internet Service Providers | 1,574 | 107 | 1,467 | 3.22 | 1.76 | +1.46 |
| Fast Food | 48,532 | 7,701 | 40,831 | 3.70 | 2.44 | +1.26 |

**Insight:** The largest gaps are concentrated in high-friction, low-expectation service categories — hospitals, airlines, ISPs, banks. Non-elite users give these categories very low ratings (often venting after bad experiences), while elite reviewers rate them more moderately. This likely reflects that elite members are more calibrated reviewers who reserve 1-star ratings for truly exceptional failures rather than general frustration.

---

### Query 7: Check-ins Per Review Ratio by City and Category

**Business question:** Which city + category combinations have the highest ratio of foot traffic (check-ins) to reviews — places people visit often but rarely write about?

**What it does:** Explodes categories, aggregates total check-ins and reviews per `city + state + category + business`, sums to category level, and computes `CHECKINS_PER_REVIEW` sorted descending.

**Sample output (highest ratio):**

| City | State | Category | Total Checkins | Total Reviews | Checkins Per Review |
|---|---|---|---|---|---|
| Saint Louis | MO | Airports | 30,686 | 105 | 292.25 |
| Philadelphia | PA | Airports | 55,857 | 196 | 284.98 |
| Philadelphia | PA | Train Stations | 16,891 | 62 | 272.44 |
| Indianapolis | IN | Airports | 21,615 | 90 | 240.17 |
| Saint Louis | MO | Stadiums & Arenas | 11,612 | 51 | 227.69 |
| Tampa | FL | Shopping Centers | 50,638 | 276 | 183.47 |

**Insight:** Airports and transit hubs generate enormous foot traffic but very few reviews per visit. Shopping centers and stadiums show similar patterns. This ratio identifies underreviewed high-volume venues where targeted review prompts could significantly increase Yelp's coverage density in those categories.

---

## Sentiment Analysis

VADER (Valence Aware Dictionary and sEntiment Reasoner) was applied to a random 100,000-row sample of `review_fact_final` to compute a sentiment score from raw review text, independently of the numeric star rating.

```python
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
analyzer = SentimentIntensityAnalyzer()

sample_df['sentiment_score'] = sample_df['review_text'].apply(
    lambda t: analyzer.polarity_scores(t)['compound'])
sample_df['sentiment_star_equiv'] = (sample_df['sentiment_score'] + 1) / 2 * 4 + 1
```

- `sentiment_score`: ranges from -1.0 (most negative) to +1.0 (most positive)
- `sentiment_star_equiv`: rescaled to the 1–5 range for direct comparison with `review_star`

**Why this matters:** The numeric star rating and the text tone don't always agree. Some reviews carry a 3-star rating but contain strongly negative language. Others carry a 1-star rating but use relatively neutral or constructive language. The sentiment score provides a second, independent satisfaction signal that can be used to flag reviews where text and stars diverge, or to build a richer quality model for businesses beyond the star average alone.

**Sample output:**

| Review Text (excerpt) | Star | Sentiment Score |
|---|---|---|
| "This small local place was decent but not overly impressive..." | 3.0 | -0.86 |
| "Negative stars!! This is one crooked business!! BEWARE!!!!" | 1.0 | +0.96 |
| "Happy they added a Riverview location. Been there 5 times since they opened." | 5.0 | +0.91 |
| "The food was not at all special... a 2+ star experience at best" | 2.0 | -0.41 |
| "Nonfat yogurt option is very bitter. Their regular ice cream is very good." | 3.0 | +0.03 |

---

## Tech Stack

| Layer | Tool |
|---|---|
| Development environment | Google Colab |
| Data processing | Python, Pandas, NumPy |
| Visualization | Matplotlib, Seaborn |
| File format (export) | Parquet |
| Cloud data warehouse | Snowflake |
| SQL interface | SnowSQL CLI |
| Sentiment analysis | VADER (`vaderSentiment`) |
| Intermediate storage | Google Drive |

---

## Repository Structure

```
yelp-dw-project/
├── notebooks/
│   └── DW_TEAM_5.ipynb          # Full project notebook (Google Colab)
├── docs/
│   └── data_dictionary.md       # Column-level definitions for all tables
├── data/
│   ├── raw/                     # Source JSON files (not committed — 8.65 GB)
│   └── processed/               # Cleaned CSVs and Parquet outputs (not committed)
├── src/                         # Extracted Python scripts (optional refactor)
├── sql/                         # Snowflake DDL and OLAP query files
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Setup and Reproduction

### 1. Clone the repo
```bash
git clone https://github.com/<your-org>/yelp-dw-project.git
cd yelp-dw-project
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
```

### 3. Download the dataset
Get the Yelp Academic Dataset from https://business.yelp.com/data/resources/open-dataset/ and place the 5 JSON files under `data/raw/`.

### 4. Run the notebook
Open `notebooks/DW_TEAM_5.ipynb` in Google Colab. Update the Google Drive base path variable at the top of the notebook to point to your own Drive folder where CSVs and Parquet files will be written.

### 5. Load to Snowflake
After the Parquet files are written to Drive, connect with SnowSQL and run the DDL and COPY commands from the notebook's Snowflake section (or from `sql/` once those scripts are extracted).

```bash
snowsql -a <account_identifier> -u <username>
```

> **Note:** Raw JSON files, all intermediate CSVs, and Parquet outputs are excluded from version control via `.gitignore`. Total uncompressed data is ~8.65 GB.

---

## Team

Team 5 — SCU BI & Data Warehousing Course, Fall 2024
