# Project Research Summary

**Project:** Project Cornerstone v1.1 — Read-Layer Domain Expansion
**Domain:** Natural Language Query Interface for Control ERP (5 new domain skills)
**Researched:** 2026-02-09
**Confidence:** HIGH

## Executive Summary

This research synthesizes findings from four parallel research tracks investigating how to expand the validated v1.0 Control ERP natural language interface (4 skills, 35/35 requirements passed) with 5 new domain areas: Financial depth, Customer intelligence, Inventory management, Production workflow, and Reports catalog. The research reveals strong consensus on build approach with one critical architectural disagreement that requires resolution.

**Recommended approach:** Build 3 new standalone skills (customers, inventory, production), extend 1 existing skill (financial with analytical depth), and integrate reports catalog into the existing glossary skill. All new skills follow the validated v1.0 pattern: single SKILL.md files with YAML frontmatter, explicit core skill dependencies, and validation against Control's Crystal Reports before shipping. The inventory and production domains introduce genuinely new table sets (1.2M+ rows of inventory/parts data, 500K+ rows of artwork/station/timecard data) requiring live database discovery queries to validate FLS-specific configurations.

**Key risks and mitigation:** The highest-severity risks are data interpretation pitfalls inherited from v1.0's architecture complexity: (1) GL sign convention inversions that make revenue appear negative, (2) inventory quantity field confusion (OnHand vs Available vs Reserved), (3) TimeCard double-counting due to parent/detail ClassTypeID patterns, and (4) DueDate semantic overloading for AR aging. All four risks have clear prevention strategies validated in existing skills. The second-tier risk is architectural: Stack research recommends creating a separate `control-erp-financial-deep` skill to avoid destabilizing the already-large (857 lines) financial skill, while Architecture research recommends extending the existing financial skill since 80% of "deep" content builds on existing queries. This disagreement must be resolved before Phase 1 begins.

## Key Findings

### Recommended Stack (from STACK.md)

Stack research focuses on additions to the validated v1.0 architecture (SQL Server via MCP, YAML skill files, wiki knowledge extracts, Crystal Reports analysis). No new packages or tooling required — all expansion happens through markdown skill files.

**Critical architectural decision:**

Stack research recommends 5 new skill files + 5 reference docs (10 files total):
1. `control-erp-financial-deep` (NEW skill, 500+ lines) — AR/AP analytics, P&L trending, cash flow, financial KPIs
2. `control-erp-customers` (NEW skill + references/segmentation_reference.md)
3. `control-erp-inventory` (NEW skill + references/part_types_reference.md)
4. `control-erp-production` (NEW skill + 2 reference docs)
5. `control-erp-reports` (NEW skill, compact catalog)

**Core technologies (all existing, validated in v1.0):**
- Database: SQL Server StoreData via MCP MSSQL tools — proven, no changes
- Skills: YAML SKILL.md + optional references/ — v1.0 shipped without references/; v1.1 introduces them for inventory/production complexity
- Validation: Crystal Reports cross-check — same pattern as v1.0 (AR report, Sales by Product, WIP reports)
- Wiki knowledge: 1,769 pages, 6 relevant extracts — production_inventory_knowledge.md, crm_payroll_system_knowledge.md, orders_accounting_knowledge.md provide complete ClassTypeID mappings and query patterns

**Build order recommendation (from Stack):**
1. Customers (lowest risk, Account table well-understood)
2. Reports (simple catalog, no new SQL patterns)
3. Financial-deep (extends validated foundation)
4. Inventory (new domain, needs live data discovery)
5. Production (most complex, needs station/artwork discovery)

### Expected Features (from FEATURES.md)

Features research identifies 67 total features across 5 domains (36 table stakes, 31 differentiators, 26 anti-features). The anti-features list is critical — it defines what NOT to build to avoid scope creep into write operations, tax compliance, payroll complexity, and Crystal Reports execution.

**Must have (table stakes):**

Financial domain (9 features, 4 already built in v1.0):
- AR/AP aging by customer/vendor (extend existing aggregates to per-entity drill-down)
- P&L for any date range (already built, ensure shortcuts work)
- Cash position (already built, add undeposited funds breakdown)
- Payment history per order (already built)
- GL entries per order (already built)

Customer domain (8 features):
- Customer lookup by name (fuzzy LIKE match on CompanyName)
- Customer profile (360-view: orders, revenue, payments, contacts, AR)
- Customer order history (TransHeader Type 1 per AccountID)
- Top customers (already exists in sales skill Template 6 — reuse)
- Customer open balances (BalanceDue > 0 aggregation)

