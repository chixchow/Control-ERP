# Feature Landscape: Analytics, Dashboards & Division Filtering

**Domain:** ERP analytics layer for Control ERP natural language interface
**Researched:** 2026-02-09
**Business context:** FLS Banners, ~$3M revenue (2025), custom flag/banner/signage & apparel manufacturer scaling to $10-15M
**Existing system:** 8 validated skills (core, sales, financial, customers, inventory, production, glossary) with 73+ SQL queries, 172 NL routing entries
**Milestone scope:** Add forecasting, trend detection, executive dashboards, division filtering, and payroll expense visibility

---

## How to Read This Document

Each feature area has four categories:

- **Table Stakes** -- Features users expect. Missing = the analytics milestone feels incomplete.
- **Differentiators** -- Features that provide genuine competitive advantage over running Crystal Reports or manual spreadsheets. Not expected, but high-value.
- **Anti-Features** -- Things to deliberately NOT build. Scope discipline is critical for a $3M company with 3-4 users.
- **Dependencies** -- What existing skills/queries already support or must be extended.

Complexity ratings: **Low** (reuses existing query with minor modification), **Med** (new query pattern, 2-3 table joins, SQL window functions), **High** (multi-query composition, complex business logic, significant new infrastructure).

---

## 1. SALES FORECASTING

**Scope constraint:** Historical pattern-based only. No external data, no ML libraries, no Python dependencies. All forecasting is SQL-computed from TransHeader/TransDetail data with 2+ years of history.

**Why this matters for FLS:** Cain built a $4M 2026 sales plan manually using historical data in a spreadsheet. The NL interface should automate the same analysis. FLS has 2+ years of validated transaction history, known seasonal patterns (Aug-Sep peak, Nov dip), and 27 product categories with tracked revenue. The data is there -- the system just needs the right queries.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Monthly revenue projection for current year | "What are we on pace for this year?" -- the single most asked forecasting question. Extrapolates YTD revenue to a full-year run rate | Low | Simple math: (YTD Revenue / days elapsed) * 365. Uses existing sales Template 1 as base. Already have validated monthly totals |
| Same-period comparison with prior year | "How does this quarter compare to last year?" -- context for any forecast | Low | Already built as sales Template 9 (YoY). Extend with percent change calculation. Customer skill already has YoY comparison logic |
| Seasonal run-rate projection | "What are we on pace for, accounting for seasonality?" -- straight-line extrapolation is misleading for a seasonal business. FLS has Aug-Sep peaks and Dec troughs | Med | Requires computing seasonal indices from 2024-2025 monthly weights, then applying the remaining months' weights to project full year. Pure SQL with CTEs. The FLS_Sales_Plan_2026.md already computed these weights manually |
| Monthly budget vs actual (plan comparison) | "Are we on track with our $4M plan?" -- Cain has a 2026 sales plan with monthly targets by product category | Med | Two approaches: (a) encode the 2026 plan as a reference table in the skill and compare at query time, or (b) use the SalesGoal table (248 rows, has GoalYear/GoalMonth/Goal columns). Check if SalesGoal is populated for 2026 |
| Revenue pace by product category | "How is Feather Flags tracking this year vs plan?" -- category-level pace check | Med | Combines sales Template 3 (DyeSub by category) with plan targets from FLS_Sales_Plan_2026.md. Must handle both DyeSub (via FP_ProductDescription) and non-DyeSub (via Description pattern) categories |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Weighted moving average forecast | More sophisticated than straight-line extrapolation -- weights recent months more heavily to capture momentum. "What would sales be next month based on the last 3-6 months weighted trend?" | Med | SQL window functions with LAG(). T-SQL supports self-join weighted average patterns. Pre-2017 SQL Server limitation: no STRING_AGG but window functions work fine. 3-month and 6-month WMA variants |
| Seasonal decomposition | "What is our deseasonalized growth rate?" -- separates seasonal pattern from underlying trend. FLS's 2025 data was wildly distorted by two outlier months (Apr $1.55M, Sep $2.57M). Removing seasonality reveals the true growth trajectory | High | Ratio-to-moving-average method, pure SQL. Compute 12-month centered moving average, divide actuals by CMA to get seasonal indices, then divide actuals by seasonal indices for deseasonalized values. Requires 24+ months of data (FLS has this) |
| Product mix shift detection | "Are our product proportions changing?" -- detects when one category is growing faster than others. Critical for capacity planning. Feather Flags went from 32% to near-100% of DyeSub in 2025 | Med | Share-of-wallet calculation by product category: current period category% vs prior period category%. Highlight categories with >5 percentage point shifts. Uses existing sales Template 3 with two periods |
| Forecast confidence range | "What is the likely range for next quarter?" -- provides high/low/expected instead of a single number. Cain's 2026 plan already has downside/base/upside scenarios ($3.2M/$4M/$5M) | Med | Use STDEV of monthly revenue to compute confidence bands. Low = projection - 1.5*STDEV, High = projection + 1.5*STDEV. Simple statistical computation in T-SQL (STDEV is available pre-2017) |
| Salesperson quota tracking | "Is Josh on pace for his annual target?" -- links SalesGoal to actual salesperson revenue | Med | Join SalesGoal table with TransHeader.SalesPerson1ID revenue actuals. Requires verifying SalesGoal data is populated. Uses sales Template 7 (revenue by salesperson) as base |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Machine learning forecasting | No Python/ML dependencies allowed. ARIMA, exponential smoothing, or regression models require libraries outside the SQL + NL interface scope. Adds complexity without proportional value for a 3-person analytics audience | Use SQL-native approaches: moving averages, seasonal indices, STDEV-based confidence. These cover 90% of a $3M company's forecasting needs |
| External data integration | Weather, economic indicators, trade show calendars -- all would improve forecasts but require ETL pipelines, API integrations, and maintenance | Use internal proxies: seasonal indices capture trade show seasonality. Customer-level ordering patterns capture demand signals. Document the limitation and suggest manual overrides for known events |
| Daily or weekly granularity forecasting | FLS has ~4,200 orders/year (~17/business day). Daily variance is high and forecasts at daily granularity are noisy and misleading | Monthly is the right granularity for FLS. Weekly summaries can be shown as actuals but forecasts should remain monthly. Note: the FLS sales plan is monthly |
| Inventory demand forecasting | "How much vinyl will we need next month?" -- inventory demand forecasting requires production BOM analysis, lead times, and supplier schedules | Defer to future milestone. Current inventory skill handles reorder alerts and stock levels. Production planning requires deeper cost/BOM integration |
| Pipeline-based forecasting from estimates | "What does our quote pipeline predict?" -- using Type 2 (estimates) to forecast future orders requires conversion rate modeling and deal-stage weighting. Control has no deal-stage tracking (estimates are binary: pending/converted/lost) | Acknowledge as future capability. Note: 56% historical conversion rate is a useful baseline, but individual estimate conversion is unpredictable without probability weighting |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Sales skill: Template 1 (total revenue) | **Reuse** | Base for all run-rate calculations |
| Sales skill: Template 2 (monthly trend) | **Reuse** | Input data for seasonal index computation |
| Sales skill: Template 3 (DyeSub by category) | **Reuse** | Product-level pace tracking |
| Sales skill: Template 9 (YoY comparison) | **Extend** | Add percent change, growth rate calculation |
| FLS_Sales_Plan_2026.md | **Import** | Encode monthly/category plan targets as skill reference data |
| Business rules validation: monthly breakdowns | **Reuse** | Validated 2025 monthly revenue provides test data |
| SalesGoal table (248 rows) | **Verify** | May be populated for salesperson quotas. Need to query live data |

