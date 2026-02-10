# Technology Stack: v1.2 Analytics, Dashboards & Division Support

**Project:** Project Cornerstone -- Control ERP Natural Language Interface
**Researched:** 2026-02-09
**Mode:** Ecosystem (subsequent milestone -- stack additions for analytics layer)
**Overall Confidence:** HIGH (all recommendations work within proven constraints)

---

## Context: What Already Exists (DO NOT CHANGE)

| Component | Status | Lines |
|-----------|--------|-------|
| SQL Server (StoreData) via MCP MSSQL tools | Validated | Core access |
| 8 YAML SKILL.md files + references/ | Validated | ~4,838 lines |
| Core skill: TransactionType, StatusID, pricing | 99.98% validated | Foundation |
| Sales skill: DyeSub container, product categories | Validated | Revenue queries |
| Financial skill: GL/Ledger, AR/AP, P&L, payments, cash flow | Validated | 888 + 412 lines |
| Customers skill: profiles, RFM, churn detection | Validated | 1,075 lines |
| Inventory skill: stock, reorder, purchasing | Validated | 663 lines |
| Production skill: artwork, stations, dwell time | Validated | 1,095 lines |
| Glossary skill: 75 NL routes, Crystal Reports catalog | Validated | 671 + 35 lines |
| 187 table schemas in output/schemas/ | Complete | Reference |
| Wiki knowledge: 1,769 pages, 12 extracts | Complete | Reference |

**Constraint: No external services, no Python ML, no pip installs.** This system runs as Claude Code skills generating SQL via MCP tools and formatting results as markdown or React artifacts. All analytics must be pure T-SQL.

---

## DECISION 1: Sales Forecasting Approach

### Recommendation: SQL-Based Linear Regression + Seasonal Adjustment

**Use pure T-SQL linear regression with CTE-based seasonal indices.** No external libraries needed.

**Why this works for FLS:**
- 2+ years of monthly sales data provides sufficient baseline for trend + seasonality
- $3M revenue business does not need ML-grade precision -- directional accuracy matters
- Claude interprets the SQL results and adds narrative context ("Q3 typically dips 15%")
- The forecasting SQL is a template in the skill file; Claude fills parameters at query time

**Confidence: HIGH** -- Linear regression is well-documented in pure T-SQL. The formulas are standard statistical calculations using SUM, AVG, COUNT, and POWER aggregates.

### Core SQL Pattern: Linear Regression

The slope/intercept calculation uses the least-squares formula in pure T-SQL:

```sql
-- Step 1: Aggregate monthly history
WITH MonthlyRevenue AS (
    SELECT
        YEAR(SaleDate) * 12 + MONTH(SaleDate) AS MonthIndex,
        YEAR(SaleDate) AS SaleYear,
        MONTH(SaleDate) AS SaleMonth,
        SUM(SubTotalPrice) AS Revenue,
        COUNT(*) AS OrderCount
    FROM TransHeader
    WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL
    GROUP BY YEAR(SaleDate), MONTH(SaleDate)
),
-- Step 2: Calculate regression components
RegressionInputs AS (
    SELECT
        MonthIndex AS X,
        Revenue AS Y,
        SaleYear, SaleMonth,
        COUNT(*) OVER () AS N,
        AVG(CAST(MonthIndex AS FLOAT)) OVER () AS X_bar,
        AVG(Revenue) OVER () AS Y_bar
    FROM MonthlyRevenue
    WHERE MonthIndex >= (YEAR(GETDATE()) - 2) * 12  -- last 2 years
),
-- Step 3: Calculate slope and intercept
Regression AS (
    SELECT
        SUM((X - X_bar) * (Y - Y_bar)) / NULLIF(SUM((X - X_bar) * (X - X_bar)), 0) AS Slope,
        AVG(Y) - (SUM((X - X_bar) * (Y - Y_bar)) / NULLIF(SUM((X - X_bar) * (X - X_bar)), 0)) * AVG(CAST(X AS FLOAT)) AS Intercept
    FROM RegressionInputs
)
-- Step 4: Project future months
SELECT ... FROM Regression CROSS JOIN future_months
```

