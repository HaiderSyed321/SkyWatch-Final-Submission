# Skywatch Analytics Engineer Take-Home Assessment
## Final Submission Summary

**Candidate**: Haider Syed
**Position**: Analytics Engineer - SkyWatch
**Date**: 27th May 2025

---

## Question 1: SQL query to find the number of new, retained, resurrected and churned users per month from January to May 2025
### a) SQL Query:

```sql
CREATE TABLE user_purchases (
    user_id INTEGER,
    purchase_date DATE,
    purchase_amount DECIMAL(10,2)
);

COPY user_purchases FROM 'user_purchases.csv' 
WITH (FORMAT CSV, HEADER TRUE);

WITH monthly_users AS (
    select 
        DATE_TRUNC('month', purchase_date) as month,
        user_id
    from user_purchases
    group by DATE_TRUNC('month', purchase_date), user_id
),
user_first_month as (
    select 
        user_id,
        MIN(month) as first_month
    from monthly_users
    group by user_id
),
monthly_segments as (
    select 
        m.month,
        m.user_id,
        f.first_month,
        LAG(m.month) OVER (PARTITION BY m.user_id ORDER BY m.month) as prev_active_month,
        CASE 
            WHEN m.month = f.first_month THEN 'new'
            WHEN LAG(m.month) OVER (PARTITION BY m.user_id ORDER BY m.month) = m.month - INTERVAL '1 month' THEN 'retained'
            ELSE 'resurrected'
        END as segment
    from monthly_users m
    join user_first_month f ON m.user_id = f.user_id
)
select 
    month,
    COUNT(CASE WHEN segment = 'new' THEN 1 END) as new_users,
    COUNT(CASE WHEN segment = 'retained' THEN 1 END) as retained_users,
    COUNT(CASE WHEN segment = 'resurrected' THEN 1 END) as resurrected_users
from monthly_segments
group by month
order by month;
```
### B) Results Table:

| Month      | New Users | Retained Users | Resurrected Users | Churned Users |
|------------|-----------|----------------|-------------------|---------------|
| 2025-01-01 | 130       | 0              | 0                 | 0             |
| 2025-02-01 | 36        | 78             | 0                 | 52            |
| 2025-03-01 | 20        | 68             | 23                | 46            |
| 2025-04-01 | 7         | 73             | 39                | 38            |
| 2025-05-01 | 2         | 68             | 37                | 51            |


---

## Question 2: Dimensional Data Model Proposal:

Our goal is to give business leaders instant answers to net-new, cohort retention, and churn/resurrection without writing SQL.

### The Core Problem

When answering questions like "How many users churned last quarter and returned this quarter?", we usually must perform nested CTEs. Analysts usually repeat that work, and business users can't self-serve those queries due to SQL complexity.

### My Proposed Solution

I'd build a dimensional model centered around three main fact tables, each optimized for different business questions:

#### Fact Tables

**1. `fact_user_activity_daily`** (user-day grain)
Pre-flags `is_new`, `is_retained`, `is_resurrected`. Row exists only if user was *active* that day, churn is derived by comparing to the previous day so we never have to generate a full user×date Cartesian product, keeping the table lean.

| Column Name | Data Type | Description | Key |
|-------------|-----------|-------------|-----|
| activity_date | DATE | Calendar date of activity | PK |
| user_id | INTEGER | Unique user identifier | PK |
| purchase_count | INTEGER | Number of purchases that day | |
| purchase_amount | DECIMAL(10,2) | Total revenue that day | |
| days_since_last_purchase | INTEGER | Recency metric for lifecycle analysis | |
| is_new | BOOLEAN | True if this is user's first purchase date | |
| is_retained | BOOLEAN | True if active today AND active yesterday | |
| is_resurrected | BOOLEAN | True if active today, inactive yesterday, but active before | |

**2. `fact_user_cohort_monthly`** (user-month-since-signup grain)
Fuels retention heat-maps and LTV curves. Includes denormalized `cohort_size` to speed percentage calculations.