---

## 2. TREND DETECTION

**Scope constraint:** Identify patterns in historical data and surface them proactively. All trends computed from SQL queries against existing tables. Focus on actionable insights, not statistical reporting.

**Why this matters for FLS:** The 2024-2025 YoY comparison revealed dramatic shifts that were only discovered through manual analysis: Feather Flags +628%, Pillow Case Frames -51%, Garments +104%. A trend detection system would have surfaced these automatically. FLS is scaling from $3M to $10-15M -- understanding growth vectors is critical.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Month-over-month revenue change | "How did this month compare to last month?" -- most basic trend indicator | Low | Already built in financial skill (MoM P&L comparison). Extend to header-level revenue with percent change. Uses LAG() window function |
| Year-over-year revenue change by month | "How does each month this year compare to last year?" -- 12-month comparison grid | Low | Already built as sales Template 9. Extend with computed columns for $ change and % change. The FLS_Sales_Plan_2026.md already has this exact table |
| Product category growth/decline ranking | "Which products are growing fastest?" -- rank all categories by YoY change | Med | Run sales Template 3 for two periods, compute growth rate, rank. Show both absolute $ change and % change. Critical: must handle categories that exist in one year but not the other (new or discontinued products) |
| Customer growth/decline ranking | "Which customers are growing vs declining?" -- customer-level trend | Med | Already built in customer skill (Spend Trend Detection query, YoY decline >30%). Extend to show both growing AND declining customers, not just at-risk ones |
| Order volume trend | "Are we getting more orders or larger orders?" -- separates volume growth from price growth | Med | COUNT vs AVG(SubTotalPrice) over time periods. Useful for understanding growth composition: more customers vs higher spend per customer |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Automatic outlier flagging | "Flag any month that is more than 2 standard deviations from the 12-month average" -- proactive anomaly detection. Would have caught the Sep 2025 $2.57M month immediately | Med | Compute trailing 12-month AVG and STDEV per product category. Flag actuals outside 2*STDEV. Pure SQL with window functions. High value: surfaces both positive anomalies (big wins) and negative (problems) |
| Revenue composition shift | "What percentage of our revenue comes from each division/product/customer segment, and how has that changed?" -- structural trend detection, not just growth rates | Med | Share-of-wallet computation: category revenue / total revenue, compared across periods. Highlight >5 point shifts. The FLS data shows DyeSub went from 88% to 85% share of revenue -- small shift overall despite huge Feather Flag growth |
| Rolling 12-month trend | "What is the trailing 12-month revenue trend?" -- smooths out monthly volatility. More informative than YTD for mid-year analysis | Med | SUM(SubTotalPrice) OVER (ORDER BY YearMonth ROWS BETWEEN 11 PRECEDING AND CURRENT ROW). Available in pre-2017 SQL Server. Shows underlying trajectory without seasonal noise |
| New customer acquisition trend | "Are we gaining new customers faster or slower than before?" -- leading indicator of growth | Med | Already exists in advanced_analytics.md (New Customers by Month). Extend with growth rate computation: this month's new customers vs same month prior year |
| Average order value trend | "Is our average deal size going up or down?" -- pricing power and mix shift indicator | Low | Simple AVG(SubTotalPrice) by month. Compare to prior year same month. FLS average was ~$731/order in 2025 ($3.05M / 4,172 orders). Track direction |
| Customer concentration trend | "Are we becoming more or less dependent on top customers?" -- risk monitoring. FLS's top 10 = 49.3% of revenue. If this is increasing, concentration risk is growing | Med | Already have Pareto analysis in customer skill. Run for two periods and compare: are top 10 getting a bigger share? Track Herfindahl-Hirschman Index (HHI) as a single-number concentration metric |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Predictive trend extrapolation | "This category grew 20% last year, so it will grow 20% next year" -- false precision. FLS's Feather Flags went +628% one year -- extrapolating that would be absurd | Show historical trends and growth rates. Let the user draw conclusions. Provide context: "Feather Flag growth was driven by bulk orders in Apr and Sep" |
| Causal trend analysis | "Revenue dropped because of trade show cancellation" -- requires external context the system does not have | Surface the WHAT (revenue dropped 40% in July) not the WHY. The user knows the business context |
| Real-time trend monitoring | "Alert me when daily sales exceed threshold" -- requires persistent monitoring, notification infrastructure, and always-on processes | Provide on-demand trend analysis when asked. The morning dashboard covers daily snapshot needs |
| Multi-variable correlation | "Does marketing spend correlate with revenue growth?" -- requires data outside the ERP (marketing spend, ad budgets, etc.) | Acknowledge limitation. Suggest: "Control does not track marketing spend. Compare revenue trends to known campaign dates manually" |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Sales skill: Template 2 (monthly) | **Reuse** | Base data for all monthly trend computation |
| Sales skill: Template 9 (YoY) | **Extend** | Add growth rate computation |
| Customer skill: Spend trend detection | **Extend** | Broaden from decline-only to growth+decline ranking |
| Customer skill: Pareto analysis | **Extend** | Add period comparison for concentration trend |
| Advanced analytics: New customers by month | **Import** | Add growth rate computation on top |
| Financial skill: MoM P&L comparison | **Reuse** | Already computes month-over-month at GL level |