### Seasonal Adjustment Pattern

Apply seasonal indices to the linear trend for more accurate month-level forecasts:

```sql
-- Calculate seasonal index per month (ratio of actual to trend)
WITH SeasonalIndex AS (
    SELECT
        MONTH(SaleDate) AS MonthNum,
        AVG(SubTotalPrice_Monthly) / AVG(SubTotalPrice_Monthly) OVER () AS SeasonalFactor
    FROM monthly_aggregated_data
    GROUP BY MONTH(SaleDate)
)
-- Adjusted forecast = LinearTrend * SeasonalFactor
```

**What Claude does with results:** Claude runs the regression SQL, gets the projected values, and provides narrative interpretation: "Based on the past 24 months, Q2 2026 is projected at $820K (up 8% YoY). Note: Q2 is historically your strongest quarter due to seasonal demand for outdoor flags."

### Alternatives Considered and Rejected

| Approach | Why Rejected |
|----------|-------------|
| Python scikit-learn / statsmodels | Cannot run Python in skill execution context; skills generate SQL only |
| SQL Server R/Python services | Not available on FLS's SQL Server instance; enterprise feature |
| Moving average only (no regression) | Too simplistic; doesn't capture growth trend, only smoothing |
| External API forecasting service | Adds dependency; latency; unnecessary for this scale |
| Exponential smoothing in SQL | Possible but recursive CTEs are fragile and hard to debug; linear regression + seasonal indices achieves similar accuracy for FLS's use case |

### Data Requirements for Forecasting

FLS has the data needed. Verified against existing schema:

| Requirement | Available | Source |
|-------------|-----------|--------|
| Monthly revenue, 24+ months | YES | TransHeader.SaleDate + SubTotalPrice |
| Order count by month | YES | COUNT(*) from TransHeader |
| Revenue by product category | YES | TransDetailParam VariableID 11053 |
| Revenue by division | YES | TransHeader.DivisionID -> DivisionData |
| Average order value trend | YES | Derived from revenue / count |

---

## DECISION 2: Trend Detection Approach

### Recommendation: Window Functions for Growth Rates + LAG/LEAD Comparisons

**Use T-SQL window functions (LAG, LEAD, running averages) for trend detection.** Claude interprets the numerical output and provides the narrative.

**Confidence: HIGH** -- Window functions are core T-SQL, already used throughout existing skills (LEAD() in production skill for dwell time).

### Core Patterns

**1. Month-over-Month Growth:**
```sql
SELECT
    SaleYear, SaleMonth, Revenue,
    LAG(Revenue) OVER (ORDER BY SaleYear, SaleMonth) AS PrevMonth,
    (Revenue - LAG(Revenue) OVER (ORDER BY SaleYear, SaleMonth))
        / NULLIF(LAG(Revenue) OVER (ORDER BY SaleYear, SaleMonth), 0) * 100 AS MoM_Growth_Pct
FROM MonthlyRevenue
```

**2. Year-over-Year by Month (seasonal comparison):**
```sql
SELECT
    SaleYear, SaleMonth, Revenue,
    LAG(Revenue, 12) OVER (ORDER BY SaleYear, SaleMonth) AS SameMonthPriorYear,
    (Revenue - LAG(Revenue, 12) OVER (ORDER BY SaleYear, SaleMonth))
        / NULLIF(LAG(Revenue, 12) OVER (ORDER BY SaleYear, SaleMonth), 0) * 100 AS YoY_Growth_Pct
FROM MonthlyRevenue
```

**3. Rolling 3-Month Average (smoothed trend):**
```sql
SELECT
    SaleYear, SaleMonth, Revenue,
    AVG(Revenue) OVER (ORDER BY SaleYear, SaleMonth ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS Rolling3Mo
FROM MonthlyRevenue
```

