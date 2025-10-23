# Comprehensive Guide: Conversion Rate Calculations Across the RevOps Analytics Platform

**Document Purpose:** This guide explains the mathematical foundations and implementation patterns for conversion rate calculations across all RevOps views, ensuring consistent and accurate metrics throughout the analytics platform.

**Target Audience:** RevOps professionals with basic statistics knowledge 

**Last Updated:** October 22, 2025

---

## Table of Contents
1. [Core Mathematical Principles](#core-principles)
2. [The Three-State Logic Problem](#three-state)
3. [Conversion Rate Formulas Across Views](#formulas)
4. [Implementation Patterns in SQL](#sql-patterns)
5. [The Cascade Model Mathematics](#cascade)
6. [Aggregation and Rollup Strategies](#aggregation)
7. [Common Pitfalls and Solutions](#pitfalls)
8. [View-Specific Implementations](#view-specific)
9. [Statistical Confidence and Volatility](#statistics)
10. [Unified Measurement Framework](#framework)

---

## Core Mathematical Principles {#core-principles}

### The Fundamental Formula

At its heart, every conversion rate follows this principle:

```
Conversion Rate = Successful Outcomes / Opportunities for Success
```

**Not:**
```
Conversion Rate = Successful Outcomes / All Records
```

This distinction is **critical** and underlies every correct implementation across our platform.

### Why This Matters: A Biological Analogy

Think of conversion rates like germination rates in biology:
- You plant 100 seeds (opportunities)
- 30 seeds are still in the packet (not planted yet)
- Of the 70 planted seeds, 49 germinate (successful outcomes)

**Wrong calculation:** 49/100 = 49% germination rate
**Right calculation:** 49/70 = 70% germination rate

The 30 seeds still in the packet haven't had the **opportunity** to germinate yet. Including them artificially depresses your rate.

---

## The Three-State Logic Problem {#three-state}

### The SQO Challenge

The most complex conversion rate in our funnel is SQL → SQO because it involves three states:

1. **Yes** - Qualified (1)
2. **No** - Not qualified (0)  
3. **NULL** - Not yet evaluated (waiting for decision)

### The Mathematical Impact

Looking at the `vw_sga_funnel.sql` (CORRECT implementation):

```sql
-- CORRECT: Three-state logic
CASE 
  WHEN LOWER(o.SQO_raw) = 'yes' THEN 1 
  WHEN LOWER(o.SQO_raw) = 'no' THEN 0
  ELSE NULL 
END AS is_sqo
```

versus `vw_sga_funnel_team_agg.sql` (INCORRECT implementation):

```sql
-- INCORRECT: Two-state logic
CASE WHEN LOWER(o.SQO_raw) = "yes" THEN 1 ELSE 0 END AS is_sqo
```

**The Mathematical Difference:**

Consider 100 SQLs:
- 40 evaluated as "yes" (SQO)
- 20 evaluated as "no" (not SQO)
- 40 not yet evaluated (NULL)

**Incorrect calculation (two-state):**
- Numerator: 40 (yes)
- Denominator: 100 (all SQLs)
- Rate: 40/100 = **40%**

**Correct calculation (three-state):**
- Numerator: 40 (yes)
- Denominator: 60 (yes + no only)
- Rate: 40/60 = **67%**

The incorrect method shows a 40% conversion rate when the actual rate is 67% - a massive 27 percentage point error!

---

## Conversion Rate Formulas Across Views {#formulas}

### Stage-by-Stage Formulas

#### 1. Contacted → MQL Rate

```sql
-- From vw_forecast_vs_actuals
SAFE_DIVIDE(
  COUNT(DISTINCT IF(is_mql = 1, Full_prospect_id__c, NULL)), 
  COUNT(DISTINCT IF(is_contacted = 1, Full_prospect_id__c, NULL))
) AS prospect_to_mql_rate
```

**Mathematical expression:**
```
Rate = MQLs / Contacted Prospects
```

**Why SAFE_DIVIDE?**
- Prevents division by zero
- Returns NULL instead of error
- Enables robust calculations even with sparse data

#### 2. MQL → SQL Rate

```sql
SAFE_DIVIDE(
  COUNT(DISTINCT IF(is_sql = 1, Full_prospect_id__c, NULL)), 
  COUNT(DISTINCT IF(is_mql = 1, Full_prospect_id__c, NULL))
) AS mql_to_sql_rate
```

**Key insight:** Both use `Full_prospect_id__c` because SQLs are still tracked as leads.

#### 3. SQL → SQO Rate (The Complex One)

```sql
-- CORRECT implementation
SAFE_DIVIDE(
  COUNT(DISTINCT CASE 
    WHEN is_sqo = 1 AND Full_Opportunity_ID__c IS NOT NULL 
    THEN Full_Opportunity_ID__c 
  END),
  COUNT(DISTINCT CASE 
    WHEN LOWER(SQO_raw) IN ('yes', 'no') AND Full_Opportunity_ID__c IS NOT NULL 
    THEN Full_Opportunity_ID__c 
  END)
) AS sql_to_sqo_rate
```

**Critical changes:**
1. Switches from `Full_prospect_id__c` to `Full_Opportunity_ID__c`
2. Denominator explicitly checks for 'yes' or 'no' (not NULL)
3. Ensures opportunity exists before counting

#### 4. SQO → Joined Rate

```sql
SAFE_DIVIDE(
  COUNT(DISTINCT CASE 
    WHEN is_joined = 1 AND Full_Opportunity_ID__c IS NOT NULL 
    THEN Full_Opportunity_ID__c 
  END),
  COUNT(DISTINCT CASE 
    WHEN is_sqo = 1 AND Full_Opportunity_ID__c IS NOT NULL 
    THEN Full_Opportunity_ID__c 
  END)
) AS sqo_to_joined_rate
```

---

## Implementation Patterns in SQL {#sql-patterns}

### Pattern 1: The IF Statement for Binary Flags

```sql
COUNT(DISTINCT IF(is_mql = 1, Full_prospect_id__c, NULL))
```

**How it works:**
1. `IF(is_mql = 1, Full_prospect_id__c, NULL)` returns:
   - The ID if condition is true
   - NULL if condition is false
2. `COUNT(DISTINCT ...)` ignores NULLs
3. Result: Count of unique MQLs

**Equivalent to but more elegant than:**
```sql
SELECT COUNT(DISTINCT Full_prospect_id__c)
FROM table
WHERE is_mql = 1
```

### Pattern 2: The CASE Statement for Complex Logic

```sql
COUNT(DISTINCT CASE 
  WHEN LOWER(SQO_raw) IN ('yes', 'no') 
    AND Full_Opportunity_ID__c IS NOT NULL 
  THEN Full_Opportunity_ID__c 
END)
```

**Why CASE instead of IF?**
- Multiple conditions
- More readable for complex logic
- Standard SQL (IF is MySQL/BigQuery specific)

### Pattern 3: Aggregation with Window Functions

From `vw_forecast_timeseries_with_confidence`:

```sql
SUM(SUM(COALESCE(a.daily_count, 0))) OVER (
  PARTITION BY dims.Channel_Grouping_Name, dims.Original_source, dims.stage 
  ORDER BY d.date_day
) AS cumulative_actual
```

**The double SUM pattern:**
- Inner `SUM()`: Aggregates within GROUP BY
- Outer `SUM() OVER`: Creates running total
- Result: Cumulative growth curve

---

## The Cascade Model Mathematics {#cascade}

### Understanding the Waterfall

From `vw_forecast_vs_actuals`, the cascade calculation for future conversions:

```sql
-- Future SQLs calculation
((COALESCE(p.open_mqls, 0) + 
  (COALESCE(p.open_prospects, 0) * COALESCE(r.prospect_to_mql_rate, 0)) + 
  (COALESCE(rf_p.remaining_value, 0) * COALESCE(r.prospect_to_mql_rate, 0)))
 * COALESCE(r.mql_to_sql_rate, 0)) AS future_sqls
```

### Breaking Down the Mathematics

Let's use actual numbers to understand:

**Given:**
- Open MQLs: 50
- Open Prospects: 100
- Remaining forecast prospects: 200
- Prospect → MQL rate: 30%
- MQL → SQL rate: 50%

**Step 1: Calculate future MQLs**
```
Future MQLs from open prospects = 100 × 0.30 = 30
Future MQLs from forecast = 200 × 0.30 = 60
Total future MQLs = 30 + 60 = 90
```

**Step 2: Calculate MQL pool**
```
Total MQL pool = Current open (50) + Future (90) = 140
```

**Step 3: Calculate future SQLs**
```
Future SQLs = 140 × 0.50 = 70
```

### The Recursive Nature

For SQOs, it gets even more complex:

```sql
-- Future SQOs = (Open SQLs + all future SQLs) × conversion rate
((COALESCE(p.open_sqls, 0) +
  ((COALESCE(p.open_mqls, 0) + 
    (COALESCE(p.open_prospects, 0) * COALESCE(r.prospect_to_mql_rate, 0)) + 
    (COALESCE(rf_p.remaining_value, 0) * COALESCE(r.prospect_to_mql_rate, 0)))
   * COALESCE(r.mql_to_sql_rate, 0)))
 * COALESCE(r.sql_to_sqo_rate, 0))
```

This is a **Markov chain** where each state's future population depends on all previous states.

---

## Aggregation and Rollup Strategies {#aggregation}

### The Challenge of Pre-Aggregated Data

When working with pre-aggregated views like `vw_sga_funnel_team_agg`, we need different approaches:

#### For Individual Records (vw_sga_funnel):
```sql
-- Can use standard aggregation
SELECT 
  SGA_Owner_Name__c,
  COUNT(DISTINCT Full_prospect_id__c) as prospects,
  SUM(is_mql) as MQLs,
  SUM(is_mql) / NULLIF(SUM(is_contacted), 0) as conversion_rate
FROM vw_sga_funnel
GROUP BY 1
```

#### For Pre-Aggregated Data (vw_sga_funnel_team_agg):
```sql
-- Already aggregated, need different approach
SELECT 
  FilterDate,
  SUM(team_mql) as total_MQLs,
  SUM(team_contacted) as total_contacted,
  SUM(team_mql) / NULLIF(SUM(team_contacted), 0) as team_conversion_rate
FROM vw_sga_funnel_team_agg
GROUP BY 1
```

### The Rollup Pattern

Creating channel-level rollups from source-level data:

```sql
-- Source level
SELECT 
  channel_grouping_name,
  original_source,
  actual_value,
  forecast_value,
  predicted_value
FROM vw_forecast_vs_actuals

UNION ALL

-- Channel rollup
SELECT 
  channel_grouping_name,
  'All' as original_source,
  SUM(actual_value),
  SUM(forecast_value),
  SUM(predicted_value)
FROM vw_forecast_vs_actuals
GROUP BY 1
```

---

## Common Pitfalls and Solutions {#pitfalls}

### Pitfall 1: Including NULL as Zero

**Wrong:**
```sql
-- This treats "not evaluated" as "no"
SUM(CASE WHEN is_sqo = 1 THEN 1 ELSE 0 END)
```

**Right:**
```sql
-- This excludes NULL from calculation
SUM(CASE WHEN is_sqo = 1 THEN 1 WHEN is_sqo = 0 THEN 0 ELSE NULL END)
```

### Pitfall 2: Wrong ID Field

**Wrong:**
```sql
-- Using lead ID for opportunity-level metric
COUNT(DISTINCT IF(is_sqo = 1, Full_prospect_id__c, NULL))
```

**Right:**
```sql
-- Using opportunity ID for opportunity metrics
COUNT(DISTINCT IF(is_sqo = 1, Full_Opportunity_ID__c, NULL))
```

### Pitfall 3: Missing DISTINCT

**Wrong:**
```sql
-- Could double-count if duplicates exist
COUNT(IF(is_mql = 1, Full_prospect_id__c, NULL))
```

**Right:**
```sql
-- Ensures unique count
COUNT(DISTINCT IF(is_mql = 1, Full_prospect_id__c, NULL))
```

---

## View-Specific Implementations {#view-specific}

### vw_funnel_lead_to_joined_v2 (Master View)

**Purpose:** Foundation for all calculations

**Key features:**
- FULL OUTER JOIN for complete coverage
- Binary flags for easy aggregation
- Composite primary key for flexibility

**Conversion rate enablement:**
```sql
-- All flags pre-calculated for downstream use
CASE WHEN Stage_Entered_Call_Scheduled__c IS NOT NULL THEN 1 ELSE 0 END AS is_mql,
CASE WHEN IsConverted IS TRUE THEN 1 ELSE 0 END AS is_sql,
-- Note: is_sqo NOT calculated here (varies by use case)
CASE WHEN o.advisor_join_date__c IS NOT NULL THEN 1 ELSE 0 END AS is_joined
```

### vw_sga_funnel (Individual SGA View)

**Purpose:** SGA-level performance tracking

**Correct implementation:**
```sql
-- Three-state SQO logic
CASE 
  WHEN LOWER(o.SQO_raw) = 'yes' THEN 1 
  WHEN LOWER(o.SQO_raw) = 'no' THEN 0
  ELSE NULL 
END AS is_sqo
```

**Active SGA filtering:**
```sql
INNER JOIN Active_SGAs a ON cd.SGA_Owner_Name__c = a.sga_name
```

### vw_sga_funnel_team_agg (Team Aggregated View)

**Purpose:** Pre-aggregated team metrics

**Current bug:**
```sql
-- INCORRECT: Two-state logic loses information
CASE WHEN LOWER(o.SQO_raw) = "yes" THEN 1 ELSE 0 END AS is_sqo
```

**Required fix:**
```sql
-- Add separate evaluation tracking
SUM(CASE WHEN LOWER(o.SQO_raw) = "yes" THEN 1 ELSE 0 END) as team_sqo,
SUM(CASE WHEN LOWER(o.SQO_raw) IN ("yes", "no") THEN 1 ELSE 0 END) as team_sqo_evaluated
```

Then calculate rates as:
```sql
team_sqo / NULLIF(team_sqo_evaluated, 0) as sqo_conversion_rate
```

### vw_forecast_vs_actuals (Prediction Engine)

**Purpose:** Current state and future predictions

**Conversion rate usage:**
- 90-day trailing rates for stability
- Applied to open pipeline for predictions
- Cascade model for multi-stage forecasting

### vw_forecast_timeseries_with_confidence (Visualization Layer)

**Purpose:** Daily projections with uncertainty

**Statistical elements:**
```sql
-- Confidence interval using historical volatility
predicted_value ± (1.96 × stddev_daily_growth × SQRT(days_forward))
```

---

## Statistical Confidence and Volatility {#statistics}

### Standard Deviation in Conversion Rates

From `vw_forecast_vs_actuals`:

```sql
STDDEV(daily_count) AS stddev_daily
```

**What this tells us:**
- Low stddev (e.g., 2): Consistent daily performance
- High stddev (e.g., 15): Volatile, unpredictable
- Used for confidence intervals

### Confidence Intervals for Rates

For a conversion rate with standard error:

```
95% CI = Rate ± 1.96 × Standard Error
```

Where Standard Error for a proportion:
```
SE = √(p × (1-p) / n)
```

Example:
- Conversion rate: 50%
- Sample size: 100
- SE = √(0.5 × 0.5 / 100) = 0.05
- 95% CI = 50% ± (1.96 × 5%) = [40%, 60%]

### The Square Root of Time

In time series projections:

```sql
SQRT(DATE_DIFF(future_date, CURRENT_DATE(), DAY))
```

**Why √time?**
- Uncertainty compounds like a random walk
- Variance grows linearly with time
- Standard deviation grows with √time
- Based on Brownian motion theory

---

## Unified Measurement Framework {#framework}

### Design Principles

1. **Consistency:** Same calculation method across all views
2. **Accuracy:** Three-state logic where applicable
3. **Robustness:** SAFE_DIVIDE and COALESCE for edge cases
4. **Scalability:** Pre-aggregation where beneficial
5. **Clarity:** Binary flags for simple operations

### The Standard Pattern

For any conversion rate calculation:

```sql
WITH base_data AS (
  -- Get records with proper flags
  SELECT * FROM vw_funnel_lead_to_joined_v2
),
calculations AS (
  SELECT
    dimension,  -- e.g., SGA, channel, source
    -- Counts for each stage
    COUNT(DISTINCT CASE WHEN is_contacted = 1 THEN Full_prospect_id__c END) as contacted,
    COUNT(DISTINCT CASE WHEN is_mql = 1 THEN Full_prospect_id__c END) as mqls,
    COUNT(DISTINCT CASE WHEN is_sql = 1 THEN Full_prospect_id__c END) as sqls,
    -- Special handling for SQO
    COUNT(DISTINCT CASE WHEN is_sqo = 1 THEN Full_Opportunity_ID__c END) as sqos,
    COUNT(DISTINCT CASE WHEN is_sqo IN (0,1) THEN Full_Opportunity_ID__c END) as sqo_evaluated,
    COUNT(DISTINCT CASE WHEN is_joined = 1 THEN Full_Opportunity_ID__c END) as joined
  FROM base_data
  GROUP BY dimension
)
SELECT
  dimension,
  contacted,
  mqls,
  sqls,
  sqos,
  joined,
  -- Conversion rates
  SAFE_DIVIDE(mqls, contacted) as contact_to_mql_rate,
  SAFE_DIVIDE(sqls, mqls) as mql_to_sql_rate,
  SAFE_DIVIDE(sqos, sqo_evaluated) as sql_to_sqo_rate,  -- Note: special denominator
  SAFE_DIVIDE(joined, sqos) as sqo_to_joined_rate
FROM calculations
```

### Testing Your Implementation

Run this diagnostic query:

```sql
-- Check for three-state logic impact
SELECT
  'Check' as validation,
  COUNT(DISTINCT CASE WHEN is_sqo = 1 THEN Full_Opportunity_ID__c END) as sqos,
  COUNT(DISTINCT CASE WHEN is_sqo = 0 THEN Full_Opportunity_ID__c END) as not_sqos,
  COUNT(DISTINCT CASE WHEN is_sqo IS NULL THEN Full_Opportunity_ID__c END) as pending,
  -- Correct rate
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN is_sqo = 1 THEN Full_Opportunity_ID__c END),
    COUNT(DISTINCT CASE WHEN is_sqo IN (0,1) THEN Full_Opportunity_ID__c END)
  ) * 100, 1) as correct_rate,
  -- Wrong rate (if using two-state)
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN is_sqo = 1 THEN Full_Opportunity_ID__c END),
    COUNT(DISTINCT Full_Opportunity_ID__c)
  ) * 100, 1) as wrong_rate,
  -- The error
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN is_sqo = 1 THEN Full_Opportunity_ID__c END),
    COUNT(DISTINCT CASE WHEN is_sqo IN (0,1) THEN Full_Opportunity_ID__c END)
  ) * 100, 1) -
  ROUND(SAFE_DIVIDE(
    COUNT(DISTINCT CASE WHEN is_sqo = 1 THEN Full_Opportunity_ID__c END),
    COUNT(DISTINCT Full_Opportunity_ID__c)
  ) * 100, 1) as percentage_point_error
FROM your_view
WHERE is_sql = 1
```

---

## Practical Applications and Business Impact

### For RevOps Teams

**Daily Operations:**
1. Use three-state logic for accurate SQO rates
2. Monitor pending evaluations separately
3. Track conversion rate trends over time
4. Use 90-day rates for stability

**Performance Management:**
```sql
-- SGA Performance Dashboard Query
SELECT 
  SGA_Owner_Name__c,
  DATE_TRUNC(FilterDate, WEEK) as week,
  SUM(is_contacted) as contacted,
  SUM(is_mql) as mqls,
  SAFE_DIVIDE(SUM(is_mql), SUM(is_contacted)) as conversion_rate,
  -- Comparison to team average
  SAFE_DIVIDE(SUM(is_mql), SUM(is_contacted)) - 
    (SELECT SAFE_DIVIDE(SUM(is_mql), SUM(is_contacted)) 
     FROM vw_sga_funnel 
     WHERE DATE_TRUNC(FilterDate, WEEK) = DATE_TRUNC(CURRENT_DATE(), WEEK)) 
  as vs_team_avg
FROM vw_sga_funnel
WHERE FilterDate >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 1, 2
```

### For Leadership

**Strategic Insights:**
- True conversion rates (not artificially depressed)
- Cascade model shows full impact of improvements
- Confidence intervals quantify uncertainty
- Predictions based on actual performance

### For Data Teams

**Implementation Checklist:**
- [ ] Three-state SQO logic implemented
- [ ] DISTINCT used in all counts
- [ ] Correct ID fields for each stage
- [ ] SAFE_DIVIDE for division operations
- [ ] FilterDate for cohort consistency
- [ ] Binary flags pre-calculated where possible
- [ ] Aggregation strategy documented

---

## Conclusion

The mathematics of conversion rates seems simple - successful outcomes divided by opportunities - but the implementation requires careful attention to detail. The three-state logic for SQO calculations alone can create 20-30 percentage point errors if implemented incorrectly.

By following these patterns consistently across all views:
1. We ensure accurate reporting
2. We enable valid comparisons
3. We support sophisticated forecasting
4. We maintain statistical rigor

The cascade model in our forecasting views demonstrates how these rates compound through the funnel, while our confidence interval calculations acknowledge the inherent uncertainty in predictions.

Most importantly, this unified framework means that whether you're looking at an individual SGA's performance, team aggregates, channel-level forecasts, or time series projections, the underlying mathematics remains consistent and correct. This consistency is what transforms raw data into trustworthy business intelligence that drives confident decision-making.

Remember: A 57% conversion rate that should be 77% isn't just a math error - it's the difference between thinking you need major process improvements versus celebrating strong performance. Getting the math right matters.