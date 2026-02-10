# Domain Pitfalls: v1.2 Analytics, Dashboards & Division Support

**Project:** Cornerstone v1.2 -- Analytics, Dashboards & Division Support
**Domains:** Sales Forecasting, Trend Detection, Executive Dashboards, Division Filtering, Payroll Expense (GL-based)
**Researched:** 2026-02-09
**Calibration:** v1.0 TransDetailParam IsActive bug = $1.3M revenue discrepancy. All severity ratings use this as the "Critical" reference point.

---

## Critical Pitfalls

Mistakes that produce wrong numbers, break the 99.98% accuracy standard, or require architectural rework. Each of these has evidence from the FLS database, the Cyrious wiki, or prior milestone discoveries.

---

### Pitfall C1: NULL DivisionID Silently Excludes Records from Division Filters

**What goes wrong:** A division filter query like `WHERE th.DivisionID = 10` silently excludes all records where `DivisionID IS NULL`. At FLS, NULL DivisionID records default to the Company Division (ID 10), so a "Banner Division sales" query misses an unknown number of orders with NULL division assignments, potentially understating revenue by tens or hundreds of thousands of dollars.

**Why it happens:** `DivisionID` is nullable in TransHeader, Ledger/GL, Account, Inventory, Journal, and TransDetail. Control's own Crystal Reports and wiki SQL scripts universally use `COALESCE(DivisionID, 10)` to handle NULLs. A developer who writes `WHERE DivisionID = 10` instead of `WHERE COALESCE(DivisionID, 10) = 10` gets different results. NULL != 10 in SQL, so NULLs are excluded from both division buckets and silently vanish from all division-filtered reports.

**Evidence:** The Cyrious wiki documents this pattern in 40+ SQL queries across integrity checks, AR reports, AP reports, and GL reconciliation scripts. Every single one uses `COALESCE(DivisionID, 10)`. The wiki troubleshooting section explicitly states: "Always use COALESCE(DivisionID, 10) -- DivisionID can be NULL and 10 is the default." The support wiki even documents a procedure to find and fix NULL DivisionID entries in the Ledger table: `SELECT * FROM Ledger WHERE ID > 0 AND DivisionID IS NULL`.

**Consequences:** Revenue by division does not sum to total revenue. "Banner Division: $2.5M + Apparel Division: $200K = $2.7M" but total revenue is $3.05M. The $350K gap is records with NULL DivisionID. This destroys trust in every division-filtered metric. Worse, it silently understates the Company Division's numbers because those are the ones that default to 10.

**Prevention:**
- MANDATORY RULE: Every query that filters or groups by DivisionID MUST use `COALESCE(DivisionID, 10)` instead of bare `DivisionID`
- This applies to: TransHeader.DivisionID, Ledger.DivisionID, GL.DivisionID, Account.DivisionID, Inventory.DivisionID, Journal.DivisionID
- Validation check: `SELECT SUM(SubTotalPrice) FROM TransHeader WHERE TransactionType=1 AND IsActive=1 AND SaleDate IS NOT NULL AND YEAR(SaleDate)=2025` must equal the sum of per-division breakdowns
- Template the COALESCE pattern into every division-aware query -- never let a bare DivisionID comparison pass code review

**Detection:** Compare total revenue (un-filtered) with SUM of per-division revenues. If they do not match, NULL DivisionIDs are being dropped.

**Severity:** CRITICAL -- This is the v1.2 equivalent of the TransDetailParam IsActive bug. It silently excludes valid records from results, producing revenue numbers that are wrong by an unquantified amount. The discrepancy could be $100K+ for the Company Division.

**Phase:** Must be the FIRST rule established in the division filtering framework before any division-filtered query is written.

---

### Pitfall C2: DivisionID vs ProductionDivisionID vs ShipFromDivisionID Confusion

**What goes wrong:** The skill uses the wrong DivisionID field for a query, producing results that assign revenue or costs to the wrong division. A "Banner Division production" query uses `TransHeader.DivisionID` (sales division) when it should use `TransDetail.ProductionDivisionID`, or vice versa.

**Why it happens:** TransHeader has THREE division-related fields:
- `DivisionID` -- The sales/billing division (which division "owns" this order for revenue purposes)
- `ProductionDivisionID` -- Which division handles production (may differ from sales division)
- `ShipFromDivisionID` -- Which division ships the order

TransDetail also has `ProductionDivisionID` (FK to DivisionData.ID). Additionally, both have `*Overridden` bit fields (`DivisionIDOverridden`, `ProductionDivisionIDOverridden`) that indicate when the value was manually changed from the default.

For FLS, an apparel order could have DivisionID = Apparel but ProductionDivisionID = Company (if the Company division does the printing). Using the wrong field produces wrong division-level metrics depending on whether the question is about revenue attribution or production workload.

**Consequences:** Revenue attributed to the wrong division, or production costs appearing in the wrong division's P&L. If Apparel orders are sometimes produced by the Company division, using ProductionDivisionID for revenue would shift Apparel revenue to Company.

**Prevention:**
- Revenue/sales queries: Always use `TransHeader.DivisionID` (or `COALESCE(TransHeader.DivisionID, 10)`)
- Production/cost queries: Use `TransDetail.ProductionDivisionID` or `TransHeader.ProductionDivisionID`
- Shipping queries: Use `TransHeader.ShipFromDivisionID`
- GL/Ledger queries: Use `GL.DivisionID` (which reflects the accounting division)
- Document which DivisionID field answers which question in the skill
- Natural language mapping: "Apparel revenue" -> TransHeader.DivisionID. "Apparel production" -> ProductionDivisionID.