**4. Product Line Trend (growth/decline detection):**
```sql
-- Flag categories growing or declining >15% YoY
SELECT Category,
    CurrYearRevenue, PriorYearRevenue,
    CASE
        WHEN YoY_Growth > 15 THEN 'GROWING'
        WHEN YoY_Growth < -15 THEN 'DECLINING'
        ELSE 'STABLE'
    END AS Trend
FROM product_yoy_comparison
```

**What Claude does with results:** Claude runs the trend SQL, then provides executive-level interpretation: "Feather Flags are up 23% YoY -- your fastest growing category. Banners are down 8% -- worth investigating whether this is seasonal or a structural shift."

### What NOT to Build

| Anti-Pattern | Why Avoid |
|-------------|-----------|
| Anomaly detection algorithms | Overkill; Claude can spot outliers from the data and explain them |
| Automated alerting system | No persistent process; this is query-on-demand |
| Predictive churn scoring | Already handled by customers skill (dormancy detection with AVG + 1.5x STDEV) |
| Automated report generation on schedule | Claude Code is interactive, not a cron job |

---

## DECISION 3: Executive Dashboard Approach

### Recommendation: Two-Tier Output -- Text Summary + React Artifact

**Tier 1: Text-based "Morning Report"** -- formatted markdown summary that Claude produces from SQL results. Always available, works everywhere.

**Tier 2: React Artifact Dashboard** -- interactive visual dashboard using Recharts in a React artifact. Available in Claude.ai and Claude desktop app. NOT available in Claude Code terminal.

**Confidence: MEDIUM-HIGH** -- React artifacts with Recharts are well-documented and widely used. The uncertainty is whether the user primarily uses Claude.ai (artifacts available) or Claude Code terminal (artifacts not rendered).

### Text Dashboard (Tier 1) -- Always Works

Claude runs 4-6 SQL queries, then formats results as a structured markdown summary:

```markdown
## FLS Banners Morning Report -- Feb 10, 2026

### Revenue Snapshot
- **MTD:** $142,350 (23 orders) -- on pace for $285K month
- **YTD:** $412,800 -- 13.5% of $3.05M annual target
- **vs Prior Year:** +11.2% YoY for same period

### Open Work
- **Orders in WIP:** 34 orders, $187,200 total value
- **Past Due:** 3 orders, $12,400 (oldest: 5 days past due)
- **Artwork Pending:** 8 proofs awaiting approval

### Alerts
- AR over 60 days: $23,400 (2 customers)
- Feather Flags trending +28% MoM -- potential capacity concern
- Apparel Division: $18,200 MTD (Gretel's dashboard)
```

**No new libraries needed.** This is Claude formatting SQL results into markdown -- the pattern used by all 8 existing skills.

### React Artifact Dashboard (Tier 2) -- Visual

The React artifact sandbox provides these pre-installed libraries:

| Library | Purpose | Status |
|---------|---------|--------|
| React 18 | Component framework | Pre-installed |
| Recharts | Charts (bar, line, pie, area, composed) | Pre-installed |
| Tailwind CSS | Styling | Pre-installed |
| Shadcn UI | Pre-built components (cards, tables, badges) | Pre-installed |
| Lucide React | Icons | Pre-installed |
| D3.js | Advanced visualizations (if Recharts insufficient) | Available via cdnjs |

**Sandbox constraints that matter:**
- No external API calls (cannot fetch data from inside artifact)
- No localStorage/sessionStorage
- No npm install -- only pre-installed packages
- External scripts only from cdnjs.cloudflare.com
- Single-file React component (no multi-file)
- Data must be embedded in the artifact as constants

**How it works in practice:**
1. Claude runs SQL queries via MCP tools
2. Claude receives the result data
3. Claude generates a React artifact with the data embedded as `const` values
4. The artifact renders charts, tables, KPI cards using Recharts + Shadcn UI + Tailwind

