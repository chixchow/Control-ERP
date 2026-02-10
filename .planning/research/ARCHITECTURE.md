# Architecture Patterns: Analytics, Dashboards & Division Filtering

**Domain:** Control ERP Natural Language Interface (Analytics Layer, v1.2)
**Researched:** 2026-02-09
**Confidence:** HIGH for division filtering (verified against database schema with FK relationships), MEDIUM for forecasting (SQL patterns well-documented, needs calibration against FLS data), HIGH for dashboard architecture (React artifacts well-understood, skill integration patterns proven through 2 milestones).

---

## 1. EXISTING ARCHITECTURE (Post v1.1)

### Current Skill Topology

```
User (natural language)
    |
    v
[Claude reads relevant skills based on question context]
    |
    +-- control-erp-glossary (75 NL routes -> owning skill)
    |       |
    |       +-- Routes to --> control-erp-sales
    |       +-- Routes to --> control-erp-financial
    |       +-- Routes to --> control-erp-customers
    |       +-- Routes to --> control-erp-inventory
    |       +-- Routes to --> control-erp-production
    |       +-- Routes to --> control-erp-core
    |
    +-- control-erp-core (foundation: business rules, TransactionType, filters)
    |       ^
    |       |
    |       +-- Depended on by --> all 5 domain skills
    |
    +-- control-erp-sales (370 lines, 9 templates)
    +-- control-erp-financial (888 lines + 412 line reference)
    +-- control-erp-customers (1,075 lines, 20+ queries)
    +-- control-erp-inventory (663 lines, 22 queries)
    +-- control-erp-production (1,095 lines, 17 queries)
```

### Skill Size Profile (Current)

| Skill | Lines | Query Templates | Reference Files |
|-------|-------|-----------------|-----------------|
| control-erp-core | 347 | 6 | 0 |
| control-erp-sales | 370 | 9 | 0 |
| control-erp-financial | 888 | 14 | 1 (412 lines) |
| control-erp-customers | 1,075 | 20+ | 0 |
| control-erp-inventory | 663 | 22 | 0 |
| control-erp-production | 1,095 | 17 | 0 |
| control-erp-glossary | 671 | 0 | 1 (35 lines) |

**Total: ~5,109 lines across 7 SKILL.md files + 447 lines in references.**

### Architecture Constraints (Proven in v1.0 + v1.1)

1. **Context window budgeting.** Core + domain skill + glossary must fit in working context. Skills were split by domain for this reason. The largest skill (production at 1,095 lines) works fine.

2. **Core dependency is absolute.** TransactionType, StatusID, date filtering, pricing rules. Every domain skill reads core first.

3. **Glossary is the NL router.** Translates natural language to "which skill and which section." Does not contain query logic.

4. **Reference files work well.** Financial skill successfully uses `references/financial-analysis.md` (412 lines) to offload detailed queries while keeping the main SKILL.md at 888 lines. This pattern should be reused.

5. **Validation is per-skill against Control reports.** Each skill must prove accuracy before shipping.

6. **Single source of truth.** Business rules live in core only. Glossary routes, never duplicates. This prevents drift.

---

## 2. DIVISION FILTERING ARCHITECTURE

### Database Discovery: Division Structure at FLS

**DivisionData table (3 rows):**

| ID | DivisionName | IsActive | IsSystem | Notes |
|----|-------------|----------|----------|-------|
| -1 | . | false | true | System default (inactive, placeholder) |
| ? | Company | true | false | Banners/Signs division (primary) |
| ? | Apparel | true | false | Apparel division (Gretel manages) |

*Note: Actual DivisionData IDs for Company and Apparel need to be queried from the live database. The schema sample shows ID=-1 as the system default. The core skill mentions Company and Apparel divisions with distinct warehouse assignments.*

### Division FK Relationships (Verified from Schema)

Division links are present across multiple tables via FK relationships:

| Table | Column | FK Target | Notes |
|-------|--------|-----------|-------|
| **TransHeader** | `DivisionID` | DivisionData.ID | **Primary** -- every order has a division |
| **TransHeader** | `ProductionDivisionID` | DivisionData.ID | Can differ from DivisionID (production may happen in different division) |
| **TransHeader** | `ShipFromDivisionID` | DivisionData.ID | Shipping origin division |
| **Ledger** | `DivisionID` | DivisionData.ID | GL entries carry division |
| **Ledger** | `ProcessedDivisionID` | DivisionData.ID | Processing division (may differ) |
| **Account** | `DivisionID` | (inferred) | Customer default division assignment |
| **Warehouse** | `DivisionID` | DivisionData.ID | **FK verified** -- each warehouse belongs to a division |

**Key insight:** Division filtering is not a simple WHERE clause -- it touches TransHeader (orders), Ledger (GL), Account (customers), and Warehouse (inventory). This confirms division filtering is a **cross-cutting concern**.

### Division Filtering Design: Clause Injection Pattern

**Recommendation: Clause injection, not query duplication.**

Instead of writing separate "sales by division" and "sales total" query templates, add a division filter clause that can be injected into existing queries.