---

## 3. EXECUTIVE DASHBOARDS

**Scope constraint:** Two delivery modes: (1) text-based morning report (NL response with formatted markdown), and (2) React visual artifacts with charts. Both query the same underlying data via MCP. The text dashboard is the MVP; visual is the stretch goal.

**Why this matters for FLS:** Cain, Taylor, and Gretel currently have no consolidated daily view of the business. Control's built-in dashboard (Dashboard table, 86 rows) stores XML-based dashboard configurations for the Control desktop app -- not accessible from the NL interface. The morning dashboard replaces opening Control, running 3-4 Crystal Reports, and mentally combining the results.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Morning sales snapshot | "How did we do yesterday?" and "How are we doing this month?" -- day-start situational awareness. Show: yesterday's orders/revenue, MTD revenue vs prior month, YTD revenue vs plan | Med | Composes 3 existing queries: (1) yesterday's TransHeader summary, (2) MTD from Template 2, (3) YTD from Template 1. Add percent-of-plan if plan data available. Format as structured text block |
| Open orders pipeline | "What orders are in progress?" -- production awareness. Show: count and value by status (New, WIP, Built), count by station | Med | TransHeader WHERE TransactionType = 1 AND StatusID IN (1, 3). Already exists as "Open Orders" in advanced_analytics.md. Add station breakdown from production skill |
| AR summary | "Who owes us money?" -- cash collection awareness. Show: total AR, aging buckets (current/30/60/90+), top 5 overdue | Med | Already built in financial skill (AR Aging Detail query). Compose into dashboard format with summary + top overdue list |
| Artwork pipeline status | "What proofs need attention?" -- production bottleneck awareness. Show: count by artwork status, stuck items (>3 days in Pending Approval), due-today items | Med | Already built in production skill (Artwork Pipeline queries). Compose into dashboard section |
| Cash position | "How much cash do we have?" -- daily financial health. Show: bank balances, undeposited funds, total liquid | Low | Already built in financial skill (Bank Balance + Undeposited Funds queries). Format as single-line summary with detail available |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Plan pace indicator | "Are we ahead or behind plan?" -- visual traffic light (green/yellow/red) based on pace vs monthly plan. No Crystal Report provides this. Cain does this mentally | Med | Compare YTD actual to YTD plan from FLS_Sales_Plan_2026.md. Green = >95% of pace, Yellow = 80-95%, Red = <80%. Simple computation but requires plan data in the skill. Text-based: show as percentage with descriptive label |
| Week-over-week velocity | "Is this week slower or faster than last week?" -- short-term momentum indicator without daily noise | Low | COUNT and SUM for current week vs prior week. TransHeader WHERE SaleDate >= start of week. Simple date math. Shows whether the current week is gaining or losing momentum |
| Division summary (when division filtering ships) | "How is Banner Division vs Apparel doing?" -- the whole point of division filtering is to see this split on the dashboard. Gretel needs visibility into her division | Med | Revenue, order count, AR by DivisionID. Requires division filtering feature (see Section 4). Composes existing queries with DivisionID filter |
| Production throughput | "How fast are we turning orders?" -- average days from order to completion for recent orders. Leading indicator of capacity constraints | Med | DATEDIFF(SaleDate, ClosedDate) for recently completed orders. Already referenced in production skill (order cycle time). Surface as single metric: "Average 5.2 days order-to-close this month" |
| Payroll summary (when payroll via GL ships) | "What did payroll cost this period?" -- see Section 5. Include as a dashboard line item showing payroll expense for current and prior pay period | Med | Depends on payroll-via-GL feature. Show as: "Last payroll: $XX,XXX (Jan 27 period), Prior: $XX,XXX" |