**Example artifact structure:**
```tsx
// Claude generates this as a React artifact
const REVENUE_DATA = [
  { month: 'Jan', revenue: 245000, priorYear: 220000 },
  { month: 'Feb', revenue: 268000, priorYear: 235000 },
  // ... data from SQL queries embedded here
];

const PRODUCT_MIX = [
  { name: 'Swing Flags', value: 615203 },
  { name: 'Feather Flags', value: 485666 },
  // ...
];

export default function Dashboard() {
  return (
    <div className="p-4 space-y-6">
      {/* KPI Cards */}
      <div className="grid grid-cols-4 gap-4">
        <Card><CardHeader>MTD Revenue</CardHeader>...</Card>
        ...
      </div>
      {/* Revenue Trend Chart */}
      <LineChart data={REVENUE_DATA}>
        <Line dataKey="revenue" />
        <Line dataKey="priorYear" strokeDasharray="5 5" />
      </LineChart>
      {/* Product Mix Pie Chart */}
      <PieChart data={PRODUCT_MIX}>...</PieChart>
    </div>
  );
}
```

### Dashboard Components to Build

| Component | Chart Type | Data Source |
|-----------|-----------|-------------|
| Revenue trend (MTD/QTD/YTD) | LineChart with prior year overlay | TransHeader monthly aggregation |
| Product mix breakdown | PieChart or horizontal BarChart | TransDetailParam VariableID 11053 |
| Division comparison | Stacked BarChart | TransHeader.DivisionID |
| AR aging waterfall | BarChart (stacked buckets) | TransHeader aging buckets |
| Order pipeline | Horizontal bar (WIP/Built/Sale) | TransHeader.StatusID counts |
| Forecast vs actual | AreaChart with projection shading | Regression output + actuals |
| Top customers | Table with sparklines | TransHeader by AccountID |

### Alternatives Considered and Rejected

| Approach | Why Rejected |
|----------|-------------|
| Mermaid diagrams | Static, no interactivity, poor for data visualization |
| SVG-only charts | Too much manual effort; Recharts handles this |
| External dashboarding tool (Metabase, Grafana) | Adds infrastructure; defeats purpose of NL interface |
| HTML artifact with Chart.js via CDN | Possible but Recharts is pre-installed and React-native |
| D3.js for everything | Overkill; Recharts covers 95% of needs with far less code |

---

## DECISION 4: Division Filtering Approach

### Recommendation: Add DivisionID Filter to Existing Query Templates

**Add an optional DivisionID JOIN/WHERE clause to existing sales, AR, and financial queries.** This is a pattern change to existing skills, not a new library or tool.

**Confidence: HIGH** -- DivisionID is a native column on TransHeader (FK to DivisionData.ID) and Ledger (FK to DivisionData.ID). The data structure is already there.

### Division Data Structure (Verified)

**DivisionData table:** 3 rows total (Row Count: 3)

| Column | Type | Purpose |
|--------|------|---------|
| ID | int | PK -- used as FK in TransHeader.DivisionID, Ledger.DivisionID |
| DivisionName | varchar(60) | "Banner Division", "Apparel Division", etc. |
| IsActive | bit | Active flag |

**Division references in key tables:**

| Table | Column | FK Verified |
|-------|--------|-------------|
| TransHeader | DivisionID | YES -- FK_TransHeader_DivisionID -> DivisionData.ID |
| TransHeader | ProductionDivisionID | YES -- FK_TransHeader_ProductionDivisionID -> DivisionData.ID |
| TransDetail | ProductionDivisionID | YES -- FK_TransDetail_ProductionDivisionID -> DivisionData.ID |
| Ledger | DivisionID | YES -- FK_Ledger_DivisionID -> DivisionData.ID |
| Ledger | ProcessedDivisionID | Present (no FK but same pattern) |
| Journal | DivisionID | YES -- FK_Journal_DivisionID -> DivisionData.ID |

