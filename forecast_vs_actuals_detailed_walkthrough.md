
# Complete Walkthrough: vw_forecast_vs_actuals (Active SGA Version) SQL Logic and Mathematics

**Document Purpose:** This guide provides a detailed, step-by-step explanation of the SQL code that powers the forecast vs actuals view filtered for active SGAs, including the mathematical logic and statistical concepts behind each calculation.

**Target Audience:** RevOps professionals with basic statistics knowledge 

**Last Updated:** October 22, 2025

**Key Difference:** This version filters all data to only include records associated with currently active Sales Generated Advisors (SGAs).

---

## Table of Contents
1. [Overview: What This View Does](#overview)
2. [CTE 1: Funnel_With_Flags - Creating the Foundation WITH Active SGA Filter](#cte1)
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

This view answers three critical questions for Q4 2025, **but only for active SGAs**:
1. **Where are we?** (Actuals - only counting active SGA performance)
2. **Where did we want to be?** (Forecast - original targets)
3. **Where will we likely end up?** (Predictions - based on active SGA conversion rates)

The key innovation in this version is filtering out inactive SGAs to get a cleaner picture of current team performance.

---

## CTE 1: Funnel_With_Flags - Creating the Foundation WITH Active SGA Filter {#cte1}

### Code:
```sql
WITH
Funnel_With_Flags AS (
  SELECT
    f.*,
    CASE WHEN LOWER(f.SQO_raw) = 'yes' THEN 1 ELSE 0 END AS is_sqo
  FROM `savvy-gtm-analytics.savvy_analytics.vw_funnel_lead_to_joined_v2` f
  -- FILTER TO ACTIVE SGAs ONLY
  INNER JOIN (
    SELECT DISTINCT sga_name 
    FROM `savvy-gtm-analytics.savvy_analytics.sga_qtly_goals_ext` 
    WHERE sga_name IS NOT NULL
  ) active_sgas
  ON CASE
    WHEN f.Full_prospect_id__c IS NULL THEN f.SGA_Owner_Name__c  -- Opp-only records
    WHEN f.SGA_Owner_Name__c = 'Savvy Marketing' AND f.converted_oppty_id IS NOT NULL THEN f.SGA_Owner_Name__c  -- Marketing-sourced that converted
    ELSE f.SGA_Owner_Name__c
  END = active_sgas.sga_name
),
```

### Logic Explanation:

This CTE creates our foundational dataset with **a critical filter for active SGAs only**.

**The Active SGA Filter - Breaking It Down:**

1. **The Subquery:**
   ```sql
   SELECT DISTINCT sga_name 
   FROM `savvy-gtm-analytics.savvy_analytics.sga_qtly_goals_ext` 
   WHERE sga_name IS NOT NULL
   ```
   - Pulls list of SGAs who have quarterly goals
   - Having goals = active SGA
   - `DISTINCT` ensures each SGA appears once
   - `WHERE sga_name IS NOT NULL` excludes blank entries

2. **The Complex JOIN Condition:**
   ```sql
   ON CASE
     WHEN f.Full_prospect_id__c IS NULL THEN f.SGA_Owner_Name__c  -- Opp-only records
     WHEN f.SGA_Owner_Name__c = 'Savvy Marketing' AND f.converted_oppty_id IS NOT NULL THEN f.SGA_Owner_Name__c
     ELSE f.SGA_Owner_Name__c
   END = active_sgas.sga_name
   ```

   **Let's decode this CASE statement:**
   
   - **Case 1: Opportunity-only records** (`Full_prospect_id__c IS NULL`)
     - Some opportunities don't have lead history (direct referrals)
     - For these, use the SGA_Owner_Name__c from the opportunity
   
   - **Case 2: Marketing-sourced conversions** 
     - When `SGA_Owner_Name__c = 'Savvy Marketing'` AND it converted
     - Keep attribution to Marketing (special SGA entity)
     - Important for measuring marketing effectiveness
   
   - **Case 3: Everything else**
     - Standard case - use the SGA owner from the lead

3. **Why INNER JOIN Instead of LEFT JOIN?**
   - `INNER JOIN` only keeps matches
   - Records without active SGA attribution are **excluded**
   - This means if an SGA left the company, their historical records don't count

**Statistical Significance:**
This filter creates a **survivorship bias** - we're only looking at currently successful SGAs. This gives us:
- More optimistic conversion rates (successful SGAs stay, unsuccessful leave)
- Cleaner predictions for current team capacity
- Better alignment with actual achievable targets

**The SQO Flag Creation:**
```sql
CASE WHEN LOWER(f.SQO_raw) = 'yes' THEN 1 ELSE 0 END AS is_sqo
```
Same as before - converts text to binary for mathematical operations.

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

**No changes from original** - This CTE standardizes forecast data exactly as before.

**Important Note:** The forecast data is NOT filtered by active SGAs. This means:
- Targets include expectations for SGAs who may have left
- Creates a potential mismatch: comparing active SGA actuals to all-SGA targets
- May need adjustment in interpretation

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

**No changes from original** - Aggregates monthly forecasts into quarterly totals.

The forecast remains the "ideal world" target, not adjusted for team changes.

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

**Structurally identical to original**, but now only counting records from active SGAs due to the filter in CTE 1.

**What This Means:**
- If an SGA left on October 10th, their Oct 1-10 MQLs are **not counted**
- Only currently active team members' contributions appear
- May show lower actuals than a full historical view

**Business Impact:**
This gives a "What can our current team deliver?" view rather than "What did we deliver historically?"

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
    
    -- [other rates same as before...]
    
  FROM Funnel_With_Flags
  WHERE DATE(FilterDate) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) AND CURRENT_DATE()
  GROUP BY 1, 2
),
```

### Logic Explanation:

**Calculates 90-day trailing conversion rates using only active SGA data.**

**Statistical Impact of Active-Only Filter:**

1. **Higher Conversion Rates Expected:**
   - Successful SGAs tend to stay
   - Underperformers leave or are let go
   - Result: Rates reflect "best of breed" performance

2. **Smaller Sample Size:**
   - Fewer SGAs = fewer data points
   - May increase variance in rates
   - Some channel/source combos might have insufficient data

3. **More Relevant Predictions:**
   - Rates reflect what current team can actually achieve
   - Better predictor of future performance
   - Removes noise from departed team members

**Example Impact:**
- Full team MQL→SQL rate: 45%
- Active SGA only rate: 52%
- The 7% difference reflects quality of retained team

---

## CTE 6: Open_Pipeline - What's In Progress {#cte6}

### Code:
```sql
Open_Pipeline AS (
  SELECT
    channel_grouping_name,
    original_source,
    COUNT(DISTINCT IF(is_contacted = 1 AND is_mql = 0, Full_prospect_id__c, NULL)) AS open_prospects,
    -- [other counts...]
  FROM Funnel_With_Flags
  WHERE DATE(FilterDate) BETWEEN '2025-10-01' AND CURRENT_DATE()
    AND (StageName IS NULL OR StageName NOT LIKE '%Closed Lost%')
  GROUP BY 1, 2
),
```

### Logic Explanation:

**Counts only open pipeline owned by active SGAs.**

**Critical Business Context:**
- If an SGA left with 50 open MQLs, those are **not** in this count
- Assumes departed SGAs' pipeline was reassigned or lost
- More conservative view of what can actually convert

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

**Creates a potentially misleading gap** because:
- `forecast_value` = target for ALL SGAs (including departed)
- `actual_value` = achievement by ACTIVE SGAs only
- `remaining_value` = inflated gap

**Example Scenario:**
- Original forecast: 100 MQLs (10 SGAs × 10 each)
- 2 SGAs left the team
- Active SGA actuals: 72 MQLs (8 SGAs × 9 each)
- Remaining shows: 28 MQLs needed
- Reality: Current team target should be 80, only 8 behind

---

## CTE 8: Future_Conversions - The Prediction Engine {#cte8}

### Code remains structurally identical to original

### Logic Explanation:

**Predictions are now based on:**
- Open pipeline from active SGAs only
- Conversion rates from active SGAs only
- But remaining forecast from original (all SGA) targets

**This Creates an Interesting Dynamic:**

The formula becomes:
```
Future MQLs = (Active SGA Open Pipeline × Active SGA Rate) + 
              (Original Remaining Target × Active SGA Rate)
