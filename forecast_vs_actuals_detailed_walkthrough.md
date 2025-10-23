# Complete Walkthrough: vw_forecast_vs_actuals SQL Logic and Mathematics

**Document Purpose:** This guide provides a detailed, step-by-step explanation of the SQL code that powers the forecast vs actuals view, including the mathematical logic and statistical concepts behind each calculation.
 
**Last Updated:** October 23, 2025

---

## Table of Contents
1. [Overview: The Dual-Track Approach](#overview)
2. [CTE 1: Funnel_With_Flags - Active Team Filter](#cte1)
3. [CTE 2: Funnel_Unfiltered - Complete Historical Data](#cte2)
4. [CTE 3: Forecast_Data - Standardizing Targets](#cte3)
5. [CTE 4: Forecast_Q4_Total - Quarterly Aggregation](#cte4)
6. [CTE 5: QTD_Actuals - What Really Happened](#cte5)
7. [CTE 6: Trailing_Rates - Learning from Past Performance](#cte6)
8. [CTE 7: Open_Pipeline - Work in Progress](#cte7)
9. [CTE 8: Remaining_Forecast - The Gap Analysis](#cte8)
10. [CTE 9: Future_Conversions - The Cascade Model](#cte9)
11. [CTE 10: Historical_Volatility - Measuring Uncertainty](#cte10)
12. [Final SELECT - Bringing It All Together](#final-select)

---

## Overview: The Dual-Track Approach {#overview}

This view answers three critical questions for Q4 2025:
1. **Where are we?** (Actuals - counting everything that happened)
2. **Where did we want to be?** (Forecast - original targets)
3. **Where will we likely end up?** (Predictions - based on current team capabilities)

The key innovation is a dual-track approach:
- **Track 1: Unfiltered Data** - Used for actuals and volatility (what actually happened)
- **Track 2: Filtered Data** - Used for conversion rates and predictions (what the current team can deliver)

Think of it like tracking a relay race: You want to know the total distance covered (all runners' contributions) but predict the finish based on who's still running.

---

## CTE 1: Funnel_With_Flags - Active Team Filter {#cte1}

### Code:
```sql
Funnel_With_Flags AS (
  SELECT
    f.*,
    CASE WHEN LOWER(f.SQO_raw) = 'yes' THEN 1 ELSE 0 END AS is_sqo
  FROM `savvy-gtm-analytics.savvy_analytics.vw_funnel_lead_to_joined_v2` f
  WHERE 
    (f.SGA_Owner_Name__c IN (
      SELECT DISTINCT Name 
      FROM `savvy-gtm-analytics.SavvyGTMData.User` 
      WHERE (IsSGA__c = TRUE OR Is_SGM__c = TRUE)
        AND IsActive = TRUE
    ))
    OR
    (f.Opportunity_Owner_Name__c IN (
      SELECT DISTINCT Name 
      FROM `savvy-gtm-analytics.SavvyGTMData.User` 
      WHERE (IsSGA__c = TRUE OR Is_SGM__c = TRUE)
        AND IsActive = TRUE
    ))
)
```

### Logic Explanation:

This CTE creates our filtered dataset - only records owned by currently active sales team members.

**The Active Team Subquery:**
```sql
SELECT DISTINCT Name 
FROM `savvy-gtm-analytics.SavvyGTMData.User` 
WHERE (IsSGA__c = TRUE OR Is_SGM__c = TRUE)
  AND IsActive = TRUE
```
This identifies who's currently on the team. Think of it as taking attendance - who's here today to help us finish the quarter?

**Why Two Owner Fields?**
- `SGA_Owner_Name__c`: The lead owner (early funnel stages)
- `Opportunity_Owner_Name__c`: The opportunity owner (later stages)
- Using OR ensures we catch records at any stage owned by active team

**Statistical Purpose:**
This filtered data will be used to calculate conversion rates that reflect current team capabilities. It's like calculating a batting average only for players still on the roster - more predictive of future performance.

---

## CTE 2: Funnel_Unfiltered - Complete Historical Data {#cte2}

### Code:
```sql
Funnel_Unfiltered AS (
  SELECT
    *,
    CASE WHEN LOWER(SQO_raw) = 'yes' THEN 1 ELSE 0 END AS is_sqo
  FROM `savvy-gtm-analytics.savvy_analytics.vw_funnel_lead_to_joined_v2`
)
```

### Logic Explanation:

This CTE pulls ALL historical data without any filtering.

**Why Keep Both Filtered and Unfiltered?**
Imagine you're tracking a marathon team's performance:
- **Unfiltered (this CTE)**: Total miles run by everyone who participated, even those who dropped out
- **Filtered (CTE 1)**: Only miles run by people still on the team

We need both because:
1. Management wants to know actual Q4 performance (unfiltered)
2. But predictions should be based on who can actually deliver going forward (filtered)

**The SQO Flag:**
```sql
CASE WHEN LOWER(SQO_raw) = 'yes' THEN 1 ELSE 0 END AS is_sqo
```
Converts text to binary (1/0) for mathematical operations. `LOWER()` handles case variations.

---

## CTE 3: Forecast_Data - Standardizing Targets {#cte3}

### Code:
```sql
Forecast_Data AS (
  SELECT
    month_key,
    CASE WHEN Channel = 'Inbound' THEN 'Marketing' ELSE Channel END AS channel_grouping_name,
    original_source,
    metric,
    CASE
      WHEN LOWER(stage) = 'mql' THEN 'mqls'
      WHEN LOWER(stage) = 'sql' THEN 'sqls'
      WHEN LOWER(stage) = 'sqo' THEN 'sqos'
      ELSE LOWER(stage)
    END AS stage,
    CAST(forecast_value AS INT64) AS forecast_value
  FROM `savvy-gtm-analytics.SavvyGTMData.q4_2025_forecast`
  WHERE metric = 'Cohort_source'
    AND original_source != 'All'
)
```

### Logic Explanation:

This standardizes naming conventions from the forecast table.

**Key Transformations:**
1. **Channel Rename:** 'Inbound' → 'Marketing' (aligning terminology)
2. **Stage Pluralization:** 'mql' → 'mqls' (consistency with other views)
3. **Data Type:** String to INT64 (ensuring mathematical operations work)

**The Filter:**
- `metric = 'Cohort_source'`: Source-level granularity
- `original_source != 'All'`: Excludes rollup rows to avoid double-counting

---

## CTE 4: Forecast_Q4_Total - Quarterly Aggregation {#cte4}

### Code:
```sql
Forecast_Q4_Total AS (
  SELECT
    channel_grouping_name,
    original_source,
    stage,
    SUM(forecast_value) AS forecast_value
  FROM Forecast_Data
  WHERE month_key IN ('2025-10', '2025-11', '2025-12')
  GROUP BY channel_grouping_name, original_source, stage
)
```

### Logic Explanation:

Aggregates monthly forecasts into quarterly totals.

**The Mathematics:**
If October target = 140, November = 145, December = 145
Then Q4 total = 140 + 145 + 145 = 430

This represents the original "stretch goal" set before the quarter began.

---

## CTE 5: QTD_Actuals - What Really Happened {#cte5}

### Code:
```sql
QTD_Actuals AS (
  SELECT channel_grouping_name, original_source, 'mqls' AS stage,
         COUNT(DISTINCT Full_prospect_id__c) AS actual_value
  FROM Funnel_Unfiltered  -- Note: Using UNFILTERED
  WHERE is_mql = 1 
    AND DATE(mql_stage_entered_ts) BETWEEN '2025-10-01' AND CURRENT_DATE()
  GROUP BY 1, 2
  
  -- Similar UNION ALL sections for sqls, sqos, joined...
)
```

### Logic Explanation:

This counts actual performance using **unfiltered data**.

**Critical Design Decision:**
Uses `Funnel_Unfiltered` because actuals should reflect total team achievement, regardless of who's still employed. If someone made 50 sales then quit, those 50 sales still count toward Q4 actuals.

**Different ID Fields by Stage:**
- MQLs/SQLs: `COUNT(DISTINCT Full_prospect_id__c)` - counting leads
- SQOs/Joined: `COUNT(DISTINCT Full_Opportunity_ID__c)` - counting opportunities

This switch happens because leads convert to opportunities at the SQL→SQO transition.

---

## CTE 6: Trailing_Rates - Learning from Past Performance {#cte6}

### Code:
```sql
Trailing_Rates AS (
  SELECT
    channel_grouping_name,
    original_source,
    
    SAFE_DIVIDE(
      COUNT(DISTINCT IF(is_sql = 1, Full_prospect_id__c, NULL)), 
      COUNT(DISTINCT IF(is_mql = 1, Full_prospect_id__c, NULL))
    ) AS mql_to_sql_rate,
    
    -- Other rates...
    
  FROM Funnel_With_Flags  -- Note: Using FILTERED
  WHERE DATE(FilterDate) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) AND CURRENT_DATE()
  GROUP BY 1, 2
)
```

### Logic Explanation:

Calculates conversion rates using **filtered data** (active team only).

**Why 90 Days?**
Statistical principle: Larger sample sizes yield more stable estimates. 90 days balances:
- Recency (relevant to current performance)
- Sample size (enough data for reliability)

**The SAFE_DIVIDE Function:**
```sql
SAFE_DIVIDE(numerator, denominator)
```
Returns NULL instead of error when denominator is zero. Essential for sources with no activity.

**Statistical Interpretation:**
These rates answer: "If the current team gets 100 MQLs, how many will likely convert to SQL?"
Not: "How did we perform historically?" (that would use unfiltered data)

---

## CTE 7: Open_Pipeline - Work in Progress {#cte7}

### Code:
```sql
Open_Pipeline AS (
  SELECT
    channel_grouping_name,
    original_source,
    COUNT(DISTINCT IF(is_mql = 1 AND is_sql = 0, Full_prospect_id__c, NULL)) AS open_mqls,
    -- Other open stages...
  FROM Funnel_With_Flags  -- Note: Using FILTERED
  WHERE DATE(FilterDate) BETWEEN '2025-10-01' AND CURRENT_DATE()
    AND (StageName IS NULL OR StageName NOT LIKE '%Closed Lost%')
  GROUP BY 1, 2
)
```

### Logic Explanation:

Counts in-progress records using **filtered data**.

**Why Filtered?**
Only the active team can work these opportunities. If someone left with 20 open MQLs and those weren't reassigned, including them would overstate what can actually convert.

**The Stage Logic:**
- `is_mql = 1 AND is_sql = 0`: Has become MQL but not yet SQL
- Excludes 'Closed Lost' (failed opportunities that won't convert)

---

## CTE 8: Remaining_Forecast - The Gap Analysis {#cte8}

### Code:
```sql
Remaining_Forecast AS (
  SELECT
    f.channel_grouping_name,
    f.original_source,
    f.stage,
    GREATEST(0, f.forecast_value - COALESCE(a.actual_value, 0)) AS remaining_value
  FROM Forecast_Q4_Total f
  LEFT JOIN QTD_Actuals a
    ON f.channel_grouping_name = a.channel_grouping_name
    AND f.original_source = a.original_source
    AND f.stage = a.stage
)
```

### Logic Explanation:

Calculates how far behind (or ahead) we are versus plan.

**The GREATEST Function:**
```sql
GREATEST(0, forecast - actual)
```
Ensures we never get negative remaining values. If we've exceeded forecast, remaining = 0.

**Example:**
- Forecast: 100 MQLs
- Actual so far: 40 MQLs  
- Remaining: GREATEST(0, 100 - 40) = 60 MQLs still needed

---

## CTE 9: Future_Conversions - The Cascade Model {#cte9}

### Code:
```sql
Future_Conversions AS (
  SELECT
    -- Future MQLs = Open prospects converting + Remaining forecast prospects converting
    (COALESCE(p.open_prospects, 0) * COALESCE(r.prospect_to_mql_rate, 0)) + 
    (COALESCE(rf_p.remaining_value, 0) * COALESCE(r.prospect_to_mql_rate, 0)) AS future_mqls,
    
    -- Future SQLs = (Open MQLs + all future MQLs) * conversion rate
    ((COALESCE(p.open_mqls, 0) + [future_mqls calculation])
     * COALESCE(r.mql_to_sql_rate, 0)) AS future_sqls,
    
    -- Continues cascading through SQO and Joined...
)
```

### Logic Explanation:

This implements a cascade/waterfall model where each stage feeds the next.

**The Mathematics:**
Think of it like a series of filters, each capturing a percentage:

1. **Future MQLs** = (Open Prospects × Rate) + (New Prospects × Rate)
2. **Future SQLs** = (Current MQLs + Future MQLs) × MQL→SQL Rate
3. **Future SQOs** = (Current SQLs + Future SQLs) × SQL→SQO Rate
4. **Future Joined** = (Current SQOs + Future SQOs) × SQO→Joined Rate

**Statistical Concept:**
This is a Markov chain - a sequence where each state depends only on the previous state. The probability of reaching "Joined" = product of all intermediate conversion rates.

---

## CTE 10: Historical_Volatility - Measuring Uncertainty {#cte10}

### Code:
```sql
Historical_Volatility AS (
  SELECT
    channel_grouping_name,
    original_source,
    stage,
    STDDEV(daily_count) AS stddev_daily
  FROM (
    SELECT DATE(mql_stage_entered_ts) AS event_date, channel_grouping_name, original_source,
           'mqls' AS stage, COUNT(DISTINCT Full_prospect_id__c) AS daily_count
    FROM Funnel_Unfiltered  -- Note: Using UNFILTERED
    WHERE is_mql = 1 
      AND DATE(mql_stage_entered_ts) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) AND CURRENT_DATE()
    GROUP BY 1, 2, 3, 4
  )
  GROUP BY 1, 2, 3
)
```

### Logic Explanation:

Calculates daily volatility using **unfiltered data** for the most complete picture.

**Why Standard Deviation?**
It measures consistency. 
- Low stddev (e.g., 2): Performance is predictable, varying by ±2 daily
- High stddev (e.g., 15): Wild swings, hard to predict

**Why Unfiltered for Volatility?**
We want to know the true historical volatility of the business, not just the current team's consistency. This gives more conservative (realistic) confidence intervals.

**Statistical Interpretation:**
If stddev = 5, then:
- 68% of days will be within ±5 of average (one standard deviation)
- 95% of days will be within ±10 of average (two standard deviations)

---

## Final SELECT - Bringing It All Together {#final-select}

### Code:
```sql
SELECT
  COALESCE(a.channel_grouping_name, f.channel_grouping_name, fc.channel_grouping_name) AS channel_grouping_name,
  -- Other dimensions...
  
  COALESCE(f.forecast_value, 0) AS forecast_value,
  COALESCE(a.actual_value, 0) AS actual_value,
  
  CASE
    WHEN stages.stage = 'prospects' THEN COALESCE(a.actual_value, 0)
    WHEN stages.stage = 'mqls' THEN COALESCE(a.actual_value, 0) + COALESCE(fc.future_mqls, 0)
    -- Other stages...
  END AS predicted_value,
  
  hv.stddev_daily
```

### Logic Explanation:

Combines all components into final metrics.

**The Prediction Formula:**
```
Predicted Value = Current Actual + Expected Future Conversions
```

**COALESCE Cascade:**
Handles missing data gracefully by checking multiple sources in order. Essential because not all channel/source combinations exist in all CTEs.

**The Output:**
- **forecast_value**: Original target (what we hoped for)
- **actual_value**: Current reality (where we are)
- **predicted_value**: Expected outcome (where we'll likely end)
- **stddev_daily**: Volatility measure (how certain we are)

---

## Statistical Summary

This view implements sophisticated predictive analytics through:

1. **Dual-Track Data Strategy**
   - Historical accuracy (unfiltered actuals)
   - Realistic predictions (filtered rates)

2. **Cascade Modeling**
   - Multi-stage conversion waterfall
   - Compound probability calculations

3. **Uncertainty Quantification**
   - Standard deviation for volatility
   - Based on comprehensive historical data

The mathematics tell a story: "Here's what happened (actuals), here's what we wanted (forecast), and based on our current team's capabilities and pipeline, here's where we'll likely end up (predictions)."

Think of it like predicting a baseball team's season: You count all runs scored so far (even by traded players), but predict future performance based on the current roster's batting averages.

---

## Practical Implications

**For Leadership:**
- **Actuals** show true Q4 performance (accountability)
- **Predictions** show realistic end-of-quarter outcomes (planning)
- **Volatility** indicates confidence level (risk assessment)

**For RevOps:**
- Conversion rates based on performers still here (actionable)
- Open pipeline filtered to workable opportunities (realistic)
- Historical volatility for confidence intervals (honest uncertainty)

The dual-track approach ensures we're neither overly optimistic (counting on departed staff) nor artificially pessimistic (ignoring their past contributions). It's the statistical equivalent of "trust but verify" - acknowledging reality while planning pragmatically.