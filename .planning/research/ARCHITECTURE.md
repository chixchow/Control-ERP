# Architecture Patterns: 5 New Domain Skills Integration

**Domain:** Control ERP Natural Language Interface (Read Layer Expansion)
**Researched:** 2026-02-09
**Confidence:** HIGH -- Based on direct analysis of existing v1.0 skill files, 187 table schemas, wiki extracts, project charter, and Crystal Reports analysis.

---

## 1. EXISTING ARCHITECTURE ANALYSIS

### Current Skill Topology (v1.0)

```
User (natural language)
    |
    v
[Claude reads relevant skills based on question context]
    |
    +-- control-erp-glossary (routing: maps terms -> owning skill)
    |       |
    |       +-- Routes to -->  control-erp-sales
    |       +-- Routes to -->  control-erp-financial
    |       +-- Routes to -->  control-erp-core
    |
    +-- control-erp-core (foundation: business rules, TransactionType, filters)
    |       ^
    |       |
    |       +-- Depended on by --> control-erp-sales
    |       +-- Depended on by --> control-erp-financial
    |
    +-- control-erp-sales (product revenue, customer revenue, sales trends)
    |
    +-- control-erp-financial (GL/Ledger, AR, AP, P&L, payments, closeouts)
```

### Current Skill Size Profile

| Skill | Lines | Sections | Query Templates | Reference Files |
|-------|-------|----------|-----------------|-----------------|
| control-erp-core | 347 | 11 | 6 | 0 |
| control-erp-sales | 370 | 7 | 9 | 0 |
| control-erp-financial | 856 | 12 | 15+ | 0 |
| control-erp-glossary | 491 | 5 | 0 | 0 |

**Observation:** The financial skill is already 2.5x the size of other skills. This is the largest single skill and already pushes the upper bound of what fits comfortably in context alongside the core skill that it depends on.

### Architectural Constraints Discovered in v1.0

1. **Context window budgeting.** The user's question + core skill + domain skill + glossary must all fit in working context. This is why skills were split by domain in v1.0.

2. **Core dependency is absolute.** Every domain skill needs core's TransactionType, StatusID, date filtering, and standard filters. Core cannot be split or removed from any query session.

3. **Glossary is the human-facing router.** It translates natural language to "which skill and which section." It does not contain query logic.

4. **No reference subdirectories exist.** Despite the project charter mentioning `references/` directories, v1.0 shipped all skills as single SKILL.md files. The planned `output/skill/references/` directory exists but is not part of the active skill architecture.

5. **Validation is per-skill, not per-query.** Each skill was validated against Control reports (AR report, AP report, Sales by Product, etc.) before shipping. This sets the quality bar.

---

## 2. KEY ARCHITECTURAL DECISION: EXTEND vs. CREATE

### Decision: Financial Depth -> EXTEND existing `control-erp-financial`

**Rationale:**

The existing `control-erp-financial` skill already contains:
- GL/Ledger architecture (complete)
- AR aging and AR snapshot (complete)
- AP aging and AP snapshot (complete)
- P&L Summary and Full P&L (complete)
- Balance Sheet key accounts (complete)
- Payment posting patterns (complete)
- Closeout/period locking (complete)

The "financial depth" domain asks for:
- AR/AP enhancements (already there)
- P&L reporting (already there)
- Cash flow analysis (new, builds on existing balance sheet queries)
- Expense drill-down (new, builds on existing P&L GLClassificationType 5002 queries)
- Bank reconciliation status (new, builds on existing Ledger.Reconciled field documented)
- Margin analysis (new, combines existing revenue + COGS patterns)

**Verdict: Extend, do not create a separate skill.** The financial skill already has the foundation. Adding 5-8 new query templates and a "Financial Analysis" section to the existing skill is cleaner than creating a `control-erp-financial-depth` skill that would duplicate 80% of the existing financial skill's context.

**Risk mitigation:** If the financial skill grows beyond ~1200 lines, consider extracting the lowest-frequency sections (closeout detail, off-balance sheet edge cases) into a `references/advanced-gl.md` file that is referenced but not always loaded. This would be the first use of the reference directory pattern.

### Decision: Customer Intelligence -> CREATE new `control-erp-customers`

**Rationale:**

The customer domain touches tables that no existing skill covers well:
- Account table (54,719 rows) -- only referenced via JOIN in sales/financial skills, never as primary entity
- AccountContact (64,851) -- not covered at all
- AccountContactUserField -- not covered
- AccountUserField -- not covered
- Address/AddressLink/PhoneNumber -- not covered
- ContactActivity (163,716) -- not covered
- CCToken/CCTransaction -- not covered
- Element table for Company Stages -- not covered
- MarketingListItem for Industry/Origin -- not covered

