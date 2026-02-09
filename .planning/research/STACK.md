# Technology Stack: v1.1 Read-Layer Domain Expansion

**Project:** Project Cornerstone — Control ERP Natural Language Interface
**Researched:** 2026-02-09
**Mode:** Ecosystem (subsequent milestone — stack additions only)

---

## Context: What Already Exists

The v1.0 stack is proven and validated. This document covers ONLY additions/changes needed for 5 new domain skills. The existing stack is:

| Component | Status | DO NOT CHANGE |
|-----------|--------|---------------|
| Database: SQL Server (StoreData) via MCP MSSQL tools | Validated | Core access pattern |
| Skills: YAML frontmatter SKILL.md + references/ | Validated | Architecture pattern |
| Wiki knowledge: 1,769 pages, 12 extracts | Available | Reference material |
| Schema docs: 187 tables in output/schemas/ | Available | Reference material |
| Crystal Report analysis: 36 reports in output/report_summary.md | Available | Reference material |
| Core skill: TransactionType, StatusID, pricing, filters | Validated 99.98% | Foundation dependency |
| Sales skill: DyeSub container, product categories, customer revenue | Validated | Revenue queries |
| Financial skill: GL/Ledger, AR/AP, P&L, payments, closeout | Validated | Financial foundation |
| Glossary skill: 20 product categories, 30+ terms, 35+ NL routes | Validated | Routing layer |

---

## New Skill Files: Recommended Structure

### Skill 1: `control-erp-financial-deep`

**Purpose:** Extends control-erp-financial with deeper AR/AP analytics, P&L trending, cash flow analysis, and financial health metrics. The existing financial skill covers the foundation (GL architecture, basic AR/AP snapshots, P&L queries). This skill adds the analytical depth.

**Why a separate skill, not extending control-erp-financial:** The current financial skill is already 857 lines and validated. Adding 500+ lines of analytical queries risks destabilizing proven patterns. A separate deep skill depends on the foundation and adds analytical capabilities without touching validated SQL.

```
skills/control-erp-financial-deep/
  control-erp-financial-deep-SKILL.md    # YAML frontmatter + analytical queries
```

**No references/ subdirectory.** All reference material (GL accounts, NodeIDs, sign conventions) already exists in control-erp-financial. This skill only adds query templates and NL routing for deeper questions.

**SKILL.md sections:**

| Section | Content | Source Material |
|---------|---------|----------------|
| AR Trends & Analytics | AR aging trend over time, customer payment behavior, DSO calculation, payment velocity | TransHeader SaleDate/PaymentTotal time series |
| AP Analytics | AP aging trends, vendor payment patterns, DPO calculation | TransHeader Type 8, Payment table |
| P&L Trending | Month-over-month, YoY comparison, margin analysis by period | GL view with GLClassificationType 4000/4001/5001/5002 |
| Cash Flow Analysis | Cash inflow/outflow by period, cash conversion cycle, runway estimation | GL NodeIDs 90/10412 (bank accounts), Payment.TenderType |
| Financial Health KPIs | Current ratio, quick ratio, working capital, gross margin trend | Balance sheet accounts via GL |
| NL Routing | "DSO", "cash runway", "margin trend", "payment velocity" | New routing patterns |

**Depends on:** `control-erp-core`, `control-erp-financial`

**Existing resources to consume:**
- `output/wiki/extracts/orders_accounting_knowledge.md` -- GL integrity checks, payment lifecycle
- `output/wiki/extracts/sql_queries_reference.md` -- GL balance integrity SQL, financial report patterns
- `output/report_join_patterns.md` -- Income Statement, Balance Sheet, AR/AP report join patterns
- `output/schemas/Ledger.md`, `output/schemas/GLAccount.md`, `output/schemas/Payment.md`

**SQL patterns to discover/validate:**
- DSO calculation: `(AR Balance / Revenue) * Days in Period` -- need to validate AR Balance derivation matches Control's AR report
- Cash conversion cycle: DSO + DIO - DPO -- need DIO from inventory skill, DPO from AP data
- Month-over-month P&L comparison -- straightforward GL query, but need to validate against Closeout periods
- Cash flow statement reconstruction from GL entries -- need to map all cash-affecting accounts

**Confidence: HIGH** -- All underlying tables and patterns are already validated in control-erp-financial. This skill adds analytical SQL on proven foundations.

---

### Skill 2: `control-erp-customers`