Inventory domain (7 features):
- Current stock levels (QuantityAvailable, not QuantityOnHand — critical distinction)
- Inventory listing (Part + Inventory join with TrackInventory = 1)
- Inventory value (SUM QuantityBilled * AverageCost, cross-check with GL NodeID 10414)
- Low stock / reorder alerts (QuantityAvailable <= ReorderPoint)
- Parts listing (Part table with PartType filter)
- Inventory by warehouse (FLS has multiple warehouses per division)

Production domain (7 features):
- Artwork status (ArtworkGroup with StatusID < 7 = in progress)
- WIP orders by station (TransHeader StatusID 1/2 with StationID)
- Station workload (COUNT/SUM per station)
- Employee time today/this week (TimeCard with ClassTypeID 20051, avoiding double-count with 20050)
- Orders due today/this week (TransHeader.DueDate for production deadline)
- Time clock status (who's clocked in — TimeClockStatus table)

Reports domain (5 features):
- Report catalog lookup (36 Crystal Reports in 6 categories)
- Report recommendation (intent-to-report mapping)
- Report parameter guidance (date range, division, entity filters)
- Report description (purpose + primary tables + key fields)

**Should have (competitive differentiators):**

Financial domain:
- Cash flow summary (payments received vs bills paid — no built-in Control report)
- Historical AR as of date (GL WHERE EntryDateTime <= @Date — wiki provides SQL)
- GL integrity checks (7+ integrity check queries from wiki, currently manual)
- Month-over-month expense trend (P&L with YoY comparison)

Customer domain:
- Customer lifetime value (SUM SubTotalPrice all-time per AccountID)
- Dormant customer detection (MAX(SaleDate) < DATEADD(MONTH, -6, GETDATE()))
- Customer concentration analysis (top 10% revenue contribution)
- RFM analysis (Recency, Frequency, Monetary value)

Inventory domain:
- Inventory turnover (annual usage / current stock)
- Slow-moving inventory (MAX(InventoryLog date) > 90 days)
- Inventory valuation vs GL reconciliation (critical for month-end close)
- Parts consumption trend (TransPart grouped by month/part)

Production domain:
- Artwork turnaround time (ArtworkGroupStatusHistory time-to-approval)
- Station utilization (SUM hours per station / available hours)
- Estimated vs actual cost comparison (TransPart.EstimatedValue vs PartUsageCard.ActualValue)
- Artwork bottleneck detection (stuck in Pending Approval > 3 days)

Reports domain:
- Report output replication (execute SQL equivalent instead of launching Crystal)
- Cross-report intelligence (metadata search across catalog)
- Report gap identification (acknowledge when no report exists, offer ad-hoc query)

**Defer (v2+ or out of scope):**

All write operations (bank reconciliation, inventory adjustments, PO creation, station changes, time clock in/out, artwork status changes) — read-only interface only.

Tax filing preparation — too high-stakes, Avalara integration required.

Payroll queries — sensitive, regulated, complex data model. Separate domain entirely for future milestone.

Budget vs actual — Control has no budget table.

Division-level reporting — deferred per charter (FLS has Banner and Apparel divisions but v1.1 focuses on company-wide queries).

HR/Payroll skill — Phase 5 per charter, lower business priority.

### Architecture Approach (from ARCHITECTURE.md)

**CRITICAL CONFLICT WITH STACK:** Architecture research contradicts Stack research on financial depth.

**Architecture recommendation:** EXTEND existing `control-erp-financial` skill (currently 857 lines) with new sections for cash flow analysis, margin analysis, AR velocity, cash requirements forecast, and bank reconciliation status. Rationale: The financial skill already contains complete GL/Ledger architecture, AR/AP snapshots, P&L templates, balance sheet queries, payment posting patterns, and closeout documentation. The "deep" queries build on these foundations (e.g., cash flow = period-over-period balance sheet changes, AR velocity = DSO calculation using existing AR queries). Creating a separate `control-erp-financial-deep` skill would duplicate 80% of context.

**Stack recommendation:** CREATE new `control-erp-financial-deep` skill (separate file). Rationale: The financial skill is already 2.5x the size of other skills at 857 lines. Adding 500+ lines of analytical queries risks destabilizing a validated skill and pushes the file toward 1,200-1,400 lines (upper limit for context window budgeting). A separate deep skill depends on the foundation and adds analytical capabilities without touching validated SQL.

**Resolution required:** This is a genuine architectural tradeoff. Extending avoids duplication and keeps related queries together. Splitting prevents destabilization and maintains skill size consistency. **Recommendation for synthesis:** Follow Architecture's recommendation to EXTEND (the v1.0 pattern was single-file skills, no skill uses references/ yet), but with a safeguard: if the extended financial skill exceeds 1,200 lines, extract low-frequency sections (closeout detail, off-balance-sheet edge cases) into `references/advanced-gl.md` as v1.1's first use of the reference pattern.

**Consensus on other 4 domains:**

Customer Intelligence → CREATE `control-erp-customers` — genuinely new domain, Account-centric queries not covered by sales/financial skills. Both Stack and Architecture agree.

Inventory → CREATE `control-erp-inventory` — 13 tables, 1.2M rows, no overlap with existing skills. Both agree.

Production → CREATE `control-erp-production` — artwork + stations + time tracking + shipping. Largest new domain. Both agree.

Reports Catalog → **DISAGREEMENT:** Stack says create `control-erp-reports` (standalone skill). Architecture says extend `control-erp-glossary` with a Crystal Reports Catalog section (too thin for standalone skill at ~100-150 lines). **Recommendation:** Follow Architecture — the glossary is already a routing skill, and a 36-report catalog naturally fits as a new section. If report catalog grows beyond 36 entries or parameter documentation becomes extensive, split later.

**Major components (skill topology after expansion):**

```
control-erp-core (foundation, unchanged)
  ├── control-erp-sales (unchanged)
  ├── control-erp-financial (EXTENDED: +5 analytical sections)
  ├── control-erp-customers (NEW: Account lookup, segmentation, CLV, churn)
  ├── control-erp-inventory (NEW: stock, parts, warehouse, consumption)
  ├── control-erp-production (NEW: artwork, stations, time, shipping)
  └── control-erp-glossary (EXTENDED: +Crystal Reports catalog section)
```

**Dependency classification:**
- Core dependency (HARD): sales, financial, customers — use TransactionType, StatusID, pricing rules on most queries
- Core dependency (SOFT): inventory, production — use some core concepts but many queries are self-contained
- Cross-references: customers ↔ sales (for revenue context), inventory ↔ financial (for GL cost accounts), production ↔ inventory (for part usage)

**Context window budget:** All configurations fit within Claude's context window. Extended financial skill (~1,000 lines) + core (347) + glossary (~600) = ~12,000 tokens. Cross-domain queries requiring 2 skills simultaneously (e.g., customer revenue + AR status) fit but are tight at ~14,000 tokens.

**Validation approach per domain:**
- Financial extension: DSO calculation vs manual AR/revenue math, cash flow vs GL balance changes
- Customers: Top 10 customers cross-checked with sales skill output (revenue totals must match)
- Inventory: Inventory valuation vs GL NodeID 10414 balance (should approximate)
- Production: WIP count vs TransHeader WHERE StatusID=1 count, artwork status vs Control UI spot-checks
- Glossary (reports): Manual routing test with 20 NL queries, 90% correct routing acceptance

### Critical Pitfalls (from PITFALLS.md)

Pitfalls research identifies 11 critical, 7 moderate, 5 minor, and 3 integration-specific pitfalls. The critical pitfalls are calibrated against v1.0's TransDetailParam IsActive bug ($1.3M revenue discrepancy) as the severity reference point.

**Top 5 critical pitfalls:**

1. **GL Sign Convention Inversion (Financial)** — Revenue queries return negative numbers because GL stores revenue as negative (credits). Prevention: Hardcode `SUM(-Amount)` for GLClassificationType IN (4000, 4001). Validation: GL revenue must match known $3,053,541.85 benchmark. Severity: HIGH — wrong sign on $3M revenue is immediately visible but subtle COGS inversion would be harder to detect.

2. **Inventory Quantity Field Confusion (Inventory)** — Skill reports QuantityOnHand instead of QuantityAvailable, overstating what can be allocated to new orders. The chain is: QuantityOnHand - QuantityReserved = QuantityAvailable. Prevention: Document NL mapping ("available" → QuantityAvailable, "on hand" → QuantityOnHand). Validation: Compare with Inventory Listing.rpt. Severity: HIGH — committing reserved inventory delays in-progress production.

3. **TimeCard ClassTypeID Double-Counting (Production)** — Time queries sum both parent (20050) and station detail (20051) records, roughly doubling hours. Prevention: "For total hours, use ClassTypeID = 20050. For time-by-station, use 20051. Never mix." Severity: CRITICAL — 2x labor hours means 2x labor cost, production domain's equivalent of the $1.3M TransDetailParam bug.

4. **DueDate Semantic Overloading (Financial AR/AP)** — AR aging uses TransHeader.DueDate (production/shipping deadline) instead of SaleDate (invoice date), producing completely wrong aging buckets. Prevention: AR aging MUST use SaleDate. AP aging MUST use DueDate (correct for bills). Severity: CRITICAL — wrong AR aging produces incorrect collections prioritization, exact analog of v1.0's SaleDate/OrderCreatedDate bug.

5. **GL View vs Ledger Table Confusion (Financial)** — Queries use Ledger table directly instead of GL view, including ~307K off-balance-sheet entries that distort financial totals. GL is a VIEW: `SELECT * FROM Ledger WHERE OffBalanceSheet = 0`. Prevention: "Use GL view for financial queries. Use Ledger only for production costs including non-accrual materials." Severity: HIGH — ~11% data distortion, off-balance-sheet entries inflate COGS.

**Integration pitfalls:**

- **Contradicting Core Skill Business Rules** — New skills hardcode TransactionType or price field rules that contradict control-erp-core. Prevention: All new skills MUST declare dependency on core and defer for TransactionType, StatusID, date fields, standard filters. Severity: CRITICAL — this is how v1.0 started with wrong mappings.

- **Natural Language Routing Ambiguity** — "Top customers" could route to sales (revenue), financial (AR), or customers (CLV). Prevention: Define clear domain boundaries, test with ambiguous prompts. Severity: MEDIUM.

- **Duplicated Query Patterns** — Financial and Customer skills both implement "revenue by customer" with different logic. Prevention: Designate one skill as authoritative per query type, cross-reference core's validated patterns. Severity: MEDIUM.

**Phase-specific warnings table:**

| Phase | Likely Pitfall | Mitigation | Severity |
|-------|---------------|------------|----------|
| Financial extension | C1 (GL sign inversion), C3 (DueDate AR aging) | Hardcode SUM(-Amount), use SaleDate not DueDate | CRITICAL |
| Customer build | C9 (CompanyName not AccountName), C10 (IsClient flags) | Use CompanyName, default IsClient=1 IsActive=1 | HIGH |
| Inventory build | C4 (Quantity field), C5 (Inventory vs Part table) | Use QuantityAvailable, query Inventory not Part | HIGH |
| Production build | C6 (TimeCard ClassTypeID), C8 (Journal multi-purpose) | Filter ClassTypeID 20050 vs 20051, JournalActivityType=45 | CRITICAL |

## Implications for Roadmap

Based on research consensus and conflict resolution, recommended phase structure for v1.1:

### Phase 1: Financial Depth Extension
**Rationale:** Lowest risk — extends already-validated skill with analytical queries building on proven foundations (GL, AR/AP, P&L templates). No new skill file to create. Validation straightforward (compare against FLS Income Statement, Balance Sheet, AR/AP reports). This resolves the Stack vs Architecture conflict by choosing to EXTEND with a safeguard (extract to references/ if >1,200 lines).

**Delivers:**
- Cash flow analysis (period-over-period balance sheet changes)
- AR velocity and DSO calculation
- P&L trending (month-over-month, YoY comparison)
- Margin analysis by period
- GL integrity checks (import 7+ wiki SQL patterns)
- Bank reconciliation status

**Features from FEATURES.md:** 5 new differentiators (cash flow summary, historical AR, GL integrity, margin trend, AR velocity)

**Avoids pitfalls:** C1 (GL sign convention — already documented in existing skill), C2 (GL view vs Ledger — already established), C3 (DueDate AR aging — existing skill uses SaleDate correctly, propagate pattern)

**Stack elements:** Wiki extracts (orders_accounting_knowledge.md, sql_queries_reference.md), existing financial skill as foundation

**Validation:** Extended financial skill must match FLS Income Statement net income within 1%, AR aging vs A_R Detail.rpt, DSO calculation vs manual math (AR ~$81K, 2025 revenue $3.05M → DSO ~9.7 days)

**Research flag:** Standard patterns (P&L, AR, AP already validated). No phase-level research needed, but specific queries (cash flow reconstruction, DSO formula) need validation against Control reports.

### Phase 2: Customer Intelligence
**Rationale:** No dependency on other new skills. High user value (customer lookup is daily use). Well-defined validation (cross-reference sales skill top customers). Account table is fully documented, CRM patterns documented in wiki. Builds confidence before tackling genuinely new domains.

**Delivers:**
- Customer lookup by name/number/contact
- Customer profile (360-view with contacts, addresses, orders, revenue, AR)
- Customer lifetime value (CLV)
- Customer segmentation (by stage, industry, origin, revenue tier)
- Churn detection (dormant customer identification)
- RFM analysis (Recency, Frequency, Monetary value)

**Features from FEATURES.md:** 8 table stakes + 8 differentiators (CLV, churn, concentration analysis, estimate conversion, segmentation)

**Avoids pitfalls:** C9 (Account.CompanyName not AccountName — documented in MEMORY.md), C10 (IsClient/IsProspect/IsActive triple-flag filtering), M3 (TransactionType polymorphism — apply core filters), I1 (contradicting core — explicit dependency declaration)

**Stack elements:** Account schema (54,719 rows), AccountContact (64,851), wiki extract (crm_payroll_system_knowledge.md for Company Stages, marketing lists)

**Depends on:** control-erp-core (TransactionType = 1 for order history, SubTotalPrice for CLV, Account.CompanyName gotcha)

**Validation:** Top 10 customers by revenue cross-checked with sales skill Template 6 (revenue totals must match within 1%), spot-check 3-5 customer profiles against Control UI

**Research flag:** Standard patterns (customer lookup, order history are straightforward joins). Company Stage query (Account.StageID → Element WHERE ClassTypeID IN 5511/5512) needs live validation against FLS data.

### Phase 3: Inventory Management
**Rationale:** New domain (13 tables, 1.2M rows) with no overlap with existing skills. Cross-references financial skill for GL cost accounts (already extended in Phase 1). Wiki documentation is comprehensive (production_inventory_knowledge.md Section 4 provides complete inventory formulas). Benefits from financial extension complete (can validate inventory value against GL NodeID 10414).

**Delivers:**
- Current stock levels (QuantityAvailable, QuantityOnHand, QuantityReserved)
- Inventory listing by part, warehouse, division
- Inventory valuation (SUM QuantityBilled * AverageCost)
- Low stock / reorder alerts
- Part lookup and cost information
- Inventory movement history (InventoryLog)
- Part consumption by order
- Open purchase orders (Type 7 TransHeader)

**Features from FEATURES.md:** 7 table stakes + 6 differentiators (turnover, slow-moving detection, GL reconciliation, consumption trend)

**Avoids pitfalls:** C4 (Quantity field confusion — QuantityAvailable vs QuantityOnHand), C5 (Inventory vs Part table quantities — always query Inventory with ClassTypeID 12200), M4 (PartType classification — filter to Material parts for physical inventory)

**Stack elements:** Inventory schema (27,268 rows), Part (7,684), InventoryLog (906,714), wiki knowledge (inventory formulas, tracking modes), TransPart (2.26M rows — performance-critical)

**Depends on:** control-erp-core (partial — GoodsItemClassTypeID 49=Product, 30=Part), control-erp-financial (cross-reference for GL cost account NodeID 10414)

**Validation:** Inventory valuation vs GL NodeID 10414 balance (should approximate), inventory listing vs Inventory Listing.rpt, spot-check 5 specific parts (query stock vs Control UI)

**Research flag:** FLS-specific warehouse configuration needs live database discovery. Query: `SELECT ID, WarehouseName, WarehouseType FROM Warehouse WHERE IsActive = 1`. How many warehouses? Which types (Standard, Kanban, Stockroom, Group)? Which parts use which warehouses? This configuration determines query patterns.

### Phase 4: Production Workflow
**Rationale:** Largest new domain (artwork + stations + time tracking + shipping). Benefits from inventory skill existing (part usage card context). Most complex due to FLS-specific station configuration and artwork status usage patterns. Needs live station/artwork discovery queries.

**Delivers:**
- Artwork approval pipeline (ArtworkGroup status tracking)
- Artwork bottleneck detection (stuck in Pending Approval)
- Station workload (orders per station, WIP by station)
- Production schedule (orders by due date, station capacity)
- Employee time tracking (hours on jobs/stations)
- Time clock status (who's clocked in)
- Shipment tracking (Shipments table, FedEx/UPS logs)
- Job costing (estimated vs actual cost per order)

**Features from FEATURES.md:** 7 table stakes + 7 differentiators (artwork turnaround time, station utilization, cost variance, bottleneck detection)

**Avoids pitfalls:** C6 (TimeCard ClassTypeID double-counting — CRITICAL), C7 (Geography columns crash queries — never SELECT *), C8 (Journal multi-purpose table — filter JournalActivityType=45 for stations), C11 (Artwork status integer magic numbers — use artwork-specific StatusIDs), M7 (PartUsageCard vs TransPart — actual vs estimated)

**Stack elements:** ArtworkGroup schema (84,770 rows), Station (98), TimeCard (159,448), Journal (5.18M rows — must filter), Shipments (70,584), wiki knowledge (artwork workflow state machine, station hierarchy, TimeCard ClassTypeID patterns)

**Depends on:** control-erp-core (StatusID 1/2 for WIP, TransactionType = 1), control-erp-inventory (soft — part usage cards reference Part table)

**Validation:** WIP count vs TransHeader WHERE StatusID=1 count (exact match), WIP value vs WIP Summary report (within 1%), artwork status spot-checks (3-5 active orders vs Control UI), Shipped By Date report cross-check

**Research flag:** HIGH — FLS-specific station configuration and artwork status usage needs live discovery. Queries needed:
- `SELECT ID, StationName, ParentID, DepartmentID, ShowOnTimeClock FROM Station WHERE IsActive = 1` (98 stations — what's the hierarchy?)
- `SELECT StatusID, COUNT(*) FROM ArtworkGroup WHERE IsActive = 1 GROUP BY StatusID` (which statuses are actively used?)
- `SELECT TOP 100 * FROM TimeCard WHERE ClassTypeID = 20050 ORDER BY ID DESC` (validate time clock pattern)

### Phase 5: Glossary Integration (Reports Catalog + Routing Update)
**Rationale:** MUST come last because it routes to ALL other skills. Requires all domain skills to exist before mapping routes. Resolves Stack vs Architecture conflict on reports by extending glossary (Architecture's recommendation) rather than creating standalone skill.

**Delivers:**
- Crystal Reports catalog (36 reports in 6 categories: Financial, WIP, Sales, Estimates, Inventory, Other)
- Intent-to-report mapping ("AR aging" → A_R Detail.rpt)
- Report vs custom query decision logic
- Report parameter guidance
- Updated NL routing for all new domain skills (inventory terms, production terms, customer terms, financial depth terms)

**Features from FEATURES.md:** 5 table stakes (catalog lookup, recommendation, parameters, description, SQL equivalents where possible) + 4 differentiators (output replication, cross-report intelligence, gap identification)

**Avoids pitfalls:** M6 (Report metadata limitations — acknowledge inferred metadata, focus on routing guidance), I2 (NL routing ambiguity — define clear domain boundaries)

**Stack elements:** report_summary.md (36 reports cataloged), report_join_patterns.md (SQL equivalents), Report table (211 rows), ReportTemplate (168 rows)

**Extends:** control-erp-glossary (add Crystal Reports Catalog section + routing entries for inventory/production/customer/financial-depth)

**Validation:** Manual routing test with 20 NL queries covering all domains, 18/20 correct routing = 90% acceptance

**Research flag:** Standard patterns (static catalog, straightforward routing table). No research needed.

### Phase Ordering Rationale

**Why this order:**

1. **Financial first** — Extends proven foundation, lowest risk, establishes analytical query patterns (cash flow, margin trend) that inform later skills
2. **Customer second** — No dependencies on new skills, high user value, builds confidence, validates Account-centric query patterns before inventory/production
3. **Inventory third** — Cross-references financial (GL cost accounts), new domain but wiki-documented formulas reduce risk
4. **Production fourth** — Largest domain, highest complexity, benefits from customer/inventory patterns established, needs most live discovery
5. **Glossary last** — Routes to all skills, must exist after skills to route to

**Dependency flow:**
```
Financial extension (no dependencies)
    ↓ (validates analytical patterns)
Customer build (no dependencies, uses core)
    ↓ (validates Account-centric patterns)
Inventory build (cross-refs financial GL accounts)
    ↓ (establishes part/warehouse patterns)
Production build (soft ref to inventory for part usage)
    ↓ (all domain skills complete)
Glossary integration (routes to ALL above)
```

**Parallelization opportunity:** Phases 1 (Financial) and 2 (Customer) have zero dependencies on each other. Could be built simultaneously by separate developers. Phases 3 (Inventory) and 4 (Production) have only a soft dependency (production references part usage cards) and could overlap if production's part-related queries are built after inventory is complete.

**How this avoids pitfalls:**

- Build order ensures core skill dependency patterns (TransactionType, StatusID, date filtering) are validated in simpler domains (customer) before complex domains (production)
- Financial first establishes GL query patterns, so inventory can validate against GL cost accounts from day 1
- Customer before production establishes Account-centric lookup patterns used in production (who's the customer for this WIP order?)
- Glossary last ensures all routing targets exist before mapping NL intents

### Research Flags

**Phases needing deeper research during planning:**

- **Phase 3 (Inventory):** FLS-specific warehouse configuration discovery required. How many warehouses? Which warehouse types? Which parts track inventory in which warehouses? Discovery query: `SELECT * FROM Warehouse WHERE IsActive = 1`. Impact: Determines whether inventory queries filter by warehouse or aggregate across all.

- **Phase 4 (Production):** Station hierarchy and artwork status usage discovery required. 98 stations exist — what's the department/workstation structure? Which statuses in `_ArtworkStatus` are actively used at FLS? Discovery queries: Station table hierarchy analysis, ArtworkGroup StatusID distribution. Impact: Determines station aggregation logic and artwork pipeline query patterns.

**Phases with standard patterns (skip research-phase):**

- **Phase 1 (Financial):** P&L, AR, AP, cash flow all use validated GL/Ledger patterns from v1.0. Wiki provides exact SQL for GL integrity checks and historical AR. Straightforward extension.

- **Phase 2 (Customer):** Account table schema fully documented, revenue queries validated in sales skill, CRM patterns documented in wiki. Standard JOIN patterns.

- **Phase 5 (Glossary):** Static catalog from report_summary.md, straightforward routing table extension. No new query patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All recommendations based on validated v1.0 architecture. No new packages. Existing MCP/SQL Server/YAML pattern proven. Wiki extracts comprehensive. |
| Features | HIGH | Table stakes identified from domain expertise + Control documentation. Differentiators validated against advanced_analytics.md templates. Anti-features clearly scoped (no write ops, no payroll). |
| Architecture | HIGH | Based on direct analysis of existing v1.0 skill files (347-857 lines each), 187 table schemas, wiki extracts, Crystal Reports catalog. Context window budgeting verified. Dependency analysis clear. |
| Pitfalls | HIGH | All critical pitfalls calibrated against v1.0 validation results ($1.3M TransDetailParam bug as reference). GL sign convention, TimeCard double-counting, DueDate semantic overload all have validated prevention strategies. |

**Overall confidence:** HIGH

Architecture and Stack research are based on existing, validated v1.0 artifacts. Features research draws from comprehensive wiki extracts (1,769 pages, 6 domain-specific extracts). Pitfalls research identifies exact analogs of v1.0 bugs in new domains (TimeCard ClassTypeID = TransDetailParam IsActive pattern). The only MEDIUM confidence area resolved: FLS-specific configurations (warehouses, stations) require live database discovery in Phases 3-4, but discovery query patterns are straightforward.

### Gaps to Address

**Architectural decision resolved:** Stack vs Architecture disagreement on financial-deep (separate skill vs extend existing). **Resolution:** EXTEND control-erp-financial per Architecture recommendation, with safeguard: if extended skill exceeds 1,200 lines, extract low-frequency sections to references/advanced-gl.md (v1.1's first use of reference pattern). This maintains v1.0's single-file pattern while establishing escape valve for future growth.

**Reports catalog location resolved:** Stack vs Architecture disagreement on standalone skill vs glossary section. **Resolution:** Extend control-erp-glossary per Architecture recommendation. 36-report catalog (~100-150 lines) is too thin for standalone skill. Glossary is already a routing skill, reports catalog is fundamentally routing ("which report answers this question?"). If catalog grows beyond 100 reports or parameter docs become extensive, split later.

**Remaining gaps (to be addressed during implementation):**

- **FLS warehouse configuration:** Number of warehouses, warehouse types in use, part-to-warehouse assignments. Discovery: `SELECT * FROM Warehouse WHERE IsActive = 1` + Inventory table analysis. Impact: Phase 3 inventory queries. Mitigation: Design queries to work with N warehouses (aggregate or filter), validate against live data.

- **FLS station hierarchy:** 98 stations — department/workstation structure specific to FLS. Discovery: `SELECT ID, StationName, ParentID, DepartmentID, ShowOnTimeClock FROM Station WHERE IsActive = 1`. Impact: Phase 4 production workload aggregation. Mitigation: Design generic department rollup pattern, populate FLS-specific hierarchy in references/station_reference.md after discovery.

- **Artwork status usage:** Which StatusIDs in `_ArtworkStatus` lookup table are actively used? Discovery: `SELECT StatusID, COUNT(*) FROM ArtworkGroup WHERE IsActive = 1 GROUP BY StatusID`. Impact: Phase 4 artwork pipeline queries. Mitigation: Query StatusID distribution, map to status names via `_ArtworkStatus` lookup, document in references/artwork_workflow_reference.md.

- **TimeCard time clock pattern:** How is "currently clocked in" represented? Discovery: `SELECT TOP 100 * FROM TimeCard WHERE ClassTypeID = 20050 ORDER BY ID DESC` + `SELECT * FROM TimeClockStatus WHERE IsActive = 1`. Impact: Phase 4 time tracking queries. Mitigation: Wiki documents ClassTypeID 20050 = parent, 20051 = detail. Validate with live data, cross-check TimeClockStatus table.

- **Customer segmentation thresholds:** What constitutes "top customer," "at-risk customer," "dormant customer" for FLS? Discovery: Revenue distribution analysis, last-order-date distribution. Impact: Phase 2 customer intelligence segmentation. Mitigation: Use industry-standard thresholds (top 10% revenue, 180 days dormant), validate against FLS data, adjust if needed.

**How to handle during planning/execution:**

All gaps are discoverable via SQL queries against live database. Each gap has a discovery query pattern documented above. Phases 3-4 require discovery queries BEFORE writing query templates. Design templates generically (work with any warehouse count, any station hierarchy), then populate FLS-specific details in skill documentation and reference files after discovery.

## Sources

### Primary (HIGH confidence)

**Existing validated v1.0 skills:**
- `/Users/cain/projects/control-db-map/skills/control-erp-core/control-erp-core-SKILL.md` — 347 lines, validated business rules, TransactionType mappings, StatusID reference, standard filters, $3,053,541.85 revenue benchmark
- `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` — 857 lines, validated GL/Ledger architecture, AR/AP snapshots, P&L templates, sign conventions, payment posting patterns
- `/Users/cain/projects/control-db-map/skills/control-erp-sales/control-erp-sales-SKILL.md` — 370 lines, validated sales queries, Template 6 (top customers by revenue)
- `/Users/cain/projects/control-db-map/skills/control-erp-glossary/control-erp-glossary-SKILL.md` — 491 lines, validated NL routing table, 20 product categories, 30+ terms

**Database schema documentation:**
- `/Users/cain/projects/control-db-map/output/schemas/` — 187 table schemas with row counts, column definitions, FK relationships from live database
- `/Users/cain/projects/control-db-map/output/domains.md` — 16 functional domains, table groupings
- `/Users/cain/projects/control-db-map/output/relationships.md` — Explicit and inferred FK relationships

**Wiki knowledge extracts:**
- `/Users/cain/projects/control-db-map/output/wiki/extracts/database_integration_knowledge.md` — Complete ClassTypeID mapping (350+ values), SQL Bridge, CHAPI, stored procs
- `/Users/cain/projects/control-db-map/output/wiki/extracts/orders_accounting_knowledge.md` — GL lifecycle, payment posting, pricing mechanics, tax handling
- `/Users/cain/projects/control-db-map/output/wiki/extracts/production_inventory_knowledge.md` — Artwork workflow, stations, inventory formulas, warehouses, shipping, parts
- `/Users/cain/projects/control-db-map/output/wiki/extracts/crm_payroll_system_knowledge.md` — CRM patterns, Company Stages, marketing lists, contact activity, payroll tables
- `/Users/cain/projects/control-db-map/output/wiki/extracts/sql_queries_reference.md` — 40+ utility/diagnostic SQL queries including GL integrity checks, AR validation, inventory valuation

**Crystal Reports analysis:**
- `/Users/cain/projects/control-db-map/output/report_summary.md` — 36 Crystal Reports cataloged with purpose, primary tables, categories
- `/Users/cain/projects/control-db-map/output/report_join_patterns.md` — Inferred SQL patterns from financial, AR, AP, WIP, sales reports

**Validation results:**
- `/Users/cain/projects/control-db-map/validation/control-erp-validation-results.md` — v1.0 validation findings, 35/35 requirements passed, TransDetailParam IsActive bug documented
- `/Users/cain/projects/control-db-map/documentation/control-erp-knowledge-log.md` — Known gotchas, field mappings, query patterns

**Project documentation:**
- `/Users/cain/projects/control-db-map/project-cornerstone-charter.md` — Project architecture, phase structure, v1.1 scope
- `/Users/cain/projects/control-db-map/CLAUDE.md` — Project instructions, schema extraction patterns, business rules reference
- `/Users/cain/projects/control-db-map/MEMORY.md` — Critical gotchas (CompanyName not AccountName, SubTotalPrice not TotalPrice, SaleDate not OrderCreatedDate, TransDetailParam IsActive filter rule, GL NodeID mappings)

### Secondary (MEDIUM confidence)

- `/Users/cain/projects/control-db-map/output/skill/references/advanced_analytics.md` — Analytics query templates (CLV, RFM, churn, turnover, station utilization) — partially validated, templates exist but need FLS-specific validation

### Tertiary (LOW confidence)

- Crystal Report internal SQL and field mappings — report_summary.md explicitly notes metadata is INFERRED from filenames, not extracted (SAP Crystal Reports proprietary OLE format cannot be parsed without SDK). Report purposes and categories are reliable, but internal SQL patterns are best-effort inference.

---

**Research completed:** 2026-02-09
**Ready for roadmap:** Yes

**Next step:** Roadmap creation using this summary as input. Phase structure and ordering established. Research flags identified for Phases 3-4. Architectural conflicts resolved (extend financial, extend glossary). All critical pitfalls documented with prevention strategies.