```
ARCHITECTURE:

Existing query template (sales skill Template 1):
  SELECT ... FROM TransHeader th
  WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) = @Year
    {{DIVISION_FILTER}}

Division filter clause:
  AND th.DivisionID = @DivisionID     -- when user specifies division
  (omit clause when "all" or unspecified)

Same pattern for GL queries:
  SELECT ... FROM GL l
  WHERE ...
    {{DIVISION_FILTER}}
  -->  AND l.DivisionID = @DivisionID
```

**Why this pattern:**

1. **No query duplication.** Every existing template gains division filtering with one additional WHERE clause. The 73+ existing SQL queries do not need to be rewritten.

2. **Opt-in, not required.** When the user does not mention a division, the clause is omitted and the query returns all-division data (current behavior). No regression.

3. **Composable with all query types.** The same clause applies to TransHeader queries (sales, customers, AR, production), Ledger queries (P&L, GL, cash flow), and can be extended to Inventory queries via Warehouse.DivisionID.

4. **NL trigger is simple.** The glossary needs to recognize "Apparel", "Banner", "Company", "by division" and inject the appropriate DivisionID.

### Division Filtering: Skill Placement

**Recommendation: Document division filtering in core skill, not a separate skill.**

Division filtering is a cross-cutting query modifier, like `IsActive = 1` or `SaleDate IS NOT NULL`. It belongs in the foundation (core) skill alongside other standard filters.

Add to `control-erp-core`:
```markdown
## Division Filtering

| Division | DivisionID | Warehouses | Primary Products |
|----------|-----------|-----------|-----------------|
| Company (Banners/Signs) | @CompanyDivID | 10, 11 | DyeSub, DyeLux, hardware |
| Apparel | @ApparelDivID | 10000, 10001 | Garments, embroidery |

### When to Apply Division Filtering

Apply `AND th.DivisionID = @DivisionID` (or `AND l.DivisionID = @DivisionID` for GL queries) when:
- User explicitly mentions "Apparel" or "Banner" division
- User asks for "by division" breakdown

When user does not mention division, omit the filter (returns all-division data).

### Division Filter Clauses

TransHeader queries:
  AND th.DivisionID = @DivisionID

Ledger/GL queries:
  AND l.DivisionID = @DivisionID

Account queries (customer default division):
  AND a.DivisionID = @DivisionID

Inventory queries (via warehouse):
  INNER JOIN Warehouse w ON i.WarehouseID = w.ID AND w.DivisionID = @DivisionID
```

**Impact on existing skills:** Each domain skill needs a small addition noting that division filtering is available. The actual DivisionID values and application logic live in core.

### Division Filtering: Build Dependencies

1. **Phase 1 prerequisite: Query DivisionData** -- Discover actual DivisionID values for Company and Apparel from live database. Currently only known that 3 rows exist with IDs -1, ?, ?.

2. **Phase 2 prerequisite: Validate TransHeader.DivisionID coverage** -- Confirm that all (or nearly all) Type 1 orders have a non-null DivisionID. If DivisionID is NULL for legacy orders, document the gap.

3. **Phase 3 prerequisite: Validate Ledger.DivisionID coverage** -- Same check for GL entries. If Ledger DivisionID is sparse, P&L-by-division will be unreliable.

4. **Phase 4: Update core skill** with Division Filtering section.

5. **Phase 5: Update glossary** with "Apparel", "Banner", "by division" NL routes.

6. **Phase 6: Validate** -- Run division-filtered revenue query and compare to full revenue. Two division totals should approximately sum to company total.

---

## 3. ANALYTICS / FORECASTING ARCHITECTURE

### Data Foundation for Forecasting

FLS has rich historical data for time series analysis:

| Data Source | Records | Time Span | Granularity |
|-------------|---------|-----------|-------------|
| TransHeader (Type 1 orders) | 232,243 | 2003-2026 | Daily (SaleDate) |
| Ledger (GL entries) | 2,748,661 | 2003-2026 | Daily (EntryDateTime) |
| TransDetail (line items) | 535,126 | 2003-2026 | Per-order |

This is more than sufficient for monthly/quarterly aggregations with 20+ years of history.

### Forecasting Approaches: SQL-Native

**Recommendation: All forecasting in SQL. No Python, no external tools.**

The existing architecture uses SQL query templates filled in by Claude based on NL questions. Adding a Python forecasting pipeline would break this pattern and require a new execution path. SQL Server provides adequate tools for the forecasting granularity FLS needs.

#### Approach 1: Moving Averages (Recommended Starting Point)

Simple Moving Average (SMA) and Weighted Moving Average (WMA) using SQL window functions:

```sql
-- 3-month simple moving average for revenue trend
SELECT
    SaleYear, SaleMonth, Revenue,
    AVG(Revenue) OVER (
        ORDER BY SaleYear, SaleMonth
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS SMA_3Month
FROM (
    SELECT YEAR(SaleDate) AS SaleYear, MONTH(SaleDate) AS SaleMonth,
           SUM(SubTotalPrice) AS Revenue
    FROM TransHeader
    WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL
    GROUP BY YEAR(SaleDate), MONTH(SaleDate)
) monthly
ORDER BY SaleYear, SaleMonth
```

**Why SMA first:**
- Simplest to implement and explain
- Works well for FLS's steady-growth business model
- Easy to validate -- user can see the math
- Window function support is native in SQL Server

#### Approach 2: Year-over-Year Growth Rate Projection

Use historical YoY growth rates to project future periods:

```sql
-- Project next year's monthly revenue using YoY growth rates
WITH MonthlyRevenue AS (
    SELECT YEAR(SaleDate) AS Y, MONTH(SaleDate) AS M,
           SUM(SubTotalPrice) AS Revenue
    FROM TransHeader
    WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL
    GROUP BY YEAR(SaleDate), MONTH(SaleDate)
),
GrowthRates AS (
    SELECT curr.M,
           AVG(CASE WHEN prev.Revenue > 0
               THEN (curr.Revenue - prev.Revenue) / prev.Revenue
               ELSE 0 END) AS AvgGrowthRate
    FROM MonthlyRevenue curr
    JOIN MonthlyRevenue prev ON curr.M = prev.M AND curr.Y = prev.Y + 1
    WHERE curr.Y >= YEAR(GETDATE()) - 3  -- last 3 years of growth
    GROUP BY curr.M
)
SELECT M AS Month,
       latest.Revenue AS LastYearRevenue,
       latest.Revenue * (1 + g.AvgGrowthRate) AS ProjectedRevenue,
       g.AvgGrowthRate AS GrowthRate
FROM GrowthRates g
JOIN MonthlyRevenue latest ON g.M = latest.M
WHERE latest.Y = YEAR(GETDATE()) - 1
ORDER BY M
```

**Why YoY growth:**
- FLS is scaling from $3M to $10-15M. Linear projection understates growth.
- Growth rate projection captures the upward trajectory.
- Easy to bound with confidence intervals (min/max historical growth rates).

#### Approach 3: Seasonal Decomposition

Combine seasonal patterns with trend:

```sql
-- Seasonal pattern: what % of annual revenue falls in each month?
WITH AnnualRevenue AS (
    SELECT YEAR(SaleDate) AS Y, SUM(SubTotalPrice) AS AnnualTotal
    FROM TransHeader
    WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL
    GROUP BY YEAR(SaleDate)
),
MonthlyRevenue AS (
    SELECT YEAR(SaleDate) AS Y, MONTH(SaleDate) AS M, SUM(SubTotalPrice) AS Revenue
    FROM TransHeader
    WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL
    GROUP BY YEAR(SaleDate), MONTH(SaleDate)
)
SELECT m.M AS Month,
       AVG(m.Revenue / a.AnnualTotal) AS SeasonalFactor,
       -- Apply to current year target
       AVG(m.Revenue / a.AnnualTotal) * @AnnualTarget AS ProjectedMonthly
FROM MonthlyRevenue m
JOIN AnnualRevenue a ON m.Y = a.Y
WHERE m.Y >= YEAR(GETDATE()) - 5  -- last 5 years for seasonal pattern
GROUP BY m.M
ORDER BY m.M
```

**Why seasonal decomposition:**
- FLS has clear seasonal patterns (event/trade show driven business)
- Allows setting an annual target and distributing it monthly
- Validates against actual YTD performance

#### Approach 4: Exponential Moving Average (Advanced)

Recursive CTE for EMA with configurable smoothing factor:

```sql
-- EMA for revenue smoothing (alpha = 2/(N+1) where N = periods)
DECLARE @alpha FLOAT = 2.0 / (12 + 1);  -- 12-month EMA

WITH MonthlyRevenue AS (
    SELECT YEAR(SaleDate) AS Y, MONTH(SaleDate) AS M,
           SUM(SubTotalPrice) AS Revenue,
           ROW_NUMBER() OVER (ORDER BY YEAR(SaleDate), MONTH(SaleDate)) AS rn
    FROM TransHeader
    WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL
    GROUP BY YEAR(SaleDate), MONTH(SaleDate)
),
EMA AS (
    SELECT Y, M, Revenue, rn,
           CAST(Revenue AS FLOAT) AS EMA_Value
    FROM MonthlyRevenue WHERE rn = 1
    UNION ALL
    SELECT m.Y, m.M, m.Revenue, m.rn,
           @alpha * m.Revenue + (1 - @alpha) * e.EMA_Value
    FROM MonthlyRevenue m
    JOIN EMA e ON m.rn = e.rn + 1
)
SELECT Y, M, Revenue, EMA_Value
FROM EMA
ORDER BY rn
OPTION (MAXRECURSION 500)
```

**Why EMA is advanced tier:** Recursive CTEs with MAXRECURSION are harder to debug and explain. Reserve for trend analysis, not initial forecasting.

### Forecasting: Skill Placement

**Recommendation: Create `control-erp-analytics` skill.**

Forecasting and trend detection are analytically distinct from the raw data queries in existing skills. They:

1. **Build on multiple domain skills** (sales data + financial data + customer data)
2. **Have unique SQL patterns** (window functions, CTEs, recursive queries)
3. **Require different confidence framing** ("projection" vs "actual")
4. **Need their own NL trigger vocabulary** ("forecast", "predict", "project", "trend", "outlook")

A new `control-erp-analytics` skill keeps the existing skills clean (they report actuals) while giving analytics its own space.

### Trend Detection Architecture

Beyond forecasting absolute numbers, trend detection identifies directional changes:

**Product-line growth/decline detection:**
```sql
-- Compare current period to same-period-last-year by product category
-- Flag categories growing >20% or declining >10%
WITH CurrentPeriod AS (...),
PriorPeriod AS (...)
SELECT
    Category,
    CurrentRevenue,
    PriorRevenue,
    (CurrentRevenue - PriorRevenue) / NULLIF(PriorRevenue, 0) * 100 AS GrowthPct,
    CASE
        WHEN (CurrentRevenue - PriorRevenue) / NULLIF(PriorRevenue, 0) > 0.20 THEN 'GROWTH'
        WHEN (CurrentRevenue - PriorRevenue) / NULLIF(PriorRevenue, 0) < -0.10 THEN 'DECLINE'
        ELSE 'STABLE'
    END AS Trend
FROM ...
```

**Customer purchasing pattern changes:**
- New customers acquired this period vs last
- At-risk customers (declining purchase frequency)
- Already partially built in `control-erp-customers` (churn detection)

**Seasonal anomaly detection:**
- Compare actual vs expected seasonal pattern
- Flag months significantly above/below seasonal norm

### Analytics: Build Dependencies

1. **Prerequisite: Monthly revenue aggregation** -- Already available in sales Template 2. May need to extend with product-line and division dimensions.

2. **Prerequisite: Multi-year data verification** -- Confirm data quality for years going back 3-5 years. Outlier years (COVID 2020?) may need special handling.

3. **Prerequisite: Baseline metrics** -- Establish current KPIs before projecting them forward. Use existing skill queries.

---

## 4. DASHBOARD ARCHITECTURE

### Two Dashboard Modes

The v1.2 requirements specify two distinct dashboard experiences:

#### Mode 1: Executive Text Summary (DASH-01)

A structured text output that Claude generates using existing SQL queries across multiple skills. This is a **prompt pattern**, not a technical component.

```
MORNING REPORT: FLS Banners
Date: February 9, 2026

REVENUE (MTD)
  February 2026: $XXX,XXX (XX orders)
  vs January 2026: +X.X%
  vs February 2025: +XX.X%

OPEN AR: $XX,XXX (XX invoices)
  Past due: $X,XXX (X invoices)
  Largest: [Customer] $X,XXX

WIP: XX orders ($XXX,XXX)
  Oldest: Order #XXXXX (X days)
  [Station] bottleneck: X orders queued

ALERTS
  - [Product line] trending +XX% (above seasonal norm)
  - [Customer] 90+ days past due
  - [Part] below reorder point
```

**Architecture:** This is a **prompt template in the analytics or dashboard skill** that tells Claude which queries to run from which skills, and how to format the results. It does not require a new execution mechanism -- Claude already runs SQL and formats results.

**Data flow:**
```
User: "morning report" / "executive summary" / "daily dashboard"
    |
    v
[Glossary routes to dashboard skill]
    |
    v
[Dashboard skill instructs Claude to run:]
    1. Sales Template 2 (MTD revenue) from control-erp-sales
    2. AR Snapshot from control-erp-financial
    3. WIP count from control-erp-production
    4. Reorder alerts from control-erp-inventory
    5. At-risk customers from control-erp-customers
    6. YoY comparison from control-erp-sales
    |
    v
[Claude assembles results into formatted text output]
```

**Key decision: The text dashboard does NOT require a separate query.** It orchestrates existing queries. The skill just needs to list which queries to run and how to format the results.

#### Mode 2: Interactive React Dashboard (DASH-02)

Claude generates a React artifact containing interactive charts, gauges, and tables. The artifact is self-contained -- all data is embedded as constants (no API calls from the artifact).

**React Artifact Architecture:**

```
[Claude runs SQL queries]
    |
    v
[Claude receives data arrays]
    |
    v
[Claude generates React component with data embedded]
    |
    v
[React artifact renders in Claude's artifact viewer]
    |
    Contains:
    +-- Revenue chart (Recharts BarChart or LineChart)
    +-- AR aging donut (Recharts PieChart)
    +-- WIP by station (Recharts horizontal BarChart)
    +-- KPI cards (revenue, margin, AR, WIP count)
    +-- Product mix chart (Recharts BarChart)
    +-- Trend indicators (up/down arrows)
```

**Technical constraints for React artifacts:**

1. **Self-contained.** No external API calls. All data must be embedded as JavaScript constants.
2. **Libraries available:** React, Recharts, Tailwind CSS, Shadcn UI, Lucide icons.
3. **No server-side rendering.** The artifact runs purely in the browser.
4. **Size limit.** Artifacts should remain reasonably sized. Embed aggregated data (monthly totals, not 232K order records).
5. **Stateless.** Each generation is a fresh snapshot. No persistent state between dashboards.

**Data pipeline for React dashboard:**

```
Step 1: Claude runs ~5-8 SQL queries against the database
Step 2: Claude receives result sets (small -- monthly aggregates, top-N lists)
Step 3: Claude generates a single React component:

  const REVENUE_DATA = [
    { month: 'Jan', revenue: 245000, prior: 220000 },
    { month: 'Feb', revenue: 280000, prior: 235000 },
    ...
  ];

  const AR_AGING = [
    { bucket: '0-30', amount: 65000 },
    { bucket: '31-60', amount: 10000 },
    ...
  ];

  // ... more embedded data constants

  export default function Dashboard() {
    return (
      <div className="grid grid-cols-2 gap-4 p-4">
        <KPICard title="YTD Revenue" value="$480,000" trend="+12%" />
        <KPICard title="Open AR" value="$80,899" trend="healthy" />
        <RevenueChart data={REVENUE_DATA} />
        <ARAgingChart data={AR_AGING} />
        ...
      </div>
    );
  }

Step 4: Artifact renders in Claude's viewer as interactive dashboard
```

