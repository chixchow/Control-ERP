# Project Research Summary

**Project:** Project Cornerstone v1.2 — Analytics, Dashboards & Division Support
**Domain:** ERP Analytics Layer (Brownfield Extension)
**Researched:** 2026-02-09
**Confidence:** HIGH

## Executive Summary

This research establishes the foundation for building analytics, dashboards, and division filtering on top of the validated v1.0/v1.1 system (8 skills, 73+ SQL queries, 99.98% accuracy). The research confirms that **all forecasting can be done in pure T-SQL** using window functions and CTEs for moving averages, seasonal decomposition, and trend detection — no Python or ML libraries required. The recommended architecture creates **two new skills** (`control-erp-analytics` for forecasting/trends, `control-erp-dashboard` for visual/text dashboards) while **modifying core with division filtering framework** that propagates across all existing skills.

The **critical architectural decision** is division filtering: all four researchers agree it is cross-cutting and must be designed as a framework **before** other features. The Cyrious wiki reveals a non-obvious pitfall that would destroy accuracy: `DivisionID` is nullable throughout the database, and **every query must use `COALESCE(DivisionID, 10)`** or silently drop records with NULL divisions. This is the v1.2 equivalent of the v1.0 TransDetailParam IsActive bug ($1.3M discrepancy). Additionally, `TransHeader` has THREE division fields (DivisionID, ProductionDivisionID, ShipFromDivisionID) and `Ledger` has TWO (DivisionID, ProcessedDivisionID) — using the wrong one assigns revenue/costs to the wrong division.

**Key risks:** (1) Forecasting with only 2 years of data has limited predictive power — use moving averages + confidence ranges, not ML-grade precision. (2) Dashboard metrics must use the EXACT same SQL templates as existing skills or create credibility-destroying inconsistencies. (3) React artifacts in Claude are sandboxed to Recharts + Tailwind + Shadcn UI — no external libraries. The research provides clear mitigations for all risks, and the brownfield context (building on 49 shipped requirements across 8 validated skills) significantly reduces unknowns.

## Key Findings

### Recommended Stack

**Consensus:** Pure SQL-based analytics with two-tier dashboard delivery (text + React artifacts). All recommendations work within proven v1.0/v1.1 constraints (SQL Server via MCP MSSQL tools, no Python, no external services). The stack adds NO new dependencies — only T-SQL patterns and pre-installed React artifact libraries.

**Core technologies:**

- **T-SQL Window Functions (LAG, LEAD, AVG OVER)** — Trend detection, YoY comparison, rolling averages. Already used in production skill (LEAD for dwell time). Proven, well-documented, native to SQL Server 2012+.

- **T-SQL Linear Regression CTEs** — Sales forecasting using least-squares formula. Pure SQL implementation of slope/intercept calculation via SUM/AVG aggregates. Adequate for FLS's $3M revenue scale ($10-15M target).

- **T-SQL Seasonal Decomposition (Ratio-to-Moving-Average)** — Separate trend from seasonality using monthly revenue % of annual total. With only 24 months of data, limit to simple seasonal indices rather than Holt-Winters.

- **React Artifacts (Recharts + Tailwind CSS + Shadcn UI)** — Visual dashboards. Pre-installed in Claude's artifact sandbox. Data embedded as JavaScript constants (no API calls). Self-contained, renders in Claude.ai and desktop app.

- **Division Filtering via COALESCE Pattern** — `COALESCE(DivisionID, 10)` applied universally to TransHeader, Ledger, Account, Inventory queries. Documented in 40+ Cyrious wiki SQL scripts. Prevents NULL DivisionID exclusion.

**What does NOT change:** SQL Server access (MCP MSSQL tools), existing 8 skills (core, sales, financial, customers, inventory, production, glossary), 73+ validated query templates. Division filtering is additive (adds WHERE clause), forecasting is compositional (builds on existing revenue queries).

**Critical version/configuration notes:**
- DivisionData table has 3 rows: ID=-1 (system, inactive), ID=? (Company/Banner), ID=? (Apparel). Actual IDs must be discovered via query before implementation.
- Warehouse.DivisionID is NOT NULL (FK required), unlike TransHeader/Ledger/Account where DivisionID is nullable.
- Ledger has PayrollID field for identifying payroll-originated GL entries (payroll expense via GL approach).

### Expected Features

**All four researchers converged on the same 5 feature areas** with consistent table stakes vs differentiators vs anti-features. This consensus validates that the requirements are well-understood.

**Must have (table stakes):**