No existing skill has customer-as-primary-entity queries. Sales skill has "top customers by revenue" but that's TransHeader-centric, not Account-centric. Financial skill has "AR by customer" but that's also TransHeader-centric.

The customer skill needs:
- Customer lookup (by name, number, phone, email, address)
- Customer profile (contact info, addresses, payment terms, pricing level)
- Customer segmentation (IsClient/IsProspect, StageID, IndustryID, DivisionID)
- Customer lifetime value (CLV = total historical SubTotalPrice)
- Customer purchase patterns (frequency, recency, average order value)
- Churn detection (days since last order)
- Customer credit management (CreditBalance, CreditLimit, HasCreditAccount)
- Contact management (primary contact, all contacts, activity history)

**This is a genuinely new domain.** Create `control-erp-customers`.

### Decision: Inventory -> CREATE new `control-erp-inventory`

**Rationale:**

Inventory touches 13 tables (1.2M rows total) that no existing skill covers:
- Inventory (27,268) -- stock levels per part/warehouse
- InventoryLog (906,714) -- all inventory transactions
- Part (7,684) -- materials, labor, equipment definitions
- PartUsageCard (286,891) -- actual consumption records
- Warehouse (5) -- warehouse locations
- AssemblyLink (19) -- bill of materials
- GoodsItemPartLink (1,388) -- product-to-part linkage
- PartInventoryConversion (2,290) -- unit conversions

The financial skill mentions inventory only as GLAccount NodeID 10414 (inventory asset value). The inventory skill needs actual stock levels, not GL balances.

**This is a genuinely new domain.** Create `control-erp-inventory`.

### Decision: Production -> CREATE new `control-erp-production`

**Rationale:**

Production spans artwork workflow + station tracking + time tracking + shipping. This involves:
- ArtworkGroup (84,770) + 17 related artwork tables
- Station (98) + PrCA.* production tables
- TimeCard (159,448) with ClassTypeID 20050/20051 pattern
- Journal (JournalActivityType=45 for station changes)
- PartUsageCard (for labor/equipment time)
- Shipments (70,584) + carrier logs

This domain is completely uncovered by existing skills. It is also large enough to warrant its own skill.

**Note on scope boundaries:** Production skill covers the production workflow from "order in WIP" through "shipped." It does NOT cover the order-level revenue or GL impacts -- those stay in sales and financial skills respectively. The boundary is: production answers "where is this order in the workflow?" and "how long did things take?", not "how much revenue did this generate?"

**This is a genuinely new domain.** Create `control-erp-production`.

### Decision: Reports Catalog -> EXTEND `control-erp-glossary`

**Rationale:**

The reports "skill" is fundamentally a routing function: "The user asked for X. Does a Crystal Report already exist for this? If yes, reference it. If no, build a custom query."

This is exactly what the glossary already does for terms and concepts. The glossary maps natural language to the right skill and section. The reports catalog maps natural language to the right Crystal Report or skill.

Creating a standalone `control-erp-reports` skill would contain:
- A list of 36 report names with descriptions
- A routing table mapping user questions to report names
- Possibly parameter documentation for each report

This is ~100-150 lines of content. Too thin for a standalone skill. More naturally lives as a new section in the glossary: "Crystal Reports Routing Table."

**Verdict: Add a "Crystal Reports Catalog" section to `control-erp-glossary`.** Include report name, purpose, when to reference it vs. building a custom query, and the key parameters each report expects.

**Exception case:** If the report catalog grows beyond 36 entries (e.g., if FLS has custom reports beyond the standard set), or if parameter documentation becomes extensive, then split into a standalone skill. But start in the glossary.

---

## 3. RESULTING ARCHITECTURE (v2.0)

### Skill Topology After Expansion

```
User (natural language)
    |
    v
[Claude reads relevant skills based on question context]
    |
    +-- control-erp-glossary (EXTENDED: + Crystal Reports catalog)
    |       |
    |       +-- Routes to -->  control-erp-sales
    |       +-- Routes to -->  control-erp-financial (EXTENDED: + depth queries)
    |       +-- Routes to -->  control-erp-customers (NEW)
    |       +-- Routes to -->  control-erp-inventory (NEW)
    |       +-- Routes to -->  control-erp-production (NEW)
    |       +-- Routes to -->  control-erp-core
    |       +-- Routes to -->  Crystal Report [name] (when existing report fits)
    |
    +-- control-erp-core (UNCHANGED: foundation for all skills)
    |       ^
    |       |
    |       +-- Depended on by --> control-erp-sales
    |       +-- Depended on by --> control-erp-financial
    |       +-- Depended on by --> control-erp-customers (NEW dependency)
    |       +-- Depended on by --> control-erp-inventory (partial dependency)
    |       +-- Depended on by --> control-erp-production (partial dependency)
    |
    +-- control-erp-sales (UNCHANGED)
    |       ^
    |       +-- Cross-referenced by --> control-erp-customers (for revenue context)
    |
    +-- control-erp-financial (EXTENDED: + cash flow, margin, expense drill-down)
    |       ^
    |       +-- Cross-referenced by --> control-erp-inventory (for GL cost accounts)
    |
    +-- control-erp-customers (NEW: account lookup, segmentation, CLV, churn)
    |
    +-- control-erp-inventory (NEW: stock, parts, warehouse, reorder, consumption)
    |
    +-- control-erp-production (NEW: artwork, stations, time tracking, shipping)
```

