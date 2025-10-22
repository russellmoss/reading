## 🎯 Project Overview
The primary goals are to **enhance the SGA dashboards** with new call tracking views and to properly integrate **"Re-Engagement Opportunities"** into our main sales funnel (`vw_funnel_lead_to_joined_v2`). This involves updates to Salesforce, Google Sheets, and BigQuery.

***

## 📊 Part 1: SGA Dashboard Enhancements (for David Hipperson)
This section focuses on creating a new dashboard view for SGAs to track key call activities and outcomes.

### New Weekly Call Views
* Create a view that shows **Initial Calls Set** and **Qualification Calls Set** by SGA.
* The view must be filterable by week to review past performance and see upcoming scheduled calls.

### Required Components
* **Scorecards:** Display the total number of initial and qualification calls for the selected week.
* **Detail Tables:** Provide a list of calls with the following columns:
    * Advisor Name (with a URL link to the Salesforce record)
    * Date of Call
    * Call Outcome (e.g., `Converted`, `Closed-Lost`, or `Pending`)

***

## 🔗 Part 2: Re-Engagement Opportunity Integration
This part addresses the main technical challenge: cleaning up and correctly reporting on re-engagement opportunities, which originate as opportunities without a preceding lead.

### Key Challenge
Re-engagement opportunities are worked from a unique record type on the Opportunity object. In the attached SQL query (`vw_funnel_lead_to_joined_v2.sql`), these will appear as records with an `opportunity_id` but no corresponding `lead_id`.

### Action Items & Questions
* **Salesforce Data Gap:** We need a way to track the "Initial Call" for re-engagement opportunities.
    * **Question:** Can we add a field to the "Re-Engagement" Opportunity object in Salesforce to capture the **Initial Call Scheduled Date**? Without this field, we cannot report on it.
* **BigQuery Funnel View Update:** The main funnel view must be modified to include these opportunities.
    * **Action:** In the `Opp_Base` CTE of the `vw_funnel_lead_to_joined_v2.sql` query, the `WHERE` clause currently filters for a single record type (`WHERE o.recordtypeid = '012Dn000000mrO3IAI'`). This must be updated to include the **Record Type ID for Re-Engagement Opportunities**.
* **Dashboard Display:** Determine how to show these on David's new dashboard.
    * **Decision:** Should re-engagement calls appear in the same table as standard MQL-generated calls, or should they be separated into their own distinct table?

***

## 📝 Part 3: Google Sheets & Reporting Updates
These are smaller enhancements for the `weekly_goals_master` Google Sheet and related reporting.

### Goal Sheet (`weekly_goals_master`)
* **Modify Goals:** Change the tracked goals from `MQL` and `SQL` to **Initial Calls Set** and **Qualification Calls Set** to align with the new dashboard.
* **Improve "Delta" Section:** Convert the delta values to be explicitly positive or negative.
* **Add Formatting:** Apply conditional formatting to highlight rows in **red** when performance is below goal and **green** when at or above goal.

### Conversion Rates Page
* Add the total numbers for **Initial Calls** and **Qualification Calls** to this page.
* Include direct links to relevant Salesforce reports for easy access.