**Purpose:** Customer intelligence -- lookup, segmentation, CLV analysis, churn detection, purchasing pattern analysis, customer health scoring.

**Why this is a new skill, not part of sales:** The sales skill focuses on product-level reporting. Customer intelligence is orthogonal -- it's about the Account entity and its relationships across all transaction types, not about product variables.

```
skills/control-erp-customers/
  control-erp-customers-SKILL.md         # YAML frontmatter + customer query templates
  references/
    segmentation_reference.md            # Customer classification criteria + SQL
```

**SKILL.md sections:**

| Section | Content | Source Material |
|---------|---------|----------------|
| Customer Lookup | Find by name, number, contact, phone, address | Account, AccountContact, PhoneNumber, Address, AddressLink |
| Customer Profile | Full 360-view: orders, revenue, payments, contacts, AR | TransHeader + Account + AccountContact + Payment |
| Segmentation | By revenue tier, order frequency, recency, industry, origin | Account flags + TransHeader aggregations |
| CLV Analysis | Lifetime value, average order value, order frequency, tenure | TransHeader time series per AccountID |
| Churn Detection | Days since last order, declining order frequency, at-risk identification | TransHeader.SaleDate per Account |
| Purchasing Patterns | Product mix, seasonal patterns, reorder cycles, cross-sell opportunities | TransHeader + TransDetail + TransDetailParam per Account |
| CRM Pipeline | Company stages, prospect-to-client conversion, salesperson attribution | Account.StageID, Element table (ClassTypeID 5511/5512), SalesPersonID1/2/3 |
| NL Routing | "who are our top customers", "customer churn", "CLV", "find customer X" | New routing patterns |

**Depends on:** `control-erp-core`

**Existing resources to consume:**
- `output/wiki/extracts/crm_payroll_system_knowledge.md` -- Company types/classification, Company Stages, marketing lists, contact activity
- `output/schemas/Account.md` -- 104 columns including IsClient, IsProspect, IsVendor, StageID, IndustryID, OriginID, CreditBalance, HasCreditAccount
- `output/schemas/AccountContact.md` -- Contact details
- `output/schemas/AccountUserField.md` -- Custom fields
- `output/schemas/ContactActivity.md` -- CRM activity tracking
- `output/schemas/MarketingListItem.md` -- Industry/origin classifications

**SQL patterns to discover/validate:**
- CLV formula: SUM(SubTotalPrice) WHERE TransactionType = 1 AND AccountID = X -- straightforward extension of core revenue formula
- RFM segmentation: Recency (MAX(SaleDate)), Frequency (COUNT orders), Monetary (SUM(SubTotalPrice)) -- need to validate grouping
- Churn threshold: DATEDIFF(DAY, MAX(SaleDate), GETDATE()) -- need to determine FLS-appropriate threshold (90 days? 180 days?)
- Company Stage query: Account.StageID -> Element table WHERE ClassTypeID IN (5511, 5512) -- wiki documents this but needs live validation
- Customer-salesperson attribution: Account.SalesPersonID1 -> Employee.ID -- straightforward FK

**references/segmentation_reference.md contents:**
- Revenue tier definitions (Top 10%, Next 20%, etc.) with SQL
- RFM scoring methodology adapted for print industry (seasonal ordering patterns)
- Industry classification mapping (MarketingListItem with MarketingListID = 10)
- Origin classification mapping (MarketingListItem with MarketingListID = 11)

**Confidence: HIGH** -- Account table schema is fully documented, CRM patterns documented in wiki extract, core revenue formula validated.

---

### Skill 3: `control-erp-inventory`

**Purpose:** Inventory levels, stock status, reorder alerts, warehouse management, part usage tracking, material cost analysis.

```
skills/control-erp-inventory/
  control-erp-inventory-SKILL.md         # YAML frontmatter + inventory query templates
  references/
    part_types_reference.md              # Part type classifications, inventory formulas
```

**SKILL.md sections:**