**Key insight:** `TransHeader.DivisionID` is the ORDER-level division assignment. `Ledger.DivisionID` carries division through to GL entries. This means both sales queries AND financial queries can be division-filtered.

### Implementation Pattern

**For sales queries (TransHeader-based):**
```sql
-- Add optional division filter to any existing query
-- @DivisionName is NULL for "all divisions" (default)
LEFT JOIN DivisionData dd ON th.DivisionID = dd.ID
WHERE ... existing filters ...
    AND (@DivisionName IS NULL OR dd.DivisionName LIKE '%' + @DivisionName + '%')
```

**For financial queries (GL/Ledger-based):**
```sql
-- Division filter on GL entries
LEFT JOIN DivisionData dd ON l.DivisionID = dd.ID
WHERE ... existing filters ...
    AND (@DivisionName IS NULL OR dd.DivisionName LIKE '%' + @DivisionName + '%')
```

**For AR queries (TransHeader-based):**
```sql
-- Same pattern -- AR uses TransHeader, which has DivisionID
LEFT JOIN DivisionData dd ON th.DivisionID = dd.ID
WHERE th.TransactionType = 1 AND th.IsActive = 1 AND th.BalanceDue > 0
    AND (@DivisionName IS NULL OR dd.DivisionName LIKE '%' + @DivisionName + '%')
```

### NL Routing for Division Queries

| User Says | Interpretation |
|-----------|---------------|
| "Apparel sales" / "Gretel's division" | Filter DivisionName LIKE '%Apparel%' |
| "Banner division revenue" | Filter DivisionName LIKE '%Banner%' |
| "Sales by division" | GROUP BY dd.DivisionName, no filter |
| "Apparel AR" / "what does Apparel owe" | AR query + DivisionName LIKE '%Apparel%' |
| "Revenue breakdown by division" | Revenue query + GROUP BY dd.DivisionName |
| "Compare divisions" | Side-by-side division aggregation |

### What NOT to Do

| Anti-Pattern | Why |
|-------------|-----|
| Create separate "Apparel skill" and "Banner skill" | Unnecessary duplication; same queries, different filter |
| Hardcode DivisionID values | Names are more maintainable; ID values might differ in other instances |
| Filter TransDetail.ProductionDivisionID for revenue | Use TransHeader.DivisionID -- that is the order's assigned division |
| Require division on every query | Default to all divisions when not specified; division is always optional |

### Division-Specific Validation Requirement

After implementation, validate:
1. SUM of division-filtered revenue = total revenue (no orphans with NULL DivisionID)
2. Apparel Division AR matches what Gretel sees in Control's AR report
3. GL entries by division sum to total (no division leakage)

---

## DECISION 5: Payroll Expense Approach

### Recommendation: GL-Based Expense Queries Using Existing Financial Skill Patterns

**Query payroll expenses through the GL (GLClassificationType 5002, Operating Expenses) where payroll-related accounts live.** Do NOT build out the Payroll/PayrollPaycheck/TimeCard tables.

**Confidence: HIGH** -- This is explicitly called out in PROJECT.md: "Deep HR/Payroll queries deferred until new time clock app integration. Payroll expense is queried via GL entries per pay period."

### Why GL, Not Payroll Tables

| Factor | GL Approach | Payroll Table Approach |
|--------|-------------|----------------------|
| Data availability | 2.7M+ Ledger entries, all payroll posts to GL | Payroll table has 1 row (essentially unused) |
| Accuracy | Matches what accountant sees on P&L | Would need to reconcile to GL anyway |
| Effort | Add 2-3 query templates to financial skill | Map 8+ tables (Payroll, PayrollPaycheck, PayrollPayItem, etc.) |
| Future-proof | GL entries persist regardless of time clock app changes | Payroll tables will change with new time clock app |
| Carrie's workflow | She runs payroll in Control; results post to GL automatically | Deep payroll queries not requested |

### Implementation