### Dependency Classification

| Skill | Core Dependency | Cross-References |
|-------|----------------|-----------------|
| control-erp-sales | HARD -- TransactionType, pricing fields, date filters | None |
| control-erp-financial | HARD -- TransactionType, StatusID, GL architecture | None |
| control-erp-customers | HARD -- TransactionType (for order history), Account.CompanyName gotcha | SOFT: sales (for revenue context in CLV) |
| control-erp-inventory | SOFT -- Part.PartTypeID reference, TransPart joins | SOFT: financial (for GL cost account NodeIDs) |
| control-erp-production | SOFT -- StatusID (WIP/Built/Sale), TransactionType 1 | SOFT: inventory (for part usage cards) |

**HARD dependency** = Skill references core business rules on most queries. Must always read core first.
**SOFT dependency** = Skill uses some core concepts but many queries are self-contained within its own domain tables.

### Context Window Budget (per query session)

A typical query session loads: core + glossary + 1 domain skill.

| Configuration | Estimated Tokens | Fits? |
|---------------|-----------------|-------|
| Core (347 lines) + Glossary (~600 lines) + Sales (370 lines) | ~8,000 | Yes |
| Core + Glossary + Financial (~1,000 lines after extension) | ~12,000 | Yes |
| Core + Glossary + Customers (~400 lines est.) | ~8,500 | Yes |
| Core + Glossary + Inventory (~500 lines est.) | ~9,000 | Yes |
| Core + Glossary + Production (~600 lines est.) | ~9,500 | Yes |
| Core + Glossary + Financial + Sales (cross-domain) | ~14,000 | Yes, but tight |

**Conclusion:** All configurations fit within Claude's context window. The extended financial skill is the largest but still manageable. Cross-domain queries (e.g., "customer revenue and AR status") may need 2 domain skills simultaneously; this works but should be the exception, not the rule.

---

## 4. INTEGRATION POINTS

### 4.1 Core Skill Integration Points

Every new skill needs these from `control-erp-core`:

| Core Concept | Used By Customer Skill | Used By Inventory Skill | Used By Production Skill |
|-------------|----------------------|------------------------|------------------------|
| TransactionType = 1 (Orders) | Yes -- order history for CLV | Partial -- TransPart joins | Yes -- WIP/production tracking |
| TransactionType = 7/8/9 (Vendor) | No | Yes -- PO receiving | No |
| StatusID reference | Yes -- active orders vs closed | Partial -- WIP status for consumption | Yes -- WIP/Built/Sale workflow |
| IsActive = 1 filter | Yes -- all Account queries | Yes -- all Part/Inventory queries | Yes -- all Artwork/Station queries |
| SaleDate filtering | Yes -- for CLV date ranges | No -- uses PostDate, ModifiedDate | No -- uses station change dates |
| SubTotalPrice | Yes -- for CLV calculations | No | No |
| Account.CompanyName (not AccountName) | Yes -- primary gotcha | No | No |

### 4.2 Financial Skill Extension Points

New financial depth queries build on existing patterns:

| Existing Pattern | New Extension |
|-----------------|--------------|
| P&L Summary (4000/4001/5001/5002) | Monthly P&L trend, YoY comparison, margin by period |
| Balance Sheet key accounts | Cash flow statement (period-over-period balance changes) |
| AR Aging Buckets | AR collection velocity, DSO calculation, customer payment pattern |
| AP Aging | Cash requirements forecast (upcoming AP due dates) |
| Payment TenderType breakdown | Payment method trend analysis |
| Closeout dates | Period comparison guardrails (only compare closed periods) |

### 4.3 Glossary Extension Points

New routing entries needed:

| User Intent | Routes To | Section |
|------------|-----------|---------|
| "look up customer" / "find company" / "customer info" | control-erp-customers | Customer Lookup |
| "customer history" / "what did [X] buy" | control-erp-customers (+ sales cross-ref) | Purchase History |
| "CLV" / "lifetime value" / "best customers" | control-erp-customers | CLV Analysis |
| "churn" / "at-risk customers" / "inactive accounts" | control-erp-customers | Churn Detection |
| "stock levels" / "inventory" / "what's in stock" | control-erp-inventory | Stock Levels |
| "reorder" / "low stock" / "need to order" | control-erp-inventory | Reorder Alerts |
| "parts" / "materials" / "what parts for [product]" | control-erp-inventory | Part Lookup |
| "warehouse" / "which warehouse" | control-erp-inventory | Warehouse Queries |
| "artwork status" / "proof status" | control-erp-production | Artwork Pipeline |
| "production schedule" / "what's in production" | control-erp-production | Station Workload |
| "where is order #NNNNN" | control-erp-production | Order Tracking |
| "time tracking" / "labor hours" / "who worked on" | control-erp-production | Time Tracking |
| "shipped" / "tracking number" / "shipments" | control-erp-production | Shipping |
| "cash flow" / "money coming in" | control-erp-financial | Cash Flow (NEW) |
| "margin by month" / "profit trend" | control-erp-financial | Margin Analysis (NEW) |
| "DSO" / "days sales outstanding" | control-erp-financial | AR Analysis (NEW) |
| "[Crystal Report name]" | Crystal Report catalog (NEW glossary section) | Report Reference |

### 4.4 Cross-Skill Query Patterns

Some user questions naturally span two skills. The architecture must handle this gracefully.

| User Question | Primary Skill | Secondary Skill | Resolution |
|--------------|--------------|----------------|------------|
| "Customer X revenue and AR balance" | customers | financial | Customer skill provides account context; financial skill provides AR aging |
| "Which products are running low?" | inventory | sales (implicit) | Inventory answers stock; product revenue context is optional |
| "Production costs for order #NNNNN" | production | financial | Production provides station/time data; financial provides GL cost entries |
| "Order #NNNNN full status" | production | sales + financial | Production for workflow position; sales for line items; financial for payment status |

**Resolution strategy:** Each skill should be self-sufficient for its primary queries. Cross-skill questions are answered by reading both skills in the same session (fits in context window) or by the user's follow-up question naturally shifting context.

---

## 5. NEW SKILL SPECIFICATIONS

### 5.1 control-erp-customers

**YAML frontmatter:**
```yaml
name: control-erp-customers
description: Look up FLS Banners customers, contacts, account status, segmentation, lifetime value, purchase patterns, and churn risk from Control ERP. Use when the user asks about a specific customer, customer segments, best customers, at-risk accounts, contact information, or customer history. Depends on control-erp-core for business rules.
```

**Primary Tables:**

| Table | Rows | Role |
|-------|------|------|
| Account | 54,719 | Primary entity -- customer/vendor records |
| AccountContact | 64,851 | Contact people per account |
| AccountContactUserField | 64,785 | Custom fields on contacts |
| AccountUserField | 54,752 | Custom fields on accounts |
| Address | 125,614 | Physical addresses |
| AddressLink | 304,987 | Address-to-entity links |
| PhoneNumber | 314,805 | Phone/fax/email |
| ContactActivity | 163,716 | CRM activity log |
| Element | 223 | Company stage definitions (ClassTypeID 5511/5512) |
| MarketingListItem | 96 | Industry/Origin/Region classifications |
| PaymentTerms | 28 | Customer payment terms |

**Key Query Templates Needed:**
1. Customer lookup by name (fuzzy LIKE match)
2. Customer profile (all details + contacts + addresses + phone)
3. Customer order history (join to TransHeader Type 1)
4. Customer lifetime value (SUM SubTotalPrice, all-time)
5. Top N customers by revenue (period-specific)
6. Customer segmentation by stage (StageID + Element lookup)
7. Customer segmentation by industry (IndustryID + MarketingListItem)
8. Churn detection (DATEDIFF from last SaleDate to now)
9. RFM analysis (Recency, Frequency, Monetary value)
10. Customer credit status (CreditBalance, CreditLimit, HasCreditAccount)
11. Contact activity history (calls, notes, emails per account)

**Core Dependency Points:**
- TransactionType = 1 for order history
- StatusID != 9 for excluding voided
- SubTotalPrice for revenue calculations
- Account.CompanyName gotcha (not AccountName)

**Validation Approach:**
- Cross-validate top customer list against sales skill Template 6
- Verify total CLV sums match total revenue from core skill
- Spot-check 3-5 individual customer profiles against Control UI

### 5.2 control-erp-inventory

**YAML frontmatter:**
```yaml
name: control-erp-inventory
description: Query FLS Banners inventory levels, part information, stock alerts, warehouse data, reorder points, material consumption, and part-product relationships from Control ERP. Use when the user asks about stock levels, what's in stock, reorder needs, parts, materials, warehouse locations, inventory history, or material consumption. Partially depends on control-erp-core for TransPart joins.
```