### Visual Dashboard Differentiators (React Artifact)

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Monthly revenue bar chart with plan overlay | Visual comparison of actual vs plan by month. The most requested executive chart. Shows the 12-month revenue trajectory with plan line | High | Requires React artifact with charting library (Recharts recommended for React integration). Data: monthly actuals (Template 2) + plan targets. Render as bar (actual) + line (plan) |
| Revenue by product pie/donut chart | "Where does our revenue come from?" -- visual product mix. Currently only available as text table | High | Same data as sales Template 3/8 but rendered as a donut chart with labels. Top 8 categories + "Other" bucket for readability |
| YoY comparison line chart | Two lines showing this year vs last year revenue by month. Visual pattern matching for seasonal trends | High | Same data as Template 9 but rendered as dual-line chart. Enables visual recognition of seasonal patterns and growth |
| AR aging horizontal bar | Visual aging buckets with dollar amounts. Immediately shows how much is overdue at a glance | High | Same data as AR aging query but as stacked horizontal bar: Current (green), 30-day (yellow), 60-day (orange), 90+ (red) |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Drag-and-drop dashboard builder | Building a configurable dashboard layout system is a product in itself. FLS has 3-4 users, not 300. One well-designed dashboard is better than an infinitely configurable one | Design a single, opinionated dashboard layout that covers 90% of needs. "Good morning Cain, here is your daily business summary:" with fixed sections |
| Real-time auto-refresh | Requires WebSocket infrastructure, persistent connections, and background polling. The NL interface is request-response | Dashboard is generated on demand: "Morning dashboard" or "Daily report." Fresh data every time the user asks |
| Mobile-optimized responsive dashboard | Native mobile requires React Native or PWA infrastructure beyond current scope | Text-based dashboard is inherently mobile-friendly. Visual artifacts work in Claude's artifact viewer. True mobile app is a future milestone |
| User-configurable KPI selection | "Let Cain see revenue but let Gretel see only her division" -- requires user preference storage, authentication, and per-user configuration | Division filtering (Section 4) handles the Gretel use case. For now, all users see all data. User identification is manual: "Show me Gretel's division" |
| Email/Slack notification delivery | Scheduled dashboard delivery requires cron jobs, email infrastructure, and notification services | On-demand only. The user asks for the dashboard when they want it. Suggest: "Set a daily reminder to ask for your morning dashboard" |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Sales skill: Templates 1, 2, 3, 9 | **Reuse** | Revenue snapshots, monthly trends, product breakdown, YoY |
| Financial skill: AR aging detail | **Reuse** | AR summary section of dashboard |
| Financial skill: Bank balance + undeposited | **Reuse** | Cash position section |
| Financial skill: Cash flow summary | **Reuse** | Cash flow section of dashboard |
| Production skill: Artwork pipeline | **Reuse** | Artwork status section |
| Production skill: Station workload | **Reuse** | Production throughput |
| Existing morning dashboard template in SKILL.md | **Extend** | Basic morning dashboard queries already exist. Need composition and formatting |
| FLS_Sales_Plan_2026.md | **Import** | Plan targets for pace comparison |

---

## 4. DIVISION-LEVEL FILTERING AND REPORTING

**Scope constraint:** FLS has two divisions: Banner Division and Apparel Division (managed by Gretel). DivisionID exists on TransHeader, Journal, Ledger, Inventory, and Account. Division filtering means adding `WHERE DivisionID = @DivisionID` to existing queries and providing division-comparison views.

