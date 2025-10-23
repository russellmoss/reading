# Complete Walkthrough: vw_timeseries_with_confidence SQL Logic and Mathematics

**Document Purpose:** This guide provides a detailed, step-by-step explanation of the SQL code that powers the time series visualization with confidence intervals, including the mathematical logic and statistical concepts behind each calculation.

**Target Audience:** RevOps professionals with basic statistics knowledge 
**Last Updated:** October 22, 2025

---

## Table of Contents
1. [Overview: Creating Time Series Predictions](#overview)
2. [CTE 1: Date_Spine - Building Our Timeline](#cte1)
3. [CTE 2: Funnel_With_Flags - Foundation Data](#cte2)
4. [CTE 3: Daily_Actuals - What Happened Each Day](#cte3)
5. [CTE 4: Cumulative_Actuals - Building the Growth Curve](#cte4)
6. [CTE 5: Monthly_Forecast_Targets - Stepped Goals with Deduplication](#cte5)
7. [CTE 6: Cumulative_Monthly_Targets - Progressive Targets](#cte6)
8. [CTE 7: Forecast_Data - Importing Predictions](#cte7)
9. [CTE 8: Current_Actuals - Today's Snapshot](#cte8)
10. [CTE 9: Historical_Daily_Volatility - Measuring Uncertainty](#cte9)
11. [CTE 10: Combined_Data - The Prediction Engine](#cte10)
12. [Final SELECT - Creating the Visualization Format](#final-select)

---

## Overview: Creating Time Series Predictions {#overview}

This view transforms point-in-time predictions into smooth daily projections with statistical confidence intervals. It answers:

1. **Historical Performance:** How have we performed each day so far?
2. **Future Trajectory:** What's our expected path to end of quarter?
3. **Uncertainty Bounds:** What's the range of likely outcomes?
4. **Target Progression:** How should we be tracking against monthly milestones?

The magic lies in linear interpolation combined with statistical confidence intervals that widen over time, plus ensuring all forecasted sources appear even if they have zero actuals.

---

## CTE 1: Date_Spine - Building Our Timeline {#cte1}

### Code:
```sql
Date_Spine AS (
  SELECT date_day
  FROM UNNEST(GENERATE_DATE_ARRAY('2025-10-01', '2025-12-31', INTERVAL 1 DAY)) AS date_day
),
```

### Logic Explanation:

This CTE creates the foundation - one row for every day in Q4 2025.

**The GENERATE_DATE_ARRAY Function:**
- Creates an array of dates from start to end
- `INTERVAL 1 DAY` means include every single day
- Generates: ['2025-10-01', '2025-10-02', ..., '2025-12-31']

**The UNNEST Function:**
- Converts the array into rows
- Transforms: ['date1', 'date2'] → separate rows
- Result: 92 rows (Oct: 31 + Nov: 30 + Dec: 31)

**Why This Matters:**
- Time series charts need continuous x-axis
- Even days with zero activity need a row
- This ensures smooth lines without gaps

---

## CTE 2: Funnel_With_Flags - Foundation Data {#cte2}

### Code:
```sql
Funnel_With_Flags AS (
  SELECT
    *,
    CASE WHEN LOWER(SQO_raw) = 'yes' THEN 1 ELSE 0 END AS is_sqo
  FROM `savvy-gtm-analytics.savvy_analytics.vw_funnel_lead_to_joined_v2`
),
```

### Logic Explanation:

Creates binary flags for mathematical operations from the base funnel view.

**The SQO Flag Creation:**
```sql
CASE WHEN LOWER(SQO_raw) = 'yes' THEN 1 ELSE 0 END AS is_sqo
```
- Converts text ('yes'/'no') to binary (1/0)
- `LOWER()` handles case variations ('Yes', 'YES', 'yes')
- Binary encoding enables `SUM()` for counting

**Note:** This pulls ALL data, not filtered by active SGAs like some other views. This means we're showing the complete historical picture.

---

## CTE 3: Daily_Actuals - What Happened Each Day {#cte3}

### Code:
```sql
Daily_Actuals AS (
  -- MQLs
  SELECT DATE(mql_stage_entered_ts) AS date_day, Channel_Grouping_Name, Original_source, 
         'mqls' AS stage, COUNT(DISTINCT Full_prospect_id__c) AS daily_count
  FROM Funnel_With_Flags
  WHERE is_mql = 1 AND DATE(mql_stage_entered_ts) BETWEEN '2025-10-01' AND '2025-12-31'
  GROUP BY 1, 2, 3
  
  UNION ALL
  
  -- SQLs (similar pattern)
  -- SQOs (similar pattern)  
  -- Joined (similar pattern)
),
```

### Logic Explanation:

This CTE calculates how many conversions happened on each specific day.

**Key Design Patterns:**

1. **Different Date Fields per Stage:**
   - MQLs: `mql_stage_entered_ts` (when they scheduled a call)
   - SQLs: `converted_date_raw` (when lead became opportunity)
   - SQOs: `Date_Became_SQO__c` (when qualified)
   - Joined: `advisor_join_date__c` (when signed)

2. **DATE() Function:**
   ```sql
   DATE(mql_stage_entered_ts)
   ```
   - Converts timestamp to date (strips time component)
   - Example: '2025-10-15 14:23:45' → '2025-10-15'

3. **COUNT(DISTINCT ...) Pattern:**
   - Ensures no double-counting
   - Critical for data integrity
   - Different ID fields: leads use `Full_prospect_id__c`, opportunities use `Full_Opportunity_ID__c`

**What This Produces:**
```
date_day    | channel   | source    | stage | daily_count
2025-10-01  | Marketing | Webinar   | mqls  | 5
2025-10-01  | Outbound  | LinkedIn  | mqls  | 3
2025-10-02  | Marketing | Webinar   | mqls  | 7
```

These are our observed daily frequencies - the raw data points that we'll accumulate to create cumulative growth curves.

---

## CTE 4: Cumulative_Actuals - Building the Growth Curve {#cte4}

### Code:
```sql
Cumulative_Actuals AS (
  SELECT 
    d.date_day,
    dims.Channel_Grouping_Name AS channel_grouping_name,
    dims.Original_source AS original_source,
    dims.stage,
    SUM(SUM(COALESCE(a.daily_count, 0))) OVER (
      PARTITION BY dims.Channel_Grouping_Name, dims.Original_source, dims.stage 
      ORDER BY d.date_day
    ) AS cumulative_actual
  FROM Date_Spine d
  CROSS JOIN (
    -- Include sources that have actuals
    SELECT DISTINCT Channel_Grouping_Name, Original_source, stage 
    FROM Daily_Actuals
    
    UNION DISTINCT
    
    -- Also include ALL sources from forecast (even with zero actuals)
    SELECT DISTINCT
      CASE WHEN Channel = 'Inbound' THEN 'Marketing' ELSE Channel END AS Channel_Grouping_Name,
      original_source AS Original_source,
      CASE
        WHEN LOWER(stage) = 'mql' THEN 'mqls'
        WHEN LOWER(stage) = 'sql' THEN 'sqls'
        WHEN LOWER(stage) = 'sqo' THEN 'sqos'
        WHEN LOWER(stage) = 'joined' THEN 'joined'
        ELSE LOWER(stage)
      END AS stage
    FROM `savvy-gtm-analytics.SavvyGTMData.q4_2025_forecast`
    WHERE metric = 'Cohort_source'
      AND original_source != 'All'
      AND month_key IN ('2025-10', '2025-11', '2025-12')
  ) dims
  LEFT JOIN Daily_Actuals a
    ON d.date_day = a.date_day
    AND dims.Channel_Grouping_Name = a.Channel_Grouping_Name
    AND dims.Original_source = a.Original_source
    AND dims.stage = a.stage
  GROUP BY 1, 2, 3, 4
),
```

### Logic Explanation:

This CTE creates cumulative totals with a **critical enhancement** - it ensures all forecasted sources appear even if they have zero actuals.

**The Enhanced dims Subquery:**

The subquery now has two parts connected by `UNION DISTINCT`:

1. **Part 1: Sources with Actual Activity**
   ```sql
   SELECT DISTINCT Channel_Grouping_Name, Original_source, stage 
   FROM Daily_Actuals
   ```
   - Gets all combinations that have had at least one conversion

2. **Part 2: All Forecasted Sources (NEW!)**
   ```sql
   SELECT DISTINCT
     CASE WHEN Channel = 'Inbound' THEN 'Marketing' ELSE Channel END AS Channel_Grouping_Name,
     original_source AS Original_source,
     [stage transformations]
   FROM `savvy-gtm-analytics.SavvyGTMData.q4_2025_forecast`
   ```
   - Pulls ALL sources from the forecast
   - Applies same naming transformations (Inbound→Marketing, mql→mqls)
   - Ensures we track sources that were forecasted but haven't produced results yet

**Why This Enhancement Matters:**

Consider this scenario:
- Forecast includes "Partner Referral" source expecting 10 MQLs
- So far, zero Partner Referrals have come in
- Without Part 2: This source wouldn't appear in the chart at all
- With Part 2: Shows a flat line at zero with target line above it

This creates a complete picture showing both performing and underperforming sources.

**The Window Function for Cumulative Totals:**
```sql
SUM(SUM(COALESCE(a.daily_count, 0))) OVER (
  PARTITION BY dims.Channel_Grouping_Name, dims.Original_source, dims.stage 
  ORDER BY d.date_day
)
```

**Example Calculation:**
```
Day 1: 5 MQLs → Cumulative: 5
Day 2: 3 MQLs → Cumulative: 8 (5+3)
Day 3: 0 MQLs → Cumulative: 8 (5+3+0)
Day 4: 6 MQLs → Cumulative: 14 (5+3+0+6)
```

For sources with no activity, the cumulative stays at 0 throughout.

---

## CTE 5: Monthly_Forecast_Targets - Stepped Goals with Deduplication {#cte5}

### Code:
```sql
Monthly_Forecast_Targets AS (
  SELECT
    CASE WHEN Channel = 'Inbound' THEN 'Marketing' ELSE Channel END AS channel_grouping_name,
    original_source,
    CASE
      WHEN LOWER(stage) = 'mql' THEN 'mqls'
      WHEN LOWER(stage) = 'sql' THEN 'sqls'
      WHEN LOWER(stage) = 'sqo' THEN 'sqos'
      ELSE LOWER(stage)
    END AS stage,
    month_key,
    SUM(CAST(forecast_value AS INT64)) AS monthly_forecast  -- SUM to handle duplicates
  FROM `savvy-gtm-analytics.SavvyGTMData.q4_2025_forecast`
  WHERE metric = 'Cohort_source'
    AND original_source != 'All'
    AND month_key IN ('2025-10', '2025-11', '2025-12')
  GROUP BY 1, 2, 3, 4  -- GROUP BY to deduplicate
),
```

### Logic Explanation:

This CTE imports monthly targets with **deduplication logic** to handle potential duplicate rows in the forecast table.

**The Deduplication Strategy:**

1. **SUM() Instead of Direct Selection:**
   ```sql
   SUM(CAST(forecast_value AS INT64)) AS monthly_forecast
   ```
   - If there's only one row: SUM = that value
   - If there are duplicates: SUM combines them
   - Assumes duplicates should be additive (not alternatives)

2. **GROUP BY All Dimensions:**
   ```sql
   GROUP BY 1, 2, 3, 4
   ```
   - Groups by channel, source, stage, and month
   - Collapses any duplicate rows into one
   - Ensures one row per unique combination

**Why Duplicates Might Exist:**
- Data entry errors in the forecast table
- Multiple forecast versions loaded
- Different business units submitting overlapping forecasts
- ETL process issues

**Example of Handling Duplicates:**
```
Before GROUP BY:
Marketing | Webinar | mqls | 2025-10 | 70
Marketing | Webinar | mqls | 2025-10 | 70  (duplicate)

After GROUP BY with SUM:
Marketing | Webinar | mqls | 2025-10 | 140
```

**Important Decision:** The choice to SUM duplicates assumes they represent different parts of the same forecast. If duplicates are errors, this could double-count. The business logic needs to determine if this is correct.

---

## CTE 6: Cumulative_Monthly_Targets - Progressive Targets {#cte6}

### Code:
```sql
-- 6. Calculate cumulative monthly targets (NO ROLLUP - just source level)
Cumulative_Monthly_Targets AS (
  SELECT
    channel_grouping_name,
    original_source,
    stage,
    month_key,
    monthly_forecast,
    SUM(monthly_forecast) OVER (
      PARTITION BY channel_grouping_name, original_source, stage 
      ORDER BY month_key
    ) AS cumulative_target
  FROM Monthly_Forecast_Targets
),
```

### Logic Explanation:

This CTE converts monthly targets into cumulative targets for the stepped line.

**The "NO ROLLUP" Comment Significance:**
- This view works at the source level (e.g., "Webinar", "LinkedIn")
- Does NOT create channel-level aggregates (e.g., all "Marketing" combined)
- Each source gets its own target line
- This is different from some dashboard views that show rolled-up totals

**The Window Function:**
```sql
SUM(monthly_forecast) OVER (
  PARTITION BY channel_grouping_name, original_source, stage 
  ORDER BY month_key
)
```

**Example Calculation for One Source:**
```
October target: 140 → Cumulative by Oct 31: 140
November target: 145 → Cumulative by Nov 30: 285 (140+145)
December target: 145 → Cumulative by Dec 31: 430 (140+145+145)
```

**Visual Result:**
Instead of three separate monthly bars, we get a line that steps up at month boundaries, showing expected progression through the quarter at the individual source level.

---

## CTE 7: Forecast_Data - Importing Predictions {#cte7}

### Code:
```sql
Forecast_Data AS (
  SELECT 
    channel_grouping_name,
    original_source,
    stage,
    forecast_value,
    predicted_value,
    stddev_daily
  FROM `savvy-gtm-analytics.savvy_analytics.vw_forecast_vs_actuals`
),
```

### Logic Explanation:

This CTE imports the sophisticated predictions from our main forecast view.

**What We're Getting:**
- `forecast_value`: Original Q4 target (e.g., 430 MQLs for a source)
- `predicted_value`: Where we'll likely end up (e.g., 580 MQLs)
- `stddev_daily`: Daily volatility for confidence intervals

**Why Import Instead of Recalculate?**
- The forecast_vs_actuals view contains complex cascade logic
- Includes conversion rate calculations
- Accounts for open pipeline
- Single source of truth principle

**This is Our Endpoint:**
The `predicted_value` tells us where we expect to be on December 31st. Our job now is to create a smooth line from today's actual to that endpoint.

---

## CTE 8: Current_Actuals - Today's Snapshot {#cte8}

### Code:
```sql
Current_Actuals AS (
  SELECT
    channel_grouping_name,
    original_source,
    stage,
    cumulative_actual AS current_actual
  FROM Cumulative_Actuals
  WHERE date_day = CURRENT_DATE()
),
```

### Logic Explanation:

This CTE captures our starting point for predictions - where we are today.

**Simple but Critical:**
- Filters cumulative actuals to just today's date
- Provides the anchor point for linear interpolation
- This is the "You are here" marker on the map

**For Zero-Activity Sources:**
- Sources with no actuals will show 0 as current_actual
- This is important for the forecast sources added in CTE 4
- Their prediction lines will start from 0 and rise to predicted value

---

## CTE 9: Historical_Daily_Volatility - Measuring Uncertainty {#cte9}

### Code:
```sql
Historical_Daily_Volatility AS (
  SELECT
    channel_grouping_name,
    original_source,
    stage,
    STDDEV(daily_growth) AS stddev_daily_growth
  FROM (
    SELECT 
      channel_grouping_name,
      original_source,
      stage,
      cumulative_actual - LAG(cumulative_actual) OVER (
        PARTITION BY channel_grouping_name, original_source, stage 
        ORDER BY date_day
      ) AS daily_growth
    FROM Cumulative_Actuals
    WHERE date_day <= CURRENT_DATE()
  )
  WHERE daily_growth IS NOT NULL
  GROUP BY 1, 2, 3
),
```

### Logic Explanation:

This CTE calculates how much daily growth varies - crucial for confidence intervals.

**The LAG Function:**
```sql
cumulative_actual - LAG(cumulative_actual) OVER (...)
```
- LAG gets the previous row's value
- Subtraction gives us daily increment
- Example: Today's 166 - Yesterday's 161 = 5 daily growth

**Two-Step Process:**

1. **Inner Query - Calculate Daily Growth:**
   ```
   Date      | Cumulative | LAG    | Daily Growth
   Oct 1     | 5          | NULL   | NULL
   Oct 2     | 8          | 5      | 3
   Oct 3     | 8          | 8      | 0
   Oct 4     | 14         | 8      | 6
   ```

2. **Outer Query - Calculate Standard Deviation:**
   ```
   STDDEV([3, 0, 6, 2, 4, ...]) = 2.1
   ```

**For Zero-Activity Sources:**
- If a source has no activity, all daily_growth = 0
- STDDEV(0, 0, 0, ...) = 0
- Results in no confidence interval (flat line)
- This is mathematically correct - no volatility means complete certainty

**Statistical Interpretation:**
If stddev = 2.1 MQLs/day:
- 68% of days will see growth within ±2.1 of average
- 95% of days will see growth within ±4.2 of average
- This quantifies our uncertainty about daily progress

---

## CTE 10: Combined_Data - The Prediction Engine {#cte10}

### Code:
```sql
Combined_Data AS (
  SELECT
    a.date_day,
    a.channel_grouping_name,
    a.original_source,
    a.stage,
    
    -- Actual value (only up to today)
    CASE 
      WHEN a.date_day <= CURRENT_DATE() THEN a.cumulative_actual
      ELSE NULL
    END AS actual_value,
    
    -- Predicted value (linear interpolation from today to end of quarter)
    CASE
      WHEN a.date_day <= CURRENT_DATE() THEN a.cumulative_actual
      ELSE 
        COALESCE(c.current_actual, 0) + 
        (DATE_DIFF(a.date_day, CURRENT_DATE(), DAY) * 
         SAFE_DIVIDE(
           COALESCE(f.predicted_value, 0) - COALESCE(c.current_actual, 0), 
           DATE_DIFF(DATE('2025-12-31'), CURRENT_DATE(), DAY)
         ))
    END AS predicted_value,
    
    -- [Confidence intervals and target calculations continue...]
```

### Logic Explanation:

This is the mathematical heart - creating smooth predictions and confidence intervals.

**Linear Interpolation Formula:**

```
Value(future_date) = Current + (Days_from_today × Daily_rate)

Where Daily_rate = (End_goal - Current) / Days_remaining
```

**Breaking Down the Calculation:**

1. **Days from today to future date:**
   ```sql
   DATE_DIFF(a.date_day, CURRENT_DATE(), DAY)
   ```
   Example: Nov 1 - Oct 21 = 11 days

2. **Required daily growth rate:**
   ```sql
   SAFE_DIVIDE(
     COALESCE(f.predicted_value, 0) - COALESCE(c.current_actual, 0), 
     DATE_DIFF(DATE('2025-12-31'), CURRENT_DATE(), DAY)
   )
   ```
   Example: (580 end goal - 166 current) / 71 days remaining = 5.83 MQLs/day

3. **Predicted value:**
   ```
   166 + (11 days × 5.83/day) = 166 + 64 = 230 MQLs on Nov 1
   ```

**For Zero-Activity Sources:**
- Current = 0, Predicted = 100 (example)
- Daily rate = 100 / 71 = 1.41 per day
- Creates line from 0 rising to 100 by Dec 31
- Shows the gap between expectation and reality

### Confidence Interval Calculation:

```sql
-- Lower bound
GREATEST(
  COALESCE(c.current_actual, 0),
  [central_prediction] - (1.96 * COALESCE(v.stddev_daily_growth, 0) * SQRT(DATE_DIFF(a.date_day, CURRENT_DATE(), DAY)))
)
```

**The Statistics Behind Confidence Intervals:**

1. **The 1.96 Factor:**
   - From normal distribution theory
   - 1.96 × σ captures 95% of probability mass
   - Industry standard for 95% confidence intervals

2. **The Square Root of Days:**
   ```sql
   SQRT(DATE_DIFF(a.date_day, CURRENT_DATE(), DAY))
   ```
   
   **Why square root?**
   - Uncertainty accumulates via "random walk" principle
   - Variance adds linearly: Var(Day_30) = 30 × Var(Day_1)
   - Standard deviation is √Variance
   - Therefore: StdDev(Day_30) = √30 × StdDev(Day_1)

3. **The GREATEST Function:**
   - Ensures lower bound never goes below current actual
   - Prevents impossible scenarios (can't have fewer than we already have)

**For Zero-Volatility Sources:**
- If stddev = 0 (no historical volatility)
- Confidence intervals = prediction line (no spread)
- Indicates complete consistency (or no activity)

### Target Line Calculation:

The target calculation creates a stepped progression that changes slope at month boundaries:

```sql
CASE 
  -- October: Linear from 0 to Oct target
  WHEN a.date_day <= DATE('2025-10-31') THEN
    [oct_cumulative_target] * DATE_DIFF(a.date_day, DATE('2025-09-30'), DAY) / 31.0
```

**Stepped Progression Logic:**

Instead of one straight line from 0 to 430, we create three segments:

1. **October segment:** 
   - Start: 0 on Sep 30
   - End: 140 on Oct 31
   - Daily rate: 140/31 = 4.52/day

2. **November segment:**
   - Start: 140 on Oct 31
   - End: 285 on Nov 30
   - Daily rate: 145/30 = 4.83/day

3. **December segment:**
   - Start: 285 on Nov 30
   - End: 430 on Dec 31
   - Daily rate: 145/31 = 4.68/day

This creates a more realistic target line that reflects monthly quotas and business cycles.

---

## Final SELECT - Creating the Visualization Format {#final-select}

### Code:
```sql
SELECT
  date_day,
  channel_grouping_name,
  original_source,
  
  -- MQL metrics
  MAX(CASE WHEN stage = 'mqls' THEN actual_value END) AS mqls_actual,
  MAX(CASE WHEN stage = 'mqls' THEN predicted_value END) AS mqls_predicted,
  MAX(CASE WHEN stage = 'mqls' THEN predicted_lower END) AS mqls_lower,
  MAX(CASE WHEN stage = 'mqls' THEN predicted_upper END) AS mqls_upper,
  MAX(CASE WHEN stage = 'mqls' THEN target_value END) AS mqls_target,
  
  -- [Repeated for SQL, SQO, Joined stages...]
  
FROM Combined_Data
GROUP BY 1, 2, 3
ORDER BY 1, 2, 3
```

### Logic Explanation:

The final SELECT pivots the data from long to wide format for visualization.

**The Pivot Pattern:**

**From (Long Format):**
```
date      | channel | source | stage | actual | predicted
Oct-21    | Market  | Web    | mqls  | 50     | 50
Oct-21    | Market  | Web    | sqls  | 25     | 25
Oct-21    | Market  | Web    | sqos  | 15     | 15
```

**To (Wide Format):**
```
date   | channel | source | mqls_actual | mqls_pred | sqls_actual | sqls_pred
Oct-21 | Market  | Web    | 50          | 50        | 25          | 25
```

**Why MAX() with CASE?**
```sql
MAX(CASE WHEN stage = 'mqls' THEN actual_value END)
```

The MAX() aggregation:
- Ignores NULLs from non-matching stages
- Returns the one non-NULL value per group
- Could use MIN() or AVG() with same result

**The 20-Column Output:**

For each stage (MQLs, SQLs, SQOs, Joined):
1. `{stage}_actual` - Historical performance
2. `{stage}_predicted` - Central forecast line
3. `{stage}_lower` - Lower confidence bound
4. `{stage}_upper` - Upper confidence bound
5. `{stage}_target` - Original goal line

5 metrics × 4 stages = 20 columns

**Handling Zero-Activity Sources:**

For sources that appear in forecast but have no activity:
- All `actual` columns will be NULL or 0
- `predicted` columns show line from 0 to predicted value
- `target` columns show the stepped monthly targets
- Creates visual gap highlighting underperformance

---

## Statistical Summary

This view implements several advanced statistical and mathematical concepts:

### 1. **Linear Interpolation**
Creates smooth transitions from current state to predicted endpoint:
```
y = y₀ + (x - x₀) × (y₁ - y₀)/(x₁ - x₀)
```

### 2. **Confidence Intervals via Random Walk Theory**
Uncertainty grows with square root of time:
```
CI(t) = μ ± 1.96 × σ × √t
```

### 3. **Complete Source Coverage**
The enhancement in CTE 4 ensures all forecasted sources appear, even with zero activity, providing a complete picture of performance vs. expectations.

### 4. **Deduplication Logic**
The SUM/GROUP BY pattern in CTE 5 handles potential data quality issues in the forecast table.

### 5. **Piecewise Linear Functions**
Stepped targets that change slope at month boundaries:
```
f(x) = { a₁x + b₁ if x ∈ [t₀,t₁]
       { a₂x + b₂ if x ∈ [t₁,t₂]
       { a₃x + b₃ if x ∈ [t₂,t₃]
```

---

## Practical Applications

### For RevOps Teams:

**Reading the Visualization:**
- **Blue solid line** (actual): Performance through today
- **Gray dashed line** (predicted): Expected trajectory
- **Gray shaded area** (confidence): Range of likely outcomes
- **Green line** (target): Original plan
- **Flat lines at zero**: Sources that haven't delivered yet

**Key Insights:**
- Sources with no activity but targets show immediate problems
- Widening confidence cone = increasing uncertainty over time
- Zero-volatility sources = highly predictable (or inactive)

### For Leadership:

**Enhanced Story:**
"This shows ALL sources we're tracking, including those that haven't produced results yet. The gaps highlight where we need intervention."

**Decision Points:**
- Zero-activity sources with high targets = immediate action needed
- Narrow confidence bands = high predictability
- Wide bands = need to understand volatility drivers

### For Data Teams:

**Data Quality Benefits:**
- Deduplication logic handles forecast table issues
- Complete source coverage prevents "missing" sources
- Robust to various data scenarios

**Monitoring Points:**
- Check for duplicate forecast entries
- Verify all forecasted sources appear
- Monitor sources with zero activity

---

## Common Issues and Solutions

### Issue: Some sources don't appear in charts

**Solution:** The enhanced CTE 4 now includes all forecasted sources, even with zero activity

### Issue: Forecast values seem doubled

**Cause:** Duplicate entries in forecast table

**Solution:** CTE 5 now includes SUM/GROUP BY to handle duplicates

### Issue: Confidence intervals missing for some sources

**Cause:** Zero historical activity means zero volatility

**Solution:** This is correct behavior - no uncertainty if no activity

### Issue: Target lines don't match expectations

**Check:** Monthly forecast values and deduplication logic

---

## Conclusion

This SQL view transforms point predictions into sophisticated time series visualizations with statistical confidence bounds. The key enhancements ensure:

1. **Complete Coverage:** All forecasted sources appear, highlighting gaps
2. **Data Quality:** Deduplication handles forecast table issues
3. **Statistical Rigor:** Confidence intervals based on actual volatility
4. **Business Alignment:** Stepped targets reflect monthly planning cycles

The mathematical foundation combines linear interpolation, random walk theory, and robust data handling to create actionable insights. The widening confidence cone elegantly captures increasing uncertainty over time, while zero-activity sources clearly show where intervention is needed.

The implementation demonstrates how SQL's analytical capabilities can handle complex statistical models while remaining resilient to data quality issues, creating visualizations that drive better business decisions through complete transparency of both successes and gaps.