**Primary Tables:**

| Table | Rows | Role |
|-------|------|------|
| Inventory | 27,268 | Current stock per part/warehouse |
| InventoryLog | 906,714 | All inventory movements |
| Part | 7,684 | Part definitions (materials, labor, equipment) |
| PartUsageCard | 286,891 | Actual consumption records |
| PartInventoryConversion | 2,290 | Unit conversion rules |
| Warehouse | 5 | Warehouse locations |
| AssemblyLink | 19 | Bill of materials |
| GoodsItemPartLink | 1,388 | Product-to-part links |
| TransPart | 2,256,954 | Parts used on orders (high volume!) |

**Key Query Templates Needed:**
1. Current stock levels by part (QuantityOnHand, QuantityAvailable)
2. Low stock alerts (QuantityOnHand <= ReorderPoint or <= RedNotificationPoint)
3. Part lookup by name/SKU
4. Inventory by warehouse
5. Inventory movement history (InventoryLog for a specific part)
6. Part consumption by order (PartUsageCard + TransHeader join)
7. Bill of materials for a product (GoodsItemPartLink)
8. Part types breakdown (PartTypeID: 0=Material, 1=Labor, etc.)
9. Average cost tracking (Inventory.AverageCost)
10. Reorder report (parts where QuantityAvailable < ReorderQuantity)
11. Inventory valuation (SUM of QuantityOnHand * AverageCost)

**Core Dependency Points:**
- PartTypeID enumeration (0-6) from wiki knowledge
- TransPart joins to TransHeader for order-context queries
- GL cost account NodeIDs (10414=Inventory, 10178=COGS) for financial cross-reference

**Performance Warning:**
- TransPart has 2.25M rows. ALWAYS filter by TransHeaderID or TransDetailID.
- InventoryLog has 906K rows. ALWAYS filter by PartID or date range.
- PartUsageCard has 286K rows. Filter by TransHeaderID or date range.

**Validation Approach:**
- Compare inventory valuation query against GL NodeID 10414 balance
- Cross-reference Inventory Listing report output
- Spot-check 5 specific parts: query stock vs. Control UI

### 5.3 control-erp-production

**YAML frontmatter:**
```yaml
name: control-erp-production
description: Track FLS Banners production workflow including artwork approval status, station workload, order production tracking, employee time on jobs, shipment status, and production scheduling from Control ERP. Use when the user asks about artwork proofs, production status, station assignments, where an order is, time tracking on jobs, labor hours, shipments, or production bottlenecks. Partially depends on control-erp-core for StatusID and TransactionType.
```

**Primary Tables:**

| Table | Rows | Role |
|-------|------|------|
| ArtworkGroup | 84,770 | Artwork proof groups per order |
| ArtworkGroupStatusHistory | 164,226 | Artwork status change audit |
| ArtworkGroupTransDetailLink | 78,998 | Links artwork to line items |
| ArtworkPlayer | 176,445 | Designers/approvers assigned |
| ArtworkProofFile | 41,741 | Proof file records |
| ArtworkComment | 4,109 | Proof comments/feedback |
| Station | 98 | Production stations/workcenters |
| TimeCard | 159,448 | Employee time records (20050=parent, 20051=station detail) |
| Journal | 5,184,393 | Station changes (ActivityType=45) |
| Shipments | 70,584 | Shipment records |
| ShippingMethod | 11 | Shipping methods |
| FedExShippingLogForShipments_FLS | 7,172 | FedEx detail |
| UPSShippingLog | 68,382 | UPS log |
| UPSShippingLogForShipments_FLS | 28,463 | UPS detail |
| PrCA.Job.Data | 383 | Production board jobs |
| PrCA.Job.TransactionLink | 383 | Job-to-order links |

**Key Query Templates Needed:**

*Artwork Pipeline:*
1. Open artwork groups by status (In Design, Pending Approval, etc.)
2. Artwork groups for a specific order
3. Artwork approval timeline (ArtworkGroupStatusHistory)
4. Overdue artwork approvals (StatusID = pending, age > threshold)
5. Artwork by designer (ArtworkPlayer.EmployeeID)

*Station Workflow:*
6. Orders by station (TransHeader.StationID or TransDetail.StationID)
7. Station workload (count/value of orders per station)
8. Station throughput (orders entering/leaving per period)