| Section | Content | Source Material |
|---------|---------|----------------|
| Inventory Architecture | How Control tracks inventory: Inventory table, PartID, WarehouseID, ClassTypeID 12200 | Wiki extract Section 4 |
| Stock Levels | Current levels by part, warehouse, division. Alert levels (yellow/red/reorder) | Inventory table: QuantityOnHand, QuantityAvailable, QuantityReserved, QuantityOnOrder |
| Inventory Valuation | Total inventory value, valuation by warehouse/category, average cost | Inventory.QuantityBilled * Inventory.AverageCost (wiki-validated pattern) |
| Reorder Analysis | Parts below reorder point, parts in alert zones, recommended PO quantities | Inventory.QuantityAvailable vs ReorderPoint, ReOrderQuantity |
| Part Usage | Actual vs estimated consumption, usage by order/product, cost analysis | TransPart (2.26M rows), PartUsageCard (287K rows) |
| Purchase Order Tracking | Open POs, on-order quantities, vendor delivery performance | TransHeader Type 7, VendorTransDetail |
| Warehouse Management | Inventory by warehouse, transfer patterns, warehouse utilization | Inventory.WarehouseID, InventoryLog with ToWarehouseID/FromWarehouseID |
| Inventory Movement Log | History of all inventory changes, consumption, receiving, adjustments | InventoryLog (907K rows) |
| NL Routing | "what's in stock", "low inventory", "reorder", "inventory value", "part usage" | New routing patterns |

**Depends on:** `control-erp-core`

**Existing resources to consume:**
- `output/wiki/extracts/production_inventory_knowledge.md` -- Sections 3-6: Parts, Inventory Management, Warehouses, POs
- `output/schemas/Inventory.md` -- 27K rows, all quantity columns documented
- `output/schemas/InventoryLog.md` -- 907K rows, movement tracking
- `output/schemas/Part.md` -- 7.7K rows, part definitions with PartType, TrackInventory, AccrueCosts
- `output/schemas/TransPart.md` -- 2.26M rows, parts linked to transactions
- `output/schemas/PartUsageCard.md` -- 287K rows, actual usage records
- `output/schemas/Warehouse.md` -- Warehouse definitions
- `output/schemas/Reservation.md` -- Inventory reservations
- `output/schemas/VendorTransDetail.md` -- PO/bill line items

**SQL patterns to discover/validate:**
- Stock level query: Inventory WHERE PartID = X AND ClassTypeID = 12200 AND WarehouseID = Y -- wiki provides pattern
- Inventory valuation: `SUM(QuantityBilled * AverageCost) WHERE ClassTypeID = 12200` -- wiki-validated
- Low stock alert: `WHERE QuantityAvailable <= ReorderPoint AND ReorderPoint > 0` -- need to validate alert logic
- Part usage analysis: TransPart with ActualValue vs EstimatedValue -- large table, need efficient query patterns
- PO on-order: VendorTransDetail WHERE TransactionType = 7, grouped by PartID -- wiki provides pattern

**references/part_types_reference.md contents:**
- Part.PartType values: 0=Material, 1=Labor, 2=Equipment, 3=Outsource, 4=Freight, 5=Other (from wiki)
- Inventory tracking modes: None, Level Only, Full Accrual (TrackInventory + AccrueCosts flags)
- Warehouse types: Standard, Kanban, Stockroom, Group
- Inventory formula: Billed + Received = OnHand; OnHand - Reserved = Available; Available + OnOrder = Expected

**Confidence: MEDIUM-HIGH** -- Wiki documentation is comprehensive. Inventory table schema is documented. Key uncertainty: FLS-specific warehouse configuration (how many warehouses? which types?) needs live database validation.

---

### Skill 4: `control-erp-production`

**Purpose:** Production workflow tracking, artwork approval status, station workload, time tracking, job costing, shipping status.

```
skills/control-erp-production/
  control-erp-production-SKILL.md        # YAML frontmatter + production query templates
  references/
    artwork_workflow_reference.md         # Artwork status machine, player roles
    station_reference.md                 # Station hierarchy (discovered from live data)
```

**SKILL.md sections:**

| Section | Content | Source Material |
|---------|---------|----------------|
| Station Architecture | Department/workstation hierarchy, ClassTypeID 26100, 98 stations defined | Station table, wiki Section 2 |
| WIP Pipeline | Orders in production by station, by due date, by customer | TransHeader StatusID 1/2, Station join |
| Artwork Workflow | Artwork group statuses, approval pipeline, proof tracking, aging | ArtworkGroup (85K rows), ArtworkGroupStatusHistory, ArtworkProofFile |
| Production Schedule | Orders by due date, station capacity, bottleneck identification | TransHeader.DueDate + Station joins |
| Time Tracking | Employee time on jobs/stations, labor cost by order/product | TimeCard (159K rows), PartUsageCard where PartType=1 |
| Job Costing | Estimated vs actual cost per order, cost breakdown by part type | TransPart (estimated vs actual), PartUsageCard |
| Shipping Status | Order shipment tracking, shipped vs unshipped, carrier breakdown | Shipments (71K rows), ShippingMethod |
| NL Routing | "WIP", "artwork status", "where's order X", "production schedule", "who's clocked in" | New routing patterns |

