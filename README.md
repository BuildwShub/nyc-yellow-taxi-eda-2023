<div align="center">

# 🚕 NYC Yellow Taxi — Optimising Operations Through EDA

### Exploratory Data Analysis of 2023 NYC TLC Yellow Taxi Trip Records

![Python](https://img.shields.io/badge/Python-3.x-blue?style=for-the-badge&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-blue?style=for-the-badge&logo=pandas&logoColor=white)
![Matplotlib](https://img.shields.io/badge/Matplotlib-blue?style=for-the-badge&logo=plotly&logoColor=white)
![Seaborn](https://img.shields.io/badge/Seaborn-blue?style=for-the-badge)
![GeoPandas](https://img.shields.io/badge/GeoPandas-green?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=for-the-badge)

</div>

| 🚖 Sampled Trips | 💰 Annual Revenue | 🗺️ Taxi Zones | 📈 Est. Revenue Uplift |
|:---:|:---:|:---:|:---:|
| **2,042,850** | **$58.2M** | **263** | **18–24%** (+$20.4M – $28.1M) |

---

## 📑 Table of Contents

1. [Project Overview](#-project-overview)
2. [Business Problem](#-business-problem)
3. [Dataset](#-dataset)
4. [Methodology](#-methodology)
5. [Key Findings](#-key-findings)
6. [Strategic Recommendations](#-strategic-recommendations)
7. [Revenue Impact Summary](#-revenue-impact-summary)
8. [Tech Stack](#-tech-stack)
9. [Project Structure](#-project-structure)
10. [How to Run](#-how-to-run)
11. [Author](#-author)

---

## 📌 Project Overview

This project applies Exploratory Data Analysis to **2,042,850 sampled NYC Yellow Taxi trips from 2023** to uncover patterns in demand, pricing, and geography across all 263 TLC taxi zones. The goal is to provide a data-driven strategy for a new taxi operator entering the NYC market — covering **when to deploy, where to deploy, and how to price**. The analysis identifies **$20.4M – $28.1M in potential annual revenue uplift** through routing optimisation, strategic zone positioning, and dynamic pricing adjustments. Insights are grounded in temporal, financial, and geographic patterns surfaced through the full EDA pipeline.

---

## ❗ Business Problem

Three core operational inefficiencies were identified in the 2023 data:

### 1. ⏰ Temporal Mismatch
**Hour 17 (5 PM)** is the slowest hour at **15.84 min/mile** — yet sits in the highest-revenue window. Congestion at peak demand reduces the number of trips a fleet can complete per hour, leaving demand uncaptured precisely when prices are highest.

### 2. 📍 Geographic Mismatch
The **top 5 pickup zones** (132, 79, 249, 48, 230) account for **22.3% of all pickups**, but fleets do not pre-position ahead of demand peaks. Drivers chase demand reactively instead of staging ahead of it.

### 3. 🌙 Night Supply Gap
**Hour 23 (11 PM)** is the highest-demand hour AND highest revenue/mile hour at **$23.14/mile**, yet night revenue is only **18.9%** of the annual total. This represents an estimated **~$11M opportunity gap** from undersupplied night coverage.

---

## 📊 Dataset

**Source:** NYC Taxi & Limousine Commission (TLC) — 2023 Yellow Taxi Trip Records
**Format:** 12 monthly Parquet files (Jan – Dec 2023)
**Raw scale:** ~38 million trip records across the year (~3 GB on disk)
**Working sample:** **1,896,400 rows** — 5% stratified sample per date-hour group, combined into a single Parquet file

### 🔍 Why sampling was necessary

The full 2023 TLC release is **12 separate monthly Parquet files totalling ~38 million rows** — too large to load into memory on Google Colab's free tier (≈12 GB RAM) without crashing the kernel during groupby aggregations, joins to the 263-zone shapefile, and choropleth rendering.

Rather than analyse one month at a time (which would distort yearly seasonality) or use a flat random 5% across the whole year (which would under-represent low-volume hours like 3 AM Tuesdays), the project uses **stratified random sampling per date-hour bucket** (`frac=0.05, random_state=42`). For every (date, hour) combination across all 12 months, exactly 5% of trips are drawn. This:

- preserves the original **temporal distribution** (peak hours stay peak hours, quiet hours stay quiet)
- keeps Q1/Q2/Q3/Q4 seasonality intact
- yields a reproducible sample (`random_state=42`)
- shrinks 38M rows → 1.9M rows — small enough to fit in RAM, large enough that hourly/zonal aggregates remain statistically stable

### 📦 Per-file sample sizes (5% stratified)

| Month | Sampled rows | Month | Sampled rows |
|---|---:|---|---:|
| 2023-01 | 152,087 | 2023-07 | 174,068 |
| 2023-02 | 168,696 | 2023-08 | 143,782 |
| 2023-03 | 163,786 | 2023-09 | 140,875 |
| 2023-04 | 139,641 | 2023-10 | 174,255 |
| 2023-05 | 144,458 | 2023-11 | 165,133 |
| 2023-06 | 162,910 | 2023-12 | 166,709 |
| | | **Combined** | **1,896,400** |

The 12 sampled per-month dataframes are concatenated and saved as a single `nyc_taxi_sampled.parquet` (22 columns), which becomes the input for all downstream cleaning and EDA — so every cell after Section 1 reads from one file instead of twelve.

### Key columns

| Column | Type | Description |
|---|---|---|
| `tpep_pickup_datetime` | Numerical | Meter engaged timestamp |
| `tpep_dropoff_datetime` | Numerical | Meter disengaged timestamp |
| `passenger_count` | Numerical | Driver-entered passenger count (1–6) |
| `trip_distance` | Numerical | Miles reported by taximeter |
| `PULocationID` | Categorical | TLC pickup zone (1–263) |
| `DOLocationID` | Categorical | TLC dropoff zone (1–263) |
| `VendorID` | Categorical | TPEP provider (1 = Creative Mobile, 2 = VeriFone) |
| `RateCodeID` | Categorical | Rate code (1 = Standard, 2 = JFK, 3 = Newark, …) |
| `payment_type` | Categorical | 1 = Credit Card, 2 = Cash, 3 = No Charge, 4 = Dispute |
| `fare_amount` | Numerical | Time-and-distance fare ($) |
| `improvement_surcharge` | Numerical | $0.30 TLC surcharge — dollar amount, not a label |
| `congestion_surcharge` | Numerical | NYS congestion charge ($2.75 where applicable) |
| `Airport_fee` | Numerical | $1.25 for JFK / LaGuardia pickups |
| `tip_amount` | Numerical | Auto-populated for credit card tips |
| `total_amount` | Numerical | Total charged (excl. cash tips) |
| `store_and_fwd_flag` | Categorical | Y = store-and-forward trip, N = not |

---

## 🔬 Methodology

```
1️⃣ Data Preparation  →  2️⃣ Data Cleaning  →  3️⃣ General EDA
                                                       ↓
                  5️⃣ Conclusions  ←  4️⃣ Detailed EDA
```

1. **Data Preparation** — Stratified sampling (5% per date-hour) across 12 monthly files, combined into a single Parquet file for efficient downstream analysis.
2. **Data Cleaning** — Column case fixes, `Airport_fee` / `airport_fee` merge, negative monetary value removal, missing-value imputation (mode for categoricals; 0 for inapplicable surcharges), outlier handling (`passenger_count > 6` dropped, `trip_distance > 250 mi` dropped, `fare_amount > 500` capped).
3. **General EDA** — Temporal analysis (hourly / daily / monthly), financial analysis (revenue distribution, fare composition), geographical choropleth mapping across all 263 zones.
4. **Detailed EDA** — Route speed analysis, weekday vs. weekend traffic, night zone demand, fare-per-mile by vendor / hour / day, tip behaviour, surcharge penetration.
5. **Conclusions & Recommendations** — Routing, zone positioning, and pricing strategy synthesised into quantified opportunity bands.

---

## 🔑 Key Findings

### 🕐 Temporal Findings

| Metric | Value | Insight |
|---|---|---|
| Peak demand hour | **11 PM (Hour 23)** | 219,269 sampled trips — 9.6% of daily volume |
| Fastest travel hour | **6 AM (Hour 6)** | 9.12 min/mile — enables 6–7 trips/hour |
| Slowest travel hour | **5 PM (Hour 17)** | 15.84 min/mile — peak congestion |
| Highest revenue/mile hour | **11 PM (Hour 23)** | $23.14/mile |
| Busiest day | **Friday** | Highest raw trip volume |
| Highest revenue/mile day | **Tuesday** | $17.70/mile |
| Peak quarter | **Q2 (Apr – Jun)** | 27.5% of annual trips |
| Lowest quarter | **Q1 (Jan – Mar)** | 23.2% — winter suppresses demand |

### 💰 Financial Findings

| Metric | Value |
|---|---|
| Annual revenue (sampled) | **$58.2M** |
| Average revenue per trip | $28.50 |
| Distance-fare correlation | **r = 0.9451** (very strong) |
| Credit card share | 78.85% of trips |
| Night revenue share | 18.9% (vs. 11 PM peak demand) |
| Fare per mile — solo rider | $16.60/mile |
| Fare per mile — 6 passengers | $2.20/mile |

### 📍 Geographic Findings

| Metric | Value |
|---|---|
| Top pickup zone | **Zone 132 (JFK Airport)** — 107,298 trips |
| Top 10 zones combined | 45.2% of all annual trips |
| Bottom 50 zones each | < 500 annual pickups |
| Demand concentration | Midtown Manhattan, Upper East / West Side, Airports |

---

## 🎯 Strategic Recommendations

### 4.1 Routing & Dispatching Optimisation

| # | Strategy | Action | Est. Annual Impact |
|---|---|---|---|
| 1 | 🌙 **Peak Night Deployment** (11 PM – 3 AM) | +30% fleet to Zones 132, 79, 249, 48, 230 from 10:30 PM | **+$2.0M – $3.0M** |
| 2 | 🌅 **Morning Commute Activation** (6 – 9 AM) | Incentives ($0.75/trip) + 10% fare discount. Target Zones 161, 236, 138 | **+$0.9M – $1.2M** |
| 3 | 🚦 **Evening Rush Routing** (4 – 7 PM) | Pre-position to Zones 161, 237, 236, 138 by 3:30 PM. Avoid toll zones 59, 187, 199 | **+$1.8M – $2.2M** |
| 4 | 📅 **Weekday/Weekend Calibration** | +3% fleet Tuesdays. −10% weekend fleet → redirect to airport zones 2 & 3 | **+$0.5M – $0.8M** |

### 4.2 Strategic Zone Positioning

#### Three-tier deployment system

| Tier | Zones | Strategy | Rationale |
|---|---|---|---|
| **Tier 1 — Always On** | 132, 79, 237, 161, 249 | 20% fleet at all times, 24/7 | 22.3% of all annual trips |
| **Tier 2 — Time-Based** | 161, 236, 138 (AM) / 48, 230 (Night) / 2, 3 (Weekend) | Deploy on demand schedule | Pre-positioning captures peaks |
| **Tier 3 — On-Demand Only** | Outer boroughs, bottom 50 zones | App dispatch only | < 500 pickups/year each |

#### Seasonal calendar

| Quarter | Strategy |
|---|---|
| **Q1 (Jan – Mar)** | Tier 1 only. Pull back Tier 2. −10% airport weekend |
| **Q2 (Apr – Jun)** | Full Tier 2. +20% airport zones for tourism. Peak quarter (27.5%) |
| **Q3 (Jul – Sep)** | Full Tier 1 + Tier 2. Extra night coverage for outdoor season |
| **Q4 (Oct – Dec)** | +25% airport Nov – Dec. Midtown event coverage for New Year |

### 4.3 Data-Driven Pricing Strategy

| # | Strategy | Detail | Est. Annual Impact |
|---|---|---|---|
| 1 | 📏 **Distance-Tiered Fares** | 0–2 mi: +5% / 5–10 mi: +8% / 10+ mi: +12% | **+$6.0M – $8.0M** |
| 2 | ⚡ **Dynamic Surge Pricing** | 11 PM – 1 AM: +20% / 4 – 7 PM: +15% / 6 – 9 AM: −10% / 2 – 5 AM: +25% | **+$3.0M – $4.0M** |
| 3 | 📆 **Day-of-Week Multipliers** | Tue: +2% / Fri: +1.5% / Sat: −2% / Sun: −3% | **+$1.5M – $2.0M** |
| 4 | 💲 **Surcharge Optimisation** | Improvement: $1.00 → $1.25 / Congestion: $2.75 → $3.00 / Airport: $1.25 → $1.75 | **+$1.1M – $1.5M** |
| 5 | 👥 **Group Pricing** | 3–4 pax: +5% / 5–6 pax: +10% (corrects 7.5× underpricing vs. solo) | **+$0.5M – $0.8M** |
| 6 | 💳 **Card Payment Incentive** | −2% fare discount for credit card. Net tip uplift ~ +$600K | **+$0.6M** |

> **Competitive guardrail:** all adjustments keep Yellow Taxi within **10% of competitor rates**. Post-adjustment premium estimated at 5–7% — safely below the ~10% switching threshold.

---

## 🔵 Revenue Impact Summary

| Strategy Area | Annual Revenue Impact |
|---|---|
| 4.1 Routing & Dispatching Optimisation | +$5.2M – $7.2M |
| 4.2 Strategic Zone Positioning | +$2.5M – $4.0M |
| 4.3 Pricing Strategy Adjustments | +$12.7M – $16.9M |
| **TOTAL ESTIMATED ANNUAL IMPACT** | **+$20.4M – $28.1M** |

> Conservative 20–40% realisation rate applied. Assumes no major TLC regulatory changes and continued 2023-level demand patterns.

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| **Python 3.x** | Core language |
| **Pandas** | Data manipulation and aggregation |
| **NumPy** | Numerical operations |
| **Matplotlib** | Static visualisations |
| **Seaborn** | Statistical plots and heatmaps |
| **GeoPandas** | Choropleth map of 263 NYC taxi zones |
| **Google Colab** | Cloud execution environment |
| **Parquet (PyArrow)** | Efficient large-file storage and loading |

---

## 📁 Project Structure

```
nyc-yellow-taxi-eda-2023/
│
├── EDA_NYC_Taxi_Analysis_Shubham.ipynb           ← Main analysis notebook
├── README.md                                      ← This file
│
├── Data/                                          ← Not included (TLC source)
│   ├── 2023-01.parquet … 2023-12.parquet         ← Raw monthly files
│   ├── nyc_taxi_sampled.parquet                  ← 5% stratified sample
│   └── nyc_taxi_cleaned.parquet                  ← Final analysis-ready file
│
└── taxi_zones/                                    ← Shapefile for geo analysis
    ├── taxi_zones.shp
    ├── taxi_zones.dbf
    ├── taxi_zones.shx
    └── taxi_zones.prj
```

---

## ▶️ How to Run

<details>
<summary><b>Step-by-step (click to expand)</b></summary>

**1. Clone the repository:**
```bash
git clone https://github.com/buildwshub/nyc-yellow-taxi-eda-2023.git
```

**2. Download the dataset from NYC TLC:**
👉 https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page

Download all 12 monthly Parquet files for 2023.

**3. Upload to Google Drive under:**
```
MyDrive/NYC_Taxi_EDA Assignment/Data Files/
```

**4. Upload the `taxi_zones` shapefile folder to:**
```
MyDrive/NYC_Taxi_EDA Assignment/taxi_zones/
```

**5. Open the notebook in Google Colab:**
`File → Open Notebook → GitHub → paste repo URL`

**6. Mount Google Drive when prompted, then `Runtime → Run all`.**

</details>

---

## 👤 Author

**Shubham Kanauji**
*Data Analyst | EDA & Business Intelligence*

[![LinkedIn](https://img.shields.io/badge/LinkedIn-buildwshub-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/buildwshub/)
[![GitHub](https://img.shields.io/badge/GitHub-buildwshub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/buildwshub)

**Dataset:** NYC TLC 2023 Yellow Taxi Trip Records (Public Domain)

---

<div align="center">

⭐ *If you found this analysis useful, consider starring the repository.*

</div>