**Detection:** If Apparel Division revenue seems unexpectedly high or low compared to Gretel's expectations, check which DivisionID field is being used.

**Severity:** HIGH -- Wrong division attribution for a significant portion of orders. The magnitude depends on how often ProductionDivisionID differs from DivisionID at FLS, which has not yet been validated.

**Phase:** Division filtering framework phase. Must be documented before any division-filtered query patterns.

---

### Pitfall C3: Ledger DivisionID vs ProcessedDivisionID for GL/Financial Queries

**What goes wrong:** A financial query for "Apparel Division P&L" uses `Ledger.DivisionID` when it should use `Ledger.ProcessedDivisionID`, or vice versa, producing a P&L that misattributes entries between divisions.

**Why it happens:** The Ledger table has TWO division fields:
- `DivisionID` -- The division of the related order/entity (populated from TransHeader.DivisionID)
- `ProcessedDivisionID` -- The division in which the GL entry was processed/posted

These can differ when a payment is posted in one division's context but applies to an order in another division. The Cyrious wiki documents a bug (CCON-5479): "Payments - Payment Ledger not updating the ProcessedDivisionID records in SQL when changing the Division a payment is made for." This means historical data may have inconsistent ProcessedDivisionID values.

The wiki's SQL integrity scripts use `DivisionID` (with COALESCE to 10) for GL balance reconciliation, NOT ProcessedDivisionID. This suggests `DivisionID` is the correct field for financial reporting by division, while `ProcessedDivisionID` is a system-level field.

**Consequences:** Division-level P&L shows wrong expense or revenue allocations. Cross-division payments appear in the wrong division's financial summary.

**Prevention:**
- Use `COALESCE(GL.DivisionID, 10)` for all division-filtered financial queries (consistent with Cyrious wiki patterns)
- Do NOT use `ProcessedDivisionID` for financial reporting unless specifically analyzing processing/posting workflow
- Validate: P&L by division should sum to total P&L (same COALESCE pattern as C1)
- Note the CCON-5479 bug: historical ProcessedDivisionID may be unreliable for cross-division payments

**Detection:** Compare `GROUP BY DivisionID` results with `GROUP BY ProcessedDivisionID` results. If they differ significantly, investigate which one matches Control's built-in reports.

**Severity:** HIGH -- Wrong division P&L. Magnitude depends on volume of cross-division transactions at FLS.

**Phase:** Division filtering framework, specifically the financial/GL integration layer.

---

### Pitfall C4: Forecasting with Insufficient Historical Data Produces Misleading Projections

**What goes wrong:** A sales forecast projects "Apparel Division will grow 40% next year" based on 6 months of data from a new division, or projects "Q1 revenue will be $900K" based on a seasonal pattern derived from only 2 years of history, which contains COVID-era distortions or business model changes.

**Why it happens:** FLS has ~2+ years of historical data. For SQL-only forecasting (no Python ML), the system can use moving averages, linear regression via window functions, and seasonal decomposition ratios. But:
- 2 years of monthly data = 24 data points -- barely enough for seasonal pattern detection
- If Apparel Division is newer, it may have even fewer data points
- Business model changes (adding product lines, losing/gaining major customers) create structural breaks that simple trend extrapolation misses
- Seasonal patterns in the sign/banner industry (trade show season, outdoor advertising season) may not follow standard retail seasonality