**Why this matters for FLS:** Gretel manages the Apparel Division and needs visibility into her division's performance without manually filtering every query. The v1.1 FEATURES.md explicitly deferred division-level reporting to "future milestone." That future is now. Control's DivisionData table has 3 rows (one is system/inactive, leaving 2 active divisions). The division model is simple but pervasive -- nearly every financial and sales table has DivisionID.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Division-filtered revenue | "How much revenue did the Apparel Division do this year?" -- the most basic division query | Low | Add `AND th.DivisionID = @DivisionID` to sales Template 1. DivisionData has only 3 rows; resolve division names to IDs in the skill. Must handle NULL DivisionID (default to company division) |
| Division-filtered AR | "What does Apparel Division AR look like?" -- division-specific collections | Med | Financial skill AR query with `AND th.DivisionID = @DivisionID`. Also filter Ledger entries by DivisionID for GL-based AR |
| Division-filtered P&L | "Show me the P&L for Banner Division" -- most critical financial view per division. Ledger.DivisionID exists for this purpose | Med | Financial skill P&L query with `AND l.DivisionID = @DivisionID`. GL entries are division-tagged. Revenue, COGS, and expenses should all respect the division filter |
| Division comparison side-by-side | "Compare Banner vs Apparel revenue" -- the natural follow-up to filtered views. Show both divisions in one table | Med | Same base query run twice (or with CASE WHEN grouping), presenting side-by-side columns. Revenue, order count, average order value per division |
| Division-aware NL routing | "How is Gretel's division doing?" -- NL should recognize "Gretel's division" = Apparel, "my division" (from Cain) = company-wide or Banner. Also: "by division" modifier on any existing query | Low | Glossary routing update: map "Apparel Division", "Gretel's division", "apparel" to DivisionID for Apparel. Map "Banner Division", "signs", "banners division" to Banner DivisionID. Add "by division" as a modifier that triggers grouping |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Division share-of-revenue trending | "Is Apparel growing as a share of total revenue?" -- strategic question for FLS's growth story. Apparel is the growth division; tracking its share shows whether diversification is working | Med | Division revenue / total revenue by month, compared across years. Shows whether Apparel is gaining share (goal) or Banner is still dominant |
| Division-filtered dashboard | "Morning dashboard for Apparel Division" -- Gretel gets a dashboard scoped to her division. All sections (revenue, AR, orders, artwork) filtered by DivisionID | Med | Compose the dashboard (Section 3) with division filter applied to all component queries. Requires DivisionID on TransHeader, Ledger, ArtworkGroup |
| Division-filtered customer ranking | "Who are the top Apparel Division customers?" -- customer intelligence scoped to division | Med | Customer skill Pareto/ranking queries with `AND th.DivisionID = @DivisionID`. Account.DivisionID shows the customer's default division, but orders can go to either division. Filter on TransHeader.DivisionID (order level), not Account.DivisionID |
| Division expense breakdown | "What are Apparel Division's expenses?" -- operating expense visibility per division from GL | Med | P&L filtered to GLClassificationType = 5002, further filtered by DivisionID. Shows which expenses are incurred by which division |
| Division cash flow | "Cash flow for Banner Division" -- division-specific cash movement | High | GL cash flow query with DivisionID filter. Complexity: bank account entries may not all have DivisionID (deposits aggregate multiple divisions). Need to verify data quality |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Inter-division elimination | In multi-entity accounting, inter-company transactions must be eliminated for consolidated reporting. FLS is a single legal entity with cost-center divisions, not separate legal entities. No elimination needed | Report each division independently and show a "Company Total" that is a simple sum. No consolidation adjustments required |
| Division-level security/access control | "Gretel can only see Apparel data" -- requires user authentication, session management, and per-user role enforcement | The NL interface does not have user authentication. All users can see all data. Gretel queries her division by asking for it explicitly. This is acceptable for a 4-person team |
| Dynamic division creation | "Add a new division" -- write operation requiring DivisionData inserts and GL configuration | Read-only system. If FLS adds a third division, the skill will be updated manually. With only 2 active divisions, this is a non-issue |
| Inter-division transfer pricing | Transfer pricing between divisions requires journal entries and complex accounting rules | FLS does not do inter-division transfers. Gretel's Apparel Division and the Banner Division are revenue centers, not transfer-pricing entities. Customer credits do have inter-division handling (documented in wiki), but that is a rare edge case |
| Division-specific chart of accounts | Some ERPs allow each division to have its own GL structure. Control uses a single COA with DivisionID as a dimension | Use the existing single COA. Filter by DivisionID. This is actually simpler and avoids mapping complexity |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Sales skill: All templates | **Extend** | Add optional DivisionID filter parameter to every template |
| Financial skill: AR, AP, P&L, cash flow | **Extend** | Add optional DivisionID filter. GL/Ledger already has DivisionID column |
| Customer skill: Rankings, Pareto | **Extend** | Filter by TransHeader.DivisionID, not Account.DivisionID |
| Production skill: Artwork, station | **Extend** | ArtworkGroup and station queries may need DivisionID (verify column availability) |
| Inventory skill: Stock levels | **Extend** | Inventory.DivisionID exists. Warehouses are division-specific: 10,11 = Company; 10000,10001 = Apparel |
| Glossary skill: NL routing | **Extend** | Add division-name-to-ID mapping and "by division" modifier |
| DivisionData table (3 rows) | **Import** | Need to query and document the exact IDs and names for both active divisions |

### Division Data Model Notes

FLS DivisionData table has 3 rows:
- ID=-1: System placeholder (IsSystem=true, IsActive=false, DivisionName='.')
- Remaining 2 rows: The active Banner and Apparel divisions (need to query for exact IDs)