### Dashboard: Skill Placement

**Recommendation: Create `control-erp-dashboard` skill.**

The dashboard skill is a **orchestration skill** -- it does not own query templates (those live in domain skills), but it owns:

1. **Which queries to run** for each dashboard type
2. **How to format text dashboards**
3. **React component templates** for visual dashboards
4. **Alert threshold definitions** (what constitutes a "warning")
5. **NL triggers** ("morning report", "dashboard", "executive summary", "show me charts")

**Why a separate skill (not extending analytics):**

- Analytics is about forecasting and trend math. Dashboard is about presentation and orchestration.
- Dashboard will reference analytics projections, sales actuals, financial KPIs -- it is inherently multi-domain.
- The React component templates are a distinct concern that would clutter an analytics skill.

**Alternative considered: No separate skill, just a prompt pattern.**

Rejected because:
- Dashboard assembly instructions are complex enough to justify a dedicated skill
- React component generation needs standardized patterns (Recharts config, Tailwind layout)
- Alert thresholds need documentation and calibration
- NL routing needs explicit entries in the glossary

### Dashboard: Data Granularity

**What gets embedded in React artifacts (small):**

| Data | Typical Size | Query Source |
|------|-------------|-------------|
| Monthly revenue (12-24 months) | 24 rows | sales Template 2 |
| AR aging buckets | 5 rows | financial AR Aging |
| WIP by station | 10-15 rows | production Station Workload |
| Top 10 customers | 10 rows | customers Top N |
| Product mix | 10-15 rows | sales Template 8 |
| P&L summary | 6 rows | financial P&L Summary |
| Forecast vs actual | 12 rows | analytics forecast |

**Total embedded data: ~100-150 rows.** This is tiny. No performance concern.

**What does NOT get embedded:**

- Individual order records (232K)
- Line-item detail (535K)
- GL entries (2.7M)
- Customer list (54K)

---

## 5. NEW SKILL TOPOLOGY (Post v1.2)

### Recommended Architecture

```
User (natural language)
    |
    v
[Claude reads relevant skills based on question context]
    |
    +-- control-erp-glossary (expanded: ~100+ NL routes)
    |       |
    |       +-- Routes to --> all domain skills (existing)
    |       +-- Routes to --> control-erp-analytics  (NEW)
    |       +-- Routes to --> control-erp-dashboard   (NEW)
    |
    +-- control-erp-core (expanded: +Division Filtering section)
    |       ^
    |       |
    |       +-- Depended on by --> all domain + analytics + dashboard skills
    |
    +-- [Existing domain skills -- UNCHANGED except minor division notes]
    |   +-- control-erp-sales
    |   +-- control-erp-financial
    |   +-- control-erp-customers
    |   +-- control-erp-inventory
    |   +-- control-erp-production
    |
    +-- control-erp-analytics (NEW -- forecasting, trends, projections)
    |       |
    |       +-- Reads from --> sales (historical revenue data)
    |       +-- Reads from --> financial (GL-based revenue/costs)
    |       +-- Reads from --> customers (acquisition/churn rates)
    |
    +-- control-erp-dashboard (NEW -- text + visual dashboards)
            |
            +-- Orchestrates --> ALL domain skills
            +-- Orchestrates --> analytics (for forecast overlay)
```

### New Component Summary

| Component | Type | Estimated Size | Purpose |
|-----------|------|---------------|---------|
| `control-erp-analytics` | NEW skill | ~400-600 lines | Forecasting, trend detection, seasonal analysis |
| `control-erp-dashboard` | NEW skill | ~300-500 lines | Text dashboards, React artifact templates, alert thresholds |
| `control-erp-core` additions | MODIFY | +50-80 lines | Division filtering section, DivisionID reference |
| `control-erp-glossary` additions | MODIFY | +30-50 NL routes | Analytics, dashboard, division terminology |
| Domain skill annotations | MODIFY (minor) | +5-10 lines each | Note that division filtering is available via core |

### What Does NOT Change

1. **Existing query templates.** None of the 73+ SQL queries in existing skills need modification. Division filtering is additive (AND clause).

2. **Core business rules.** TransactionType, StatusID, pricing logic, date filtering -- all unchanged.

3. **Skill dependency direction.** Core is still the foundation. Domain skills are still independent. No circular dependencies.

4. **Validation methodology.** New analytics outputs are validated by running them against known historical periods.

---

## 6. COMPONENT BOUNDARIES

### control-erp-analytics (NEW)

**Owns:**
- Moving average / smoothing query templates
- YoY growth rate projection queries
- Seasonal decomposition queries
- Product-line trend detection queries
- Customer trend queries (new vs declining)
- Confidence interval / range projection