```

**Interpretation Challenge:**
- Uses superior conversion rates (active SGAs)
- Applied to potentially inflated remaining targets
- May overpredict if team size has decreased significantly

---

## CTE 9: Historical_Volatility - Measuring Uncertainty {#cte9}

### Code remains the same structurally

### Logic Explanation:

**Calculates daily volatility using only active SGA performance.**

**Statistical Implications:**
- Lower volatility expected (consistent performers retained)
- Confidence intervals may be artificially narrow
- Doesn't account for risk of further SGA departures

---

## Final SELECT - Bringing It All Together {#final-select}

### Code remains the same structurally

### Logic Explanation:

**The output now represents:**
- `forecast_value`: Original target (all SGAs)
- `actual_value`: Current achievement (active SGAs only)
- `predicted_value`: Expected end-of-quarter (based on active SGA performance)
- `stddev_daily`: Volatility of active SGAs

**Key Interpretation Notes:**

1. **Forecast vs Actual Comparison:**
   - Not apples-to-apples if team size changed
   - Consider adjusting forecast proportionally to team size

2. **Prediction Reliability:**
   - More accurate for current team capacity
   - Doesn't account for hiring/departures during quarter
   - Optimistic bias from survivor effect

3. **Recommended Adjustments:**
   ```sql
   Adjusted_Forecast = Original_Forecast × (Active_SGAs / Original_SGAs)
   ```

---

## Statistical Summary

This version introduces **survivorship bias** into the analysis:

**Advantages:**
- More accurate view of current team capability
- Better conversion rates for planning
- Cleaner data without departed SGA noise

**Disadvantages:**
- Historical comparisons become invalid
- Forecast targets may be unrealistic for smaller team
- Confidence intervals too narrow

**The Mathematics:**
```
Active_SGA_Prediction = Current_Actuals(active) + 
                       Pipeline(active) × Rates(active) +
                       Remaining_Forecast(all) × Rates(active)