Tables with DivisionID:
- **TransHeader**: Order-level division (most important for revenue filtering)
- **Ledger/GL**: GL entry-level division (most important for P&L filtering)
- **Journal**: Journal entry division
- **Inventory**: Inventory by division (warehouse segregation)
- **Account**: Customer default division (less useful than order-level)
- **Warehouse**: Division-specific warehouses (10/11 = Company, 10000/10001 = Apparel)

**Critical design decision:** Filter on TransHeader.DivisionID for revenue queries, not Account.DivisionID. A Banner Division customer can place an Apparel order. The order's division is what matters for financial reporting.

---

## 5. PAYROLL EXPENSE VIA GL ENTRIES

**Scope constraint:** GL-based payroll expense reporting only. No deep TimeCard, Payroll, or PayrollPaycheck analysis. Show payroll as an expense line item derived from GL entries, grouped by pay period. The v1.1 milestone explicitly deferred "payroll queries" as a separate domain. This milestone provides GL-level payroll expense visibility without building a full payroll skill.

**Why this matters for FLS:** Carrie Goetelman handles payroll processing in Control. Cain needs to see payroll costs per pay period for cash flow planning and P&L analysis. The financial skill already categorizes "Payroll" as a cash flow category (via `Ledger.PayrollID IS NOT NULL`). Expanding this to show payroll expense by period and category fills a gap without requiring deep TimeCard/Payroll domain expertise.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Payroll expense total by pay period | "What was last payroll?" -- most basic payroll question. Show total payroll expense for the most recent and prior pay periods | Med | GL entries with PayrollID IS NOT NULL, grouped by PayrollID or pay period date range. The Ledger.PayrollID links to the Payroll table which has pay period dates. Alternative: SUM(Amount) WHERE GLClassificationType = 5002 AND GLAccountName LIKE '%Payroll%' or '%Wage%' by month |
| Payroll as P&L line item | "Show payroll in the P&L" -- payroll is already included in Operating Expenses (GLClassificationType 5002) but is not broken out separately. Users expect to see payroll called out | Low | Already appears in P&L as individual GL accounts under 5002. May need to group payroll-related accounts (wages, employer taxes, benefits) into a "Total Payroll" subtotal |
| Monthly payroll trend | "Is payroll going up?" -- 6-12 month payroll expense trend | Med | SUM(Amount) for payroll GL accounts by month. Same pattern as monthly cash flow but filtered to payroll accounts. Identify payroll GL accounts by name pattern or explicit NodeID list |
| Payroll as percent of revenue | "What is our payroll-to-revenue ratio?" -- key profitability metric. For a $3M company, payroll is typically the largest expense | Med | (Total payroll expense / Total revenue) * 100 by period. Combines financial P&L with sales revenue. Useful for tracking as the company scales |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Payroll expense by category | "How much do we spend on wages vs taxes vs benefits?" -- breaks payroll into components using GL account names. Useful for identifying cost drivers | Med | Group payroll GL accounts into categories: Wages/Salaries (straight, OT, salary), Employer Taxes (FICA, FUTA, SUTA, Medicare), Benefits (401K, insurance), Other (PTO accruals). Requires mapping GL accounts to categories |
| Payroll per-employee from GL | "What did we pay Josh last month?" -- per-employee payroll from GL entries. Ledger has EmployeeID column | Med | SUM(Amount) WHERE PayrollID IS NOT NULL AND EmployeeID = @EmployeeID. Note: EmployeeID on Ledger entries may not be populated for all payroll entries. Verify data quality before exposing |
| Payroll by division | "What is Apparel Division payroll?" -- division-level payroll expense. Ledger.DivisionID enables this | Med | SUM(Amount) for payroll GL accounts WHERE DivisionID = @DivisionID. Combines with division filtering feature |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Paycheck detail (deductions, taxes, net pay) | Requires deep PayrollPaycheck, PayrollPaycheckPayItem, PayrollTaxTable analysis. Sensitive, regulated data. The data model is complex (tax tables, deductions, comp codes) | GL-level aggregates only. If user asks for paycheck details, route to: "Paycheck detail is available in Control under Payroll > View Payroll" |
| TimeCard-based labor costing | Linking time card hours to payroll costs requires hour-rate calculations, overtime rules, and shift differential handling. v1.1 explicitly deferred this | Show GL payroll totals. For labor hours, route to production skill (station hours). Do not attempt to compute hourly labor costs from TimeCard data |
| Tax filing or compliance queries | "What is our FICA liability?" -- payroll tax compliance requires PayrollTaxTable, TaxTableRow, and tax year calculations. High stakes, high complexity | Show GL tax expense amounts for informational purposes. Route compliance questions to: "Consult your accountant or use Control's payroll reports" |
| Payroll processing or adjustments | Write operations to Payroll, PayrollPaycheck, Journal tables | Read-only. Route to: "Process payroll in Control under Payroll > Process Payroll" |
| W-2 or year-end payroll reporting | Requires complete payroll data, tax table validation, and regulatory formatting | Route to: "Run W-2 reports in Control's payroll module" |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Financial skill: Cash flow "Payroll" category | **Extend** | Already identifies payroll GL entries via `Ledger.PayrollID IS NOT NULL`. Expand to per-period detail |
| Financial skill: P&L with OpEx breakdown | **Extend** | Payroll accounts already appear in OpEx. Add grouping/subtotaling |
| Financial skill: GL sign conventions | **Reuse** | Payroll expenses are positive amounts (debits) in GL |
| Core skill: Employee table reference | **Reuse** | Employee.ID for per-employee filtering |
| Wiki: Payroll GL setup | **Reference** | Payroll setup documentation describes GL account mapping (wage expense account, tax accrual accounts) |