**Does NOT own:**
- Raw data queries (those stay in domain skills)
- Presentation / formatting (that is dashboard's job)
- DivisionID filtering logic (that is core's job)
- Business rules (core)

**Depends on:**
- `control-erp-core` (standard filters, TransactionType rules)
- Data generated by `control-erp-sales` queries (monthly revenue)
- Data generated by `control-erp-financial` queries (GL-based revenue)

**NL triggers:**
- "forecast", "predict", "project", "projection", "next quarter"
- "trend", "trending", "growth", "decline", "seasonal"
- "what will", "expected", "outlook"
- "compared to last year" (routes to sales Template 9 first, then analytics for projection)

### control-erp-dashboard (NEW)

**Owns:**
- Text dashboard templates (morning report, weekly summary)
- React artifact component patterns (Recharts configs, Tailwind layouts)
- Alert threshold definitions (what is "concerning" vs "healthy")
- Multi-query orchestration instructions
- KPI definitions (which metric, which query, how to calculate)

**Does NOT own:**
- SQL queries (references queries from other skills)
- Data analysis / forecasting (that is analytics's job)
- Business rules (core)

**Depends on:**
- `control-erp-core` (standard filters)
- ALL domain skills (pulls data from each)
- `control-erp-analytics` (optional: forecast overlay on charts)

**NL triggers:**
- "morning report", "daily report", "executive summary"
- "dashboard", "overview", "snapshot"
- "show me a chart", "visualize", "graph"
- "KPI", "metrics", "how are we doing"

### control-erp-core (MODIFIED)

**New section added: Division Filtering**

Content:
- DivisionData table reference with actual IDs
- Division filter clause templates for TransHeader, Ledger, Account, Warehouse
- NL trigger mapping ("Apparel" -> DivisionID X, "Banner" -> DivisionID Y)
- When to apply vs omit division filter

**Size impact:** +50-80 lines. Core grows from 347 to ~420-430 lines. Well within context budget.

### control-erp-glossary (MODIFIED)

**New NL routes added:**

| User Says | Route To | Notes |
|-----------|----------|-------|
| "forecast sales" / "projected revenue" | control-erp-analytics | Forecasting queries |
| "sales trend" / "product growth" | control-erp-analytics | Trend detection |
| "morning report" / "executive summary" | control-erp-dashboard | Text dashboard |
| "show me a dashboard" / "charts" | control-erp-dashboard | React artifact |
| "Apparel sales" / "Apparel AR" | Route to domain + core division filter | Cross-cutting |
| "by division" / "division breakdown" | Applicable skill + core division filter | Cross-cutting |
| "Banner division" / "Company division" | Route to domain + core division filter | Cross-cutting |

**Size impact:** +30-50 lines. Glossary grows from 671 to ~710-720 lines.

---

## 7. DATA FLOW DIAGRAMS

### Forecasting Data Flow

```
Historical Data (TransHeader)
    |
    [Sales Template 2: Monthly Revenue by Year]
    |
    v
Monthly Revenue Table (24-60 rows, 2-5 years)
    |
    +--> [SMA: Window function over 3-month window]
    |        --> Moving Average Trend Line
    |
    +--> [YoY: Compare each month to same-month-prior-year]
    |        --> Growth Rate per Month
    |        --> Projected Next Year = Last Year * (1 + Avg Growth)
    |
    +--> [Seasonal: Each month's % of annual total]
    |        --> Seasonal Factors (12 values)
    |        --> Target Distribution = Annual Target * Seasonal Factor
    |
    v
Forecast Output:
    - Projected quarterly revenue (Q1-Q4)
    - Projected annual revenue
    - Confidence range (based on historical variance)
    - Seasonal adjustments applied
```

### Dashboard Data Flow (Text Mode)

```
User: "morning report"
    |
    v
[Dashboard skill: Morning Report Template]
    |
    +-- Query 1: SELECT ... (sales MTD + prior month + prior year MTD)
    +-- Query 2: SELECT ... (AR snapshot)
    +-- Query 3: SELECT ... (WIP count by station)
    +-- Query 4: SELECT ... (reorder alerts)
    +-- Query 5: SELECT ... (at-risk customers, optional)
    +-- Query 6: SELECT ... (forecast vs actual, optional)
    |
    v
[Claude formats results into text template]
    |
    v
Formatted text output
```

### Dashboard Data Flow (React Mode)

```
User: "show me a visual dashboard"
    |
    v
[Dashboard skill: Visual Dashboard Template]
    |
    +-- Query 1-6: Same as text mode
    |
    v
[Claude generates React component:]
    - Embeds query results as const data arrays
    - Uses Recharts for charts (LineChart, BarChart, PieChart)
    - Uses Tailwind for layout (grid, cards)
    - Uses Lucide for icons (TrendingUp, AlertTriangle, etc.)
    |
    v
React artifact rendered in Claude's artifact viewer
```

### Division-Filtered Query Flow

```
User: "Apparel division sales this year"
    |
    v
[Glossary: "Apparel" + "sales" -> control-erp-sales + division filter]
    |
    v
[Core: Division Filtering section -> DivisionID = @ApparelDivID]
    |
    v
[Sales Template 1 + division filter]:
    SELECT SUM(SubTotalPrice) AS Revenue
    FROM TransHeader
    WHERE TransactionType = 1
        AND IsActive = 1
        AND SaleDate IS NOT NULL
        AND YEAR(SaleDate) = 2026
        AND DivisionID = @ApparelDivID  -- <-- injected
    |
    v
Result: Apparel division revenue
```

---

## 8. BUILD ORDER AND DEPENDENCIES

### Recommended Phase Structure

```
Phase 14: Division Discovery + Core Update
    |
    +-- Discover DivisionData IDs (query live DB)
    +-- Validate DivisionID coverage on TransHeader and Ledger
    +-- Add Division Filtering section to core skill
    +-- Add division NL routes to glossary
    +-- Validate: division-filtered revenue sums to total
    |
Phase 15: Analytics Skill
    |
    +-- Create control-erp-analytics skill
    +-- Implement SMA, YoY growth, seasonal decomposition
    +-- Implement product-line trend detection
    +-- Add analytics NL routes to glossary
    +-- Validate: forecast for known historical period matches actuals
    |
Phase 16: Dashboard Skill
    |
    +-- Create control-erp-dashboard skill
    +-- Implement text dashboard template (morning report)
    +-- Implement React artifact template (visual dashboard)
    +-- Define alert thresholds
    +-- Add dashboard NL routes to glossary
    +-- Validate: text dashboard pulls correct data from all domains
    +-- Validate: React artifact renders correctly
    |
Phase 17: Integration + Glossary Update + Validation
    |
    +-- End-to-end testing across all new features
    +-- Division + analytics + dashboard combined tests
    +-- Glossary final sync (all new NL routes)
    +-- Milestone audit
```

### Build Order Rationale

1. **Division filtering first (Phase 14)** because:
   - Requires DB discovery (actual DivisionID values)
   - Cross-cuts all other features (analytics and dashboard both need it)
   - Simplest to validate (just adds a WHERE clause)
   - Unblocks division-filtered forecasts and division-filtered dashboards

2. **Analytics second (Phase 15)** because:
   - Builds on existing sales/financial data (no new DB discovery needed)
   - Produces outputs that dashboard consumes (forecast data)
   - Can be validated independently against historical data
   - Division filtering from Phase 14 can be applied to forecasts

3. **Dashboard third (Phase 16)** because:
   - Orchestrates all other skills (needs them to be stable)
   - Consumes analytics output (needs analytics to exist)
   - Is the presentation layer -- should be built last
   - React artifacts are self-contained (no backend changes)

4. **Integration last (Phase 17)** because:
   - Tests the full chain: NL -> glossary -> skill -> query -> format
   - Validates cross-feature combinations (division-filtered dashboard)
   - Final glossary sync ensures all new NL routes are wired

### Dependency Graph

```
Phase 14 (Division) ──────> Phase 15 (Analytics)
         |                          |
         |                          v
         +────────────────> Phase 16 (Dashboard)
                                    |
                                    v
                            Phase 17 (Integration)
```

Phase 14 must complete before 15 or 16 can start (division filtering is a prerequisite).
Phase 15 should complete before 16 (dashboard uses forecast data).
Phase 16 cannot start until 15 is done (React dashboard includes forecast overlay).
Phase 17 requires all three prior phases.

---

## 9. ANTI-PATTERNS TO AVOID

### Anti-Pattern 1: Query Duplication for Division Filtering

**Wrong:** Create separate "Apparel sales" and "Banner sales" query templates in the sales skill.

**Why bad:** Every new query template would need 2 division-specific variants. 73+ existing queries would become 219+. Unmaintainable.

**Instead:** Single query template + optional WHERE clause injection from core.

### Anti-Pattern 2: Python Forecasting Pipeline

**Wrong:** Run Python scripts for forecasting (pandas, statsmodels, prophet).

**Why bad:** Breaks the existing SQL-query-template architecture. Requires a new execution path (Python runtime via MCP). Adds a dependency that complicates the skill pattern.

**Instead:** SQL-native forecasting using window functions and CTEs. Adequate for FLS's needs (monthly/quarterly projections, not tick-level trading).

### Anti-Pattern 3: Real-Time Dashboard with API Calls

**Wrong:** Build a React dashboard that makes live API calls to the database.

**Why bad:** React artifacts in Claude are sandboxed. No external API access. Also, MCP MSSQL is Claude's tool, not the artifact's.

**Instead:** Claude runs queries, embeds results as static data in the React component, artifact renders with that embedded data.

### Anti-Pattern 4: Monolithic Dashboard Skill

**Wrong:** Put all dashboard queries, formatting logic, chart configs, alert thresholds, and analytics forecasts in one massive skill.

**Why bad:** Would be 2000+ lines. Exceeds comfortable context budget. Mixes concerns (math vs presentation).

**Instead:** Analytics skill (math, projections) + Dashboard skill (presentation, orchestration). Clear separation.

### Anti-Pattern 5: Modifying Existing Skills to Add Division

**Wrong:** Add DivisionID handling to every query template in every existing skill.

**Why bad:** Spreads division logic across 7 skills. When a third division is added (FLS is scaling), every skill needs updating.

**Instead:** Document division filtering once in core. Each domain skill notes "division filtering available -- see core" with a one-line reference.

### Anti-Pattern 6: Forecasting Without Confidence Bounds

**Wrong:** Present a single projected number ("Q3 2026 revenue will be $850,000").

**Why bad:** False precision. Forecasts are estimates. A single number implies certainty that does not exist.

**Instead:** Always present range: "Q3 2026 projected: $780,000 - $920,000 (based on 3-year growth rates, +/- 1 STDEV)". This is honest and more useful.

---

## 10. SCALABILITY CONSIDERATIONS

### Division Scaling

FLS currently has 2 active divisions (Company + Apparel). The architecture should handle a third without structural changes.

**Current design handles this:** DivisionID is a FK lookup. Adding a row to DivisionData and updating the core skill's Division Reference table is the only change needed.

### Query Count Growth

v1.2 adds ~15-25 new query templates across analytics and dashboard skills. Combined with existing 73+, total reaches ~90-100 queries.

**No structural concern.** Queries are distributed across independent skill files. Each file stays within context budget.

### React Artifact Complexity

As dashboard features grow, the React artifact could become large. Mitigation:

1. **Multiple dashboard types** (sales dashboard, financial dashboard, production dashboard) rather than one mega-dashboard.
2. **Focused artifacts** -- each generated artifact covers one view, not everything.
3. **Data aggregation** -- always embed aggregated data (monthly totals), never raw records.

---

## 11. CONFIDENCE ASSESSMENT

| Area | Confidence | Rationale |
|------|------------|-----------|
| Division filtering DB structure | HIGH | FK relationships verified in schema. TransHeader.DivisionID -> DivisionData.ID confirmed. Ledger.DivisionID -> DivisionData.ID confirmed. |
| Division filtering architecture | HIGH | Clause injection is a proven pattern. Analogous to existing IsActive=1 and SaleDate IS NOT NULL filters. |
| DivisionID actual values | LOW | Need to query live DB to discover IDs for Company and Apparel. Schema sample only shows ID=-1 (system default). |
| SQL forecasting patterns | MEDIUM | Window functions and CTEs for SMA/YoY are well-documented. Need to test against FLS data for edge cases (COVID dip, rapid growth periods). |
| Seasonal decomposition | MEDIUM | Assumes FLS has clear seasonal patterns. Need to verify with actual monthly data before committing to seasonal model. |
| React artifact capabilities | HIGH | Well-documented: Recharts, Tailwind, Shadcn UI. Self-contained, embedded data. Proven by community usage. |
| Text dashboard architecture | HIGH | No new technical mechanism. Claude already runs multiple queries and formats results. This is a prompt template pattern. |
| New skill sizes | MEDIUM | Estimated 400-600 lines for analytics, 300-500 for dashboard. Based on comparable skill sizes (sales=370, financial=888). May need reference files if larger. |
| Build order | HIGH | Dependency chain is clear: division first (cross-cutting), analytics second (produces data), dashboard third (consumes data), integration last. |

---

## 12. OPEN QUESTIONS FOR PHASE-LEVEL RESEARCH

1. **What are the actual DivisionData IDs?** Must query `SELECT * FROM DivisionData WHERE IsActive = 1` in Phase 14.

2. **What percentage of TransHeader records have DivisionID populated?** If many records have NULL DivisionID, division filtering will miss data.

3. **Does FLS have clear seasonal patterns?** Monthly revenue over 3-5 years will reveal whether seasonal decomposition is worthwhile or if growth trend dominates.

4. **What are appropriate alert thresholds for FLS?** "AR >90 days is a concern" vs "AR >60 days" -- these need business input or calibration against historical norms.

5. **How many queries can Claude comfortably run for a single dashboard?** If 8 queries take too long, we may need to reduce the dashboard scope or batch them.

6. **Does Gretel need division-filtered reports in specific formats?** The Apparel Division user may have specific reporting requirements that should shape the division filtering implementation.

---

## SOURCES

- TransHeader schema: `/Users/cain/projects/control-db-map/output/schemas/TransHeader.md` -- DivisionID (line 103), ProductionDivisionID (line 189) with FK to DivisionData.ID confirmed
- DivisionData schema: `/Users/cain/projects/control-db-map/output/schemas/DivisionData.md` -- 3 rows, DivisionName column
- Ledger schema: `/Users/cain/projects/control-db-map/output/schemas/Ledger.md` -- DivisionID (line 36) with FK to DivisionData.ID confirmed
- Account schema: `/Users/cain/projects/control-db-map/output/schemas/Account.md` -- DivisionID (line 71)
- Warehouse schema: `/Users/cain/projects/control-db-map/output/schemas/Warehouse.md` -- DivisionID with FK to DivisionData.ID
- Existing core skill: `/Users/cain/projects/control-db-map/skills/control-erp-core/control-erp-core-SKILL.md` -- Multi-Division Context section (lines 332-338)
- Existing financial skill: `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` -- Ledger DivisionID reference
- SQL Server forecasting patterns: [MSSQLTips EMA](https://www.mssqltips.com/sqlservertip/6448/exponential-moving-average-calculation-in-sql-server/), [SQLServerCentral Forecasting](https://www.sqlservercentral.com/articles/forecasting-with-sql)
- React artifacts: [Anthropic Claude Artifacts](https://claude.com/blog/claude-powered-artifacts), [LogRocket Artifacts](https://blog.logrocket.com/implementing-claudes-artifacts-feature-ui-visualization/)
- v1.1 Milestone Audit: `/Users/cain/projects/control-db-map/.planning/milestones/v1.1-MILESTONE-AUDIT.md`
- Project state: `/Users/cain/projects/control-db-map/.planning/PROJECT.md`