**Depends on:** `control-erp-core`

**Existing resources to consume:**
- `output/wiki/extracts/production_inventory_knowledge.md` -- Sections 1-2, 7: Artwork workflow, stations, shipping
- `output/schemas/ArtworkGroup.md` -- 85K rows, StatusID, TransHeaderID
- `output/schemas/ArtworkGroupStatusHistory.md` -- Status change audit trail
- `output/schemas/ArtworkGroupTransDetailLink.md` -- Links to line items
- `output/schemas/ArtworkPlayer.md` -- Designer/approver roles
- `output/schemas/ArtworkProofFile.md` -- Proof images
- `output/schemas/Station.md` -- 98 stations
- `output/schemas/TimeCard.md` -- 159K rows, ClassTypeID 20050/20051
- `output/schemas/Shipments.md` -- 71K rows, tracking, carriers
- `output/schemas/TransPart.md` -- 2.26M rows, cost tracking

**SQL patterns to discover/validate:**
- WIP by station: TransHeader JOIN Station ON th.StationID = Station.ID WHERE StatusID IN (1,2) -- need to verify TransHeader has StationID or if it's via TransDetail
- Artwork pipeline: ArtworkGroup WHERE StatusID < 7 GROUP BY StatusID -- wiki confirms StatusID 7 = Produced
- Artwork aging: DATEDIFF(DAY, ArtworkGroup.GroupCreatedDT, GETDATE()) WHERE StatusID < 7 -- straightforward
- Employee clocked in: TimeCard WHERE ClassTypeID = 20050 AND IsActive = 1 AND <active time window> -- need to discover how "currently clocked in" is represented
- Job cost variance: TransPart SUM(EstimatedCost) vs SUM(ActualCost) per TransHeaderID -- straightforward but need efficient query on 2.26M row table
- Shipping status: Shipments WHERE TransHeaderID = X, IsShipped flag -- straightforward

**references/artwork_workflow_reference.md contents:**
- Complete status machine: In Design -> Pending Approval -> Approved -> Production Ready -> Produced (StatusID values from wiki)
- Player roles: Designer (Employee), Approver (Contact)
- Notification triggers: 24h, 72h, 168h reminders
- Status auto-transitions: WIP->Sale/Built triggers artwork->Produced

**references/station_reference.md contents:**
- To be populated from live Station table query (98 stations with hierarchy)
- Department/workstation structure specific to FLS
- Station-to-workflow mapping (which stations are order-level vs line-item-level)
- Time clock eligible stations (ShowOnTimeClock = 1)

**Confidence: MEDIUM** -- Wiki documentation is strong for architecture patterns. Key uncertainty: FLS's actual station configuration, which artwork statuses are actively used, and how time tracking flows at FLS specifically. These require live database discovery queries.

---

### Skill 5: `control-erp-reports`

**Purpose:** Crystal Report catalog -- knows which reports exist, what they contain, when to reference them vs building ad-hoc queries. This is a routing/reference skill, not a query skill.

```
skills/control-erp-reports/
  control-erp-reports-SKILL.md           # YAML frontmatter + report catalog + routing logic
```

**No references/ subdirectory.** The report catalog is compact enough to live in the SKILL.md itself. The 36 Crystal Reports + 211 Report table records fit in a single well-organized file.

**SKILL.md sections:**

| Section | Content | Source Material |
|---------|---------|----------------|
| Report Architecture | How Crystal Reports work in Control: Report table (211 rows), ReportTemplate (168 rows), CrystalSystemReports, .rpt files | Schema docs + wiki |
| Crystal Report Catalog | 36 identified .rpt files with purpose, parameters, primary tables, when to use | output/report_summary.md |
| Report-vs-Query Decision Logic | When to point user to existing Crystal Report vs build ad-hoc SQL | Decision tree based on report capability match |
| Report Categories | Financial (5), WIP (4), Sales (14), Estimates (4), Inventory (6), Other (3) | output/report_summary.md |
| NL-to-Report Routing | "print AR aging report" -> A_R Detail.rpt, "WIP by station" -> WIP By Line Item Station.rpt | Mapping table |
| Ad-Hoc Alternative Patterns | When user wants something a Crystal Report nearly-but-not-quite covers, SQL alternative | Derived from report_join_patterns.md |