**Payroll expense from GL:**
```sql
-- Total payroll expense for a period
SELECT
    ga.AccountName,
    SUM(l.Amount) AS Amount
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 5002  -- Operating Expenses
    AND ga.AccountName LIKE '%Payroll%'
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY ga.AccountName
ORDER BY Amount DESC
```

**Payroll by pay period (uses Ledger.PayrollID to group):**
```sql
-- If Ledger entries have PayrollID populated, can group by pay period
SELECT
    l.PayrollID,
    MIN(l.EntryDateTime) AS PayDate,
    SUM(l.Amount) AS TotalPayrollExpense
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 5002
    AND ga.AccountName LIKE '%Payroll%'
    AND YEAR(l.EntryDateTime) = @Year
GROUP BY l.PayrollID
ORDER BY PayDate
```

**Payroll expense by division:**
```sql
-- Division-filtered payroll expense
SELECT
    dd.DivisionName,
    SUM(l.Amount) AS PayrollExpense
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
LEFT JOIN DivisionData dd ON l.DivisionID = dd.ID
WHERE ga.GLClassificationType = 5002
    AND ga.AccountName LIKE '%Payroll%'
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY dd.DivisionName
```

### Payroll-Related GL Account Names (to discover at build time)

The existing skill documents GLClassificationType 5002 as "Operating Expenses -- ~92 accounts (payroll, insurance, rent, etc.)". At build time, query:
```sql
SELECT ID, AccountName FROM GLAccount
WHERE GLClassificationType = 5002 AND AccountName LIKE '%Payroll%'
ORDER BY AccountName
```
This will reveal the exact account names (e.g., "Payroll - Wages", "Payroll - FICA", "Payroll - Benefits", etc.) to build precise queries.

---

## New Skill Architecture: Where These Go

### Option A (RECOMMENDED): Extend Existing Skills + Add Analytics Skill

| Feature | Where It Lives | Rationale |
|---------|---------------|-----------|
| Forecasting SQL templates | NEW `control-erp-analytics` skill | New capability, cross-domain |
| Trend detection templates | NEW `control-erp-analytics` skill | Cross-domain analysis |
| Dashboard templates | NEW `control-erp-analytics` skill | Presentation layer |
| Division filtering | MODIFY existing sales, financial, customers skills | Adds WHERE clause to existing queries |
| Payroll expense | MODIFY existing financial skill | Already has P&L, just add payroll specifics |
| NL routing updates | MODIFY glossary skill | Add 15-20 new routing entries |

**New skill: `control-erp-analytics`**
```
skills/control-erp-analytics/
  control-erp-analytics-SKILL.md     # Forecasting, trends, dashboard templates
  references/
    forecast-models.md               # Regression + seasonal SQL templates
    dashboard-components.md          # React artifact component library
```

**Why a separate analytics skill:**
- Forecasting and dashboards are cross-domain (pull from sales, financial, production)
- Dashboard generation is a presentation concern, not a data domain
- Keeps existing validated skills focused on their domain
- Analytics skill depends on all other skills' data but has its own query patterns

### Option B (REJECTED): Distribute Everything into Existing Skills

Would scatter forecasting across sales skill, financial skill, etc. Makes maintenance harder and loses the "dashboard as unified view" concept.

---

## Complete Stack for v1.2

### What Gets Added

| Component | Version | Purpose | Confidence |
|-----------|---------|---------|------------|
| T-SQL linear regression CTEs | N/A (pure SQL) | Sales forecasting | HIGH |
| T-SQL window functions (LAG/LEAD) | N/A (already used) | Trend detection | HIGH |
| Recharts | Pre-installed in artifacts | Dashboard charts | HIGH |
| Shadcn UI | Pre-installed in artifacts | Dashboard layout | HIGH |
| Tailwind CSS | Pre-installed in artifacts | Dashboard styling | HIGH |
| React 18 | Pre-installed in artifacts | Dashboard framework | HIGH |

### What Gets Modified