### Payroll GL Account Identification

To identify payroll-related GL accounts, look for:
- GLClassificationType = 5002 (Operating Expenses) with AccountName containing: Wages, Salary, Payroll, FICA, FUTA, SUTA, Medicare, 401K, Insurance (employee benefit), Workers Comp, PTO
- The Payroll module maps each pay item to a GL Expense account and GL Accrual account
- Ledger.PayrollID IS NOT NULL reliably identifies entries created by the payroll module
- Manual journal entries for payroll adjustments may not have PayrollID -- cross-reference with EntryType = 1

---

## CROSS-FEATURE DEPENDENCIES

Features in this milestone are deeply interconnected. Build order matters.

### Dependency Graph

```
Division Filtering (foundational)
  |
  +---> Sales Forecasting (adds division-filtered forecasts)
  |       |
  |       +---> Executive Dashboard (composes forecasts into dashboard)
  |
  +---> Trend Detection (adds division-level trends)
  |       |
  |       +---> Executive Dashboard (composes trends into dashboard)
  |
  +---> Payroll via GL (adds division-level payroll)
          |
          +---> Executive Dashboard (adds payroll as dashboard section)
```

### Implementation Order Recommendation

1. **Division Filtering FIRST** -- It is a cross-cutting filter that modifies every other feature. Building forecasting without division filtering means rebuilding queries later.

2. **Trend Detection SECOND** -- Uses existing query patterns (already built in sales/customer/financial skills). Mostly computation-on-top-of-existing-queries, not new domain exploration.

3. **Sales Forecasting THIRD** -- Builds on trend detection (seasonal indices require monthly trend data). The forecast queries are new patterns but use the same base tables.

4. **Payroll via GL FOURTH** -- Independent of the above three. Can technically be built in parallel. Small scope, well-defined GL queries.

5. **Executive Dashboard LAST** -- Composes all of the above into a unified view. Cannot be completed until the component features are built. Text version first, visual artifacts second.

### Shared Business Logic

These rules apply across all 5 feature areas and are already encoded in existing skills:

| Rule | Features Affected | Source |
|------|------------------|--------|
| Use SubTotalPrice for revenue | Forecasting, Trends, Dashboard, Division | Core skill |
| Use SaleDate for date filtering | Forecasting, Trends, Dashboard, Division | Core skill |
| TransactionType = 1 for sales | All 5 | Core skill |
| StatusID NOT IN (9) exclude voided | All 5 | Core skill |
| DivisionID on TransHeader for order division | Division, all filtered queries | New (this milestone) |
| DivisionID on Ledger for GL division | Division, Payroll, P&L | New (this milestone) |
| COALESCE(DivisionID, default) for NULL handling | Division filtering | Core skill (referenced) |

---

## MVP RECOMMENDATION

For MVP of this milestone, prioritize features that compose existing validated queries into new views. The foundational data access is already built and validated to 99.98%. This milestone is about computation and composition, not data exploration.

### Phase 1: Division Filtering + Payroll GL (infrastructure)
1. **Query DivisionData** to document exact IDs and names for Banner and Apparel
2. **Division-filtered revenue, AR, P&L** -- add DivisionID filter to existing queries
3. **Division comparison view** -- side-by-side output format
4. **Payroll expense from GL** -- identify payroll GL accounts, query by pay period
5. **NL routing for division terms** -- glossary updates

### Phase 2: Trend Detection (analytical layer)
6. **Product category growth ranking** -- YoY by category with growth rate
7. **Customer growth/decline ranking** -- extend customer skill spend trend
8. **Revenue composition shift** -- share-of-wallet trending
9. **Rolling 12-month trend** -- trailing 12-month smoothed revenue
10. **Outlier flagging** -- 2-STDEV anomaly detection

### Phase 3: Sales Forecasting (predictive layer)
11. **Run-rate projection** -- simple and seasonal-adjusted
12. **Plan vs actual comparison** -- encode 2026 plan, compare monthly
13. **Weighted moving average** -- 3-month and 6-month WMA
14. **Forecast confidence range** -- STDEV-based high/low/expected

### Phase 4: Executive Dashboard (composition layer)
15. **Text-based morning dashboard** -- compose all features into structured report
16. **Visual revenue chart** -- React artifact with bar + plan line
17. **Visual AR aging** -- horizontal bar chart
18. **Division dashboard variant** -- filtered dashboard for Gretel

---

## COMPLEXITY SUMMARY