**Consequences:** Overconfident forecasts that mislead business planning. "We project $4.2M next year" when the uncertainty range is actually $3.0M-$5.5M. Or seasonal adjustments that amplify noise in thin months (e.g., a single large order in January 2025 makes the model predict a strong January 2026 that doesn't materialize).

**Prevention:**
- Always show confidence context: "Based on [N] months of history" and "Historical range: $X-$Y"
- Use simple moving averages (3-month, 6-month) rather than complex seasonal decomposition with only 24 data points
- For annual forecasts, present as a range: "Projected: $X-$Y based on [method]"
- Flag when a single large customer or order distorts a period (FLS's top customer FLASH is ~14% of revenue -- if they have an unusually large or small month, it swings the trend)
- Compare forecast against naive baseline: "vs. same period last year" and "vs. trailing average"
- Never forecast Apparel Division separately if it has fewer than 12 months of division-tagged data -- use blended company data instead
- SQL approach should be: moving averages via window functions (proven, simple), NOT attempting Holt-Winters or exponential smoothing in pure SQL (error-prone, hard to validate)

**Detection:** If a forecast shows >20% growth or >20% decline from trailing 12-month actual, flag it as "outside historical range" and require human review. Compare projection against what actually happened in prior periods.

**Severity:** HIGH -- Bad forecasts drive bad business decisions (overhiring, over-ordering materials, under-investing). Not as immediately detectable as wrong revenue numbers, which makes it more insidious.

**Phase:** Sales forecasting phase. Must establish methodology constraints before any forecast queries are written.

---

### Pitfall C5: Dashboard Metrics Inconsistent with Existing Skill Results

**What goes wrong:** The executive dashboard shows "YTD Revenue: $2,980,000" while the sales skill returns "$3,052,952.52" for the same query. The user sees two different numbers from the same system and loses trust in both.

**Why it happens:** The dashboard aggregates data independently from the existing skills. If the dashboard query uses slightly different filters (e.g., missing `SaleDate IS NOT NULL`, using `TotalPrice` instead of `SubTotalPrice`, or not applying the StatusID 9 exclusion consistently), it produces different numbers. This is the v1.2 manifestation of v1.1's Pitfall I1 (Contradicting Core Skill Business Rules) and I3 (Duplicated Query Patterns).

With dashboards, the risk is amplified because:
- Dashboards display multiple metrics side-by-side, making inconsistencies visible
- React artifact dashboards are generated per-request, so each generation could use slightly different SQL
- A "morning report" that shows revenue alongside AR alongside production metrics must be consistent across ALL three source skills

**Consequences:** Two numbers for the same metric visible to the same user at the same time. This is a credibility-destroying outcome for a system validated to 99.98% accuracy.

**Prevention:**
- Dashboard queries MUST use the EXACT same SQL patterns as the existing skills -- not "similar" patterns
- Create a shared query template library that both skills and dashboards reference
- The dashboard skill should call the SAME revenue formula from control-erp-core:
  ```sql
  SELECT SUM(SubTotalPrice) FROM TransHeader
  WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL
  ```
- Every metric on the dashboard must trace back to a specific skill and query template
- Validation: dashboard metrics must match individual skill results for the same parameters
- Consider generating dashboard content by running the same NL queries that the skills already handle, rather than building separate dashboard-specific SQL

**Detection:** Run the dashboard and the individual skill queries side-by-side. Any discrepancy > $1 indicates a query inconsistency.

**Severity:** CRITICAL -- Visible inconsistency between dashboard and conversational queries. Users see the dashboard first thing in the morning and then ask follow-up questions via NL. If numbers differ, the entire system's credibility is destroyed.

**Phase:** Dashboard phase. Must be addressed in the dashboard architecture before any React artifacts are built.

---

### Pitfall C6: Division Filtering on GL Does Not Match Division Filtering on TransHeader

**What goes wrong:** "Apparel Division revenue (TransHeader)" shows $200K but "Apparel Division revenue (GL)" shows $180K. The discrepancy is because GL entries may have different DivisionID values than their parent TransHeader due to timing, manual corrections, or the CCON-5479 bug.

**Why it happens:** Revenue can be measured two ways:
1. **TransHeader-based:** `SUM(SubTotalPrice) WHERE TransactionType=1 AND COALESCE(DivisionID, 10) = @DivisionID` -- the validated 99.98% approach
2. **GL-based:** `SUM(-Amount) WHERE GLAccountID IN (revenue accounts) AND COALESCE(DivisionID, 10) = @DivisionID`

These SHOULD agree, but the Ledger/GL entries are created during status transitions and may inherit a different DivisionID than the TransHeader if:
- The order's division was changed after GL entries were posted
- The CCON-5479 bug caused ProcessedDivisionID to not update
- Manual GL adjustments don't carry the correct DivisionID
- Off-balance-sheet entries have different division tagging

The existing v1.1 pitfall m5 (Ledger.EntryDateTime vs TransHeader.SaleDate timing) already documented minor discrepancies between the two approaches. Adding division filtering amplifies this because you're now partitioning an already-small discrepancy across divisions.

**Consequences:** Division-level P&L (GL-based) does not reconcile with division-level revenue (TransHeader-based). The P&L might show Apparel at $180K while the sales dashboard shows Apparel at $200K. This is confusing and erodes trust.

**Prevention:**
- Establish a single authoritative source for each metric:
  - **Revenue by division:** TransHeader-based (primary, validated) with COALESCE pattern
  - **P&L by division:** GL-based (necessary for expense breakdown) with COALESCE pattern
  - **Document the expected discrepancy** between the two approaches and set tolerance thresholds
- If the GL vs TransHeader discrepancy for a division exceeds 2%, flag it and investigate
- Consider adding a reconciliation query to the skill that identifies orders where `COALESCE(TransHeader.DivisionID, 10) != COALESCE(GL.DivisionID, 10)` for GL entries related to that order
- The dashboard should clearly label which data source each metric uses

**Detection:** Compare TransHeader revenue by division with GL revenue by division. Expect minor discrepancies (<1%) but flag anything larger.

**Severity:** HIGH -- Confusing but not catastrophically wrong. The discrepancy is bounded by the overall GL-vs-TransHeader variance (previously validated as small).

**Phase:** Division filtering framework phase, specifically when integrating division filtering into the financial skill.

---

## Moderate Pitfalls

Mistakes that cause delays, confusing results, or technical debt but do not produce catastrophically wrong numbers.

---

### Pitfall M1: Seasonal Pattern Overfitting with Only 24 Monthly Data Points

**What goes wrong:** The forecasting model detects "seasonality" that is actually noise. January 2025 had a large one-time order, so the model projects a strong January 2026. Or a model built on COVID-recovery years (2024-2025) extrapolates growth rates that are unsustainable.

**Why it happens:** With only ~24 months of monthly data (2024-2025 at FLS), the ratio-to-moving-average seasonal decomposition has exactly 2 data points per month. Any single anomalous month has 50% weight in the seasonal index for that month. This is statistically insufficient for reliable seasonal pattern detection.

**Prevention:**
- For seasonal decomposition, require at minimum 3 full years (36 months) before claiming seasonal patterns -- until then, use trailing moving averages only
- Present all seasonality findings with the caveat: "Based on limited history (N months). Patterns may not repeat."
- Use 3-month moving averages for smoothing (wide enough to dampen noise, narrow enough to capture genuine trends)
- For any month where a single customer represents >20% of that month's revenue, note it as a potential outlier
- Prefer YoY comparison over seasonal projection: "This month last year was $X" is more trustworthy than "Our model predicts $Y"

**Detection:** If the seasonal index for any month is >1.3x or <0.7x the average, the limited data may be overfitting to an outlier.

**Severity:** MEDIUM -- Misleading but clearly labeled forecasts are less dangerous than silently wrong revenue numbers.

**Phase:** Forecasting phase.

---

### Pitfall M2: Division Filter Not Applied Consistently Across Cross-Skill Queries

**What goes wrong:** A user asks "Apparel Division morning report" and the dashboard shows Apparel-filtered revenue (correct) but company-wide AR aging (wrong) and company-wide production pipeline (wrong), because the division filter was only applied to the sales query but not propagated to financial and production sub-queries.

**Why it happens:** Division filtering is cross-cutting -- it needs to work across sales (TransHeader.DivisionID), financial (GL.DivisionID), customer intelligence (Account.DivisionID), inventory (Inventory.DivisionID / Warehouse.DivisionID), and production (TransDetail.ProductionDivisionID). Each domain has a different DivisionID location and different COALESCE semantics. A skill that adds division filtering to its own queries but relies on other skills (which lack division filtering) produces an inconsistent result.

**Consequences:** Dashboard shows a mix of division-filtered and unfiltered metrics. "Apparel revenue: $200K" alongside "AR aging: $450K" (total, not Apparel). The user is confused about what they're seeing.

**Prevention:**
- Division filtering must be implemented as a FRAMEWORK that all skills can consume, not as ad-hoc WHERE clauses in individual skills
- Define a "division context" parameter that every query template accepts
- Map which DivisionID field to use in each table:
  | Table | Division Field | COALESCE Default |
  |-------|---------------|-----------------|
  | TransHeader | DivisionID | 10 |
  | GL/Ledger | DivisionID | 10 |
  | Account | DivisionID | 10 |
  | Inventory | DivisionID | ? (needs validation) |
  | Warehouse | DivisionID | N/A (not nullable) |
  | TransDetail | ProductionDivisionID | ? (needs validation) |
- Test the dashboard with explicit division filtering and verify ALL metrics are filtered

**Detection:** Run the dashboard for "Apparel Division" and check every metric. If any metric seems too large (close to company-wide total), the division filter was not applied.

**Severity:** MEDIUM -- Confusing but detectable. The user would notice "wait, that AR number seems high for Apparel."

**Phase:** Division filtering framework phase. Must be completed BEFORE the dashboard phase, which consumes it.

---

### Pitfall M3: React Artifact Dashboard Imports Unsupported Libraries

**What goes wrong:** The dashboard React artifact fails to render because it imports a library not available in the Claude artifact sandbox. The code references `date-fns`, `d3`, `@tremor/react`, or another charting library that is not in the supported set.

**Why it happens:** Claude's React artifact sandbox supports a specific set of libraries: React, Tailwind CSS, Shadcn UI components, Lucide icons, and Recharts. It does NOT support external data fetching, arbitrary npm packages, or libraries outside this set. A dashboard design that assumes `d3.js` for complex visualizations or `date-fns` for date formatting will fail at render time.

**Prevention:**
- Constrain all dashboard designs to: React + Tailwind CSS + Recharts (for charts) + Lucide (for icons) + Shadcn UI (for components)
- Do NOT attempt: date-fns (use native JS Date), d3 (use Recharts), @tremor/react (not available), moment.js (not available)
- Design the dashboard data flow as: SQL query -> formatted data -> passed as props to React component
- The React artifact receives pre-computed data, not raw SQL results. All data transformation happens in the skill/query layer, not in the React component.
- Test each chart type (line, bar, pie, table) individually before combining into a dashboard

**Detection:** The React artifact shows an error or blank screen instead of the expected dashboard.

**Severity:** MEDIUM -- Blocks dashboard delivery but does not produce wrong data. Easy to fix by substituting supported libraries.

**Phase:** Dashboard phase. Establish library constraints before any React artifact development.

---

### Pitfall M4: Moving Average Window Functions Misconfigured for Forecast Projection

**What goes wrong:** The SQL forecast query uses `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` for a 3-month moving average but fails to handle the edge cases at the beginning of the data series (first 2 months have fewer than 3 data points) or at the end (the forecast period has no actual data, so the moving average cannot be computed).

**Why it happens:** SQL window functions compute moving averages over existing rows. They cannot project forward into periods that do not yet exist. A forecast requires:
1. Computing the moving average/trend from historical data
2. Extrapolating the trend forward into future periods
3. Optionally applying seasonal adjustment factors

Step 2 cannot be done with a simple window function -- it requires generating future period rows (via a date/number table or CTE) and then applying the computed trend/seasonal factors. Many SQL forecasting tutorials only show step 1 and leave the reader to figure out steps 2-3.

**Prevention:**
- Use a CTE to generate future period rows before applying the forecast formula:
  ```sql
  ;WITH Months AS (
    SELECT 1 AS MonthNum UNION ALL SELECT MonthNum + 1 FROM Months WHERE MonthNum < 24
  ),
  HistoricalData AS (
    -- Actual monthly revenue
  ),
  Trend AS (
    -- Moving average and growth rate from historical data
  )
  SELECT ... FROM Months LEFT JOIN HistoricalData ...
  ```
- Keep the forecast SQL simple: 3-month or 6-month trailing average projected forward, optionally with YoY seasonal ratio
- Do NOT attempt Holt-Winters exponential smoothing in pure SQL -- it requires iterative computation that is fragile in T-SQL
- Validate the forecast against the last known actual period: if the model would have predicted last month within 15%, the methodology is reasonable

**Detection:** If the forecast query returns NULL or zero for future periods, the projection step is missing.

**Severity:** MEDIUM -- Produces missing forecasts rather than wrong ones. The user gets "no data" for future periods instead of a projection.

**Phase:** Forecasting phase.

---

### Pitfall M5: Payroll GL Expense Queries Pick Up Non-Payroll Entries

**What goes wrong:** "Total payroll expense" includes non-payroll items from the same GL account categories, or misses payroll items that are posted to unexpected accounts.

**Why it happens:** Payroll expense at FLS is queried via GL entries (per the project scope -- no deep TimeCard/Payroll tables). But GL expense accounts (GLClassificationType 5002) contain both payroll-related entries (salaries, taxes, benefits) and non-payroll expenses (rent, insurance, utilities). Without filtering to payroll-specific GL account NodeIDs, a "payroll expense" query returns all operating expenses.

Additionally, the Ledger table has `PayrollID` and `EmployeeID` fields that can help identify payroll-related entries, but these are only populated for entries created by the payroll module -- manual payroll adjustments may not have these fields set.

**Prevention:**
- Map the specific GL account NodeIDs that represent payroll at FLS (need to query: `SELECT ID, AccountName FROM GLAccount WHERE AccountName LIKE '%payroll%' OR AccountName LIKE '%salary%' OR AccountName LIKE '%wage%' OR AccountName LIKE '%benefit%'`)
- Use `Ledger.PayrollID IS NOT NULL` as a secondary filter to identify payroll-originated entries
- For "total payroll expense," sum the mapped payroll GL accounts, not all 5002-classified accounts
- Present the breakdown: "Salaries: $X, Taxes: $Y, Benefits: $Z, Total Payroll: $Sum" so the user can verify reasonableness
- Cross-validate: Carrie Goetelman runs payroll in Control and can confirm approximate totals

**Detection:** If "payroll expense" is close to "total operating expenses," the filter is too broad. FLS payroll should be a fraction of the ~$3M revenue, not close to total OpEx.

**Severity:** MEDIUM -- Wrong payroll numbers, but the payroll expense feature is explicitly lightweight ("GL-based, no deep TimeCard/Payroll tables"), so expectations are moderate.

**Phase:** Payroll expense phase (likely a smaller sub-phase).

---

### Pitfall M6: Account.DivisionID Does Not Always Match TransHeader.DivisionID

**What goes wrong:** A customer intelligence query filtered by "Apparel Division customers" uses `Account.DivisionID` and returns a different customer list than "Apparel Division orders" which uses `TransHeader.DivisionID`. Customer X appears in Apparel's customer list but their orders show up in the Company Division.

**Why it happens:** `Account.DivisionID` is the default division assigned to a customer account. `TransHeader.DivisionID` is the division assigned to a specific order, which can be overridden per-order (controlled by `TransHeader.DivisionIDOverridden`). A customer assigned to the Company Division could place an order manually assigned to the Apparel Division, or vice versa.

**Consequences:** Division-filtered customer lists and division-filtered order lists disagree about which customers belong to which division. Dashboard showing "Apparel Division: 50 customers, $200K revenue" but querying those 50 customers' orders shows $180K because some of their orders were tagged to the Company Division.

**Prevention:**
- For customer-level division filtering: use `Account.DivisionID` with COALESCE (answers "which customers are assigned to Apparel")
- For order-level division filtering: use `TransHeader.DivisionID` with COALESCE (answers "which orders are attributed to Apparel")
- For "Apparel Division revenue by customer": use `TransHeader.DivisionID` on the order, not `Account.DivisionID` on the customer
- Document the distinction: "Division customers" != "customers with division orders"
- The dashboard should be consistent: if it shows "Apparel Division" context, ALL metrics should use order-level division (TransHeader.DivisionID), not customer-level division

**Detection:** Compare customer counts between Account.DivisionID-filtered and TransHeader.DivisionID-filtered queries. Differences indicate customers with cross-division orders.

**Severity:** MEDIUM -- Confusing but bounded. The delta is likely small (most customers probably order within their assigned division).

**Phase:** Division filtering framework, specifically the customer integration layer.

---

### Pitfall M7: Dashboard Text Summary and React Visual Show Different Time Ranges

**What goes wrong:** The morning text report says "February revenue: $180K" but the React chart shows a bar for February at $195K. The text used MTD through yesterday, the chart used MTD through today (or vice versa), or the text uses fiscal month boundaries while the chart uses calendar month boundaries.

**Why it happens:** The dashboard has two output modes: text summary and React visual. If they are generated by different queries or at different times, they can use different date boundaries. Additionally, if "this month" is computed differently (`MONTH(GETDATE()) = MONTH(SaleDate)` vs `SaleDate >= DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()), 0) AND SaleDate < GETDATE()`), the results diverge.

**Consequences:** The user reads the text summary and then looks at the chart. If numbers differ, they wonder which is right.

**Prevention:**
- Define a single date computation for each time period and share it between text and visual:
  ```sql
  DECLARE @MTDStart DATETIME = DATEADD(MONTH, DATEDIFF(MONTH, 0, GETDATE()), 0);
  DECLARE @MTDEnd DATETIME = GETDATE();
  ```
- Both text and chart queries use the same @MTDStart and @MTDEnd
- Include the date range in both outputs: "Revenue MTD (Feb 1-9): $180K"
- The React artifact receives the same data array that the text summary was generated from -- do not re-query

**Detection:** Compare any metric that appears in both text and chart. They should be identical.

**Severity:** MEDIUM -- Confusing but easily caught and fixed.

**Phase:** Dashboard phase.

---

## Minor Pitfalls

Mistakes that cause annoyance but are quickly fixable.

---

### Pitfall m1: DivisionData Table Has a System Row That Must Be Excluded

**What goes wrong:** Division queries include the system/default row (ID = -1, DivisionName = '.', IsActive = false) in results, showing an extra "division" that does not exist.

**Prevention:** Always filter `WHERE IsActive = 1 AND ID > 0` when querying DivisionData. The sample data shows ID=-1 is a system placeholder row (IsSystem=true, IsActive=false, DivisionName='.').

**Severity:** LOW -- Extra row in results, not wrong data.

---

### Pitfall m2: SalesGoal Table Exists but May Not Be Division-Aware

**What goes wrong:** The forecasting skill uses the SalesGoal table (248 rows, with GoalYear/GoalMonth/Goal columns) for comparison but does not realize SalesGoal has no DivisionID column. Goals cannot be broken down by division unless they are duplicated per division.

**Prevention:** Verify SalesGoal structure before building goal-vs-actual comparisons. If SalesGoal lacks DivisionID, it represents company-wide goals only. Division-level goal comparison would require either a new data source or acknowledging the limitation.

**Severity:** LOW -- Missing feature, not wrong data.

---

### Pitfall m3: React Artifact Recharts Number Formatting Differences

**What goes wrong:** The React chart shows "$3052952.52" instead of "$3,052,953" because Recharts does not automatically format currency values. Or percentages show as "0.147" instead of "14.7%".

**Prevention:** Always include number formatters in the React artifact:
- Currency: `new Intl.NumberFormat('en-US', {style: 'currency', currency: 'USD', maximumFractionDigits: 0}).format(value)`
- Percentage: `(value * 100).toFixed(1) + '%'`
- Apply formatters to Recharts `tickFormatter`, `labelFormatter`, and tooltip props

**Severity:** LOW -- Cosmetic issue, easily fixed.

---

### Pitfall m4: Trend Detection Confuses Customer Concentration Changes with Market Trends

**What goes wrong:** The trend analysis shows "Swing Flags declining 15% YoY" when the actual situation is "FLASH (our largest customer) reduced their Swing Flag orders" -- it is a customer-specific change, not a market trend. The system presents it as a product line trend without context.

**Prevention:** For any product line showing >10% change, drill into whether the change is driven by a small number of customers or a broad pattern. Report both the trend and the concentration: "Swing Flags -15% YoY. Top driver: FLASH reduced orders by $50K (60% of the decline)."

**Severity:** LOW -- Misleading but not numerically wrong. Better context prevents misinterpretation.

---

### Pitfall m5: Warehouse DivisionID Is Not Nullable (Unlike Other Tables)

**What goes wrong:** A division filtering query applies the `COALESCE(DivisionID, 10)` pattern to `Warehouse.DivisionID`, but Warehouse.DivisionID is NOT nullable (defined as `int NOT NULL`). The COALESCE is unnecessary but harmless. However, this creates a false pattern expectation that Warehouse follows the same rules as other tables.

**Prevention:** Note that Warehouse.DivisionID is a reliable, non-nullable FK. Use it directly: `WHERE Warehouse.DivisionID = @DivisionID`. The COALESCE pattern is needed for TransHeader, Ledger, Account, Inventory, but NOT Warehouse.

**Severity:** LOW -- No functional impact, but documents a schema inconsistency that could cause confusion.

---

## Integration Pitfalls

Mistakes specific to adding v1.2 features alongside the existing 8-skill system.

---

### Pitfall I1: Forecasting Skill Creates a Parallel Revenue Calculation Path

**What goes wrong:** The forecasting skill builds its own "historical revenue by month" query as a foundation for projections, and this query produces slightly different monthly totals than the existing sales skill's "Monthly Revenue Trend" template (Template 2). Now there are two different sources of truth for monthly revenue.

**Why it happens:** The forecasting skill needs historical monthly revenue as input. It is tempting to write a fresh query optimized for forecasting (e.g., including Type 2 estimates as "pipeline" or using a different date aggregation). But any deviation from the core revenue formula creates inconsistency.

**Prevention:**
- The forecasting skill MUST use the EXACT same monthly revenue query as the sales skill (Template 2)
- If the forecast needs additional data (pipeline estimates, etc.), those should be SEPARATE from the base revenue series
- Validation: the "actuals" portion of the forecast must match the sales skill's monthly trend output exactly, to the penny
- Structure the forecast as: `actuals (from core revenue formula) + projection (computed from actuals)`

**Severity:** MEDIUM -- Silent inconsistency between skills. Detectable by comparing forecast actuals with sales skill actuals.

**Phase:** Forecasting phase.

---

### Pitfall I2: Division Filtering Breaks Existing Validated Queries

**What goes wrong:** Adding division filtering to the revenue query introduces a regression: the undivided total no longer matches $3,052,952.52 because the COALESCE pattern or JOIN to DivisionData changes the query behavior.

**Why it happens:** The validated revenue formula is simple:
```sql
SELECT SUM(SubTotalPrice) FROM TransHeader
WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL AND YEAR(SaleDate) = 2025
```
Adding division filtering means either:
- Adding `AND COALESCE(DivisionID, 10) = @DivisionID` (changes results if @DivisionID is not set)
- Adding a JOIN to DivisionData (could exclude records with no matching DivisionData row)

If the "all divisions" case is not carefully handled (e.g., omitting the WHERE clause when no division is selected, or using `@DivisionID = -1` to mean "all"), the base case regresses.

**Prevention:**
- Division filtering MUST be additive: when no division is specified, the query should be IDENTICAL to the existing validated formula
- Implementation pattern: `AND (@DivisionID = -1 OR COALESCE(th.DivisionID, 10) = @DivisionID)` where -1 means "all divisions"
- This matches the Cyrious wiki pattern: `?Division_DivisionID = -1 or ?Division_DivisionID = Ledger.DivisionID`
- After adding division filtering, re-run the validation suite. The "all divisions" results must still match $3,052,952.52 for 2025.
- DO NOT use a JOIN to DivisionData for filtering (it would exclude orphaned records). Use a WHERE clause.

**Detection:** Re-run the 21-test validation suite after adding division filtering. Any regression is immediately visible.

**Severity:** MEDIUM -- Detectable through existing validation, but would block delivery until fixed.

**Phase:** Division filtering framework phase. Must include regression testing.

---

### Pitfall I3: Dashboard NL Routing Conflicts with Existing Skill Routing

**What goes wrong:** A user asks "show me this month's revenue" and the system routes to the dashboard skill (showing a React chart) when the user wanted a simple number from the sales skill. Or the user asks "morning report" and the system does not know whether to use the dashboard skill's text summary or the glossary's routing table.

**Why it happens:** The glossary skill (control-erp-glossary) has 75 NL routing entries that map natural language to existing skills. Adding a dashboard skill creates routing ambiguity: "monthly revenue" could go to sales (number) or dashboard (chart). "Morning report" is a new concept that does not exist in current routing.

**Prevention:**
- Define explicit routing triggers for the dashboard skill that do NOT overlap with existing skills:
  - Dashboard triggers: "morning report", "executive summary", "dashboard", "visual report", "chart of...", "show me a chart"
  - Sales triggers (unchanged): "monthly revenue", "sales this month", "revenue trend"
- When ambiguous, default to the text-based skill (simpler, faster, validated). Only route to dashboard when the user explicitly requests a visual or dashboard.
- Update the glossary routing table with dashboard entries
- Test with the 25 existing routing test queries to ensure no regressions

**Detection:** Run the 25 routing test queries after adding dashboard entries. Any routing change is a regression.

**Severity:** LOW -- Annoyance (wrong output format), not wrong data. User can clarify.

**Phase:** Dashboard phase. Must update glossary routing table.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation | Severity |
|-------------|---------------|------------|----------|
| Division framework | C1 (NULL DivisionID exclusion) | COALESCE(DivisionID, 10) everywhere | CRITICAL |
| Division framework | C2 (Wrong DivisionID field) | Map which field per table per query type | HIGH |
| Division framework | I2 (Regression of validated queries) | Re-run 21-test suite after every change | MEDIUM |
| Division + Financial | C3 (DivisionID vs ProcessedDivisionID) | Use DivisionID, not ProcessedDivisionID, for financial | HIGH |
| Division + Financial | C6 (GL vs TransHeader division disagreement) | Document expected variance, set tolerance | HIGH |
| Division + Customers | M6 (Account vs TransHeader DivisionID) | Use TransHeader.DivisionID for order-level metrics | MEDIUM |
| Division + Inventory | M2 (Inconsistent cross-skill filtering) | Division filter framework consumed by all skills | MEDIUM |
| Forecasting | C4 (Insufficient data for reliable forecasts) | Moving averages only, show confidence context | HIGH |
| Forecasting | M1 (Seasonal overfitting) | Min 36 months before claiming seasonal patterns | MEDIUM |
| Forecasting | I1 (Parallel revenue calculation) | Use core revenue formula for actuals portion | MEDIUM |
| Forecasting | M4 (Window function projection gap) | Generate future period rows via CTE, then apply trend | MEDIUM |
| Dashboard (text + visual) | C5 (Inconsistent metrics vs skills) | Same SQL templates for dashboard and skills | CRITICAL |
| Dashboard (visual) | M3 (Unsupported React libraries) | Only Recharts + Tailwind + Shadcn + Lucide | MEDIUM |
| Dashboard (text + visual) | M7 (Divergent time ranges) | Single date computation shared by both outputs | MEDIUM |
| Dashboard (routing) | I3 (NL routing conflicts) | Explicit dashboard triggers, no overlap with existing | LOW |
| Payroll GL | M5 (Non-payroll entries in payroll query) | Map specific payroll GL NodeIDs | MEDIUM |
| Trend detection | m4 (Customer concentration vs market trend) | Drill into top-customer contribution for any >10% change | LOW |

---

## Discovery Tasks Required Before Implementation

These are unknowns that MUST be resolved by querying the live database before writing query templates. Failure to resolve these creates risk of building on wrong assumptions.

| Question | Query | Why It Matters |
|----------|-------|---------------|
| What are the actual DivisionData IDs and names at FLS? | `SELECT ID, DivisionName, IsActive FROM DivisionData WHERE ID > 0` | We refer to "10" as default but have not confirmed the actual Apparel Division ID |
| How many TransHeader records have NULL DivisionID? | `SELECT COUNT(*) FROM TransHeader WHERE DivisionID IS NULL AND TransactionType=1 AND IsActive=1` | Quantifies the C1 pitfall severity |
| How many GL entries have NULL DivisionID? | `SELECT COUNT(*) FROM GL WHERE DivisionID IS NULL` | Quantifies the C1 pitfall for financial queries |
| Do DivisionID and ProcessedDivisionID usually agree in Ledger? | `SELECT COUNT(*) FROM Ledger WHERE DivisionID != ProcessedDivisionID AND DivisionID IS NOT NULL AND ProcessedDivisionID IS NOT NULL` | Quantifies the C3 pitfall severity |
| Which GL accounts are payroll-related? | `SELECT ID, AccountName FROM GLAccount WHERE ClassTypeID=8001 AND (AccountName LIKE '%payroll%' OR AccountName LIKE '%salary%' OR AccountName LIKE '%wage%' OR AccountName LIKE '%benefit%')` | Required for M5 (payroll GL mapping) |
| Does Apparel Division have enough history for forecasting? | `SELECT YEAR(SaleDate) Y, MONTH(SaleDate) M, COUNT(*), SUM(SubTotalPrice) FROM TransHeader WHERE TransactionType=1 AND IsActive=1 AND SaleDate IS NOT NULL AND COALESCE(DivisionID,10) != 10 GROUP BY YEAR(SaleDate), MONTH(SaleDate) ORDER BY Y, M` | Determines if per-division forecasting is feasible (C4) |
| How many months of total history exist? | `SELECT MIN(SaleDate), MAX(SaleDate) FROM TransHeader WHERE TransactionType=1 AND IsActive=1 AND SaleDate IS NOT NULL` | Determines forecast methodology ceiling |

---

## Sources

- `/Users/cain/projects/control-db-map/output/schemas/TransHeader.md` -- TransHeader schema with DivisionID, ProductionDivisionID, ShipFromDivisionID fields (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/schemas/Ledger.md` -- Ledger schema with DivisionID and ProcessedDivisionID fields (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/schemas/DivisionData.md` -- DivisionData table: 3 rows, nullable columns documented (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/schemas/Account.md` -- Account.DivisionID FK to DivisionData (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/schemas/Warehouse.md` -- Warehouse.DivisionID NOT NULL (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/sql_queries_reference.md` -- 40+ wiki SQL scripts using COALESCE(DivisionID, 10) pattern (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/howto_troubleshooting_knowledge.md` -- "Always use COALESCE(DivisionID, 10)" rule, common gotchas (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/support/external_exception_when_printing.md` -- NULL DivisionID in Ledger documented as known issue (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/control/control_release_notes_6.1.2107.0701.md` -- CCON-5479 ProcessedDivisionID bug documented (HIGH confidence)
- `/Users/cain/projects/control-db-map/skills/control-erp-core/control-erp-core-SKILL.md` -- Validated revenue formula, business rules (HIGH confidence)
- `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` -- GL sign conventions, Ledger field reference (HIGH confidence)
- `/Users/cain/projects/control-db-map/skills/control-erp-sales/control-erp-sales-SKILL.md` -- Monthly revenue trend template (HIGH confidence)
- `/Users/cain/projects/control-db-map/skills/control-erp-glossary/control-erp-glossary-SKILL.md` -- NL routing table, division glossary entry (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/skill/references/field_values.md` -- Warehouse IDs by division, DivisionID reference (HIGH confidence)
- `/Users/cain/projects/control-db-map/.planning/PROJECT.md` -- v1.2 requirements, project context (HIGH confidence)
- [Forecasting with SQL - SQLServerCentral](https://www.sqlservercentral.com/articles/forecasting-with-sql) -- SQL Server forecasting patterns (MEDIUM confidence, community source)
- [Time Series Forecasting in SQL with Moving Averages - Towards Data Science](https://towardsdatascience.com/how-to-conduct-time-series-forecasting-in-sql-with-moving-averages-fd5e6f2a456/) -- Window function patterns for forecasting (MEDIUM confidence, community source)
- [Silota: Ratio to Moving Average Seasonal Index](http://www.silota.com/docs/recipes/sql-ratio-to-moving-average-seasonal-index.html) -- Seasonal decomposition in SQL (MEDIUM confidence, tutorial)
- [Claude Artifacts limitations - Medium](https://medium.com/@intranetfactory/claude-artifacts-a-game-changer-held-back-by-frustrating-limits-6adcacdd95a7) -- React artifact sandbox constraints (MEDIUM confidence, Jan 2025)
- [Common Data Analysis Mistakes - NetSuite](https://www.netsuite.com/portal/resource/articles/data-warehouse/data-mistakes.shtml) -- General analytics pitfalls (MEDIUM confidence, vendor source)
- [Biggest Forecasting Mistakes 2026 - Mosaic](https://www.mosaicapp.com/post/the-biggest-forecasting-mistakes-leaders-still-make-in-2026) -- Forecasting methodology pitfalls (MEDIUM confidence, vendor source)