*Time Tracking:*
9. Employee time on a specific order (TimeCard + Journal)
10. Labor hours by station for a period
11. Employee time clock status (who's clocked in, on what)

*Shipping:*
12. Shipments for an order
13. Shipments by date range
14. Carrier breakdown (FedEx vs UPS, by volume and cost)

**Core Dependency Points:**
- StatusID 1=WIP, 2=Built, 3=Sale for production workflow queries
- TransactionType = 1 for order production tracking
- TransHeader.StationID for current order station

**Special Patterns:**
- TimeCard ClassTypeID: 20050 = clock in/out parent, 20051 = station time detail
- ArtworkGroup StatusID: < 7 = in progress, 7 = Produced
- Journal JournalActivityType = 45 for station time tracking
- Journal table is 5.1M rows -- MUST filter by TransactionID or date range

**Validation Approach:**
- Cross-reference WIP reports (WIP Summary, WIP By Line Item Station, WIP By Machine)
- Spot-check artwork status for 3-5 active orders against Control UI
- Compare Shipped By Date report output against shipment queries

---

## 6. FINANCIAL SKILL EXTENSION SPECIFICATION

### New Sections to Add to `control-erp-financial`

**Section: Cash Flow Analysis**
```sql
-- Cash flow = period-over-period change in cash accounts
-- Cash accounts: NodeID 90 (Checking), 10412 (MM)
SELECT
    YEAR(l.EntryDateTime) AS Year,
    MONTH(l.EntryDateTime) AS Month,
    SUM(CASE WHEN ga.GLClassificationType = 1000 THEN l.Amount ELSE 0 END) AS CashFlow,
    SUM(CASE WHEN ga.GLClassificationType IN (4000,4001) THEN -l.Amount ELSE 0 END) AS Revenue,
    SUM(CASE WHEN ga.GLClassificationType IN (5001,5002) THEN l.Amount ELSE 0 END) AS Expenses
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY YEAR(l.EntryDateTime), MONTH(l.EntryDateTime)
ORDER BY Year, Month
```

**Section: Margin Analysis**
- Gross margin by month (Revenue - COGS) / Revenue
- Margin by product line (using GLAccount revenue account NodeIDs)
- Margin trend over time

**Section: AR Velocity**
- DSO (Days Sales Outstanding) = (AR Balance / Revenue) * Days in Period
- Collection rate = Payments received / Opening AR balance
- Customer payment pattern analysis (average days to pay)

**Section: Cash Requirements Forecast**
- Upcoming AP due dates by period
- Open PO commitments (Type 7 with StatusID in (25,26,27))
- Expected collections from AR by aging bucket

**Section: Bank Reconciliation Status**
- Unreconciled GL entries (Ledger.Reconciled = 0)
- Reconciliation summary by account

### Glossary Routing Additions for Financial Depth

| User Says | Maps To |
|-----------|---------|
| "cash flow" / "money in vs money out" | Financial -> Cash Flow Analysis |
| "gross margin" / "profit margin" / "margin trend" | Financial -> Margin Analysis |
| "DSO" / "days sales outstanding" / "how fast customers pay" | Financial -> AR Velocity |
| "cash forecast" / "what do we owe this month" | Financial -> Cash Requirements Forecast |
| "reconciliation status" / "unreconciled entries" | Financial -> Bank Reconciliation |

---

## 7. GLOSSARY EXTENSION: CRYSTAL REPORTS CATALOG

### New Section Structure

Add after the existing "Natural Language Routing Table" section:

```markdown
## CRYSTAL REPORTS CATALOG

When a user's question matches an existing Crystal Report, reference the report
first. Build custom queries only when no suitable report exists or when the user
needs data the report does not provide.

### Financial Reports
| Report | Use When User Asks | Parameters |
|--------|-------------------|------------|
| FLS Income Statement.rpt | "P&L", "income statement" | Date range, Division |
| FLS Balance Sheet.rpt | "balance sheet", "assets and liabilities" | As-of date, Division |
| A_R Detail.rpt | "AR detail", "invoice detail by customer" | As-of date |
| AR Report - Summary.rpt | "AR summary" | As-of date |
| A_P Aging Detail.rpt | "AP aging", "vendor aging" | As-of date |

### Sales Reports
[14 reports mapped]

### WIP Reports
[4 reports mapped]

### Inventory Reports
[6 reports mapped]

### Other Reports
[7 reports mapped]

### When to Build Custom vs Reference Report
- Reference report: User wants standard view with standard filters
- Build custom: User wants date range/grouping/filter the report does not support
- Build custom: User wants data from multiple reports combined
- Build custom: User wants calculations not in any report (CLV, churn, RFM)
```

---

## 8. DATA FLOW: NL QUESTION -> SQL -> OUTPUT

### Current Flow (v1.0)

```
1. User asks question in natural language
2. Claude identifies relevant skill(s) from context + glossary
3. Claude reads core skill for business rules
4. Claude reads domain skill for query templates
5. Claude adapts template to specific question (date range, filters, etc.)
6. Claude executes SQL via MCP MSSQL read_data tool
7. Claude formats results and presents to user
```

### Updated Flow (v2.0)

```
1. User asks question in natural language
2. Claude checks glossary for routing:
   a. Does a Crystal Report exist for this exact question? -> Reference it
   b. Which domain skill(s) does this question map to? -> Load those
3. Claude reads core skill for business rules
4. Claude reads primary domain skill
5. If cross-domain: Claude reads secondary domain skill
6. Claude adapts template to specific question
7. Claude executes SQL via MCP MSSQL read_data tool
8. Claude formats results and presents to user
```

**Key difference:** Step 2a is new -- the Crystal Reports catalog check. This prevents building custom queries when a validated report already exists.

---

## 9. SUGGESTED BUILD ORDER

### Rationale for Order

The build order is driven by:
1. **Dependency depth:** Skills that depend on nothing (beyond core) go first
2. **Validation complexity:** Skills where we can validate against existing reports go earlier
3. **Cross-reference readiness:** Skills referenced by others must exist first
4. **User value:** Higher-value domains prioritized

### Recommended Build Order

**Phase 1: Financial Depth Extension** (extends existing skill)
- Lowest risk: extending an already-validated skill
- No new skill file to create, just adding sections
- Validation: compare against FLS Income Statement, Balance Sheet, AR/AP reports
- Glossary impact: 5-8 new routing entries

**Phase 2: Customer Intelligence** (new skill)
- No dependency on other new skills
- High user value (customer lookup is daily use)
- Well-defined validation (cross-reference with sales skill top customers)
- Glossary impact: 10-15 new routing entries + terms

**Phase 3: Inventory** (new skill)
- No dependency on other new skills
- Validation against Inventory Listing, All Parts Listing reports
- Cross-references financial skill for GL cost accounts (already done in Phase 1)
- Glossary impact: 8-12 new routing entries + terms

**Phase 4: Production** (new skill)
- Largest domain (artwork + stations + time + shipping)
- Benefits from inventory skill existing (part usage card context)
- Validation against WIP reports, Shipped By Date
- Glossary impact: 15-20 new routing entries + terms

**Phase 5: Reports Catalog + Glossary Update** (extends glossary)
- Must come LAST because it routes to ALL other skills
- Requires all domain skills to exist before mapping routes
- Also performs the comprehensive glossary update for all new domain terms
- Validation: manual walk-through of 20+ NL queries to verify routing

### Dependency Diagram

```
Phase 1: Financial Depth
    |
    +-- (no dependency)
    |
Phase 2: Customer Intelligence
    |
    +-- (no dependency on Phase 1, can parallelize)
    |
Phase 3: Inventory
    |
    +-- Soft cross-ref to financial (GL cost accounts) -- Phase 1
    |
Phase 4: Production
    |
    +-- Soft cross-ref to inventory (part usage) -- Phase 3
    |
Phase 5: Reports Catalog + Glossary Update
    |
    +-- Requires ALL above phases complete (routes to all skills)
```

**Parallelization opportunity:** Phases 1 and 2 have zero dependencies on each other and could be built simultaneously. Phases 3 and 4 have a soft dependency (production references part usage cards from inventory domain) but could also be partially parallelized.

---

## 10. NEW REFERENCE DOCUMENTS PER SKILL

### Whether to Use Reference Directories

v1.0 shipped without reference directories. The question is whether v2.0 should introduce them.

**Recommendation: Not yet.** Keep each skill as a single SKILL.md file. Reasons:
1. Consistency with v1.0 pattern
2. Simpler to load (one file, not file + references)
3. No skill currently exceeds the point where splitting helps
4. The financial skill at ~1000 lines is still manageable

**When to reconsider:** If any skill exceeds ~1200 lines, extract low-frequency sections into `references/` files that are loaded on-demand. The most likely candidate is `control-erp-financial` after the depth extension.

### Source Documents Available Per New Skill

Each new skill can draw from:

| New Skill | Schema Files | Wiki Extracts | Report Analysis |
|-----------|-------------|---------------|-----------------|
| Customers | Account.md, AccountContact.md, AccountContactUserField.md, AccountUserField.md, Address.md, AddressLink.md, PhoneNumber.md, ContactActivity.md, CCToken.md, CCTransaction.md | crm_payroll_system_knowledge.md | Last Contact Aging.rpt, Sales By Customer reports |
| Inventory | Inventory.md, InventoryLog.md, Part.md, PartUsageCard.md, PartInventoryConversion.md, Warehouse.md, AssemblyLink.md, GoodsItemPartLink.md | production_inventory_knowledge.md, parts_knowledge.md | Inventory Listing.rpt, Inventory History.rpt, All Parts Listing.rpt |
| Production | ArtworkGroup.md, ArtworkGroupStatusHistory.md, ArtworkGroupTransDetailLink.md, ArtworkPlayer.md, ArtworkProofFile.md, ArtworkComment.md, Station.md, TimeCard.md, Shipments.md, PrCA.*.md | production_inventory_knowledge.md | WIP reports (4), Shipped By Date.rpt, Work Order with Proof.rpt |
| Financial (ext) | Already loaded in existing skill | orders_accounting_knowledge.md, sql_queries_reference.md | FLS Income Statement.rpt, Balance Sheet.rpt, AR/AP reports |

---

## 11. VALIDATION APPROACH PER DOMAIN

### Validation Matrix

| Skill | Control Report to Validate Against | Key Metric | Acceptance Criteria |
|-------|-----------------------------------|------------|-------------------|
| Financial (ext) | FLS Income Statement, FLS Balance Sheet | Net income, total assets | Match within 1% for closed periods |
| Customers | Sales By Customer (Volume) | Top 10 customer revenue | Match top 10 ranking and revenue within 1% |
| Inventory | Inventory Listing | Stock levels for 10 parts | Exact match (inventory is point-in-time) |
| Production | WIP Summary - Order Level | WIP count and total value | Match count exactly, value within 1% |
| Glossary (reports) | Manual routing test | 20 NL queries routed correctly | 18/20 correct routing (90%) |

### Validation Sequence

1. Build skill with query templates
2. Run 3-5 spot-check queries against Control UI
3. Run full validation query against matching Crystal Report output
4. Document any discrepancies with root cause
5. Adjust skill based on findings
6. Re-validate until acceptance criteria met
7. Update glossary routing entries

---

## 12. ANTI-PATTERNS TO AVOID

### Anti-Pattern 1: Duplicating Core Business Rules

**What:** Copying TransactionType, StatusID, or pricing rules into domain skills.
**Why bad:** Creates maintenance burden. When rules are corrected (as happened in v1.0), multiple files need updating.
**Instead:** Reference core skill. Write "See control-erp-core for StatusID reference" and use the values without re-documenting them.

### Anti-Pattern 2: Cross-Domain Query Templates

**What:** Putting a "customer revenue" query template in the customer skill that duplicates the sales skill's Template 6.
**Why bad:** Two sources of truth for the same query.
**Instead:** Customer skill should have a "Customer Purchase History" template that uses Account as primary entity. Sales skill's Template 6 uses TransHeader as primary entity. Different perspectives on the same data, not duplicates.

### Anti-Pattern 3: Oversized Skills

**What:** Putting inventory, production, and shipping all in one skill because they're "operations."
**Why bad:** Context window waste. A question about stock levels does not need artwork approval workflow in context.
**Instead:** Split by query affinity. Group tables that are commonly queried together.

### Anti-Pattern 4: Incomplete IsActive/StatusID Filtering

**What:** Forgetting to include IsActive = 1 on new domain tables.
**Why bad:** Returns inactive/deleted records, producing incorrect counts and sums.
**Instead:** Every new skill must include a "Standard Filters" section mirroring core skill's pattern. For each primary table, document the required WHERE clause.

### Anti-Pattern 5: Neglecting the Glossary

**What:** Building new skills without updating the glossary routing table.
**Why bad:** User says "customer info" and Claude doesn't know to load the customer skill.
**Instead:** Phase 5 (glossary update) is mandatory, not optional. Every new skill must have corresponding glossary entries.

---

## SOURCES

All findings in this document are based on direct analysis of:

- `/Users/cain/projects/control-db-map/skills/control-erp-core/control-erp-core-SKILL.md` (v1.0 core skill)
- `/Users/cain/projects/control-db-map/skills/control-erp-sales/control-erp-sales-SKILL.md` (v1.0 sales skill)
- `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` (v1.0 financial skill)
- `/Users/cain/projects/control-db-map/skills/control-erp-glossary/control-erp-glossary-SKILL.md` (v1.0 glossary)
- `/Users/cain/projects/control-db-map/output/domains.md` (187 tables, 16 domains)
- `/Users/cain/projects/control-db-map/output/schemas/` (187 table schemas)
- `/Users/cain/projects/control-db-map/output/report_summary.md` (36 Crystal Reports)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/` (12 wiki knowledge extracts)
- `/Users/cain/projects/control-db-map/project-cornerstone-charter.md` (architecture vision)
- `/Users/cain/projects/control-db-map/project-cornerstone-updated-plan.md` (current plan)

**Confidence: HIGH.** All recommendations are derived from existing, validated project artifacts. No external sources or unverified claims.