| Feature Area | Table Stakes | Differentiators | Anti-Features | Total Features |
|-------------|-------------|-----------------|---------------|----------------|
| Sales Forecasting | 5 | 5 | 5 | 10 features + 5 anti |
| Trend Detection | 5 | 6 | 4 | 11 features + 4 anti |
| Executive Dashboards | 5 (text) + 4 (visual) | 5 + 4 (visual) | 5 | 18 features + 5 anti |
| Division Filtering | 5 | 5 | 5 | 10 features + 5 anti |
| Payroll via GL | 4 | 3 | 5 | 7 features + 5 anti |
| **Totals** | **28** | **28** | **24** | **56 features + 24 anti** |

### Key Insights

1. **This milestone is composition-heavy, not exploration-heavy.** Unlike v1.1 which required discovering and validating data models for 5 new domains, this milestone builds on top of 73+ validated SQL queries across 8 existing skills. The risk is in the computation and presentation, not the data access.

2. **Division filtering is cross-cutting.** It touches every skill. The simplest implementation is a skill-level utility pattern: "For any query Q, Q(DivisionID) = Q WHERE ... AND DivisionID = @DivisionID." This must be designed once and applied consistently.

3. **The FLS_Sales_Plan_2026.md is a gift.** Cain already created a detailed monthly-by-product-category plan. Encoding this as skill reference data enables plan-vs-actual comparison immediately. No external data source needed.

4. **Text dashboard is the MVP; visual is the stretch.** The text dashboard composes existing queries into a formatted response. The visual dashboard requires React artifact infrastructure (Recharts, component design). Separate them into distinct phases.

5. **Payroll is intentionally shallow.** GL-level payroll expense visibility satisfies 80% of the use case (how much did payroll cost, is it trending up) without requiring deep TimeCard/Payroll domain expertise. The remaining 20% (per-employee detail, tax compliance) is deferred.

---

## SOURCES

### Internal (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/skill/SKILL.md` -- Main skill with all query templates
- `/Users/cain/projects/control-db-map/output/skill/references/advanced_analytics.md` -- Existing trend/analytics queries
- `/Users/cain/projects/control-db-map/output/skill/references/business_rules_validation.md` -- Validated revenue formula, monthly breakdowns
- `/Users/cain/projects/control-db-map/skills/control-erp-sales/control-erp-sales-SKILL.md` -- Sales templates 1-9
- `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` -- GL architecture, AR/AP, P&L, cash flow
- `/Users/cain/projects/control-db-map/skills/control-erp-financial/references/financial-analysis.md` -- YoY P&L, MoM P&L, cash flow, product-line margin
- `/Users/cain/projects/control-db-map/skills/control-erp-customers/control-erp-customers-SKILL.md` -- Customer ranking, spend trend, Pareto, churn
- `/Users/cain/projects/control-db-map/skills/control-erp-production/control-erp-production-SKILL.md` -- Artwork pipeline, station workload
- `/Users/cain/projects/control-db-map/output/FLS_Sales_Plan_2026.md` -- 2026 sales plan with monthly/category targets
- `/Users/cain/projects/control-db-map/output/schemas/DivisionData.md` -- Division table schema (3 rows)
- `/Users/cain/projects/control-db-map/output/schemas/SalesGoal.md` -- Sales goal table (248 rows, has GoalYear/GoalMonth)
- `/Users/cain/projects/control-db-map/output/schemas/Dashboard.md` -- Dashboard table (86 rows, XML-based, not useful for NL)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/crm_payroll_system_knowledge.md` -- Payroll system architecture, GL mapping
- `/Users/cain/projects/control-db-map/.planning/milestones/v1.1-MILESTONE-AUDIT.md` -- v1.1 audit with deferred items (payment forecast, payroll)

### External (MEDIUM confidence)
- [How to Conduct Time Series Forecasting in SQL with Moving Averages](https://towardsdatascience.com/how-to-conduct-time-series-forecasting-in-sql-with-moving-averages-fd5e6f2a456/) -- SQL forecasting patterns
- [Weighted vs Simple Moving Average with SQL Server T-SQL Code](https://www.mssqltips.com/sqlservertip/6312/weighted-vs-simple-moving-average-with-sql-server-tsql-code/) -- T-SQL WMA implementation
- [Forecasting in presence of Seasonal effects using the Ratio to Moving Average method](http://www.silota.com/docs/recipes/sql-ratio-to-moving-average-seasonal-index.html) -- Seasonal decomposition in SQL
- [SQL Growth Analysis - Calculating MoM and YoY Changes](https://sqlguroo.com/blog/sql-growth-calculations/) -- Growth rate computation patterns
- [Top 5 React Chart Libraries 2026](https://www.syncfusion.com/blogs/post/top-5-react-chart-libraries) -- Recharts recommended for React dashboard integration
- [Executive Dashboards: 13+ Examples, Templates & Best Practices](https://improvado.io/blog/executive-dashboards) -- Dashboard KPI selection guidance
- [Manufacturing KPI Dashboard 2026 Guide](https://www.method.me/blog/manufacturing-kpi-dashboard/) -- Manufacturing-specific KPIs

**Confidence:** HIGH for all feature categories. This milestone builds entirely on validated data models (99.98% accuracy) and existing SQL patterns. The primary risk is in composition complexity, not data uncertainty. Division filtering requires a one-time query to verify DivisionData IDs. Payroll GL accounts require a one-time query to identify payroll NodeIDs.