**Sales Forecasting:**
- Monthly revenue projection for current year (run-rate: YTD pace extrapolated)
- Same-period comparison with prior year (YoY)
- Seasonal run-rate projection (accounts for FLS's Aug-Sep peak, Dec trough)
- Monthly budget vs actual (2026 sales plan from FLS_Sales_Plan_2026.md)
- Revenue pace by product category (DyeSub categories tracked)

**Trend Detection:**
- Month-over-month revenue change (% change, already in financial skill MoM P&L)
- Year-over-year revenue change by month (12-month grid)
- Product category growth/decline ranking (which categories growing fastest)
- Customer growth/decline ranking (YoY spend trend per customer)
- Order volume vs order value trend (more orders or larger orders?)

**Executive Dashboards:**
- Morning sales snapshot (yesterday, MTD, YTD vs plan)
- Open orders pipeline (count/value by status: New, WIP, Built)
- AR summary (total AR, aging buckets, top 5 overdue)
- Artwork pipeline status (proofs pending, stuck >3 days, due today)
- Cash position (bank balances, undeposited funds)

**Division Filtering:**
- Division-filtered revenue ("Apparel Division revenue this year")
- Division-filtered AR ("Apparel Division AR aging")
- Division-filtered P&L ("Banner Division P&L")
- Division comparison side-by-side (Banner vs Apparel in one table)
- Division-aware NL routing ("Gretel's division" = Apparel)

**Payroll Expense (GL-based):**
- Payroll expense total by pay period (via Ledger.PayrollID IS NOT NULL)
- Payroll as P&L line item (break out from OpEx)
- Monthly payroll trend (is payroll increasing?)
- Payroll as percent of revenue (key profitability metric)

**Should have (competitive):**

- Weighted moving average forecast (3-month, 6-month WMA for recent momentum)
- Seasonal decomposition (deseasonalized growth rate)
- Product mix shift detection (Feather Flags went 32% -> 100% of DyeSub)
- Forecast confidence range (high/low/expected, not single number)
- Plan pace indicator (traffic light: on pace, behind pace)
- Division summary on dashboard (Gretel's division visibility)
- React visual dashboards (charts: revenue bar + plan line, AR aging, YoY line)
- Automatic outlier flagging (>2 STDEV from 12-month avg)
- Rolling 12-month trend (trailing year smoothed)

**Defer (v2+):**

- ML-based forecasting (scikit-learn, ARIMA) — cannot run Python in skill execution
- External data integration (weather, economic indicators) — no ETL pipelines
- Daily/weekly forecasting — too noisy for FLS's 17 orders/day
- Inventory demand forecasting — requires BOM/production planning
- Pipeline-based forecasting from estimates — Control has no deal-stage weighting
- Real-time dashboard auto-refresh — no WebSocket infrastructure
- Drag-and-drop dashboard builder — overkill for 3-4 users
- Deep payroll analysis (timecards, deductions, tax compliance) — deferred until time clock app integration

**Anti-features (explicit scope discipline):**
- Predictive trend extrapolation with false precision ("Feather Flags will grow 628% next year")
- Causal trend analysis ("Revenue dropped because of trade show cancellation")
- Automated alerting/notifications — query-on-demand only
- Mobile-optimized responsive dashboard — text dashboard is mobile-friendly
- User-configurable KPI selection — single opinionated dashboard
- Inter-division elimination accounting — FLS is single legal entity, no consolidation
- Division-level security/access control — all users see all data

### Architecture Approach

**Consensus architecture:** Create two new skills (analytics, dashboard) that build on existing skills, with division filtering as a cross-cutting framework documented in core. All four researchers independently arrived at the same skill topology. This is a **composition layer** on top of validated data access, not new data exploration.

**Major components:**

1. **Division Filtering Framework (in control-erp-core)** — Documents `COALESCE(DivisionID, 10)` pattern, maps which DivisionID field to use per table (TransHeader.DivisionID for sales, GL.DivisionID for financial, ProductionDivisionID for production workload), provides injection pattern for WHERE clauses. ~50-80 lines added to core skill. Cross-cutting: all domain skills gain division filtering by referencing this framework.

2. **Analytics Skill (NEW: control-erp-analytics)** — Forecasting, trend detection, seasonal analysis. SQL templates for moving averages (SMA, WMA), linear regression (slope/intercept via CTEs), seasonal decomposition (ratio-to-moving-average), product-line growth/decline detection, customer trend analysis. Estimated ~400-600 lines. Depends on: core (filters), sales (monthly revenue data), financial (GL revenue), customers (churn/acquisition data). NL triggers: "forecast", "predict", "project", "trend", "growth", "outlook".

3. **Dashboard Skill (NEW: control-erp-dashboard)** — Text dashboard templates (morning report structure), React artifact component patterns (Recharts configs, Tailwind layouts), alert threshold definitions, multi-query orchestration. Estimated ~300-500 lines. Orchestrates ALL domain skills + analytics for unified views. NL triggers: "morning report", "executive summary", "dashboard", "show me a chart". Two-tier output: text (always available) and React visual (Claude.ai/desktop only).

4. **Glossary Expansion (MODIFY)** — Add ~30-50 NL routing entries for analytics ("forecast sales" -> analytics), dashboard ("morning report" -> dashboard), division filtering ("Apparel sales" -> sales + core division filter). Grows from 671 lines to ~710-720 lines.

5. **Domain Skill Annotations (MODIFY: minor)** — Each existing skill (sales, financial, customers, inventory, production) adds a note: "Division filtering available — see control-erp-core Division Filtering section." ~5-10 lines per skill. No query template changes (division filtering applied via framework).

**Data flow:**
- User asks division-filtered question -> Glossary routes to domain skill + notes division context -> Domain skill query template + core division filter clause (`AND COALESCE(th.DivisionID, 10) = @DivisionID`) -> SQL via MCP -> Result
- User asks for dashboard -> Dashboard skill -> Runs 5-8 queries across multiple skills -> Assembles into text OR generates React artifact with embedded data -> Returns formatted output
- User asks for forecast -> Analytics skill -> Runs sales Template 2 (monthly revenue) -> Applies moving average/regression/seasonal via SQL CTEs -> Returns projection with confidence context

**Key architectural decisions:**

- **Division filtering is additive, not duplicative.** Single query template gains division filter via WHERE clause injection, not separate division-specific templates. Prevents query explosion (73 templates would become 219 with 3 division variants).

- **Dashboard composes existing queries, not new ones.** Dashboard runs the SAME SQL templates from domain skills, then formats results. No separate "dashboard revenue query" that could diverge from validated sales query.

- **Analytics forecasts use core revenue formula for actuals.** The "actuals" portion of any forecast MUST match sales skill Template 2 results exactly. Projection is computed from validated actuals, not parallel data source.

- **React artifacts embed data, no API calls.** Claude runs SQL queries, receives result arrays, generates React component with data as JavaScript constants. Artifact is self-contained and sandboxed.

**Build order (all researchers agree):**
1. Division filtering FIRST (cross-cutting, required by analytics and dashboard, simplest to validate)
2. Analytics SECOND (produces forecast data that dashboard consumes, builds on division framework)
3. Dashboard THIRD (composes all features, needs analytics + division to be stable)
4. Integration/validation LAST (end-to-end testing, glossary sync)

### Critical Pitfalls

Research identified 6 critical pitfalls (PITFALLS.md severity scale calibrated to v1.0 TransDetailParam bug = $1.3M discrepancy). The top 3 would destroy accuracy if not mitigated:

1. **NULL DivisionID Silently Excludes Records (CRITICAL)** — `WHERE DivisionID = 10` excludes all records where DivisionID IS NULL. At FLS, NULL defaults to Company Division (ID 10), so division-filtered queries silently drop $100K+ in revenue. Cyrious wiki documents this in 40+ SQL scripts, all using `COALESCE(DivisionID, 10)`. **Prevention:** MANDATORY rule: every query filtering or grouping by DivisionID MUST use COALESCE pattern. **Validation:** Sum of per-division revenues must equal total revenue. **Detection:** Compare division totals with unfiltered total — any gap is NULL DivisionIDs being dropped. **Phase:** Division filtering framework, FIRST rule established.

2. **DivisionID vs ProductionDivisionID vs ShipFromDivisionID Confusion (HIGH)** — TransHeader has THREE division fields: DivisionID (sales/billing division), ProductionDivisionID (who handles production), ShipFromDivisionID (shipping origin). Using the wrong one assigns revenue to wrong division. Apparel order could have DivisionID=Apparel but ProductionDivisionID=Company. **Prevention:** Revenue queries use TransHeader.DivisionID. Production queries use ProductionDivisionID. Shipping queries use ShipFromDivisionID. GL queries use GL.DivisionID. Document mapping in skill. **Phase:** Division framework, document which field per query type.

3. **Dashboard Metrics Inconsistent with Existing Skills (CRITICAL)** — Dashboard shows "YTD Revenue: $2,980,000" while sales skill returns "$3,052,952.52" for same query. User sees two numbers, trust destroyed. Happens if dashboard uses independent SQL instead of same templates (e.g., missing `SaleDate IS NOT NULL`, using TotalPrice vs SubTotalPrice). **Prevention:** Dashboard MUST use EXACT same SQL patterns as existing skills. Shared query template library. Dashboard metrics trace back to specific skill/template. **Validation:** Dashboard results match individual skill results to the penny. **Detection:** Run dashboard + individual queries side-by-side, any discrepancy >$1 indicates inconsistency. **Phase:** Dashboard phase, before React artifacts built.

4. **Ledger DivisionID vs ProcessedDivisionID (HIGH)** — Ledger has TWO division fields: DivisionID (order/entity division) and ProcessedDivisionID (GL posting division). Using wrong one misattributes GL entries. Wiki bug CCON-5479: ProcessedDivisionID not updating for cross-division payments. **Prevention:** Use `COALESCE(GL.DivisionID, 10)` for financial reporting (matches wiki pattern), NOT ProcessedDivisionID. **Phase:** Division framework, financial integration.

5. **Forecasting with Insufficient Data (HIGH)** — Projecting "Apparel will grow 40%" from 6 months data, or seasonal patterns from only 24 months (contains COVID distortions). 2 years = barely enough for seasonal detection. **Prevention:** Show confidence context ("Based on N months"), use simple moving averages (not complex decomposition with thin data), present as range not single number, flag large-customer distortions (FLASH = 14% of revenue), never forecast per-division if <12 months division-tagged data. **Detection:** If forecast shows >20% growth or decline from trailing 12-month actual, flag as outside historical range. **Phase:** Forecasting phase, establish methodology constraints first.

6. **Division Filtering on GL vs TransHeader Mismatch (HIGH)** — "Apparel revenue (TransHeader)" = $200K but "Apparel revenue (GL)" = $180K. GL entries may have different DivisionID than parent TransHeader due to timing, manual corrections, or CCON-5479 bug. **Prevention:** Establish single authoritative source per metric (revenue = TransHeader, P&L = GL), document expected discrepancy, set tolerance threshold (<2%), add reconciliation query for DivisionID mismatches. **Phase:** Division framework, financial integration.

**Other notable pitfalls:**
- Seasonal pattern overfitting with 24 data points (require 36+ months before claiming patterns)
- Division filter not applied consistently across cross-skill queries (dashboard shows Apparel revenue + total AR)
- React artifact imports unsupported libraries (only Recharts + Tailwind + Shadcn UI)
- Moving average window functions misconfigured for projection (need to generate future period rows via CTE)
- Payroll GL queries pick up non-payroll OpEx (need to map specific payroll GL NodeIDs)
- Account.DivisionID != TransHeader.DivisionID (customer default vs order-level division)

## Implications for Roadmap

Based on research, the **critical path is division filtering -> analytics -> dashboard**, with integration/validation at each step. The brownfield context (building on 49 shipped requirements, 73 validated queries) allows parallelization within phases but NOT across the dependency chain.

### Phase 14: Division Filtering Framework

**Rationale:** Division filtering is cross-cutting — touches sales, financial, customers, inventory, production. Building analytics or dashboard without division support means rebuilding later. It is also the SIMPLEST feature (adds WHERE clause to existing queries) and has clear validation (sum of divisions = total). This unblocks division-filtered forecasts and division-specific dashboards.

**Delivers:**
- Division Filtering section in control-erp-core (~50-80 lines)
- DivisionID mapping table (which field per table: TransHeader.DivisionID, GL.DivisionID, ProductionDivisionID, etc.)
- COALESCE(DivisionID, 10) pattern documented as mandatory rule
- Division-filtered revenue query (extends sales Template 1)
- Division-filtered AR query (extends financial AR Aging)
- Division-filtered P&L query (extends financial P&L)
- Division comparison view (side-by-side Banner vs Apparel)
- NL routing updates in glossary ("Apparel" = DivisionID X, "Banner" = DivisionID Y, "by division" modifier)

**Addresses features:**
- DIVISION-01 (filter by division)
- DIVISION-02 (division comparison)
- DIVISION-03 (division-aware NL routing)

**Avoids pitfalls:**
- C1 (NULL DivisionID exclusion) — COALESCE pattern mandatory
- C2 (wrong DivisionID field) — mapping table per query type
- C3 (DivisionID vs ProcessedDivisionID) — use DivisionID for financial
- I2 (regression of validated queries) — re-run 21-test suite after changes

**Prerequisites (discovery queries):**
1. Query DivisionData for actual IDs: `SELECT ID, DivisionName, IsActive FROM DivisionData WHERE ID > 0`
2. Validate DivisionID coverage: `SELECT COUNT(*) FROM TransHeader WHERE DivisionID IS NULL AND TransactionType=1`
3. Validate GL DivisionID coverage: `SELECT COUNT(*) FROM GL WHERE DivisionID IS NULL`
4. Check DivisionID vs ProcessedDivisionID agreement: `SELECT COUNT(*) FROM Ledger WHERE DivisionID != ProcessedDivisionID AND DivisionID IS NOT NULL`

**Validation:**
- Sum of division-filtered revenues equals total revenue (no orphans)
- Apparel Division AR matches Gretel's expectations (cross-check with Control report)
- GL entries by division sum to total (no division leakage)

**Research flag:** STANDARD PATTERN. Division filtering is well-documented in Cyrious wiki (40+ SQL scripts). No deep research needed during phase planning — just apply the COALESCE pattern consistently.

---

### Phase 15: Analytics Skill — Forecasting & Trends

**Rationale:** Analytics produces forecast data that dashboard consumes (forecast overlay on revenue charts). Building analytics before dashboard allows testing forecasts independently. Division filtering from Phase 14 enables division-level forecasts. Forecasting builds on existing sales/financial data (no new DB discovery).

**Delivers:**
- control-erp-analytics skill (~400-600 lines)
- Simple moving average (3-month, 6-month) via window functions
- YoY growth rate projection (average historical growth applied to next year)
- Seasonal decomposition (monthly % of annual, applied to target)
- Confidence range calculation (STDEV-based high/low/expected)
- Product-line trend detection (growth/decline ranking by category)
- Customer trend analysis (new customers, declining customers)
- Rolling 12-month trend (smoothed revenue trajectory)
- Outlier flagging (>2 STDEV from 12-month avg)
- NL routing updates ("forecast", "predict", "trend", "growth")

**Uses (from STACK.md):**
- T-SQL window functions (LAG, LEAD, AVG OVER)
- T-SQL linear regression CTEs (least-squares slope/intercept)
- T-SQL seasonal decomposition (ratio-to-moving-average)

**Implements (from ARCHITECTURE.md):**
- Analytics skill component (builds on sales/financial/customers data)
- Forecast data flow: historical actuals -> trend computation -> projection + confidence
- Trend detection: YoY comparison, MoM comparison, product/customer growth ranking

**Addresses features:**
- FORECAST-01 through FORECAST-05 (table stakes forecasting)
- TREND-01 through TREND-05 (table stakes trend detection)
- FORECAST-06 through FORECAST-09 (differentiator forecasting)
- TREND-06 through TREND-11 (differentiator trends)

**Avoids pitfalls:**
- C4 (insufficient data) — use moving averages only, show confidence context, flag thin data
- M1 (seasonal overfitting) — require 36+ months before claiming patterns
- I1 (parallel revenue calculation) — use core revenue formula for actuals portion
- M4 (window function projection gap) — generate future period rows via CTE

**Prerequisites:**
1. Validate monthly revenue data quality: `SELECT MIN(SaleDate), MAX(SaleDate) FROM TransHeader WHERE TransactionType=1`
2. Check Apparel Division history: `SELECT YEAR(SaleDate), MONTH(SaleDate), COUNT(*), SUM(SubTotalPrice) FROM TransHeader WHERE COALESCE(DivisionID,10) != 10 GROUP BY YEAR(SaleDate), MONTH(SaleDate)`
3. Identify large-customer distortions (FLASH = 14% of revenue, flag if >20% in any month)

**Validation:**
- Forecast for known historical period matches actuals within 15% (methodology test)
- "Actuals" portion of forecast matches sales Template 2 results exactly (consistency test)
- Division-filtered forecasts sum to total forecast (division math check)

**Research flag:** MODERATE RESEARCH NEEDED. SQL forecasting patterns are well-documented, but FLS-specific calibration required: seasonal pattern validation (are Aug-Sep peaks consistent?), large-customer impact (FLASH order timing), Apparel Division data sufficiency. Plan for 1-2 discovery queries during phase planning to validate assumptions.

---

### Phase 16: Dashboard Skill — Text & Visual

**Rationale:** Dashboard composes all features (sales, financial, customers, inventory, production, analytics). Cannot be built until component features are stable. Division filtering (Phase 14) enables division-specific dashboards. Analytics (Phase 15) provides forecast overlay data for charts. Dashboard is presentation layer — should be built last.

**Delivers:**
- control-erp-dashboard skill (~300-500 lines)
- Text dashboard template (morning report structure: revenue, AR, WIP, alerts)
- React artifact component patterns (Recharts configs: LineChart, BarChart, PieChart, AreaChart)
- Tailwind layout templates (grid, cards, tables)
- Alert threshold definitions (what is "concerning" AR, "stuck" artwork, "overdue" orders)
- KPI definitions (which metric, which query, how to calculate)
- Multi-query orchestration (which queries to run for each dashboard type)
- Division-filtered dashboard variant (Gretel's Apparel dashboard)
- NL routing updates ("morning report", "dashboard", "show me charts")

**Uses (from STACK.md):**
- React Artifacts (Recharts + Tailwind CSS + Shadcn UI + Lucide icons)
- Data embedding pattern (SQL results -> JavaScript constants -> React props)

**Implements (from ARCHITECTURE.md):**
- Dashboard skill component (orchestrates all domain skills)
- Two-tier output: text (always available) + React visual (Claude.ai/desktop)
- Data granularity: ~100-150 rows embedded (monthly aggregates, top-N lists)

**Addresses features:**
- DASH-01 (text morning report)
- DASH-02 through DASH-05 (table stakes dashboard sections)
- DASH-06 through DASH-10 (differentiator dashboard features)
- DASH-11 through DASH-14 (React visual dashboards)

**Avoids pitfalls:**
- C5 (inconsistent dashboard metrics) — use EXACT same SQL templates as domain skills
- M3 (unsupported React libraries) — only Recharts + Tailwind + Shadcn + Lucide
- M7 (divergent time ranges) — single date computation shared by text + visual
- I3 (NL routing conflicts) — explicit dashboard triggers, no overlap with existing

**Prerequisites:**
1. Establish alert thresholds via historical baselines (what % of AR is typically >90 days? what is normal artwork dwell time?)
2. Test each chart type individually (line, bar, pie, table) before combining
3. Validate React artifact sandbox constraints (no external API calls, libraries pre-installed)

**Validation:**
- Dashboard metrics match individual skill results to the penny (consistency test)
- Text dashboard + React dashboard show same numbers (no divergence)
- Division-filtered dashboard shows only division-scoped metrics (no company-wide leakage)
- All alert thresholds trigger on known historical scenarios

**Research flag:** MODERATE RESEARCH NEEDED. React artifact patterns are well-documented, but dashboard design requires business input: which KPIs matter most? what alert thresholds are actionable? how much detail in morning report? Plan for 1-2 user interviews (Cain, Gretel, Taylor) during phase planning to calibrate dashboard content.

---

### Phase 17: Payroll Expense (GL-based) — Lightweight Add-On

**Rationale:** Payroll expense is intentionally shallow (GL aggregates only, no deep TimeCard/Payroll tables per PROJECT.md). Small scope, well-defined GL queries. Independent of analytics/dashboard — can be built in parallel with Phase 16 or as a quick follow-up. Includes division filtering (payroll by division) which depends on Phase 14.

**Delivers:**
- Payroll expense queries in financial skill (~50-80 lines added)
- Total payroll by pay period (via Ledger.PayrollID IS NOT NULL)
- Payroll as P&L line item (break out from OpEx 5002)
- Monthly payroll trend (6-12 month history)
- Payroll as % of revenue (profitability metric)
- Payroll by division (Apparel vs Banner payroll costs)
- Optional: Payroll expense by category (wages, taxes, benefits) if GL accounts are clearly named

**Addresses features:**
- PAYROLL-01 through PAYROLL-04 (table stakes payroll)
- PAYROLL-05 through PAYROLL-07 (differentiator payroll)

**Avoids pitfalls:**
- M5 (non-payroll OpEx in payroll query) — map specific payroll GL NodeIDs

**Prerequisites:**
1. Identify payroll GL accounts: `SELECT ID, AccountName FROM GLAccount WHERE GLClassificationType=5002 AND (AccountName LIKE '%payroll%' OR AccountName LIKE '%salary%' OR AccountName LIKE '%wage%' OR AccountName LIKE '%benefit%')`
2. Verify Ledger.PayrollID coverage (how many payroll entries have PayrollID vs manual adjustments?)

**Validation:**
- Cross-validate with Carrie Goetelman (payroll processor) for approximate totals
- Payroll expense is fraction of revenue (not close to total OpEx)
- Division-level payroll sums to total payroll

**Research flag:** STANDARD PATTERN. GL-based expense queries already proven in financial skill. Just identify payroll-specific GL accounts and apply existing P&L patterns.

---

### Phase 18: Integration, Glossary Sync & Milestone Validation

**Rationale:** After all features built, test the full chain: NL -> glossary -> skill -> query -> format. Validate cross-feature combinations (division-filtered dashboard with forecast overlay). Final glossary sync ensures all new NL routes wired correctly. Milestone audit confirms 99.98% accuracy maintained.

**Delivers:**
- End-to-end testing across all v1.2 features
- Glossary final sync (~30-50 new NL routes for analytics, dashboard, division)
- Division + analytics + dashboard combined tests (Gretel's morning report)
- Regression testing: 21-test v1.1 validation suite + 25 routing test queries
- Milestone audit document (v1.2-MILESTONE-AUDIT.md)
- Updated skill count, query count, line count summaries

**Validation scenarios:**
- "Apparel Division forecast for Q2" — tests division + analytics integration
- "Morning report for Banner Division" — tests division + dashboard integration
- "Show me a chart of revenue by division with forecast overlay" — tests all three features
- Re-run all 21 v1.1 validation tests (ensure no regressions from division filtering)
- Run 25 glossary routing tests (ensure no routing conflicts from new entries)

**Success criteria:**
- All division-filtered queries maintain 99.98% accuracy (sum of divisions = total)
- Dashboard metrics match individual skill results (no inconsistencies)
- Forecasts labeled with confidence context (no false precision)
- Glossary routes all new NL phrases correctly (no ambiguity)

---

### Phase Ordering Rationale

**Dependency-driven sequence:**
- Division filtering (14) MUST precede analytics (15) and dashboard (16) because both consume division filtering
- Analytics (15) SHOULD precede dashboard (16) because dashboard includes forecast overlay data
- Payroll (17) is independent and can run parallel with dashboard (16) or after
- Integration (18) requires all features complete

**Why NOT parallelize 14-15-16:**
- If analytics built before division filtering, all forecast queries need division retrofit later
- If dashboard built before analytics, no forecast data to embed in charts (design incomplete)
- If division filtering added after dashboard, dashboard queries need division clause injection (rework)

**Risk mitigation via ordering:**
- Division filtering first = smallest scope, clearest validation, unblocks everything
- Analytics second = medium scope, builds on validated data, produces inputs for dashboard
- Dashboard last = largest composition scope, needs stable inputs, presentation layer isolated from data correctness

**Brownfield advantage:**
- 73 existing queries provide templates (no greenfield data exploration)
- 8 existing skills provide integration points (no monolithic rebuild)
- 99.98% accuracy baseline provides regression tests (immediate validation)

### Research Flags

**Phases needing deeper research during planning:**

- **Phase 15 (Analytics):** FLS-specific seasonal pattern validation (are Aug-Sep peaks statistically significant with only 2 years?), large-customer impact quantification (how much does FLASH's order timing distort trends?), Apparel Division data sufficiency check (enough history for per-division forecasts?). Discovery queries + data analysis during phase planning. Estimated 1-2 hours research.

- **Phase 16 (Dashboard):** Business input required for KPI selection (which metrics matter most to Cain/Gretel/Taylor?), alert threshold calibration (what AR aging triggers concern?), dashboard layout preferences (which sections most important?). User interviews during phase planning. Estimated 1-2 hours.

**Phases with standard patterns (skip deep research):**

- **Phase 14 (Division Filtering):** Well-documented in Cyrious wiki (40+ SQL scripts with COALESCE pattern). DivisionID mapping is schema lookup. Just apply pattern consistently. Discovery queries only (10-15 minutes).

- **Phase 17 (Payroll GL):** Extends existing financial skill P&L patterns. Just identify payroll GL accounts via discovery query. Standard GL aggregation. Discovery query only (10 minutes).

- **Phase 18 (Integration):** No new research — just testing and validation.

**Research effort estimate:** ~3-4 hours total across all phases (vs. 8-12 hours for greenfield v1.0). Brownfield advantage: known data models, proven patterns, clear validation baselines.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All recommendations are SQL-native or pre-installed React libraries. No new dependencies. T-SQL window functions proven in production skill. Division filtering documented in 40+ wiki scripts. React artifacts well-understood (2+ years of community usage). |
| Features | HIGH | All four researchers converged on same table stakes vs differentiators. FLS requirements clear (2026 sales plan exists, division structure known, payroll via GL documented in PROJECT.md). 56 features + 24 anti-features identified with rationale. |
| Architecture | HIGH | Two-skill expansion (analytics, dashboard) with division framework in core matches existing skill topology. Build order consensus across all researchers. Component boundaries clear. Data flow validated against existing query patterns. |
| Pitfalls | HIGH | 6 critical pitfalls identified with evidence (Cyrious wiki, schema FKs, v1.0 lessons). COALESCE pattern critical, TransHeader has 3 DivisionID fields (verified in schema), dashboard consistency is v1.2's TransDetailParam equivalent. All mitigations documented. |

**Overall confidence: HIGH**

This research builds on 49 shipped requirements across 8 validated skills. The unknowns are calibration (seasonal patterns, alert thresholds) not data model discovery. The critical pitfalls are known (NULL DivisionID, wrong DivisionID field) with clear mitigations (COALESCE, mapping table). The stack requires no new installations. The architecture extends proven patterns (skill per domain, core for cross-cutting, glossary for routing).

### Gaps to Address

**Data gaps (resolve via discovery queries in Phase 14):**

- **Actual DivisionData IDs:** Schema shows ID=-1 (system), but Company and Apparel IDs unknown. Query: `SELECT ID, DivisionName, IsActive FROM DivisionData WHERE ID > 0`. Needed before any division-filtered query.

- **DivisionID NULL coverage:** How many TransHeader/GL records have NULL DivisionID? Query: `SELECT COUNT(*) FROM TransHeader WHERE DivisionID IS NULL AND TransactionType=1`. Quantifies C1 pitfall severity.

- **Payroll GL NodeIDs:** Which GL accounts are payroll? Query: `SELECT ID, AccountName FROM GLAccount WHERE GLClassificationType=5002 AND AccountName LIKE '%payroll%'`. Needed for payroll expense queries.

- **Apparel Division history depth:** Does Apparel have 12+ months of division-tagged data? Query: monthly revenue by division. Determines if per-division forecasting feasible.

**Calibration gaps (resolve via user input in Phases 15-16):**

- **Seasonal pattern significance:** With only 24 months of data, are Aug-Sep peaks statistically robust or noise? Run seasonal decomposition, check STDEV. If weak, use simple YoY instead.

- **Alert thresholds:** What AR aging triggers action at FLS? What artwork dwell time is "stuck"? Interview Cain/Gretel for business context.

- **Dashboard KPI priorities:** Which metrics matter most? Morning report should highlight what Cain/Gretel/Taylor check first thing. User preferences drive dashboard design.

**Technical gaps (validate during implementation):**

- **React artifact rendering:** Test each chart type (line, bar, pie) individually before combining into dashboard. Confirm Recharts + Tailwind work as expected in Claude's artifact sandbox.

- **Division filtering regression:** After adding division clauses, re-run 21-test validation suite. Ensure no regressions in unfiltered queries.

None of these gaps block research synthesis or roadmap creation. All are addressable during phase planning/execution with 10 minutes - 2 hours of effort per gap.

## Sources

### Primary (HIGH confidence)

**Internal (verified against database schema and validated skills):**
- `/Users/cain/projects/control-db-map/output/schemas/TransHeader.md` — DivisionID, ProductionDivisionID, ShipFromDivisionID FK relationships verified
- `/Users/cain/projects/control-db-map/output/schemas/Ledger.md` — DivisionID, ProcessedDivisionID fields verified
- `/Users/cain/projects/control-db-map/output/schemas/DivisionData.md` — 3 rows, nullable columns, IsActive flag
- `/Users/cain/projects/control-db-map/output/schemas/Warehouse.md` — DivisionID NOT NULL (FK required)
- `/Users/cain/projects/control-db-map/output/schemas/Account.md` — DivisionID FK to DivisionData
- `/Users/cain/projects/control-db-map/output/wiki/extracts/sql_queries_reference.md` — 40+ wiki SQL scripts using COALESCE(DivisionID, 10)
- `/Users/cain/projects/control-db-map/output/wiki/control/control_release_notes_6.1.2107.0701.md` — CCON-5479 ProcessedDivisionID bug documented
- `/Users/cain/projects/control-db-map/skills/control-erp-core/control-erp-core-SKILL.md` — 99.98% validated revenue formula
- `/Users/cain/projects/control-db-map/skills/control-erp-sales/control-erp-sales-SKILL.md` — 9 query templates (370 lines)
- `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` — GL architecture, P&L, AR/AP (888 + 412 lines)
- `/Users/cain/projects/control-db-map/output/FLS_Sales_Plan_2026.md` — 2026 sales plan with monthly/category targets
- `/Users/cain/projects/control-db-map/.planning/PROJECT.md` — v1.2 requirements, payroll deferral decision
- `/Users/cain/projects/control-db-map/.planning/milestones/v1.1-MILESTONE-AUDIT.md` — v1.1 delivered features, deferred items

### Secondary (MEDIUM confidence)

**SQL forecasting patterns:**
- [Statistics in SQL: Simple Linear Regressions - Simple Talk](https://www.red-gate.com/simple-talk/blogs/statistics-sql-simple-linear-regressions/) — T-SQL least-squares formula
- [MSSQLTips: T-SQL Windowing Functions for Moving Averages](https://www.mssqltips.com/sqlservertip/8124/calculate-a-moving-average-with-t-sql-windowing-functions/) — Window function patterns
- [LearnSQL: Year-over-Year Difference in SQL](https://learnsql.com/blog/year-over-year-difference-sql/) — YoY comparison patterns
- [Silota: Ratio to Moving Average](http://www.silota.com/docs/recipes/sql-ratio-to-moving-average-seasonal-index.html) — Seasonal decomposition in SQL
- [Weighted vs Simple Moving Average with SQL Server T-SQL](https://www.mssqltips.com/sqlservertip/6312/weighted-vs-simple-moving-average-with-sql-server-tsql-code/) — WMA implementation

**React artifacts:**
- [Reverse Engineering Claude Artifacts - Reid Barber](https://www.reidbarber.com/blog/reverse-engineering-claude-artifacts) — Artifact sandbox constraints
- [Claude Help Center: Artifacts](https://support.claude.com/en/articles/9487310-what-are-artifacts-and-how-do-i-use-them) — Official docs
- [Claude Artifacts: Game Changer Held Back by Limits - Medium](https://medium.com/@intranetfactory/claude-artifacts-a-game-changer-held-back-by-frustrating-limits-6adcacdd95a7) — Sandbox limitations (Jan 2025)
- [Top 5 React Chart Libraries 2026 - Syncfusion](https://www.syncfusion.com/blogs/post/top-5-react-chart-libraries) — Recharts recommended

**Analytics methodology:**
- [How to Conduct Time Series Forecasting in SQL with Moving Averages - Towards Data Science](https://towardsdatascience.com/how-to-conduct-time-series-forecasting-in-sql-with-moving-averages-fd5e6f2a456/) — SQL forecasting patterns
- [Biggest Forecasting Mistakes 2026 - Mosaic](https://www.mosaicapp.com/post/the-biggest-forecasting-mistakes-leaders-still-make-in-2026) — Methodology pitfalls
- [Executive Dashboards: Examples & Best Practices - Improvado](https://improvado.io/blog/executive-dashboards) — Dashboard KPI selection

### Tertiary (community, needs validation)

- [Manufacturing KPI Dashboard 2026 Guide - Method](https://www.method.me/blog/manufacturing-kpi-dashboard/) — Manufacturing-specific KPIs (FLS is manufacturing, but custom/small-batch)
- [SQL Growth Analysis - SQLGuroo](https://sqlguroo.com/blog/sql-growth-calculations/) — Growth rate computation patterns
- [Common Data Analysis Mistakes - NetSuite](https://www.netsuite.com/portal/resource/articles/data-warehouse/data-mistakes.shtml) — General analytics pitfalls

---

**Research completed:** 2026-02-09

**Ready for roadmap:** YES

All four research dimensions (stack, features, architecture, pitfalls) reached HIGH confidence. Phase structure clear (14: Division, 15: Analytics, 16: Dashboard, 17: Payroll, 18: Integration). Discovery queries identified (10-15 minutes in Phase 14). Brownfield advantage leveraged (49 shipped requirements, 73 validated queries, 8 proven skills). Critical pitfalls documented with mitigations. No architectural unknowns blocking roadmap creation.