| Column Name | Data Type | Description | Key |
|-------------|-----------|-------------|-----|
| signup_month | DATE | First day of user's signup month | PK |
| user_id | INTEGER | Unique user identifier | PK |
| months_since_signup | INTEGER | 0, 1, 2, 3... months since signup | PK |
| is_active_in_period | BOOLEAN | True if user purchased in this month | |
| purchase_count | INTEGER | Number of purchases in this month | |
| purchase_amount | DECIMAL(10,2) | Total revenue in this month | |
| cumulative_revenue | DECIMAL(10,2) | Total revenue from user since signup | |
| cohort_size | INTEGER | Total users in this signup cohort (denormalized) | |

**3. `v_user_lifecycle_quarterly`** (view on monthly fact)
Rolls monthly fact to quarters and adds `is_churned_and_returned`. Implemented as a view to avoid a third ETL path - do the simplest thing first, optimize later.

| Column Name | Data Type | Description | Key |
|-------------|-----------|-------------|-----|
| quarter | DATE | First day of quarter | PK |
| user_id | INTEGER | Unique user identifier | PK |
| was_active_current_quarter | BOOLEAN | True if user purchased in this quarter | |
| was_active_previous_quarter | BOOLEAN | True if user purchased in previous quarter | |
| is_churned_and_returned | BOOLEAN | True if churned last quarter but active this quarter | |
| quarterly_revenue | DECIMAL(10,2) | Total revenue in this quarter | |

#### Dimension Tables

**`dim_date`** (conformed dimension)
Standard date dimension used by all facts for consistent time-based filtering. Includes calendar + fiscal attributes.

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| date | DATE | Calendar date (Primary Key) |
| year | INTEGER | Calendar year |
| quarter | INTEGER | Calendar quarter (1-4) |
| month | INTEGER | Calendar month (1-12) |
| week_of_year | INTEGER | Week number within year |
| day_of_week | INTEGER | Day of week (1=Monday, 7=Sunday) |
| is_weekend | BOOLEAN | True for Saturday/Sunday |
| month_name | VARCHAR(20) | Full month name |
| quarter_name | VARCHAR(10) | Q1, Q2, Q3, Q4 |

**`dim_user`** (SCD Type 2)
User attributes with slowly changing dimensions for attributes like plan_tier, acquisition_channel.

| Column Name | Data Type | Description |
|-------------|-----------|-------------|
| user_id | INTEGER | Unique user identifier (Primary Key) |
| first_purchase_date | DATE | Date of user's first purchase |
| signup_cohort_month | DATE | First day of signup month |
| acquisition_channel | VARCHAR(50) | How user was acquired |
| country | VARCHAR(50) | User's country |
| region | VARCHAR(50) | User's region |
| plan_tier | VARCHAR(20) | Current subscription tier |
| customer_segment | VARCHAR(30) | Business-defined user segment |
| total_lifetime_purchases | INTEGER | Total purchases to date |
| total_lifetime_value | DECIMAL(10,2) | Total revenue to date |
| last_purchase_date | DATE | Most recent purchase date |
| user_status | VARCHAR(20) | Current status (active/churned/new) |

### Semantic Layer (Looker Explores)

These objects are views or materialized views in Snowflake cloud or other softwares, and are surfaced in Looker as Explores so non-SQL users can drag-and-drop:

**Daily KPIs** - DAU, Net-New, Churned, Resurrected, Revenue
**Weekly/Monthly KPIs** - WAU, MAU, growth %, MRR, retention rates  
**Cohort Retention** - signup_month × months_since_signup heat-map with pre-calculated retention rates

### How This Answers the example Specific Business Questions

**"How many net new users did we have last month?"**
Filter `summary_monthly_metrics` to last month, read `net_new_users` column directly.

**"What % of users who signed up in January are still purchasing three months later?"**
Filter `cohort_retention_summary` where `signup_month = 'January 2025'` and `period_number = 3`, read `retention_rate`.

**"What's the retention rate for users by signup cohort?"**
Create heatmap: rows = `signup_month`, columns = `period_label`, values = `retention_rate`.