```

---

## Practical Applications

### For RevOps Teams:

**Adjustments Needed:**
1. Track active SGA count vs original planning count
2. Adjust forecast targets proportionally
3. Consider separate "coverage" metric

**Example Adjustment:**
```sql
-- Add this CTE for context
Team_Size_Adjustment AS (
  SELECT 
    COUNT(DISTINCT sga_name) as active_sga_count,
    10 as original_sga_count,  -- Or pull from planning table
    COUNT(DISTINCT sga_name) / 10.0 as size_ratio
  FROM active_sgas
)
```

### For Leadership:

**The Story Changes To:**
"Based on our **current active team's** pipeline and performance, here's where we'll end the quarter. Note that our forecast was set for a larger team."

**Key Questions:**
- Should we adjust targets for team size?
- Are we hiring to fill gaps?
- Is the higher conversion rate sustainable?

### For Data Teams:

**Consider Creating Two Views:**
1. This version (active SGAs) for forward-looking predictions
2. Original version (all historical) for true performance tracking
3. Reconciliation view showing the impact of team changes

---

## Common Issues and Solutions

### Issue: Predictions exceed forecast despite being behind

**Cause:** Superior active SGA rates applied to full remaining forecast

**Solution:** Adjust remaining forecast for team size

### Issue: Historical trends broken

**Cause:** Comparing active-only to historical all-SGA metrics

**Solution:** Maintain parallel tracking or note the methodology change

### Issue: Some sources show no data

**Cause:** Only inactive SGAs worked those sources

**Solution:** Document and consider reassignment strategy

---

## Conclusion

This active-SGA-filtered version provides a more realistic view of what the current team can achieve, but at the cost of historical comparability. The survivorship bias creates optimistic conversion rates that may not be sustainable if team composition changes. 

The key insight is that this view answers "What can our current team deliver?" rather than "How are we performing against original plans?" Both questions are valuable, but it's critical to understand which one you're answering when making decisions based on this data.

For maximum value, this view should be used alongside team size metrics and potentially adjusted forecasts that account for staffing changes. The mathematics remain sound, but the interpretation requires more nuance when the underlying population (active SGAs) differs from the planning population (all originally planned SGAs).