**Depends on:** `control-erp-core`

**Existing resources to consume:**
- `output/report_summary.md` -- 36 reports cataloged with purpose, primary tables
- `output/report_join_patterns.md` -- Inferred join patterns for financial, AR, AP, WIP, sales reports
- `output/reports/*.md` -- Individual report analysis files (if populated)
- `output/schemas/Report.md` -- 211 rows, ReportName, TemplateID, ShowOnMenu, CriteriaOptions
- `output/schemas/ReportTemplate.md` -- 168 rows, TemplateName, ReportFileName, CriteriaOptions
- `output/wiki/extracts/crm_payroll_system_knowledge.md` -- Reports section with Crystal Reports configuration

**SQL patterns to discover/validate:**
- Report catalog query: `SELECT ReportName, Description, ShowOnMenu FROM Report WHERE IsActive = 1` -- discover which reports are accessible in FLS's Control installation
- Template lookup: `SELECT TemplateName, ReportFileName FROM ReportTemplate WHERE IsActive = 1` -- discover actual .rpt file mappings
- These are discovery queries to populate the catalog, not ongoing queries the skill will run

**Confidence: HIGH** -- Report catalog is already documented in output/report_summary.md. The Report/ReportTemplate tables provide authoritative data. Main work is organizing into a routing-oriented skill format.

---

## Alternatives Considered and Rejected

### Rejected: Single monolithic "control-erp-advanced" skill
**Why not:** Each domain has distinct table sets, query patterns, and NL routing. A single skill would be 2000+ lines and impossible to maintain. The proven pattern from v1.0 is focused skills with clear boundaries.

### Rejected: Splitting financial-deep into AR-deep and AP-deep
**Why not:** AR and AP analytics share the same underlying patterns (TransHeader-based aging with GL verification). Splitting would create two tiny skills with redundant architecture sections. Keep them together.

### Rejected: Merging customers into sales
**Why not:** Customer intelligence queries are Account-centric (segmentation, CLV, churn). Sales queries are TransDetail-centric (product categories, revenue by product). Different primary tables, different join patterns, different NL intents. They share TransHeader as a common join point but that's true of every skill.

### Rejected: Separate shipping skill
**Why not:** Shipping is a subset of production workflow. At 71K rows in Shipments, it's a single section within the production skill, not enough for a standalone skill. Shipping questions ("where's my order?") are really production tracking questions.

### Rejected: HR/Payroll skill in v1.1
**Why not:** The project charter lists HR as Phase 5 (lower priority). The payroll tables (PayrollPaycheck, PayrollPayItem, etc.) are complex with tax tables and deduction logic. Adding payroll to v1.1 would expand scope beyond the 5-domain target. Defer to v1.2.

---

## Glossary Skill Updates Required

The existing `control-erp-glossary` must be updated when new skills are created. New routing entries needed:

| User Says | Route To |
|-----------|----------|
| "inventory", "stock", "reorder", "parts on hand" | control-erp-inventory |
| "artwork status", "proof approval", "WIP by station" | control-erp-production |
| "shipping", "tracking number", "what shipped today" | control-erp-production |
| "customer profile", "top customers", "CLV", "churn" | control-erp-customers |
| "DSO", "cash flow", "cash runway", "margin trend" | control-erp-financial-deep |
| "Crystal Report", "print report", "existing report" | control-erp-reports |

**Update glossary AFTER all new skills are validated**, not before. Do not route to skills that do not yet exist.

---

## Dependency Graph

```
control-erp-core (foundation, DO NOT MODIFY)
  |
  +-- control-erp-financial (foundation, DO NOT MODIFY)
  |     |
  |     +-- control-erp-financial-deep (NEW: extends with analytics)
  |
  +-- control-erp-sales (foundation, DO NOT MODIFY)
  |
  +-- control-erp-customers (NEW: customer intelligence)
  |
  +-- control-erp-inventory (NEW: stock/parts/warehouse)
  |
  +-- control-erp-production (NEW: workflow/artwork/shipping)
  |
  +-- control-erp-reports (NEW: Crystal Report catalog)
  |
  +-- control-erp-glossary (UPDATE: add new routing)
```

