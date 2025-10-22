# Complete Walkthrough: vw_timeseries_with_confidence_current SQL Logic and Mathematics

**Document Purpose:** This guide provides a detailed, step-by-step explanation of the SQL code that powers the time series visualization with confidence intervals, including the mathematical logic and statistical concepts behind each calculation.

**Target Audience:** RevOps professionals with basic statistics knowledge (undergraduate biology level)

**Last Updated:** October 22, 2025

---

## Table of Contents
1. [Overview: Creating Time Series Predictions](#overview)
2. [CTE 1: Date_Spine - Building Our Timeline](#cte1)
3. [CTE 2: Funnel_With_Flags - Foundation Data](#cte2)
4. [CTE 3: Daily_Actuals - What Happened Each Day](#cte3)
5. [CTE 4: Cumulative_Actuals - Building the Growth Curve](#cte4)
6. [CTE 5: Monthly_Forecast_Targets - Stepped Goals](#cte5)
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

The magic lies in linear interpolation combined with statistical confidence intervals that widen over time.

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

**Statistical Context:**
This creates our temporal domain - the continuous timeline over which we'll interpolate values. In time series analysis, having complete temporal coverage is crucial for proper visualization and preventing misleading gaps.

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

Identical to the forecast_vs_actuals view - creates binary flags for mathematical operations.

**Reusing the Pattern:**
- Same base data as main forecast view
- Ensures consistency across all analytics
- Binary encoding enables SUM() counting

**Note:** This is the raw funnel data before any time-based aggregation. Each row represents a lead or opportunity at a point in time.

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
  
  -- SQLs
  SELECT DATE(converted_date_raw) AS date_day, Channel_Grouping_Name, Original_source,
         'sqls' AS stage, COUNT(DISTINCT Full_prospect_id__c) AS daily_count
  FROM Funnel_With_Flags
  WHERE is_sql = 1 AND DATE(converted_date_raw) BETWEEN '2025-10-01' AND '2025-12-31'
  GROUP BY 1, 2, 3
  
  UNION ALL
  
  -- SQOs
  SELECT DATE(Date_Became_SQO__c) AS date_day, Channel_Grouping_Name, Original_source,
         'sqos' AS stage, COUNT(DISTINCT Full_Opportunity_ID__c) AS daily_count
  FROM Funnel_With_Flags
  WHERE is_sqo = 1 AND DATE(Date_Became_SQO__c) BETWEEN '2025-10-01' AND '2025-12-31'
  GROUP BY 1, 2, 3
  
  UNION ALL
  
  -- Joined
  SELECT DATE(advisor_join_date__c) AS date_day, Channel_Grouping_Name, Original_source,
         'joined' AS stage, COUNT(DISTINCT Full_Opportunity_ID__c) AS daily_count
  FROM Funnel_With_Flags
  WHERE is_joined = 1 AND DATE(advisor_join_date__c) BETWEEN '2025-10-01' AND '2025-12-31'
  GROUP BY 1, 2, 3
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
   
   Each stage has its own timestamp representing when that event occurred.

2. **DATE() Function:**
   ```sql
   DATE(mql_stage_entered_ts)
   ```
   - Converts timestamp to date (strips time component)
   - Groups all events from same day together
   - Example: '2025-10-15 14:23:45' → '2025-10-15'

3. **UNION ALL Structure:**
   - Combines all stages into one dataset
   - Preserves stage identification via literal string
   - Creates long format: (date, channel, source, stage, count)

**What This Produces:**
```
date_day    | channel   | source    | stage | daily_count
2025-10-01  | Marketing | Webinar   | mqls  | 5
2025-10-01  | Outbound  | LinkedIn  | mqls  | 3
2025-10-02  | Marketing | Webinar   | mqls  | 7
```

**Statistical Importance:**
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
  CROSS JOIN (SELECT DISTINCT Channel_Grouping_Name, Original_source, stage FROM Daily_Actuals) dims
  LEFT JOIN Daily_Actuals a
    ON d.date_day = a.date_day
    AND dims.Channel_Grouping_Name = a.Channel_Grouping_Name
    AND dims.Original_source = a.Original_source
    AND dims.stage = a.stage
  GROUP BY 1, 2, 3, 4
),
```

### Logic Explanation:

This CTE creates cumulative totals - the running sum of all activity from Q4 start through each date.

**Complex Join Logic Explained:**

1. **The CROSS JOIN:**
   ```sql
   Date_Spine d
   CROSS JOIN (SELECT DISTINCT ... FROM Daily_Actuals) dims
   ```
   - Creates every combination of date × channel × source × stage
   - If we have 92 dates and 20 channel/source/stage combos = 1,840 rows
   - Ensures we have a row even for days with zero activity

2. **The LEFT JOIN:**
   ```sql
   LEFT JOIN Daily_Actuals a ON [four conditions]
   ```
   - Attaches actual counts where they exist
   - Leaves NULL where no activity occurred
   - COALESCE converts NULL to 0

3. **The Window Function Magic:**
   ```sql
   SUM(SUM(COALESCE(a.daily_count, 0))) OVER (
     PARTITION BY dims.Channel_Grouping_Name, dims.Original_source, dims.stage 
     ORDER BY d.date_day
   )
   ```
   
   **Breaking this down:**
   - Inner `SUM()`: Aggregates within the GROUP BY
   - Outer `SUM() OVER`: Creates running total
   - `PARTITION BY`: Separate running totals for each channel/source/stage
   - `ORDER BY d.date_day`: Accumulate in chronological order

**Example Calculation:**
```
Day 1: 5 MQLs → Cumulative: 5
Day 2: 3 MQLs → Cumulative: 8 (5+3)
Day 3: 0 MQLs → Cumulative: 8 (5+3+0)
Day 4: 6 MQLs → Cumulative: 14 (5+3+0+6)
```

**Statistical Significance:**
Cumulative functions create monotonically increasing curves - they can only go up or stay flat, never decrease. This property is crucial for our confidence interval calculations later.

---

## CTE 5: Monthly_Forecast_Targets - Stepped Goals {#cte5}

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
    CAST(forecast_value AS INT64) AS monthly_forecast
  FROM `savvy-gtm-analytics.SavvyGTMData.q4_2025_forecast`
  WHERE metric = 'Cohort_source'
    AND original_source != 'All'
    AND month_key IN ('2025-10', '2025-11', '2025-12')
),
```

### Logic Explanation:

This CTE imports monthly targets to create stepped progression lines (not just end-of-quarter targets).

**Why Monthly Instead of Quarterly?**
- Business reality: Different monthly targets reflect seasonality
- October might be lower due to ramp-up
- December might be higher due to year-end push
- Creates more realistic target progression

**Data Transformations:**
Same as forecast_vs_actuals:
- Channel renaming (Inbound → Marketing)
- Stage pluralization (mql → mqls)
- Type casting for numeric operations

**The Result:**
```
channel    | source   | stage | month_key | monthly_forecast
Marketing  | Webinar  | mqls  | 2025-10   | 140
Marketing  | Webinar  | mqls  | 2025-11   | 145
Marketing  | Webinar  | mqls  | 2025-12   | 145
```

This gives us monthly milestones for the target line.

---

## CTE 6: Cumulative_Monthly_Targets - Progressive Targets {#cte6}

### Code:
```sql
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

**The Window Function:**
```sql
SUM(monthly_forecast) OVER (
  PARTITION BY channel_grouping_name, original_source, stage 
  ORDER BY month_key
)
```

**Example Calculation:**
```
October target: 140 → Cumulative by Oct 31: 140
November target: 145 → Cumulative by Nov 30: 285 (140+145)
December target: 145 → Cumulative by Dec 31: 430 (140+145+145)
```

**Why Cumulative?**
- Our actuals are cumulative
- Comparison requires same scale
- Shows progressive achievement toward final goal

**Visual Result:**
Instead of three separate monthly bars, we get a line that steps up at month boundaries, showing expected progression through the quarter.

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
- `forecast_value`: Original Q4 target (430 MQLs)
- `predicted_value`: Where we'll likely end up (580 MQLs)
- `stddev_daily`: Daily volatility for confidence intervals

**Why Import Instead of Recalculate?**
- Reuses complex cascade logic from main view
- Ensures consistency across dashboards
- Single source of truth for predictions

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
- This is the "You are here" marker

**Example:**
If today is October 21st and we have 166 MQLs so far, this CTE captures that 166 as our starting point for projecting forward.

**Statistical Importance:**
In interpolation, boundary conditions are crucial. This provides our initial condition - the known value from which we project.

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

**Why Daily Growth Instead of Daily Count?**
- We're predicting cumulative curves
- Need to understand growth volatility
- Some days might have negative growth (data corrections)

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

### Confidence Interval Calculation:

```sql
-- Lower bound
CASE
  WHEN a.date_day <= CURRENT_DATE() THEN a.cumulative_actual
  ELSE 
    GREATEST(
      COALESCE(c.current_actual, 0),
      [central_prediction] - (1.96 * COALESCE(v.stddev_daily_growth, 0) * SQRT(DATE_DIFF(a.date_day, CURRENT_DATE(), DAY)))
    )
END AS predicted_lower,

-- Upper bound
CASE
  WHEN a.date_day <= CURRENT_DATE() THEN a.cumulative_actual
  ELSE 
    [central_prediction] + (1.96 * COALESCE(v.stddev_daily_growth, 0) * SQRT(DATE_DIFF(a.date_day, CURRENT_DATE(), DAY)))
END AS predicted_upper,
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
   ```sql
   GREATEST(COALESCE(c.current_actual, 0), [calculation])
   ```
   - Ensures lower bound never goes below current actual
   - Prevents impossible scenarios (can't have fewer than we already have)

**Example Confidence Interval:**
- Prediction 30 days out: 341 MQLs
- Daily volatility: 4.2 MQLs
- Uncertainty: 1.96 × 4.2 × √30 = 1.96 × 4.2 × 5.48 = 45 MQLs
- 95% Confidence Interval: [296, 386]

### Target Line Calculation:

```sql
-- CORRECTED Target line (stepped monthly progression)
CASE 
  -- October: Linear from 0 to Oct target
  WHEN a.date_day <= DATE('2025-10-31') THEN
    COALESCE([oct_cumulative_target], 0) * DATE_DIFF(a.date_day, DATE('2025-09-30'), DAY) / 31.0
    
  -- November: Linear from Oct target to Nov target  
  WHEN a.date_day <= DATE('2025-11-30') THEN
    COALESCE([oct_cumulative_target], 0) +
    ([nov_cumulative_target] - [oct_cumulative_target]) * DATE_DIFF(a.date_day, DATE('2025-10-31'), DAY) / 30.0
    
  -- December: Linear from Nov target to Dec target
  ELSE
    COALESCE([nov_cumulative_target], 0) +
    ([dec_cumulative_target] - [nov_cumulative_target]) * DATE_DIFF(a.date_day, DATE('2025-11-30'), DAY) / 31.0
END AS target_value
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

This creates a more realistic target line that reflects monthly quotas.

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
  
  -- SQL metrics
  MAX(CASE WHEN stage = 'sqls' THEN actual_value END) AS sqls_actual,
  MAX(CASE WHEN stage = 'sqls' THEN predicted_value END) AS sqls_predicted,
  -- ... repeated for all stages
  
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

1. **CASE returns:**
   - Value when condition matches
   - NULL when it doesn't

2. **GROUP BY creates groups:**
   - One group per date/channel/source combination
   - Multiple rows (one per stage) in each group

3. **MAX() collapses the group:**
   - Ignores NULLs
   - Returns the one non-NULL value
   - Could also use MIN() or AVG() - same result with one value

**The 20-Column Output:**

For each stage (MQLs, SQLs, SQOs, Joined):
1. `{stage}_actual` - Historical performance
2. `{stage}_predicted` - Central forecast line
3. `{stage}_lower` - Lower confidence bound
4. `{stage}_upper` - Upper confidence bound
5. `{stage}_target` - Original goal line

5 metrics × 4 stages = 20 columns

**Why This Format?**

Looker Studio can easily plot multiple series from columns:
- Each column becomes a line on the chart
- Shared x-axis (date_day)
- Different line styles for actual vs predicted
- Shaded areas between upper/lower bounds

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

### 3. **Cumulative Distribution Functions**
Monotonically increasing functions that represent accumulated progress:
```
F(t) = Σ(events from start to t)
```

### 4. **Piecewise Linear Functions**
Stepped targets that change slope at month boundaries:
```
f(x) = { a₁x + b₁ if x ∈ [t₀,t₁]
       { a₂x + b₂ if x ∈ [t₁,t₂]
       { a₃x + b₃ if x ∈ [t₂,t₃]
```

### 5. **Window Functions for Running Totals**
SQL's analytical capabilities for cumulative calculations:
```
SUM(value) OVER (ORDER BY date ROWS UNBOUNDED PRECEDING)
```

---

## Practical Applications

### For RevOps Teams:

**Reading the Visualization:**
- **Blue solid line** (actual): Performance through today
- **Gray dashed line** (predicted): Expected trajectory
- **Gray shaded area** (confidence): Range of likely outcomes
- **Green line** (target): Original plan

**Key Insights:**
- Widening confidence cone = increasing uncertainty over time
- Actual crossing above/below target = ahead/behind plan
- Predicted vs target at Dec 31 = will we hit goal?

### For Leadership:

**The Story:**
"Based on our current pace and historical patterns, here's our expected path to end of quarter. The shaded area shows our confidence range - we're 95% confident we'll end up within these bounds."

**Decision Points:**
- If predicted line below target: Need intervention
- If lower bound above target: High confidence of success
- If upper bound below target: Very unlikely to hit goal

### For Data Teams:

**Reusability:**
This pattern works for any metric with:
- Historical daily data
- Future endpoint prediction
- Need for uncertainty quantification

**Extensions:**
- Monte Carlo simulation using the volatility measures
- Seasonal adjustments to the linear interpolation
- Multi-scenario planning with different endpoints

---

## Common Issues and Solutions

### Issue: Confidence intervals go negative

**Solution:** The GREATEST function ensures lower bound ≥ current actual

### Issue: Jagged actual line

**Cause:** Sparse data with many zero days

**Solution:** Consider smoothing or moving averages

### Issue: Confidence intervals too wide/narrow

**Adjust:** The 1.96 factor (use 1.645 for 90%, 2.576 for 99%)

### Issue: Target line doesn't match expectations

**Check:** Monthly forecast values and cumulative logic

---

## Conclusion

This SQL view transforms point predictions into sophisticated time series visualizations with statistical confidence bounds. The mathematical foundation combines:

- Linear interpolation for smooth projections
- Random walk theory for uncertainty quantification  
- Cumulative functions for monotonic growth
- Piecewise functions for realistic targets

The result is a powerful visualization that conveys not just where we expect to be, but how confident we are in that expectation. The widening confidence cone elegantly captures the fundamental truth that uncertainty increases with time, while the stepped target line reflects real business planning cycles.

The entire implementation demonstrates how SQL's analytical capabilities can implement sophisticated statistical models, creating actionable insights from raw funnel data. The key innovation is making complex statistics accessible through intuitive visualizations that drive better business decisions.