**"How many users churned last quarter and came back this quarter?"**
Count rows in `v_user_lifecycle_quarterly` where `is_churned_and_returned = true` for current quarter.

**"What's the trend in DAU, WAU, and MAU?"**
Line charts from `daily_active_users`, `weekly_active_users`, `monthly_active_users` in respective summary tables.

### Why Business Leaders would prefer this, rather than complex SQL queries:

- **Drag-and-drop cohorts, no SQL** - Self-service analytics in Looker
- **Pre-aggregated views = sub-second dashboards** on tables with 100M+ rows
- **One definition of "new user"** used company-wide, eliminating metric inconsistencies
- **Ready-made visualizations** for common questions

### Operations & Governance

All facts are partitioned on their period key (`activity_date`, `signup_month`) and clustered on `user_id` so even 100M user-days scan in seconds. Access is enforced through a `business_intelligence` role with read-only grants on the views, keeping raw tables safe from accidental modifications.

**Roles**: `etl_writer` maintains facts/dimensions, `bi_reader` accesses curated semantic layer.

This model transforms complex user segmentation analysis into simple, accessible business intelligence,

---

## Question 3: Forecasting Quarterly Active Users

### SkyWatch-Specific Forecasting Framework

For a geospatial data platform like SkyWatch, I'd build a forecasting system that goes beyond a traditional time series analysis to capture the unique drivers of satellite imagery demand.

**Target Metric**: Quarterly Active Organizations (distinct `organization_id` with ≥1 imagery order)  
**Forecast Horizon**: Through quarter-end, refreshed weekly  
**Update Cadence**: Every Monday morning for stakeholder planning

### Multi-Layer Model Architecture

**1. Baseline Time Series Foundation**
- SARIMAX model capturing weekly seasonality and yearly agricultural cycles
- Fourier terms for complex seasonal patterns (crop seasons, fiscal quarter-end rushes)
- 3+ years of historical organization activity data as foundation

**2. Business Driver Uplift Model**
- XGBoost or CATBoost regression on week-over-week changes in active customers
- Key input features:
  - Sales pipeline data from HubSpot/Salesforce or acquire data from SkyWatch's alternative approach to capturing Sales Data (2-week lag)
  - Marketing campaign spend and timing
  - New satellite provider launches (+demand)
  - Weather events and cloud cover forecasts
  - Industry events (GEOINT conference, harvest seasons)

**3. Capacity Reality Check**
- Post-processing constraint: forecasted demand ≤ available satellite tasking capacity
- Prevents "paper demand" exceeding actual imagery acquisition slots
- Monitors provider utilization rates and capacity limits

### Industry-Specific Considerations

**Agricultural Seasonality**: Spring planting and fall harvest create predictable demand spikes  
**Weather Dependencies**: Clear sky forecasts drive urgent re-tasking requests (3-day lag)  
**Fiscal Calendar Effects**: Q4 budget spending creates December ordering surges  
**Satellite Capacity**: New provider launches increase available imagery, driving customer growth  
**Negative Events**: Solar eclipses, satellite failures, or provider outages temporarily reduce activity

### Key Assumptions & Monitoring

| Assumption | Validation Method |
|------------|------------------|
| Lead-to-order conversion stays ~18% | Validation: Weekly funnel report comparing MQL → SQL → won rates. Trigger: Retrain uplift model if the win-rate drifts by more than ±5 percentage points (e.g. from 18 % to below 13 % or above 23 %). |
| Marketing cost-per-acquisition within ±20% of plan | Validation: Compare actual spend vs. qualified leads generated each week. Trigger: Flag when CPA deviates more than 20 % from budgeted rate, then reevaluate spend efficacy. |
| No step-change in baseline demand |Validation: Prophet’s automated changepoint detection plus a weekly manual review of the raw DAU/WAU trends in Looker. Trigger: If a new structural shift is detected (e.g. sustained MAPE > 15 % on the baseline model), investigate whether to add a new regressor or adjust seasonality parameters |
| Satellite capacity isn't constraining | Validation: Ops dashboard showing % utilization per provider. Trigger: If any key provider’s tasking slots exceed 90 % booked, apply a post-forecast clip so we don’t overshoot “paper” demand. |