No circular dependencies. Each new skill depends only on core (and financial-deep additionally depends on financial). Skills do not depend on each other.

---

## Installation / File Creation

No package installations needed. This project uses Claude Code skills (markdown files), not application code.

**Files to create:**

```bash
# Skill 1: Financial Deep
skills/control-erp-financial-deep/control-erp-financial-deep-SKILL.md

# Skill 2: Customers
skills/control-erp-customers/control-erp-customers-SKILL.md
skills/control-erp-customers/references/segmentation_reference.md

# Skill 3: Inventory
skills/control-erp-inventory/control-erp-inventory-SKILL.md
skills/control-erp-inventory/references/part_types_reference.md

# Skill 4: Production
skills/control-erp-production/control-erp-production-SKILL.md
skills/control-erp-production/references/artwork_workflow_reference.md
skills/control-erp-production/references/station_reference.md

# Skill 5: Reports
skills/control-erp-reports/control-erp-reports-SKILL.md
```

**Total new files: 10** (5 SKILL.md + 5 reference docs)

**File to update: 1** (control-erp-glossary/control-erp-glossary-SKILL.md -- add routing)

---

## Validation Strategy Per Skill

Each skill needs validation before shipping. The v1.0 pattern (test against known numbers) applies:

| Skill | Validation Approach | Known Reference Point |
|-------|--------------------|-----------------------|
| financial-deep | DSO calculation verified against manual AR/revenue math | AR ~$81K, 2025 revenue $3.05M -> DSO ~9.7 days |
| customers | Top 10 customers by revenue cross-checked with sales skill output | Revenue totals must match control-erp-sales |
| inventory | Inventory valuation vs GL NodeID 10414 balance | GL Inventory account balance should approximate valuation |
| production | WIP count vs TransHeader WHERE StatusID=1 count | WIP + Built count from core |
| reports | Report catalog matches Report table active records | Report WHERE IsActive=1 row count |

---

## Build Order Recommendation

Based on dependency analysis and validation complexity:

1. **control-erp-customers** (lowest risk, Account table well-understood, extends core patterns)
2. **control-erp-reports** (reference/routing skill, no new SQL patterns to validate)
3. **control-erp-financial-deep** (extends validated financial foundation)
4. **control-erp-inventory** (new domain, needs live data discovery)
5. **control-erp-production** (most complex, needs live station/artwork discovery)

**Rationale:** Start with what we know well (customers = Account + TransHeader), build confidence with a simple catalog skill (reports), extend proven foundations (financial-deep), then tackle genuinely new domains that require database discovery (inventory, production).

---

## What NOT to Build

| Temptation | Why Skip |
|------------|----------|
| Analytics/forecasting skill | v1.1 is read-layer domain coverage. Statistical modeling is v1.2+ intelligence layer |
| Dashboard skill | Requires React frontend. Out of scope for skill-file architecture |
| Write-path skills | Phase 5 per charter. Read-layer must be complete first |
| HR/Payroll skill | Deferred to v1.2. Complex tax/deduction logic, lower business priority |
| Orchestrator skill | Routing handled by glossary updates. Dedicated orchestrator is premature |

---

## Sources

| Source | Type | Confidence |
|--------|------|------------|
| skills/control-erp-core/control-erp-core-SKILL.md | Existing validated skill | HIGH |
| skills/control-erp-financial/control-erp-financial-SKILL.md | Existing validated skill | HIGH |
| skills/control-erp-sales/control-erp-sales-SKILL.md | Existing validated skill | HIGH |
| skills/control-erp-glossary/control-erp-glossary-SKILL.md | Existing validated skill | HIGH |
| output/wiki/extracts/production_inventory_knowledge.md | Wiki extract | HIGH |
| output/wiki/extracts/crm_payroll_system_knowledge.md | Wiki extract | HIGH |
| output/wiki/extracts/orders_accounting_knowledge.md | Wiki extract | HIGH |
| output/wiki/extracts/sql_queries_reference.md | Wiki extract | HIGH |
| output/report_summary.md | Crystal Report analysis | MEDIUM (inferred, not parsed) |
| output/report_join_patterns.md | Crystal Report analysis | MEDIUM (inferred, not parsed) |
| output/schemas/*.md | Schema documentation | HIGH (from live database) |
| project-cornerstone-charter.md | Project architecture | HIGH (owner-authored) |
