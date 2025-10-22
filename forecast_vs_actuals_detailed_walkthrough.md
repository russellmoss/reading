# Complete Walkthrough: vw_forecast_vs_actuals SQL Logic and Mathematics

**Document Purpose:** This guide provides a detailed, step-by-step explanation of the SQL code that powers the forecast vs actuals view, including the mathematical logic and statistical concepts behind each calculation.

**Target Audience:** RevOps professionals with basic statistics knowledge (undergraduate biology level)

**Last Updated:** October 22, 2025

---

## Table of Contents
1. [Overview: What This View Does](#overview)
2. [CTE 1: Funnel_With_Flags - Creating the Foundation](#cte1)
3. [CTE 2: Forecast_Data - Importing Our Targets](#cte2)
4. [CTE 3: Forecast_Q4_Total - Aggregating Quarterly Goals](#cte3)
5. [CTE 4: QTD_Actuals - Measuring Current Performance](#cte4)
6. [CTE 5: Trailing_Rates - The Heart of Prediction](#cte5)
7. [CTE 6: Open_Pipeline - What's In Progress](#cte6)
8. [CTE 7: Remaining_Forecast - Gap Analysis](#cte7)
9. [CTE 8: Future_Conversions - The Prediction Engine](#cte8)
10. [CTE 9: Historical_Volatility - Measuring Uncertainty](#cte9)
11. [Final SELECT - Bringing It All Together](#final-select)

---

## Overview: What This View Does {#overview}

This view answers three critical questions for Q4 2025:
1. **Where are we?** (Actuals)
2. **Where did we want to be?** (Forecast)
3. **Where will we likely end up?** (Predictions)

The magic lies in using historical conversion rates to predict how our current pipeline will mature by quarter's end.

---

## CTE 1: Funnel_With_Flags - Creating the Foundation {#cte1}

### Code:
```sql
WITH
Funnel_With_Flags AS (
  SELECT
    *,
    CASE WHEN LOWER(SQO_raw) = 'yes' THEN 1 ELSE 0 END AS is_sqo
  FROM `savvy-gtm-analytics.savvy_analytics.vw_funnel_lead_to_joined_v2`
),
```

### Logic Explanation:

This Common Table Expression (CTE) creates our foundational dataset by:

1. **Pulling all columns** from the base funnel view (`SELECT *`)
2. **Adding a calculated flag** for SQOs (Sales Qualified Opportunities)

**Why the CASE statement?**
- The source data stores SQO status as text ('yes'/'no')
- We convert it to binary (1/0) for easier mathematical operations
- `LOWER()` ensures we catch 'Yes', 'YES', 'yes' - all variations

**Statistical Significance:**
Binary flags (0/1) allow us to use `SUM()` to count occurrences and `AVG()` to calculate rates. This is a common technique in data science called "one-hot encoding."

---

## CTE 2: Forecast_Data - Importing Our Targets {#cte2}

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
),
```

### Logic Explanation:

This CTE standardizes and filters the forecast data that was set at the end of Q3.

**Key Transformations:**

1. **Channel Renaming:**
   ```sql
   CASE WHEN Channel = 'Inbound' THEN 'Marketing' ELSE Channel END
   ```
   - Historical naming inconsistency: forecast uses "Inbound", but everywhere else uses "Marketing"
   - This ensures consistency across all views

2. **Stage Pluralization:**
   ```sql
   WHEN LOWER(stage) = 'mql' THEN 'mqls'
   ```
   - Converts singular forms (mql) to plural (mqls) for consistency
   - Uses `LOWER()` to handle any capitalization variations

3. **Data Type Conversion:**
   ```sql
   CAST(forecast_value AS INT64)
   ```
   - Ensures numeric operations work correctly
   - INT64 chosen because we're counting discrete entities (leads, opportunities)

4. **Filtering Logic:**
   - `metric = 'Cohort_source'`: Selects source-level granularity (not channel rollups)
   - `original_source != 'All'`: Excludes pre-aggregated totals to avoid double-counting

---

## CTE 3: Forecast_Q4_Total - Aggregating Quarterly Goals {#cte3}

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
  GROUP BY 1, 2, 3
),
```

### Logic Explanation:

This CTE aggregates monthly forecasts into quarterly totals.

**Mathematical Operation:**
- Takes three monthly values (Oct, Nov, Dec) and sums them
- Example: If Oct MQL target = 140, Nov = 145, Dec = 145, then Q4 total = 430

**Why GROUP BY 1, 2, 3?**
- Groups by channel, source, and stage
- Creates one row per unique combination
- The numbers refer to column positions (cleaner than repeating column names)

**Statistical Context:**
This represents our null hypothesis in statistical terms - the expected outcome if everything goes according to plan.

---

## CTE 4: QTD_Actuals - Measuring Current Performance {#cte4}

### Code:
```sql
QTD_Actuals AS (
  SELECT channel_grouping_name, original_source, 'prospects' AS stage, 
         COUNT(DISTINCT Full_prospect_id__c) AS actual_value
  FROM Funnel_With_Flags
  WHERE DATE(CreatedDate) BETWEEN '2025-10-01' AND CURRENT_DATE()
  GROUP BY 1, 2
  
  UNION ALL
  
  SELECT channel_grouping_name, original_source, 'mqls' AS stage,
         COUNT(DISTINCT Full_prospect_id__c) AS actual_value
  FROM Funnel_With_Flags
  WHERE is_mql = 1 
    AND DATE(mql_stage_entered_ts) BETWEEN '2025-10-01' AND CURRENT_DATE()
  GROUP BY 1, 2
  
  -- Similar patterns for sqls, sqos, joined...
),
```

### Logic Explanation:

This CTE calculates quarter-to-date (QTD) actual performance for each funnel stage.

**Key Design Decisions:**

1. **COUNT(DISTINCT ...):**
   - Ensures we don't double-count if a record appears multiple times
   - Critical for data integrity

2. **Different ID Fields:**
   - Prospects/MQLs/SQLs use `Full_prospect_id__c` (lead-based)
   - SQOs/Joined use `Full_Opportunity_ID__c` (opportunity-based)
   - This reflects the Salesforce data model where leads convert to opportunities

3. **Date Range Logic:**
   ```sql
   BETWEEN '2025-10-01' AND CURRENT_DATE()
   ```
   - Captures everything from Q4 start through today
   - `CURRENT_DATE()` makes this dynamic - updates daily

4. **UNION ALL Pattern:**
   - Combines results from different stages into one dataset
   - Each SELECT becomes a set of rows in the final result
   - `UNION ALL` keeps all rows (vs `UNION` which would deduplicate)

**Statistical Importance:**
These are our observed values - the actual data points we'll compare against predictions.

---

## CTE 5: Trailing_Rates - The Heart of Prediction {#cte5}

### Code:
```sql
Trailing_Rates AS (
  SELECT
    channel_grouping_name,
    original_source,
    
    -- Prospect to MQL rate
    SAFE_DIVIDE(
      COUNT(DISTINCT IF(is_mql = 1, Full_prospect_id__c, NULL)), 
      COUNT(DISTINCT IF(is_contacted = 1, Full_prospect_id__c, NULL))
    ) AS prospect_to_mql_rate,
    
    -- MQL to SQL rate
    SAFE_DIVIDE(
      COUNT(DISTINCT IF(is_sql = 1, Full_prospect_id__c, NULL)), 
      COUNT(DISTINCT IF(is_mql = 1, Full_prospect_id__c, NULL))
    ) AS mql_to_sql_rate,
    
    -- SQL to SQO rate (fixed denominator)
    SAFE_DIVIDE(
      COUNT(DISTINCT CASE 
        WHEN is_sqo = 1 AND Full_Opportunity_ID__c IS NOT NULL 
        THEN Full_Opportunity_ID__c 
      END),
      COUNT(DISTINCT CASE 
        WHEN LOWER(SQO_raw) IN ('yes', 'no') AND Full_Opportunity_ID__c IS NOT NULL 
        THEN Full_Opportunity_ID__c 
      END)
    ) AS sql_to_sqo_rate,
    
    -- SQO to Joined rate
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
    
  FROM Funnel_With_Flags
  WHERE DATE(FilterDate) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) AND CURRENT_DATE()
  GROUP BY 1, 2
),
```

### Logic Explanation:

This CTE calculates conversion rates between funnel stages using the last 90 days of data.

**Statistical Concept: Conversion Rate Calculation**

The basic formula is:
```
Conversion Rate = (Count of Successful Outcomes) / (Count of Opportunities)
```

**Breaking Down Each Rate:**

1. **Prospect to MQL Rate:**
   ```sql
   COUNT(DISTINCT IF(is_mql = 1, Full_prospect_id__c, NULL)) / 
   COUNT(DISTINCT IF(is_contacted = 1, Full_prospect_id__c, NULL))
   ```
   - Numerator: How many became MQLs
   - Denominator: How many were contacted (had the chance to become MQL)
   - Example: 30 MQLs from 100 contacted = 30% conversion rate

2. **The IF() Trick:**
   ```sql
   IF(is_mql = 1, Full_prospect_id__c, NULL)
   ```
   - Returns the ID if condition is true, NULL otherwise
   - COUNT ignores NULLs, so this effectively filters and counts in one step
   - More efficient than a WHERE clause in this context

3. **SQL to SQO Rate - The Bug Fix:**
   ```sql
   COUNT(DISTINCT CASE WHEN LOWER(SQO_raw) IN ('yes', 'no') ... END)
   ```
   - Original bug: denominator included all SQLs
   - Problem: Some SQLs haven't been evaluated yet (SQO_raw is NULL)
   - Fix: Only count SQLs with a decision (yes OR no)
   - This gives us the true "decision rate" not biased by pending evaluations

4. **SAFE_DIVIDE Function:**
   - Prevents division by zero errors
   - Returns NULL instead of error if denominator is 0
   - Critical for robustness when some channels might have no activity

**Why 90 Days?**

This is a classic bias-variance tradeoff:
- **Too short (e.g., 30 days):** High variance, rates fluctuate wildly
- **Too long (e.g., 180 days):** High bias, slow to reflect improvements
- **90 days:** Optimal balance for B2B sales cycles

**Statistical Significance:**
With 90 days of data, we typically have enough sample size for the Central Limit Theorem to apply, meaning our rates approximate a normal distribution.

---

## CTE 6: Open_Pipeline - What's In Progress {#cte6}

### Code:
```sql
Open_Pipeline AS (
  SELECT
    channel_grouping_name,
    original_source,
    COUNT(DISTINCT IF(is_contacted = 1 AND is_mql = 0, Full_prospect_id__c, NULL)) AS open_prospects,
    COUNT(DISTINCT IF(is_mql = 1 AND is_sql = 0, Full_prospect_id__c, NULL)) AS open_mqls,
    COUNT(DISTINCT IF(is_sql = 1 AND is_sqo = 0 AND LOWER(SQO_raw) IS NULL, Full_Opportunity_ID__c, NULL)) AS open_sqls,
    COUNT(DISTINCT IF(is_sqo = 1 AND is_joined = 0, Full_Opportunity_ID__c, NULL)) AS open_sqos
  FROM Funnel_With_Flags
  WHERE DATE(FilterDate) BETWEEN '2025-10-01' AND CURRENT_DATE()
    AND (StageName IS NULL OR StageName NOT LIKE '%Closed Lost%')
  GROUP BY 1, 2
),
```

### Logic Explanation:

This CTE identifies records currently "in flight" - started but not yet completed their journey.

**The Logic of "Open" Records:**

1. **Open Prospects:**
   ```sql
   is_contacted = 1 AND is_mql = 0
   ```
   - We've reached out but they haven't scheduled a call yet
   - These have potential to become MQLs

2. **Open MQLs:**
   ```sql
   is_mql = 1 AND is_sql = 0
   ```
   - Qualified leads that haven't converted to opportunities
   - In the nurture phase

3. **Open SQLs:**
   ```sql
   is_sql = 1 AND is_sqo = 0 AND LOWER(SQO_raw) IS NULL
   ```
   - Converted to opportunity but not yet evaluated for SQO
   - The `IS NULL` check is crucial - excludes those with 'no' decisions

4. **Exclusion Filter:**
   ```sql
   AND (StageName IS NULL OR StageName NOT LIKE '%Closed Lost%')
   ```
   - Removes dead opportunities
   - Closed Lost = definitely won't convert

**Statistical Importance:**
These open records represent our "inventory" - they will convert at historical rates to generate future outcomes. This is similar to a Markov chain where entities move through states with known transition probabilities.

---

## CTE 7: Remaining_Forecast - Gap Analysis {#cte7}

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
),
```

### Logic Explanation:

This CTE calculates how much of our original forecast remains to be achieved.

**The Mathematics:**
```
Remaining = Original Target - What We've Achieved
```

**Key Functions:**

1. **GREATEST(0, ...):**
   - Ensures we never get negative values
   - If we've exceeded forecast, remaining = 0
   - Example: Target was 100, we have 120, remaining = 0 (not -20)

2. **COALESCE(a.actual_value, 0):**
   - Handles cases where we have no actuals yet (NULL)
   - Treats NULL as 0 for calculation purposes
   - Prevents NULL arithmetic (anything + NULL = NULL)

3. **LEFT JOIN Logic:**
   - Keeps all forecast rows even if no actuals exist
   - Important for new sources that haven't produced results yet

**Business Context:**
This tells us the "gap to goal" - critical for understanding how much ground we need to make up.

---

## CTE 8: Future_Conversions - The Prediction Engine {#cte8}

### Code:
```sql
Future_Conversions AS (
  SELECT
    COALESCE(p.channel_grouping_name, r.channel_grouping_name, rf_p.channel_grouping_name) AS channel_grouping_name,
    COALESCE(p.original_source, r.original_source, rf_p.original_source) AS original_source,
    
    -- Future MQLs = Open prospects converting + Remaining forecast prospects converting
    (COALESCE(p.open_prospects, 0) * COALESCE(r.prospect_to_mql_rate, 0)) + 
    (COALESCE(rf_p.remaining_value, 0) * COALESCE(r.prospect_to_mql_rate, 0)) AS future_mqls,
    
    -- Future SQLs = (Open MQLs + all future MQLs) * conversion rate
    ((COALESCE(p.open_mqls, 0) + 
      (COALESCE(p.open_prospects, 0) * COALESCE(r.prospect_to_mql_rate, 0)) + 
      (COALESCE(rf_p.remaining_value, 0) * COALESCE(r.prospect_to_mql_rate, 0)))
     * COALESCE(r.mql_to_sql_rate, 0)) AS future_sqls,
    
    -- Future SQOs = (Open SQLs + all future SQLs) * conversion rate
    ((COALESCE(p.open_sqls, 0) +
      ((COALESCE(p.open_mqls, 0) + 
        (COALESCE(p.open_prospects, 0) * COALESCE(r.prospect_to_mql_rate, 0)) + 
        (COALESCE(rf_p.remaining_value, 0) * COALESCE(r.prospect_to_mql_rate, 0)))
       * COALESCE(r.mql_to_sql_rate, 0)))
     * COALESCE(r.sql_to_sqo_rate, 0)) AS future_sqos,
    
    -- Future Joined = (Open SQOs + all future SQOs) * conversion rate
    ((COALESCE(p.open_sqos, 0) +
      ((COALESCE(p.open_sqls, 0) +
        ((COALESCE(p.open_mqls, 0) + 
          (COALESCE(p.open_prospects, 0) * COALESCE(r.prospect_to_mql_rate, 0)) + 
          (COALESCE(rf_p.remaining_value, 0) * COALESCE(r.prospect_to_mql_rate, 0)))
         * COALESCE(r.mql_to_sql_rate, 0)))
       * COALESCE(r.sql_to_sqo_rate, 0)))
     * COALESCE(r.sqo_to_joined_rate, 0)) AS future_joined
    
  FROM Open_Pipeline p
  FULL OUTER JOIN Trailing_Rates r
    ON p.channel_grouping_name = r.channel_grouping_name 
    AND p.original_source = r.original_source
  FULL OUTER JOIN (
    SELECT * FROM Remaining_Forecast WHERE stage = 'prospects'
  ) rf_p
    ON COALESCE(p.channel_grouping_name, r.channel_grouping_name) = rf_p.channel_grouping_name 
    AND COALESCE(p.original_source, r.original_source) = rf_p.original_source
),
```

### Logic Explanation:

This is the mathematical heart of our forecasting system - it predicts future outcomes using a cascade model.

**The Cascade Mathematics:**

Think of this like a waterfall where each pool feeds the next:

1. **Future MQLs Formula:**
   ```
   Future MQLs = (Open Prospects × Conversion Rate) + (New Prospects × Conversion Rate)
   ```
   
   Example with numbers:
   - 50 open prospects × 30% rate = 15 future MQLs
   - 200 remaining forecast × 30% rate = 60 future MQLs
   - Total = 75 future MQLs

2. **Future SQLs - The Cascade Begins:**
   ```
   Total Future MQL Pool = Open MQLs + Future MQLs
   Future SQLs = Total Future MQL Pool × MQL-to-SQL Rate
   ```
   
   Breaking it down:
   - We start with MQLs already in pipeline
   - Add MQLs we'll create (from step 1)
   - Multiply total by conversion rate

3. **Future SQOs - Double Cascade:**
   ```
   Total Future SQL Pool = Open SQLs + Future SQLs
   Future SQOs = Total Future SQL Pool × SQL-to-SQO Rate
   ```
   
   This is recursive - Future SQLs already includes the cascade from MQLs!

4. **Future Joined - Triple Cascade:**
   The most complex calculation - includes conversions from all previous stages

**Why FULL OUTER JOIN?**

```sql
FULL OUTER JOIN Trailing_Rates r
```

This ensures we capture:
- Sources with pipeline but no historical rates (new sources)
- Sources with rates but no current pipeline (dormant sources)
- Sources with both (normal case)

**Statistical Model:**

This implements a Markov chain model where:
- States = Funnel stages
- Transition probabilities = Conversion rates
- Future state distribution = Current state × Transition matrix

**The Power of COALESCE:**

```sql
COALESCE(p.open_prospects, 0) * COALESCE(r.prospect_to_mql_rate, 0)
```

Handles three edge cases elegantly:
1. No open prospects (NULL × rate = NULL, becomes 0 × rate = 0)
2. No historical rate (count × NULL = NULL, becomes count × 0 = 0)
3. Both exist (normal calculation)

---

## CTE 9: Historical_Volatility - Measuring Uncertainty {#cte9}

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
    FROM Funnel_With_Flags
    WHERE is_mql = 1 
      AND DATE(mql_stage_entered_ts) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) AND CURRENT_DATE()
    GROUP BY 1, 2, 3, 4
    
    UNION ALL
    -- Similar for other stages...
  )
  GROUP BY 1, 2, 3
)
```

### Logic Explanation:

This CTE calculates the standard deviation of daily performance - a measure of volatility.

**Statistical Concepts:**

1. **Standard Deviation (STDDEV):**
   - Measures spread of data around the mean
   - Formula: σ = √(Σ(x - μ)²/n)
   - In context: "How much does daily performance vary?"

2. **Why Daily Counts?**
   - We count events per day over 90 days
   - Creates ~90 data points per channel/source/stage
   - Enough for meaningful statistical analysis

3. **Interpretation:**
   - Low stddev (e.g., 2): Consistent daily performance
   - High stddev (e.g., 15): Volatile, unpredictable performance
   - Used later for confidence intervals

**Example Calculation:**
```
Daily MQLs over 5 days: [10, 12, 8, 11, 9]
Mean = 10
Deviations: [0, 2, -2, 1, -1]
Squared deviations: [0, 4, 4, 1, 1]
Variance = 10/5 = 2
Stddev = √2 = 1.41
```

This tells us daily MQL generation is fairly consistent (±1.41 MQLs typical variation).

---

## Final SELECT - Bringing It All Together {#final-select}

### Code:
```sql
SELECT
  COALESCE(a.channel_grouping_name, f.channel_grouping_name, fc.channel_grouping_name) AS channel_grouping_name,
  COALESCE(a.original_source, f.original_source, fc.original_source) AS original_source,
  stages.stage,
  COALESCE(f.forecast_value, 0) AS forecast_value,
  COALESCE(a.actual_value, 0) AS actual_value,
  
  CASE
    WHEN stages.stage = 'prospects' THEN COALESCE(a.actual_value, 0)
    WHEN stages.stage = 'mqls' THEN COALESCE(a.actual_value, 0) + COALESCE(fc.future_mqls, 0)
    WHEN stages.stage = 'sqls' THEN COALESCE(a.actual_value, 0) + COALESCE(fc.future_sqls, 0)
    WHEN stages.stage = 'sqos' THEN COALESCE(a.actual_value, 0) + COALESCE(fc.future_sqos, 0)
    WHEN stages.stage = 'joined' THEN COALESCE(a.actual_value, 0) + COALESCE(fc.future_joined, 0)
    ELSE COALESCE(a.actual_value, 0)
  END AS predicted_value,
  
  hv.stddev_daily
  
FROM (
  SELECT DISTINCT channel_grouping_name, original_source, stage 
  FROM QTD_Actuals
  UNION DISTINCT
  SELECT DISTINCT channel_grouping_name, original_source, stage 
  FROM Forecast_Q4_Total
) stages
LEFT JOIN QTD_Actuals a ON ...
LEFT JOIN Forecast_Q4_Total f ON ...
LEFT JOIN Future_Conversions fc ON ...
LEFT JOIN Historical_Volatility hv ON ...
```

### Logic Explanation:

The final SELECT assembles all components into the output table.

**Key Design Patterns:**

1. **The Stages Subquery:**
   ```sql
   FROM (
     SELECT DISTINCT ... FROM QTD_Actuals
     UNION DISTINCT
     SELECT DISTINCT ... FROM Forecast_Q4_Total
   ) stages
   ```
   - Creates a complete list of all channel/source/stage combinations
   - Ensures we have rows even if missing actuals OR forecast
   - Acts as the "spine" of our result set

2. **Multiple COALESCE for Robustness:**
   ```sql
   COALESCE(a.channel_grouping_name, f.channel_grouping_name, fc.channel_grouping_name)
   ```
   - Tries each source in order until finding non-NULL
   - Ensures we always have identifying information

3. **The Prediction Formula:**
   ```sql
   COALESCE(a.actual_value, 0) + COALESCE(fc.future_mqls, 0)
   ```
   
   **Simple but powerful:**
   - End-of-Quarter Prediction = Current Actuals + Expected Future Conversions
   
   **Why prospects are different:**
   ```sql
   WHEN stages.stage = 'prospects' THEN COALESCE(a.actual_value, 0)
   ```
   - Prospects don't have "future" conversions to prospect stage
   - They ARE the starting point

4. **LEFT JOIN Strategy:**
   - Keeps all rows from stages (our spine)
   - Attaches data where available
   - NULLs where data missing (handled by COALESCE)

**Final Output Structure:**

Each row contains:
- **Identifiers:** channel, source, stage
- **Three Key Metrics:**
  - `forecast_value`: Original Q4 target
  - `actual_value`: Current achievement
  - `predicted_value`: Expected end-of-quarter result
- **Uncertainty Measure:** `stddev_daily` for confidence intervals

---

## Statistical Summary

This view implements several statistical concepts:

1. **Conversion Rate Analysis:** Using historical probabilities to predict future outcomes
2. **Cascade/Markov Model:** Each stage feeds the next with known transition probabilities  
3. **Volatility Measurement:** Standard deviation quantifies uncertainty
4. **Robust Estimation:** COALESCE and SAFE_DIVIDE handle edge cases gracefully

The mathematics can be summarized as:

```
Prediction = Current_Actuals + (Open_Pipeline × Historical_Conversion_Rates) + 
             (Remaining_Forecast × Historical_Full_Funnel_Rate)
```

This provides a statistically grounded forecast that gets more accurate as the quarter progresses and uncertainty decreases.

---

## Practical Application

**For RevOps Teams:**
- Monitor `predicted_value` vs `forecast_value` to see if you'll hit targets
- Use `stddev_daily` to understand confidence in predictions
- Track conversion rates over time to spot improvements or degradations

**For Leadership:**
- Simple story: "Based on current pipeline and historical performance, here's where we'll end the quarter"
- Confidence intervals (using stddev) answer "What's our worst/best case scenario?"

**For Data Teams:**
- This pattern (actuals + pipeline × rates) is reusable for any funnel
- The cascade model can extend to revenue predictions
- Historical volatility provides input for Monte Carlo simulations

---

## Conclusion

This SQL view elegantly combines current performance, historical patterns, and pipeline dynamics to create a sophisticated yet interpretable forecasting system. The mathematical foundation ensures predictions improve as more data becomes available, while the statistical measures provide appropriate confidence bounds for decision-making.

The key innovation is the cascade calculation that properly accounts for how each funnel stage feeds the next, creating a comprehensive picture of expected end-of-quarter performance based on rigorous statistical analysis rather than wishful thinking.