| Component | Change | Impact |
|-----------|--------|--------|
| `control-erp-sales` | Add DivisionID optional filter | ~20 lines per query template |
| `control-erp-financial` | Add DivisionID filter + payroll expense queries | ~50-80 lines |
| `control-erp-customers` | Add DivisionID filter (customer segmentation by division) | ~15 lines per query |
| `control-erp-glossary` | Add 15-20 NL routing entries for analytics/division | ~30-40 lines |

### What Does NOT Change

| Component | Why Unchanged |
|-----------|--------------|
| `control-erp-core` | Foundation -- no analytics concerns |
| `control-erp-inventory` | Division filtering lower priority (not in requirements) |
| `control-erp-production` | Division filtering lower priority (not in requirements) |
| MCP MSSQL tools | Same access pattern |
| Database schema | Read-only; no DDL changes |

---

## Installation

No installation needed. This milestone uses:
1. Pure T-SQL queries (no new database features)
2. Pre-installed React artifact libraries (Recharts, Shadcn, Tailwind)
3. Existing MCP MSSQL tools
4. New/modified SKILL.md files (markdown documents)

The "installation" is writing skill files and validating queries.

---

## Key Risks and Mitigations

| Risk | Severity | Mitigation |
|------|----------|------------|
| Forecast accuracy on 2 years of data | Medium | Clearly communicate confidence intervals; "projection" not "prediction" |
| DivisionID NULL on some TransHeaders | Medium | Validate at build time: count NULL DivisionID on Type 1 orders |
| React artifacts not available in Claude Code terminal | Low | Always produce text dashboard (Tier 1); artifact is enhancement |
| Payroll GL accounts not obviously named | Low | Discovery query at build time; document exact NodeIDs |
| Seasonal patterns weak with only 2 years | Medium | Use YoY comparison rather than formal decomposition if data insufficient |

---

## Sources

### SQL Forecasting
- [Calculate Moving Averages in SQL - GeeksforGeeks](https://www.geeksforgeeks.org/sql/calculate-moving-averages-in-sql/)
- [Statistics in SQL: Simple Linear Regressions - Simple Talk](https://www.red-gate.com/simple-talk/blogs/statistics-sql-simple-linear-regressions/)
- [MSSQLTips: T-SQL Windowing Functions for Moving Averages](https://www.mssqltips.com/sqlservertip/8124/calculate-a-moving-average-with-t-sql-windowing-functions/)
- [LearnSQL: Year-over-Year Difference in SQL](https://learnsql.com/blog/year-over-year-difference-sql/)
- [YOY Growth in SQL: The Ultimate Guide](https://runsql.com/blog/posts/2025-03-01-yoy-growth-guide/)

### React Artifacts
- [Reverse Engineering Claude Artifacts - Reid Barber](https://www.reidbarber.com/blog/reverse-engineering-claude-artifacts)
- [Claude Help Center: Artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them)
- [Claude Artifacts: Game Changer Held Back by Limits - Medium](https://medium.com/@intranetfactory/claude-artifacts-a-game-changer-held-back-by-frustrating-limits-6adcacdd95a7)
- [Claude Artifacts Runner - GitHub](https://github.com/claudio-silva/claude-artifact-runner)

### Existing System (Internal)
- TransHeader schema: `/Users/cain/projects/control-db-map/output/schemas/TransHeader.md` -- DivisionID FK verified
- DivisionData schema: `/Users/cain/projects/control-db-map/output/schemas/DivisionData.md` -- 3 rows
- Ledger schema: `/Users/cain/projects/control-db-map/output/schemas/Ledger.md` -- DivisionID FK verified
- Financial skill: `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` -- P&L with GLClassificationType 5002
- Sales skill: `/Users/cain/projects/control-db-map/skills/control-erp-sales/control-erp-sales-SKILL.md` -- existing query templates to modify
- PROJECT.md: `/Users/cain/projects/control-db-map/.planning/PROJECT.md` -- payroll deferral decision documented
