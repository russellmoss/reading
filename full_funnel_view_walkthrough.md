# Complete Walkthrough: vw_funnel_lead_to_joined_v2 - The Foundation View

**Document Purpose:** This guide provides a detailed, step-by-step explanation of the main funnel view that serves as the foundation for all RevOps analytics, including how it combines lead and opportunity data into a unified funnel.

**Target Audience:** RevOps professionals with basic statistics knowledge (undergraduate biology level)

**Last Updated:** October 22, 2025

**Critical Context:** This view is being enhanced to support Re-Engagement Opportunities - a new pathway where advisors restart their journey directly as opportunities without going through the lead process.

---

## Table of Contents
1. [Overview: The Master Funnel View](#overview)
2. [CTE 1: Lead_Base - Processing Lead Data](#cte1)
3. [CTE 2: Opp_Base - Processing Opportunity Data](#cte2)
4. [The Main SELECT - The FULL OUTER JOIN Magic](#main-select)
5. [Key Field Calculations](#key-fields)
6. [The SGA Attribution Logic](#sga-logic)
7. [FilterDate Strategy for Cohort Analysis](#filterdate)
8. [Current Limitations and Upcoming Enhancements](#limitations)

---

## Overview: The Master Funnel View {#overview}

This view is the **single source of truth** for the entire sales funnel, combining:
- **Lead data** from Salesforce (prospects being nurtured)
- **Opportunity data** from Salesforce (qualified prospects in sales process)
- **Attribution mapping** to channels and sources
- **Binary flags** for mathematical operations in downstream views

**The Key Innovation:** Using a FULL OUTER JOIN to capture three types of records:
1. Leads that haven't converted yet
2. Leads that converted to opportunities
3. Opportunities that started without leads (direct referrals, re-engagements)

This comprehensive approach ensures we never miss revenue-generating activities.

---

## CTE 1: Lead_Base - Processing Lead Data {#cte1}

### Code Structure:
```sql
WITH Lead_Base AS (
  SELECT
    Full_prospect_id__c,
    Name AS Prospect_Name,
    CreatedDate,
    OwnerID,
    SGA_Owner_Name__c,
    Company,
    -- [additional fields...]
    
    -- Binary flag creation
    CASE WHEN stage_entered_contacting__c IS NOT NULL THEN 1 ELSE 0 END AS is_contacted,
    CASE WHEN Stage_Entered_Call_Scheduled__c IS NOT NULL THEN 1 ELSE 0 END AS is_mql,
    CASE WHEN initial_call_scheduled_date__c IS NOT NULL THEN 1 ELSE 0 END AS is_initial_call,
    CASE WHEN IsConverted IS TRUE THEN 1 ELSE 0 END AS is_sql,
    
    -- Week bucketing for reporting
    FORMAT_DATE('%m/%d/%Y', DATE_TRUNC(DATE(initial_call_scheduled_date__c), WEEK(MONDAY))) AS Week_Bucket_MQL_Call,
    
    -- FilterDate calculation
    GREATEST(
      IFNULL(CreatedDate, TIMESTAMP('1900-01-01')),
      IFNULL(stage_entered_new__c, TIMESTAMP('1900-01-01')),
      IFNULL(stage_entered_contacting__c, TIMESTAMP('1900-01-01'))
    ) AS FilterDate
    
  FROM `savvy-gtm-analytics.SavvyGTMData.Lead`
),
```

### Logic Explanation:

**Field Renaming for Clarity:**
```sql
Name AS Prospect_Name
Stage_Entered_Call_Scheduled__c AS mql_stage_entered_ts
ConvertedDate AS converted_date_raw
```
- Creates intuitive field names
- `_ts` suffix indicates timestamp fields
- `_raw` suffix indicates unprocessed data

**Binary Flag Creation Pattern:**
```sql
CASE WHEN stage_entered_contacting__c IS NOT NULL THEN 1 ELSE 0 END AS is_contacted
```

**Why binary flags?**
- Enables `SUM(is_mql)` to count MQLs
- Enables `AVG(is_mql)` to calculate rates
- More efficient than repeated CASE statements in queries

**The Three Key Milestone Flags:**

1. **is_contacted:**
   - Lead has been reached out to
   - First touch in the sales process
   - Gate for prospect → MQL conversion

2. **is_mql (Marketing Qualified Lead):**
   - Call has been scheduled
   - Primary qualification milestone
   - Most important early-funnel metric

3. **is_initial_call:**
   - Initial call actually occurred
   - Different from scheduled (no-show tracking)
   - **Gap Alert:** Re-engagement opportunities lack this field

4. **is_sql (Sales Qualified Lead):**
   - Lead converted to opportunity
   - Crosses the lead-opportunity boundary
   - Uses Salesforce's `IsConverted` field

**Week Bucketing for Cohort Analysis:**
```sql
DATE_TRUNC(DATE(Stage_Entered_Contacting__c), WEEK(MONDAY)) AS Week_Bucket_MQL_Date
```
- Truncates dates to week start (Monday)
- Enables week-over-week comparisons
- Critical for SGA performance tracking

**The FilterDate Calculation:**
```sql
GREATEST(
  IFNULL(CreatedDate, TIMESTAMP('1900-01-01')),
  IFNULL(stage_entered_new__c, TIMESTAMP('1900-01-01')),
  IFNULL(stage_entered_contacting__c, TIMESTAMP('1900-01-01'))
) AS FilterDate
```

**Logic breakdown:**
1. **IFNULL** converts NULL to ancient date (1900-01-01)
2. **GREATEST** picks the latest non-NULL date
3. **Result:** When the lead truly "started" in our system

**Why not just use CreatedDate?**
- Some leads are imported but not immediately worked
- `stage_entered_new__c` captures when work actually began
- `stage_entered_contacting__c` catches leads that skipped stages

---

## CTE 2: Opp_Base - Processing Opportunity Data {#cte2}

### Code Structure:
```sql
Opp_Base AS (
  SELECT
    o.Full_Opportunity_ID__c,
    o.Name AS Opp_Name,
    o.CreatedDate AS Opp_CreatedDate,
    sga_user.Name AS sga_name_from_opp,
    manager_user.Name AS sgm_name,
    -- [financial fields...]
    o.SQL__c AS SQO_raw,
    o.Date_Became_SQO__c,
    o.advisor_join_date__c,
    -- [additional fields...]
    
  FROM `savvy-gtm-analytics.SavvyGTMData.Opportunity` o
  LEFT JOIN `savvy-gtm-analytics.SavvyGTMData.User` sga_user
    ON o.SGA__c = sga_user.Id
  LEFT JOIN `savvy-gtm-analytics.SavvyGTMData.User` opp_owner_user
    ON o.OwnerId = opp_owner_user.Id
  LEFT JOIN `savvy-gtm-analytics.SavvyGTMData.User` manager_user
    ON opp_owner_user.ManagerId = manager_user.Id
  WHERE o.recordtypeid = '012Dn000000mrO3IAI'  -- ⚠️ LIMITATION: Only standard opportunities
)
```

### Logic Explanation:

**User Table Joins for Attribution:**
```sql
LEFT JOIN `savvy-gtm-analytics.SavvyGTMData.User` sga_user
  ON o.SGA__c = sga_user.Id
```
- Gets human-readable SGA name from User ID
- `LEFT JOIN` ensures we keep opportunities even if SGA is missing
- Critical for proper sales attribution

**Manager Hierarchy:**
```sql
LEFT JOIN `savvy-gtm-analytics.SavvyGTMData.User` manager_user
  ON opp_owner_user.ManagerId = manager_user.Id
```
- Two-step join: Opportunity → Owner → Manager
- Captures management hierarchy for rollup reporting
- SGM (Sales Group Manager) oversight tracking

**Key Fields for Funnel Stages:**

1. **SQO_raw (`SQL__c`):**
   - Stores 'yes'/'no'/NULL values
   - Indicates if opportunity is Sales Qualified
   - Note the confusing naming: `SQL__c` field contains SQO data

2. **Date_Became_SQO__c:**
   - Timestamp of qualification decision
   - Critical for time-to-SQO metrics
   - Used in conversion rate calculations

3. **advisor_join_date__c:**
   - When advisor officially joined platform
   - Final success metric
   - Revenue recognition point

**⚠️ CRITICAL LIMITATION - Record Type Filter:**
```sql
WHERE o.recordtypeid = '012Dn000000mrO3IAI'
```

**Current state:**
- Only includes standard advisor opportunities
- Excludes re-engagement opportunities
- Creates blind spot in funnel reporting

**Planned enhancement:**
```sql
WHERE o.recordtypeid IN (
  '012Dn000000mrO3IAI',  -- Standard opportunities
  'XXXXXXXXXXXXXXXXXXXXX'  -- Re-engagement opportunities (TBD)
)
```

---

## The Main SELECT - The FULL OUTER JOIN Magic {#main-select}

### Code Structure:
```sql
SELECT
  -- Composite Primary Key
  COALESCE(l.Full_prospect_id__c, o.Full_Opportunity_ID__c) AS primary_key,
  
  -- [Lead fields...]
  -- [Opportunity fields...]
  
FROM Lead_Base l
FULL OUTER JOIN Opp_Base o
  ON l.converted_oppty_id = o.Full_Opportunity_ID__c
LEFT JOIN `savvy-gtm-analytics.SavvyGTMData.Channel_Group_Mapping` g
  ON COALESCE(o.Opp_Original_Source, l.Lead_Original_Source) = g.Original_Source_Salesforce;
```

### Logic Explanation:

**The FULL OUTER JOIN Strategy:**
```sql
FROM Lead_Base l
FULL OUTER JOIN Opp_Base o
  ON l.converted_oppty_id = o.Full_Opportunity_ID__c
```

**What this captures:**

1. **Left Side Only (Lead without Opportunity):**
   - Unconverted leads still in nurture
   - Example: 500 leads being contacted

2. **Both Sides (Converted Lead-Opportunity Pairs):**
   - Standard funnel progression
   - Example: 200 leads that became opportunities

3. **Right Side Only (Opportunity without Lead):**
   - Direct opportunities (referrals)
   - Re-engagement opportunities
   - Example: 50 direct referrals from partners

**Why not INNER or LEFT JOIN?**
- `INNER JOIN`: Would miss unconverted leads AND direct opportunities
- `LEFT JOIN`: Would miss opportunity-only records
- `FULL OUTER JOIN`: Complete picture of all revenue paths

**Composite Primary Key Creation:**
```sql
COALESCE(l.Full_prospect_id__c, o.Full_Opportunity_ID__c) AS primary_key
```
- Creates unique identifier for any record type
- Prefers lead ID if available
- Falls back to opportunity ID for direct opportunities

---

## Key Field Calculations {#key-fields}

### The SGA Attribution Logic {#sga-logic}

```sql
CASE
  WHEN l.Full_prospect_id__c IS NULL THEN o.sga_name_from_opp  -- Opp-only records
  WHEN l.SGA_Owner_Name__c = 'Savvy Marketing' THEN o.sga_name_from_opp  -- Marketing conversion
  ELSE l.SGA_Owner_Name__c
END AS SGA_Owner_Name__c
```

**Logic breakdown:**

1. **Case 1: Opportunity-only records**
   ```sql
   WHEN l.Full_prospect_id__c IS NULL THEN o.sga_name_from_opp
   ```
   - No lead exists, use opportunity's SGA
   - Handles referrals and re-engagements

2. **Case 2: Marketing-sourced conversions**
   ```sql
   WHEN l.SGA_Owner_Name__c = 'Savvy Marketing' THEN o.sga_name_from_opp
   ```
   - Lead was marketing-generated
   - But SGA took over at opportunity stage
   - Gives SGA credit for the close

3. **Case 3: Standard SGA-owned leads**
   - SGA owned from the beginning
   - Maintains original attribution

**Why this complexity?**
- Marketing generates leads but doesn't close deals
- SGAs should get credit for opportunities they close
- Accurate commission and performance tracking

### The Advisor Name Logic

```sql
COALESCE(o.Opp_Name, l.Prospect_Name) AS advisor_name
```
- Opportunity names are usually more accurate (updated during sales)
- Falls back to lead name if no opportunity exists
- Single field for reporting regardless of funnel stage

### Channel Attribution

```sql
LEFT JOIN `savvy-gtm-analytics.SavvyGTMData.Channel_Group_Mapping` g
  ON COALESCE(o.Opp_Original_Source, l.Lead_Original_Source) = g.Original_Source_Salesforce

-- Then in SELECT:
IFNULL(g.Channel_Grouping_Name, 'Other') AS Channel_Grouping_Name
COALESCE(o.Opp_Original_Source, l.Lead_Original_Source, 'Unknown') AS Original_source
```

**Three-level fallback for source:**
1. Opportunity source (most reliable if updated)
2. Lead source (original attribution)
3. 'Unknown' (data quality safety net)

**Channel mapping:**
- Maps granular sources to channel groups
- Example: "LinkedIn Ad" → "Marketing"
- 'Other' for unmapped sources

---

## FilterDate Strategy for Cohort Analysis {#filterdate}

### The Complete FilterDate Logic

```sql
-- FilterDate with fallback for opportunity-only records
COALESCE(
  l.FilterDate,                       -- Lead's calculated start date
  o.Opp_CreatedDate,                 -- When opportunity was created
  o.Date_Became_SQO__c,              -- When qualified
  TIMESTAMP(o.advisor_join_date__c)  -- When joined (last resort)
) AS FilterDate
```

**Why this cascade?**

1. **Prefer Lead FilterDate:**
   - Most accurate for standard funnel
   - Already handles multiple lead date scenarios

2. **Fallback to Opp_CreatedDate:**
   - For direct opportunities
   - Reasonable proxy for start date

3. **Fallback to SQO Date:**
   - If opportunity created date missing
   - At least captures when in funnel

4. **Last Resort - Join Date:**
   - Ensures every record has a date
   - Better than NULL for cohort analysis

**Statistical Importance:**
FilterDate enables cohort analysis - comparing groups that started at the same time. Without consistent dating, conversion rates become meaningless.

---

## The is_joined Calculation

```sql
-- *** NEW: is_joined calculated in BigQuery (not Looker) ***
CASE WHEN o.advisor_join_date__c IS NOT NULL THEN 1 ELSE 0 END AS is_joined
```

**Why the "NEW" comment?**
- Previously calculated in Looker Studio
- Moving to BigQuery improves:
  - Performance (calculated once, not per query)
  - Consistency (same logic everywhere)
  - Maintainability (single source of truth)

---

## Current Limitations and Upcoming Enhancements {#limitations}

### 1. Re-Engagement Opportunity Integration

**Current Problem:**
```sql
WHERE o.recordtypeid = '012Dn000000mrO3IAI'  -- Only standard opportunities
```

**Impact:**
- Re-engagement opportunities invisible in funnel
- Understating conversion metrics
- Missing important revenue source

**Required Changes:**
1. Update WHERE clause to include re-engagement record type ID
2. Add Initial Call field to re-engagement opportunity object
3. Modify SGA attribution logic for re-engagements

### 2. Missing Initial Call Tracking for Re-Engagements

**Current State:**
- Leads have `initial_call_scheduled_date__c`
- Re-engagement opportunities lack equivalent field
- Cannot track this critical milestone

**Proposed Solution:**
```sql
-- After adding field to Salesforce
CASE 
  WHEN l.initial_call_scheduled_date__c IS NOT NULL THEN l.initial_call_scheduled_date__c
  WHEN o.Re_Engagement_Initial_Call__c IS NOT NULL THEN o.Re_Engagement_Initial_Call__c
  ELSE NULL
END AS initial_call_date
```

### 3. Qualification Call Tracking

**Enhancement Needed:**
- Add qualification call tracking similar to initial calls
- Critical for new SGA dashboard requirements
- Needs consistent fields across lead and opportunity paths

---

## Statistical and Business Impact

### This View Powers:
1. **All conversion rate calculations** (via binary flags)
2. **Cohort analysis** (via FilterDate)
3. **Attribution reporting** (via channel mapping)
4. **SGA performance tracking** (via SGA logic)
5. **Executive dashboards** (complete funnel visibility)

### Mathematical Foundation:
```sql
-- Conversion rates enabled by binary flags
MQL_Conversion_Rate = AVG(is_mql) = SUM(is_mql) / COUNT(*)
SQL_Conversion_Rate = SUM(is_sql) / SUM(is_mql)
SQO_Conversion_Rate = SUM(is_sqo=1) / SUM(is_sqo IN (0,1))
```

### Data Quality Principles:
- **COALESCE** for NULL handling
- **IFNULL** for default values
- **Multiple fallbacks** for critical fields
- **FULL OUTER JOIN** for completeness

---

## Practical Applications

### For RevOps Teams:

**Query Pattern Examples:**
```sql
-- Weekly MQL performance by SGA
SELECT 
  SGA_Owner_Name__c,
  Week_Bucket_MQL_Date,
  SUM(is_mql) as MQLs,
  AVG(is_sql) as conversion_rate
FROM `vw_funnel_lead_to_joined_v2`
GROUP BY 1, 2
```

**Data Quality Checks:**
```sql
-- Find opportunities without leads (candidates for re-engagement classification)
SELECT COUNT(*)
FROM `vw_funnel_lead_to_joined_v2`
WHERE Full_prospect_id__c IS NULL
  AND Full_Opportunity_ID__c IS NOT NULL
```

### For Data Teams:

**Performance Considerations:**
- FULL OUTER JOIN is expensive but necessary
- Consider materialized views for historical data
- Partition by FilterDate for better performance

**Testing New Enhancements:**
```sql
-- Validate re-engagement opportunities appear correctly
SELECT 
  primary_key,
  Full_prospect_id__c,
  Full_Opportunity_ID__c,
  Original_source,
  SGA_Owner_Name__c
FROM `vw_funnel_lead_to_joined_v2`
WHERE Full_prospect_id__c IS NULL
  AND Original_source = 'Re-Engagement'
```

---

## Conclusion

The `vw_funnel_lead_to_joined_v2` view is the cornerstone of RevOps analytics, elegantly handling the complexity of multiple revenue paths through a sophisticated JOIN strategy and careful field calculations. Its binary flags enable efficient mathematical operations, while its comprehensive fallback logic ensures no opportunity is missed.

The upcoming enhancements for re-engagement opportunities will make this view even more complete, capturing the full customer lifecycle including win-back scenarios. The key to its success is the FULL OUTER JOIN pattern combined with robust NULL handling, creating a unified view of the entire revenue funnel regardless of entry point.

This foundation enables all downstream analytics, from simple conversion rates to complex cohort analyses, making it the most critical view in the entire analytics infrastructure. Understanding its logic is essential for anyone working with RevOps data at scale.d