### Validation Strategy

**Statistical Validation**
- Walk-forward testing on last 8 quarters targeting sMAPE ≤12%
- Time series cross-validation across multiple seasonal cycles
- 80% and 90% confidence intervals using Monte Carlo simulation (surfaced on dashboard so stakeholders understand forecast uncertainty)

**Business Logic Checks**
- Lower bound: Current actives × 0.8 (conservative retention)
- Upper bound: (Pipeline × max win-rate) + current actives
- Sanity check: Average km² per organization shouldn't exceed 2× historical maximum

**Driver Attribution**
- SHAP analysis on uplift model to explain forecast changes
- Growth team reviews: "Agricultural summer peak adds +310 organizations"
- Human validation if forecast implies unrealistic usage patterns

**Revenue Impact Modeling**
- Multiply forecasted active organization count by expected quarterly spend per org
- Project quarterly bookings ceiling based on customer activity forecast
- Segment revenue projections by customer tier (enterprise vs. commercial)

### External Factor Integration

**Weather & Environmental** (Primary Focus)
- NOAA 10-day cloud cover forecasts for agricultural regions (moonshot feature - even coarse weather predictions can lift forecast accuracy 1-2 percentage points)
- Seasonal weather pattern analysis for demand timing
- Natural disaster monitoring for emergency imagery requests

**Market & Industry**
- Industry conference schedules and their demand impact (GEOINT, agricultural trade shows)
- Competitor satellite launch schedules affecting market capacity
- Agricultural commodity prices influencing farm investment decisions

**Regulatory & Economic**
- Government budget cycles for defense/intelligence customers
- Regulatory changes affecting imagery access and licensing
- Economic indicators for commercial real estate monitoring demand

### Implementation & Operations

**Data Pipeline**
- dbt builds forecast feature mart nightly
- Airflow DAG runs Monday 2AM UTC: train → backtest → publish to `forecast_active_orgs_by_week`
- Looker / Tableau dashboard: actual (blue), forecast (green), confidence intervals (shaded uncertainty bands)

**Performance Tracking & Alerts**
- `forecast_performance_tracking` table logs weekly accuracy metrics
- Slack webhook or email alerts when weekly actives deviate >1 standard deviation from trend
- Automated model performance tracking and retraining triggers in `model_performance_log`
- Weekly forecast accuracy reports for stakeholder confidence

Using this, we can specifically addresses the geospatial industry's unique characteristics:

- **Satellite capacity constraints** which ties straight into SkyWatch's business model
- **Weather dependencies** that create urgent, unpredictable demand spikes  
- **Agricultural seasonality** driving predictable annual patterns
- **Government/enterprise budget cycles** creating fiscal quarter effects
- **Provider ecosystem dynamics** where new satellite launches expand addressable market

All of this considered, we can create weekly-updated (or longer depending on business needs) forecast that gives a realistic view of paying customer growth while identifying the specific levers (marketing spend, weather patterns, capacity additions) that drive those numbers.

---

## Question 4: Debugging MAU Drop

### Systematic Investigation Process

First I'd sanity-check the raw landing table: run `SELECT count_distinct(user_id) FROM events_raw WHERE event_date BETWEEN ...` and compare to the same query for the prior month. If the raw count hasn't changed, I move upstream—look for failed or delayed ingests in Airflow/dbt logs (schema drift, API quota, S3 object size zero, etc.). If the raw count is healthy, I diff the most recent dbt run artifacts: did a model or `is_active` definition change in yesterday's PR? a filter accidentally switched from `>=` to `=`? Finally, I open the the Dashboading software where the results are being displayed such as Looker, explore behind the MAU tile to be sure the dashboard isn't double-filtering (e.g., last month's data got tagged with a new fiscal calendar flag). That three-stop drill—**raw rows → pipeline logs → semantic layer diff**—usually surfaces the main issue within 15 minutes.


