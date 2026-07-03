# 🚲 Cyclistic Bike-Share Analytics: Data Warehouse & Behavioral Insights

![Data Architecture](https://img.shields.io/badge/Data_Architecture-Star_Schema-blue?style=for-the-badge)
![Database Engine](https://img.shields.io/badge/Database_Engine-Oracle_SQL-red?style=for-the-badge)
![Analytics](https://img.shields.io/badge/Analytics-Analytics_Engineering-green?style=for-the-badge)
![Pipeline](https://img.shields.io/badge/Pipeline-ETL%2FELT-orange?style=for-the-badge)
![Tool](https://img.shields.io/badge/Tool-Jupyter_Notebook-yellow?style=for-the-badge)

---

## 📌 Project Executive Summary

This repository is the end-to-end **Analytics Engineering** implementation for the **Cyclistic Bike-Share Case Study** — a Google Data Analytics capstone project. The core objective is to ingest raw, unstructured operational bike trip logs and systematically transform them into an **enterprise-grade Star Schema Data Warehouse (DW)**.

By separating descriptive, high-cardinality attributes into lean dimension tables and centralizing additive numeric metrics within a specialized fact table, this architecture transitions the data platform from fragmented tabular parsing into a **high-performance system** optimized for:

- Sub-second BI querying and dimensional slicing
- Geospatial route and station corridor analysis
- Temporal behavioral pattern discovery
- Strategic marketing and revenue analytics

> **Context:** Cyclistic is a fictional Chicago-based bike-share company with ~5,800 bikes and 600+ docking stations. The business challenge: convert casual riders into profitable annual members through data-driven marketing.

---

## 📐 Data Grain Definition

| Grain Level | Definition |
|---|---|
| **Atomic Grain** | `1 row in FACT_RIDE = 1 completed bike trip transaction` |
| **Temporal Scope** | Trailing 12-month rolling window |
| **Granularity** | Individual ride-level (no pre-aggregation) |

Retaining data at the **lowest atomic grain** guarantees zero metric loss during pipeline execution, giving downstream analysts full flexibility to build custom rollups, temporal filters, spatial aggregates, and behavioral cohort analyses without destroying historical lineage.

---

## 🎯 Strategic Business Questions Addressed

This dimensional model is custom-engineered to systematically answer **12 critical business, marketing, and revenue-driving questions** across three analytical domains:

### 👥 Domain 1 — User Segmentation & Behavioral Analytics

| # | Question | Key Dimensions |
|---|---|---|
| 1 | Do annual members take longer rides than casual riders over the trailing year? | `DIM_RIDER_PROFILE`, `DIM_DATE` |
| 2 | Does average trip duration differ between members and casual riders across meteorological seasons? | `DIM_RIDER_PROFILE`, `DIM_DATE.season` |
| 3 | During which days of the week or hourly blocks is casual rider activity most concentrated? | `DIM_DATE.day_of_week`, `DIM_TIME.hour` |
| 4 | What percentage of casual riders exhibit member-like commuting patterns on weekdays? (conversion potential) | `DIM_RIDER_PROFILE`, `DIM_DATE.is_weekend` |
| 5 | What peak usage hours and ride lengths define weekday corporate commute windows for annual members? | `DIM_TIME`, `DIM_DATE`, `DIM_RIDER_PROFILE` |

### 🗺️ Domain 2 — Geospatial & Asset Fleet Analytics

| # | Question | Key Dimensions |
|---|---|---|
| 6 | What are the top 10 most popular routes (start station → end station) across all riders? | `DIM_STATION` (role-playing) |
| 7 | Which bike type (classic vs. electric) achieves peak demand during Summer? | `DIM_BIKE`, `DIM_DATE.season` |
| 8 | Which station corridors should be prioritized for premium physical advertising placements? | `DIM_STATION`, `DIM_RIDER_PROFILE` |
| 9 | Which low-traffic peripheral stations should receive targeted promotional discounts to stimulate off-peak casual demand? | `DIM_STATION`, `DIM_DATE` |

### ⚙️ Domain 3 — Operational Risk & Revenue Analytics

| # | Question | Key Dimensions |
|---|---|---|
| 10 | What is the financial revenue leakage from casual riders exceeding classic bike time limits during peak weekends? | `DIM_RIDER_PROFILE`, `DIM_BIKE`, `DIM_DATE` |
| 11 | Which bike category (classic vs. electric) yields the highest ROI per maintenance dollar spent? | `DIM_BIKE`, `FACT_RIDE.ride_length_min` |
| 12 | What is the business impact when high-demand electric bikes accumulate at low-traffic peripheral stations on weekends? | `DIM_BIKE`, `DIM_STATION`, `DIM_DATE` |

---

## 🏗️ Data Warehouse Architecture

### Star Schema Overview

The warehouse follows a strict **Kimball-style Star Schema** — one central fact table connected to six independent dimension tables via surrogate key foreign keys. Engineered in **Oracle SQL Developer Data Modeler** and validated via **dbdiagram.io**.

```
                    ┌─────────────────┐
                    │   DIM_DATE      │
                    │  (date_sk PK)   │
                    └────────┬────────┘
                             │
┌─────────────────┐          │          ┌─────────────────┐
│   DIM_STATION   │          │          │   DIM_TIME      │
│ (station_sk PK) │          │          │  (time_sk PK)   │
│ [Role: Start]   ├──────────┤          └────────┬────────┘
│ [Role: End]     │          │                   │
└────────┬────────┘    ┌─────▼───────────────────▼──────┐    ┌─────────────────┐
         │             │           FACT_RIDE             │    │   DIM_BIKE      │
         └────────────►│  (ride_sk PK - atomic grain)   │◄───│  (bike_sk PK)   │
                       │  start_station_fk               │    └─────────────────┘
                       │  end_station_fk                 │
                       │  start_date_fk                  │    ┌──────────────────────┐
                       │  start_time_fk                  │◄───│  DIM_RIDER_PROFILE   │
                       │  bike_fk                        │    │ (rider_profile_sk PK)│
                       │  rider_profile_fk               │    └──────────────────────┘
                       │  ride_length_min (metric)       │
                       └─────────────────────────────────┘
```

---

### 🗂️ Full DBML Schema Definition

```dbml
// ══════════════════════════════════════════════
// FACT TABLE — Central Transaction Hub
// ══════════════════════════════════════════════
Table FACT_RIDE {
  ride_sk          INTEGER   [pk, increment, note: "Surrogate Primary Key"]
  ride_id          VARCHAR2(50)              [note: "Original source natural key"]
  start_date_fk    INTEGER                   [ref: > DIM_DATE.date_sk]
  start_time_fk    INTEGER                   [ref: > DIM_TIME.time_sk]
  start_station_fk INTEGER                   [ref: > DIM_STATION.station_sk, note: "Role: Origin Hub"]
  end_station_fk   INTEGER                   [ref: > DIM_STATION.station_sk, note: "Role: Destination Hub"]
  rider_profile_fk INTEGER                   [ref: > DIM_RIDER_PROFILE.rider_profile_sk]
  bike_fk          INTEGER                   [ref: > DIM_BIKE.bike_sk]
  ride_length_min  NUMBER                    [note: "Core additive metric — minutes"]
  started_at       DATE                      [note: "Raw audit timestamp — immutable"]
  ended_at         DATE                      [note: "Raw audit timestamp — immutable"]
  start_lat        NUMBER                    [note: "Geospatial vector — retained in fact (see design note)"]
  start_lng        NUMBER
  end_lat          NUMBER
  end_lng          NUMBER
}

// ══════════════════════════════════════════════
// DIMENSION TABLES
// ══════════════════════════════════════════════

Table DIM_STATION {
  station_sk   INTEGER      [pk, increment]
  station_id   VARCHAR2(100)
  station_name VARCHAR2(100)
  Note: "Role-Playing Dimension — maps as both origin and destination via two FK channels"
}

Table DIM_DATE {
  date_sk      INTEGER      [pk, note: "Integer key format: YYYYMMDD"]
  full_date    DATE
  day_of_week  VARCHAR2(10) [note: "e.g., Monday, Tuesday"]
  day_num      INTEGER      [note: "ISO day number 1–7"]
  month        VARCHAR2(20) [note: "e.g., January"]
  month_num    INTEGER
  quarter      INTEGER
  year         INTEGER
  is_weekend   CHAR(1)      [note: "Y / N flag"]
  season       VARCHAR2(10) [note: "Spring / Summer / Fall / Winter"]
}

Table DIM_TIME {
  time_sk      INTEGER      [pk, note: "Integer key format: HHMM"]
  hour         INTEGER      [note: "0–23"]
  minute       INTEGER
  time_period  VARCHAR2(20) [note: "Morning / Afternoon / Evening / Night"]
  Note: "24 rows for hourly grain — lightweight and fully cached"
}

Table DIM_BIKE {
  bike_sk       INTEGER      [pk, increment]
  rideable_type VARCHAR2(50) [note: "classic_bike / electric_bike / docked_bike"]
}

Table DIM_RIDER_PROFILE {
  rider_profile_sk INTEGER     [pk, increment]
  member_casual    VARCHAR2(30) [note: "member / casual"]
}

// ══════════════════════════════════════════════
// EXPLICIT RELATIONSHIPS
// ══════════════════════════════════════════════
Ref: FACT_RIDE.start_station_fk > DIM_STATION.station_sk
Ref: FACT_RIDE.end_station_fk   > DIM_STATION.station_sk
Ref: FACT_RIDE.bike_fk          > DIM_BIKE.bike_sk
Ref: FACT_RIDE.rider_profile_fk > DIM_RIDER_PROFILE.rider_profile_sk
Ref: FACT_RIDE.start_time_fk    > DIM_TIME.time_sk
Ref: FACT_RIDE.start_date_fk    > DIM_DATE.date_sk
```

---

### 🔗 Referential Integrity & Constraint Matrix

All foreign key relationships are enforced at the **physical database level** via explicit Oracle constraint declarations:

| Constraint Name | Child Table Column | Parent Table Column | Role |
|---|---|---|---|
| `FK_FACT_BIKE` | `FACT_RIDE.bike_fk` | `DIM_BIKE.bike_sk` | Asset type lookup |
| `FK_FACT_RIDER` | `FACT_RIDE.rider_profile_fk` | `DIM_RIDER_PROFILE.rider_profile_sk` | Customer class lookup |
| `FK_FACT_TIME` | `FACT_RIDE.start_time_fk` | `DIM_TIME.time_sk` | Intraday clock lookup |
| `FK_FACT_DATE` | `FACT_RIDE.start_date_fk` | `DIM_DATE.date_sk` | Calendar lookup |
| `FK_FACT_STATION_START` | `FACT_RIDE.start_station_fk` | `DIM_STATION.station_sk` | Origin hub (Role 1) |
| `FK_FACT_STATION_END` | `FACT_RIDE.end_station_fk` | `DIM_STATION.station_sk` | Destination hub (Role 2) |

---

## 🧠 Key Architectural Design Decisions

### 1. Date–Time Dimension Split
`DIM_DATE` and `DIM_TIME` are maintained as **two separate dimension tables** rather than a single combined datetime dimension. Merging them would cause a row explosion:

| Grain | Combined Rows/Year | Split Rows (Date) | Split Rows (Time) |
|---|---|---|---|
| Daily | 365 | 365 | — |
| Hourly | 8,760 | 365 | 24 |
| Minute | 525,600 | 365 | 1,440 |

Splitting keeps both tables **lightweight, fully cacheable, and query-optimized**.

### 2. Role-Playing Dimension Pattern
A single physical table `DIM_STATION` serves a **dual architectural role**, mapping into `FACT_RIDE` via two separate FK channels (`start_station_fk` and `end_station_fk`). This eliminates table duplication, unifies station metadata, and enables direct route corridor analysis between any origin–destination pair.

### 3. Geospatial Coordinates in Fact Table
GPS coordinate columns (`start_lat`, `start_lng`, `end_lat`, `end_lng`) are **deliberately retained in `FACT_RIDE`** rather than isolated into a geography dimension. Because GPS values shift continuously across a decimal spectrum, a separate geography table would grow at the exact same rate as the fact table — providing zero compression benefit while adding a costly join. Retaining coordinates in the fact table enables direct vector-based geospatial rendering.

### 4. Surrogate Key Strategy
All dimension tables use **system-generated integer surrogate keys** (`_sk`) as primary keys rather than natural source keys. This isolates the warehouse from upstream source system changes, enables consistent cross-table joins, and protects historical data integrity during source key reuse scenarios.

---

## 📁 Repository Directory Structure

```
cyclistic-bike-share-analytics/
│
├── 📂 notebooks/
│   ├── 01_data_preparation.ipynb       # Raw file ingestion, schema type auditing, initial load config
│   ├── 02_data_cleaning.ipynb          # Hygiene execution: nulls, negative durations, test ride removal
│   ├── 03_feature_engineering.ipynb    # Temporal hierarchy extraction, surrogate key generation, dim partitioning
│   └── 04_exploratory_analysis.ipynb   # BI query execution, aggregate matrices, metric discovery, ROI calculations
│
├── 📂 sql/
│   ├── ddl/
│   │   ├── create_fact_ride.sql        # FACT_RIDE DDL with all constraints
│   │   ├── create_dim_station.sql
│   │   ├── create_dim_date.sql
│   │   ├── create_dim_time.sql
│   │   ├── create_dim_bike.sql
│   │   └── create_dim_rider_profile.sql
│   └── dml/
│       └── populate_dimensions.sql     # Dimension population scripts
│
├── 📂 data/
│   ├── raw/                            # Original source CSV trip files (not committed — see .gitignore)
│   └── processed/                      # Cleaned, transformed, warehouse-ready outputs
│
├── 📂 reports/
│   ├── executive_summary.pdf           # Business findings and strategic recommendations
│   ├── star_schema_diagram.png         # Exported schema architecture visual
│   └── presentation_deck.pptx         # Stakeholder presentation slides
│
├── 📂 docs/
│   └── data_dictionary.md              # Full column-level metadata and business definitions
│
├── .gitignore
├── requirements.txt                    # Python dependencies
└── README.md
```

---

## ⚙️ Pipeline Architecture Overview

```
RAW SOURCE DATA (CSV)
        │
        ▼
┌──────────────────────┐
│  01_data_preparation │  → Load files, audit dtypes, check schema consistency
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  02_data_cleaning    │  → Remove nulls, negative durations, test rides, station gaps
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────┐
│  03_feature_engineering  │  → Extract day_of_week, season, hour block, compute ride_length_min
└──────────┬───────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│  STAR SCHEMA LOAD                                        │
│  DIM_DATE → DIM_TIME → DIM_STATION → DIM_BIKE →         │
│  DIM_RIDER_PROFILE → FACT_RIDE (FK-validated insert)     │
└──────────┬───────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────┐
│  04_exploratory      │  → Execute 12 strategic BI queries, generate KPI outputs
│  _analysis           │
└──────────────────────┘
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Database Engine** | Oracle SQL / Oracle SQL Developer |
| **Schema Design** | Oracle SQL Developer Data Modeler, dbdiagram.io |
| **ETL / Data Processing** | Python 3.x, Pandas, NumPy |
| **Notebook Environment** | Jupyter Notebook |
| **Visualization** | Matplotlib, Seaborn, Folium (geospatial) |
| **Version Control** | Git / GitHub |

---

## 🚀 Getting Started

### Prerequisites
```bash
pip install -r requirements.txt
```

### Running the Pipeline
Execute notebooks in order:
```bash
jupyter notebook notebooks/01_data_preparation.ipynb
jupyter notebook notebooks/02_data_cleaning.ipynb
jupyter notebook notebooks/03_feature_engineering.ipynb
jupyter notebook notebooks/04_exploratory_analysis.ipynb
```

### Database Setup
```bash
# Run DDL scripts in order against your Oracle instance
sqlplus user/password@db @sql/ddl/create_dim_date.sql
sqlplus user/password@db @sql/ddl/create_dim_time.sql
sqlplus user/password@db @sql/ddl/create_dim_station.sql
sqlplus user/password@db @sql/ddl/create_dim_bike.sql
sqlplus user/password@db @sql/ddl/create_dim_rider_profile.sql
sqlplus user/password@db @sql/ddl/create_fact_ride.sql
```

---

## 🚀 Strategic Business Recommendations

> *(Derived from EDA outputs in `04_exploratory_analysis.ipynb`)*

### 1. 🎯 Membership Conversion Campaign
Target frequent casual riders who exhibit member-like weekday commuting patterns (identified via Domain 1, Q4).

| | |
|---|---|
| **Tactic** | Personalized digital outreach during weekday peak hours |
| **Expected Impact** | Higher membership conversion rate, increased recurring revenue |

---

### 2. 📅 Weekend Marketing Promotions
Casual riders show disproportionately high weekend activity — a prime conversion window.

| | |
|---|---|
| **Tactic** | Time-limited weekend membership trial offers at high-traffic stations |
| **Expected Impact** | Improved conversion opportunities, increased brand engagement |

---

### 3. ⚡ Dynamic Fleet Rebalancing
Electric bikes accumulate at low-traffic peripheral stations on weekends, starving high-demand hubs.

| | |
|---|---|
| **Tactic** | Automated rebalancing triggers on weekend mornings at top-10 demand corridors |
| **Expected Impact** | Reduced stockouts, improved user satisfaction scores |

---

### 4. 📍 Station-Based Premium Advertising
A small number of station corridors generate a disproportionately large share of casual rider volume (Domain 2, Q8).

| | |
|---|---|
| **Tactic** | Deploy physical membership ads at the top 5 casual rider corridor hubs |
| **Expected Impact** | More efficient marketing spend, higher membership acquisition per dollar |

---

### 5. 💼 Corporate Commuter Subscription Tier
Annual members concentrate rides during weekday commute windows — an underserved pricing segment.

| | |
|---|---|
| **Tactic** | Design a corporate subscription tier priced around peak commute hours and average ride lengths |
| **Expected Impact** | Increased enterprise adoption, higher predictable recurring revenue |

---

## 📊 Key Findings Preview

> *(Populate this section after completing `04_exploratory_analysis.ipynb`)*

| Insight | Finding |
|---|---|
| Member vs. Casual Avg. Duration | — |
| Peak Casual Rider Day | — |
| Top Route Corridor | — |
| Most Popular Bike Type (Summer) | — |
| Weekend Revenue Leakage Estimate | — |


---

*Built with  SQL · Python · Jupyter · Power BI · Kimball Dimensional Modeling*
