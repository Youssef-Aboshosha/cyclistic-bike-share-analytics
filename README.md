# Cyclistic Bike-Share Analytics: Data Warehouse Design & Behavioral Insights

![Data Architecture](https://img.shields.io/badge/Data_Architecture-Star_Schema-blue?style=for-the-badge)
![Database Engine](https://img.shields.io/badge/Database_Engine-Oracle_SQL-red?style=for-the-badge)
![Analytics](https://img.shields.io/badge/Analytics-Analytics_Engineering-green?style=for-the-badge)

## 📌 Project Executive Summary
This repository contains the end-to-end Analytics Engineering implementation for the **Cyclistic Bike-Share Case Study**. The primary objective of this project is to ingest raw, unstructured operational bike trip logs and transform them into an enterprise-grade **Star Schema Data Warehouse (DW)**. 

By separating descriptive, high-cardinality attributes into clean dimension tables and centralizing numeric metrics within a specialized fact table, this architecture transitions the data platform from fragmented tabular parsing into a high-performance system optimized for sub-second business intelligence querying, dimensional slicing, and advanced behavioral analysis.

### 📐 The Mathematical Data Grain
* **The Atomic Grain Definition:** `One single row in the fact table = One single completed bike ride / trip transaction instance.`
* Retaining data at this lowest atomic level guarantees **zero column or metric loss** during the ETL/ELT pipeline execution. This provides downstream analysts with absolute flexibility to build custom temporal filters, spatial aggregates, and complex behavioral rollups without destroying historical lineage.

---

## 🎯 Strategic Business & Marketing Questions Addressed
This dimensional model is custom-engineered from the ground up to systematically solve 12 critical strategic business, marketing, and revenue-driving inquiries:

### 👥 User Segmentation & Behavior Analytics
1. **Rider Value & Duration Benchmarking:** Do annual members take longer rides than casual riders based on the historical trailing year window?
2. **Meteorological Behavioral Fluctuations:** Does the average trip duration and usage behavior differ between members and casual riders across different meteorological seasons?
3. **Temporal Demand Concentration:** During which specific days of the week or hourly blocks of the day is casual rider activity most heavily concentrated?
4. **Subscription Conversion Potential:** What is the exact conversion potential for casual riders to become annual subscribers based on the percentage of casual users who exhibit traditional member-like commuting patterns on weekdays?
5. **Commute Window Pricing Profiling:** What specific corporate subscription tier pricing parameters should be designed based on the peak usage hours and ride lengths of annual members during weekday corporate commute windows?

### 🗺️ Geospatial & Asset Fleet Analytics
6. **Route Corridor Discovery:** What are the top 10 most popular physical routes (explicit Start Station Key to End Station Key) heavily utilized by riders?
7. **Asset Demand Fluctuations:** Which specific type of bike asset (classic vs. electric) achieves peak popularity during the specific Summer season?
8. **Premium Advertising Placement:** Which specific station corridors should be targeted for premium physical advertising placements based on having the highest volume of high-value casual rider transactions?
9. **Low-Traffic Demand Stimulation:** Which low-traffic or peripheral station locations should receive targeted promotional pricing discounts or marketing campaigns to stimulate casual rider demand during off-peak weekdays?

### ⚙️ Operational Risk & Revenue Driving Analytics
10. **Revenue Leakage Tracking:** What is the exact financial revenue leakage caused by casual riders keeping classic bikes past the standard single-pass time limits during peak weekend periods?
11. **Asset Return on Investment (ROI):** Which bike asset categories (classic vs. electric) yield the highest return on investment (ROI) in terms of total rental revenue generated per maintenance dollar spent?
12. **Fleet Imbalance Logistics Impact:** What is the systemic business impact on fleet availability and user satisfaction when high-demand electric bikes are disproportionately left at low-traffic peripheral stations on weekends?

---

## 🏗️ Data Warehouse Architecture (Star Schema)
To deliver optimal query execution plans and eliminate structural redundancy, the pipeline adheres to a strict dimensional design engineered inside **Oracle SQL Developer Data Modeler** and validated via **dbdiagram.io**.

### 🌟 Dimensional Schema Configuration Code (DBML)
The relational system structure is declared using the following standardized Database Markup Language (DBML) properties, isolating discrete dimensions from the central fact hub:

```dbml
// --- FACT TABLE ---
Table FACT_RIDE {
  ride_sk INTEGER [pk, increment]
  ride_id VARCHAR2(50)
  start_date_fk INTEGER
  start_time_fk INTEGER
  start_station_fk INTEGER
  end_station_fk INTEGER
  rider_profile_fk INTEGER
  bike_fk INTEGER
  ride_length_min NUMBER
  started_at DATE
  ended_at DATE
  start_lat NUMBER
  start_lng NUMBER
  end_lat NUMBER
  end_lng NUMBER
}

// --- DIMENSION TABLES ---
Table DIM_STATION {
  station_sk INTEGER [pk, increment]
  station_id VARCHAR2(100)
  station_name VARCHAR2(100)
}

Table DIM_DATE {
  date_sk INTEGER [pk]
  Date DATE
  day_of_week VARCHAR2(10)
  month VARCHAR2(20)
  is_weekend CHAR(1)
  season VARCHAR2(10)
}

Table DIM_TIME {
  time_sk INTEGER [pk]
  hour INTEGER
  time_period VARCHAR2(20)
}

Table DIM_BIKE {
  bike_sk INTEGER [pk, increment]
  rideable_type VARCHAR2(50)
}

Table DIM_RIDER_PROFILE {
  rider_profile_sk INTEGER [pk, increment]
  member_casual VARCHAR2(30)
}

// --- RELATIONSHIPS ---
Ref: FACT_RIDE.start_station_fk > DIM_STATION.station_sk
Ref: FACT_RIDE.end_station_fk > DIM_STATION.station_sk
Ref: FACT_RIDE.bike_fk > DIM_BIKE.bike_sk
Ref: FACT_RIDE.rider_profile_fk > DIM_RIDER_PROFILE.rider_profile_sk
Ref: FACT_RIDE.start_time_fk > DIM_TIME.time_sk
Ref: FACT_RIDE.start_date_fk > DIM_DATE.date_sk

🗂️ Architectural Highlights & Design Justifications
The Date & Time Split Optimization: DIM_DATE and DIM_TIME are strictly separated into two distinct lookup tables. Merging them into a single continuous datetime dimension would cause a massive Row Explosion (e.g., exploding the table to 8,760 rows per year for hourly grain, or over 525,000 rows for minute grain). Splitting them maintains lightweight, highly cached dimension tables (365 rows for a 1-year date lookup and 24 rows for an hourly clock lookup), guaranteeing lightning-fast filtering.

The Role-Playing Dimension Pattern: A single physical table (DIM_STATION) serves a dual architectural role. It maps into the central fact table via two separate foreign key channels: start_station_fk and end_station_fk. This eliminates table duplication, saves storage, and keeps metadata unified.

Overcoming the Low-Cardinality Dimension Trap: Spatial telemetry coordinate metrics (start_lat, start_lng, end_lat, end_lng) are retained directly within FACT_RIDE rather than being isolated into an external geography dimension. Because GPS values shift continuously across a decimal spectrum, a separate geography table would grow at the exact same rate as the fact table itself. Keeping them in the fact table ensures fast, vector-based geospatial mapping calculations.

🔗 Explicit Referential Integrity & Constraints Matrix
To guarantee total database consistency and prevent orphaned transactions, referential integrity is strictly enforced through explicit physical database keys, eliminating all messy automated column artifacts:

Fact_Sales_Dim_Bike_FK binds FACT_RIDE(bike_fk) directly to DIM_BIKE(bike_sk)

Fact_Sales_Dim_Rider_Profile_FK binds FACT_RIDE(rider_profile_fk) directly to DIM_RIDER_PROFILE(rider_profile_sk)

Fact_Sales_Dim_Time_FK binds FACT_RIDE(start_time_fk) directly to DIM_TIME(time_sk)

Fact_Sales_Dim_Date_FK binds FACT_RIDE(start_date_fk) directly to DIM_DATE(date_sk)

Fact_Sales_Dim_Station_FK [Role: Start Hub] binds FACT_RIDE(start_station_fk) directly to DIM_STATION(station_sk)

Fact_Sales_Dim_Station_FKv1 [Role: End Hub] binds FACT_RIDE(end_station_fk) directly to DIM_STATION(station_sk)

🚀 Repository Directory Roadmap
The analytical engineering process is logically sequenced into specialized, self-documenting Jupyter Notebook phases:

📂 Notebooks/01_data_preparation.ipynb: Manages the loading of massive raw data files, schema data type auditing, and initial ingestion configurations.

📂 Notebooks/02_data_cleaning.ipynb: Executes rigorous data hygiene standards, filters out physical anomalies (negative durations, test rides), handles station lookup gaps safely, and calibrates the core ride_length_min measure.

📂 Notebooks/03_feature_engineering.ipynb: Extracts rich temporal and calendar hierarchies from raw timestamps to construct optimized keys for the target dimensional tables.

📂 Notebooks/04_exploratory_analysis.ipynb: Houses the formal execution of business intelligence queries, aggregate matrices, and primary metric discovery calculations to drive strategic ROI insights.

📂 reports/: Houses executive summarization decks, presentation slides, and formal architecture compilation specifications.


