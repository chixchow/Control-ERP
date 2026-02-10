# Phase 13: Glossary Integration + Reports Catalog - Research

**Researched:** 2026-02-09
**Domain:** Skill routing, Crystal Reports catalog, NL query classification
**Confidence:** HIGH

## Summary

This phase extends the existing `control-erp-glossary` skill with two capabilities: (1) a Crystal Reports catalog mapping 36 reports to user intents with coverage status, and (2) expanded NL routing to all 6 domain skills (core, sales, financial, customers, inventory, production). The existing glossary is already a well-structured routing skill at 492 lines with product terminology, technical terminology, and a 37-entry routing table. The Crystal Reports analysis exists in `output/report_summary.md` (36 reports cataloged) and `output/reports/*.md` (36 individual analyses), providing sufficient material without re-analyzing .rpt files. A gap log file and 20+ test queries round out the deliverables.

The primary challenge is **routing ambiguity at domain boundaries** -- several query types overlap between skills (e.g., "top customers" routes to both sales and customers skills, "inventory value" touches both inventory and financial). The CONTEXT.md decision to "ask the user to clarify" on ambiguous queries eliminates the need for a complex disambiguation algorithm. The secondary challenge is **honest coverage assessment** -- mapping each of the 36 Crystal Reports to a coverage status (Replaced/Partially covered/Not covered) against the 6 domain skills.

**Primary recommendation:** Add a CRYSTAL REPORTS CATALOG section to the glossary (~100-150 lines) and expand the NL ROUTING TABLE to cover all domain skills (~40-50 additional entries), keeping the glossary under 700 lines total. Create a separate `skills/control-erp-glossary/gap-log.md` for uncovered domain tracking.

## Standard Stack

This phase involves no external libraries or frameworks. All work is markdown skill file editing.

### Core
| Component | Version | Purpose | Why Standard |
|-----------|---------|---------|--------------|
| Glossary SKILL.md | N/A | Main routing and term reference | Existing file, extend in place |
| gap-log.md | N/A | Gap tracking for uncovered domains | Separate file for easy monitoring |
| output/report_summary.md | N/A | Crystal Reports catalog source | Already created in Phase 2, 36 reports |
| output/reports/*.md | N/A | Individual report analyses | Already created in Phase 2 |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Centralized catalog in glossary | Distributed catalog across domain skills | Centralized is better: reports cross domain boundaries, single lookup is more token-efficient, glossary is already the routing layer |
| Separate gap-log.md file | Gap section in glossary SKILL.md | Separate file is better: easier to check, won't bloat the skill, can be updated without reloading the skill |
| Re-analyzing .rpt files | Using existing report_summary.md | Existing analysis is sufficient: SQL was inferred from filenames/schema, not extracted from binaries (acknowledged in report_summary.md). Re-analysis would yield same results. |

## Architecture Patterns

### Current Glossary Structure (492 lines)

```
control-erp-glossary-SKILL.md
├── YAML frontmatter (3 lines)
├── Introduction / How to use (14 lines)
├── FLS PRODUCT LINE TERMINOLOGY (~200 lines)
│   ├── DyeSub Print Container Products
│   └── Non-DyeSub Product Groups
├── CONTROL TECHNICAL TERMINOLOGY (~190 lines)
│   ├── Core System Components
│   ├── Database Fields and Patterns
│   ├── Account and Customer Concepts
│   ├── Pricing and Product Concepts
│   ├── Production and Workflow Concepts
│   ├── Financial Concepts
│   └── System and Integration Concepts
├── ORDER LIFECYCLE AND WORKFLOW TERMINOLOGY (~50 lines)
├── NATURAL LANGUAGE ROUTING TABLE (~37 entries, ~37 lines)
└── Footer (2 lines)
```

### Target Structure After Phase 13

```
control-erp-glossary-SKILL.md
├── YAML frontmatter (3 lines)
├── Introduction / How to use (14 lines)
├── FLS PRODUCT LINE TERMINOLOGY (~200 lines) [unchanged]
├── CONTROL TECHNICAL TERMINOLOGY (~190 lines) [unchanged]
├── ORDER LIFECYCLE AND WORKFLOW TERMINOLOGY (~50 lines) [unchanged]
├── CRYSTAL REPORTS CATALOG (~100-130 lines) [NEW]
│   ├── How to use this catalog (guidance text)
│   ├── Financial Reports (5 reports)
│   ├── WIP Reports (4 reports)
│   ├── Sales Reports (14 reports)
│   ├── Estimate Reports (4 reports)
│   ├── Inventory & Product Reports (6 reports)
│   └── Other Reports (3 reports)
├── NATURAL LANGUAGE ROUTING TABLE (~80+ entries) [EXPANDED]
│   ├── Product & Sales queries → control-erp-sales
│   ├── Financial queries → control-erp-financial
│   ├── Customer queries → control-erp-customers
│   ├── Inventory queries → control-erp-inventory
│   ├── Production queries → control-erp-production
│   ├── Technical/terminology queries → control-erp-glossary (self)
│   ├── Report discovery queries → Crystal Reports Catalog (self)
│   └── Cross-domain & ambiguous → ask user to clarify
└── Footer (2 lines)

skills/control-erp-glossary/gap-log.md [NEW FILE]
├── Purpose and update instructions
├── Uncovered domains list
└── Example queries that triggered gaps
```

### Pattern 1: Report Catalog Entry Format

Each report entry should include: name, category, purpose, and coverage status. Per CONTEXT.md, "no parameter details" (users already know their reports).

**Format:**
```markdown
| Report | Purpose | Coverage |
|--------|---------|----------|
| FLS Income Statement.rpt | P&L showing revenue and expenses by GL hierarchy | Replaced by control-erp-financial (P&L queries) |
| A_R Detail.rpt | AR aging with invoice-level detail per customer | Replaced by control-erp-financial (AR Aging Buckets) |
| WIP By Line Item Station.rpt | WIP detail by station per line item | Replaced by control-erp-production (LI WIP by Station, 6b) |
| Product Parameters.rpt | Product parameter definitions | Not yet covered |
```

**Coverage status values:**
- `Replaced by [skill] ([section/template])` -- skill fully covers this report's intent
- `Partially covered by [skill]` -- skill covers some but not all of the report's purpose
- `Not yet covered` -- no skill handles this report's data domain

### Pattern 2: NL Routing Entry Format

The existing routing table uses this format:
```markdown
| User Says | Route To | Key Pattern/Section |
|-----------|----------|-------------------|
```

This format works well. Expand by adding entries from each domain skill's NL table to create a comprehensive routing index. Use the existing format verbatim.

### Pattern 3: Gap Log Format

```markdown
# Control ERP Glossary - Gap Log

**Purpose:** Track every query domain that falls outside current skill coverage.
**Update policy:** Add entries whenever a user query cannot be answered by any skill.

## Uncovered Domains

| Domain | Example Queries | Priority | Notes |
|--------|----------------|----------|-------|
| Payroll/TimeCard | "show me timecards", "who worked overtime" | Medium | Tables exist (Payroll, TimeCard) but no skill |
| Commissions | "commission rates", "sales commission report" | Low | Tables exist (CommissionPlan, CommissionRate) |
| Tax compliance | "sales tax collected", "tax by state" | Medium | Tables exist (TaxClass, TaxLink) |
| Service tickets | "open service tickets", "service history" | Low | 1 record in 2025, minimal usage |
```

### Anti-Patterns to Avoid

- **Duplicating query templates in the glossary:** The glossary is a routing layer. It should point to the right skill, not contain SQL templates. All templates live in domain skills.
- **Over-routing (guessing):** Per CONTEXT.md: "CRITICAL: Never guess. If a query falls outside covered domains, say it's not covered." Log the gap instead.
- **Hardcoding report IDs or internal references:** Report entries should use filename and human-readable purpose only. Users identify reports by name.
- **Bloating the glossary beyond 700 lines:** The glossary is loaded on every query. Keep it lean. If the reports catalog exceeds budget, move detailed SQL from reports to a reference file.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Report-to-skill mapping | Manual analysis of each report | Cross-reference report_summary.md against skill NL tables | 36 reports vs 6 skills is a manual but finite task |
| NL routing patterns | New classification system | Aggregate existing NL tables from all skills | Each skill already has a validated NL table |
| Coverage assessment | Automated SQL comparison | Human review of report purpose vs skill queries | Report SQL is inferred, not extracted; manual judgment needed |

## Common Pitfalls

### Pitfall 1: TransactionType Values in Reports are WRONG
**What goes wrong:** The report_summary.md and report_join_patterns.md contain incorrect TransactionType mappings inherited from early analysis (e.g., Type 3=Order, Type 4=Invoice, StatusID 1600/1700). These were corrected in the core skill but NOT in the report analysis files.
**Why it happens:** The report analysis was done before TransactionType validation. The inferred SQL in reports uses wrong type codes.
**How to avoid:** When mapping reports to coverage status, use the CORRECT TransactionType mappings from core skill (Type 1=Order, Type 2=Estimate, etc.), NOT the inferred SQL from report files. The report analysis files are useful for categorization and purpose, but their inferred SQL should not be cited verbatim in the catalog.
**Warning signs:** Any reference to TransactionType=3 (Order), TransactionType=4 (Invoice), StatusID=1600, or StatusID=1700.

### Pitfall 2: "Top Customers" Ambiguity
**What goes wrong:** "Top customers" could reasonably route to control-erp-sales (Template 6: Top Customers by Revenue), control-erp-customers (Customer Ranking section), or control-erp-financial (Revenue by Product Line with customer drill-down).
**Why it happens:** Multiple skills have customer revenue analysis capabilities with overlapping NL triggers.
**How to avoid:** Define clear domain boundaries in the routing table:
- "top customers" / "biggest customers" / "best customers" → control-erp-customers (primary home for customer-centric queries)
- "customer revenue" / "[customer] sales" → control-erp-sales (revenue-focused context)
- "customer AR" / "customer balance" → control-erp-financial (financial context)
Per CONTEXT.md: current glossary routes "top customers" to control-erp-sales. The customer skill also handles this. Decision: Route to control-erp-customers (it has the more comprehensive ranking section with YoY, Pareto, segmentation).

### Pitfall 3: Report Metadata Quality Caveat
**What goes wrong:** Users may expect exact parameter lists or SQL from the report catalog. The report analysis was inferred from filenames, not extracted from binary .rpt files.
**Why it happens:** Crystal Reports uses proprietary encrypted OLE format; SQL cannot be extracted without the SDK.
**How to avoid:** Include a clear disclaimer in the catalog header: "Report details were inferred from filenames and database schema analysis. Actual report SQL and parameters may differ." Per CONTEXT.md: "Report entries show name and purpose only -- no parameter details."

### Pitfall 4: Stale Routing After Skill Updates
**What goes wrong:** A domain skill adds new NL patterns but the glossary routing table isn't updated, causing queries to miss the right skill.
**Why it happens:** Routing table is manually maintained.
**How to avoid:** Include a "Last synchronized" date in the routing table header, and a note that the routing table should be updated whenever a domain skill's NL patterns change.

### Pitfall 5: Gap Log Becomes a Dumping Ground
**What goes wrong:** The gap log fills up with one-off queries that don't represent real business needs, making it noisy and hard to prioritize.
**Why it happens:** No criteria for what constitutes a "gap" vs a "one-off curiosity."
**How to avoid:** Only log gaps that represent recurring business needs or domains with known tables. Include a "Priority" column (High/Medium/Low) based on whether the data exists in the database and whether users are likely to ask again.

## Code Examples

### Example 1: Crystal Reports Catalog Section

```markdown
## CRYSTAL REPORTS CATALOG

Crystal Reports are the legacy reporting engine at FLS Banners, running for 15+ years with battle-tested SQL. Skills are their modern replacement. When a report is "Replaced by" a skill, use the skill's query instead of running the Crystal Report. When a report is "Not yet covered", the Crystal Report is the only option -- run it in Control.

**LLM answers first.** Only recommend running the Crystal Report in Control when:
- The skill doesn't cover that report's data domain
- The report has complex formatting (invoices, work orders with proof images) that a query can't reproduce
- The user specifically asks to run a Crystal Report

> Report details were inferred from filenames and database schema analysis, not extracted from binary .rpt files. Actual report definitions may differ.

### Financial Reports

| Report | Purpose | Coverage |
|--------|---------|----------|
| FLS Income Statement.rpt | P&L: revenue, COGS, expenses by GL hierarchy | Replaced by control-erp-financial (Full P&L, P&L Summary) |
| FLS Balance Sheet.rpt | Assets, liabilities, equity snapshot | Replaced by control-erp-financial (Balance Sheet Key Accounts) |
| A_R Detail.rpt | AR aging with invoice-level detail per customer | Replaced by control-erp-financial (AR Aging Buckets, AR Snapshot) |
| AR Report - Summary.rpt | AR summary totals by customer | Replaced by control-erp-financial (AR by Payment Terms) |
| A_P Aging Detail.rpt | AP aging with vendor invoice detail | Replaced by control-erp-financial (AP Aging, AP Snapshot) |
```

### Example 2: Expanded Routing Table Entries (New Domains)

```markdown
| "check stock for X" / "what's in stock" | control-erp-inventory | Stock Level Check |
| "what needs reordering" / "low stock" | control-erp-inventory | Reorder Alert |
| "what does X cost" / "price of X" | control-erp-inventory | Last Price Paid (PO history) |
| "top vendors" / "who do we buy from" | control-erp-inventory | Top Vendors by PO |
| "artwork pipeline" / "pending approvals" | control-erp-production | Artwork Pipeline Summary |
| "what's in production" / "WIP" | control-erp-production | Order-Level WIP by Station |
| "bottlenecks" / "which stations are backed up" | control-erp-production | Bottleneck Detection |
| "dwell time" / "how long at [station]" | control-erp-production | Average Dwell Time |
| "customer profile for [X]" / "look up [customer]" | control-erp-customers | Customer Search + Profile |
| "customer segmentation" / "RFM analysis" | control-erp-customers | RFM Segmentation |
| "at-risk customers" / "churn risk" | control-erp-customers | At-Risk Detection |
```

### Example 3: Report Discovery Routing Entries

```markdown
| "is there a report for X" / "what report shows X" | control-erp-glossary | Crystal Reports Catalog |
| "AR aging report" / "AR report" | control-erp-glossary | Crystal Reports Catalog → then route to skill |
| "sales by product report" | control-erp-glossary | Crystal Reports Catalog → then route to skill |
| "inventory report" / "parts listing report" | control-erp-glossary | Crystal Reports Catalog → then route to skill |
| "work order report" / "WIP report" | control-erp-glossary | Crystal Reports Catalog → then route to skill |
```

### Example 4: Test Query Set (20+ queries for routing validation)

```markdown
## Test Queries for Routing Validation

| # | Query (as user would type) | Expected Route | Category |
|---|----------------------------|----------------|----------|
| 1 | "whats our revenue this year" | control-erp-sales (Template 1) | Sales |
| 2 | "show me feather flag sales" | control-erp-sales (Template 3) | Sales/Product |
| 3 | "whos our biggest customer" | control-erp-customers (Top Customers) | Customer |
| 4 | "AR aging" | control-erp-financial (AR Aging Buckets) | Financial |
| 5 | "what do we owe vendors" | control-erp-financial (AP Snapshot) | Financial |
| 6 | "whats in stock for FR 66 knit" | control-erp-inventory (Stock Check) | Inventory |
| 7 | "what needs reordering" | control-erp-inventory (Reorder Alert) | Inventory |
| 8 | "pending artwork approvals" | control-erp-production (Artwork Pipeline) | Production |
| 9 | "is there a report for sales by product" | control-erp-glossary (Reports Catalog) | Reports |
| 10 | "whats a ClassTypeID" | control-erp-glossary (Technical Terms) | Terminology |
| 11 | "P&L this quarter" | control-erp-financial (P&L Summary) | Financial |
| 12 | "top 10 customers by revenue" | control-erp-customers (Top Customers) | Customer |
| 13 | "how long does artwork approval take" | control-erp-production (Turnaround) | Production |
| 14 | "customer profile for FLASH" | control-erp-customers (Customer Profile) | Customer |
| 15 | "gross margin" | control-erp-financial (P&L Summary) | Financial |
| 16 | "swing flag orders this month" | control-erp-sales (Template 3) | Sales/Product |
| 17 | "which stations are backed up" | control-erp-production (Bottleneck) | Production |
| 18 | "last price paid for polyester fabric" | control-erp-inventory (Last Price Paid) | Inventory |
| 19 | "sales by rep" | control-erp-sales (Template 7) | Sales |
| 20 | "what report shows inventory levels" | control-erp-glossary (Reports Catalog) | Reports |
| 21 | "dormant customers" | control-erp-customers (At-Risk Detection) | Customer |
| 22 | "cash position" | control-erp-financial (Balance Sheet) | Financial |
| 23 | "how much do we use of FR 66 knit per month" | control-erp-inventory (Consumption) | Inventory |
| 24 | "compare this year to last year" | control-erp-sales (Template 9) | Sales |
| 25 | "commission rates for sales team" | NOT COVERED (gap-log) | Gap |
```

## State of the Art

This phase is pure content authoring (markdown skill files), not software development. No "state of the art" technology considerations apply. The "art" is in routing accuracy and coverage assessment quality.

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Run Crystal Report in Control | Ask LLM, get immediate query results | Phase 13 makes this explicit | Users get answers faster; Crystal Reports become fallback only |
| Glossary covers sales + finance only | Glossary routes to ALL 6 domain skills | Phase 13 | Complete question coverage |

## Open Questions

### 1. Routing Overlap: "Top Customers"
- **What we know:** Both control-erp-sales (Template 6) and control-erp-customers (Customer Ranking section) handle "top customers." The customer skill has the richer implementation (YoY, Pareto, segmentation).
- **What's unclear:** Should the glossary routing table duplicate entries pointing to both skills, or pick one?
- **Recommendation:** Route to control-erp-customers as primary (it has more comprehensive analysis). Add a note that control-erp-sales also has a simpler top-customer template. Update the existing glossary entry (currently routes to control-erp-sales Template 6) to route to control-erp-customers instead.

### 2. Report Coverage for "Work Order with Proof"
- **What we know:** The Work Order with Proof report generates a formatted work order printout with embedded proof images. No skill can reproduce this.
- **What's unclear:** Is this a "Not yet covered" or a permanent "Run in Control"?
- **Recommendation:** Mark as "Not yet covered -- requires formatted output with embedded images (use Crystal Report in Control)". This acknowledges the gap while explaining WHY it can't be replaced by a query.

### 3. How Many Reports are "Not Yet Covered"?
- **Preliminary assessment based on research:**
  - **Fully Replaced (18-20):** Income Statement, Balance Sheet, AR Detail, AR Summary, AP Aging, WIP Summary, WIP by Line Item Station, WIP by Machine, WIP by Calendar Month, Sales by Product, Sales by Customer (Volume/Frequency), Sales by Salesperson, Sales by Primary Salesperson, Sales Report (general), Yearly Sales by Month, Yearly Sales by Salesperson, Orders Placed Between, Sales Graph
  - **Partially Covered (6-8):** Sales by Product Category (skill uses variable-based grouping, report uses Product entity), Sales by Sales Account, Based On Sales by Price, Based on Orders Placed by Price, Converted Estimates, Pending Estimates, Est vs Act Cost Summary
  - **Not Yet Covered (8-10):** All Parts Listing, Catalog Listing, Inventory Listing, Inventory History, Product Detail, Product Parameters, Last Contact Aging, Shipped By Date, Work Order with Proof, Last Order Aging
- **Confidence: MEDIUM** -- exact coverage status requires judgment call per report

### 4. Inventory Reports in Gap vs Skill
- **What we know:** control-erp-inventory handles stock levels, reorder monitoring, and PO history. But the "All Parts Listing", "Catalog Listing", and "Inventory Listing" Crystal Reports may not be fully equivalent.
- **What's unclear:** Does the inventory skill's stock check query fully replace "Inventory Listing"?
- **Recommendation:** Mark as "Partially covered" -- the skill handles individual part lookups and broad overviews but may not match the exact Crystal Report format. Note this in the catalog.

## Coverage Assessment: 36 Crystal Reports vs 6 Domain Skills

### DETAILED MAPPING (the core deliverable for the planner)

#### Financial Reports (5 reports)

| # | Report | Skill Coverage | Status |
|---|--------|---------------|--------|
| 1 | FLS Income Statement.rpt | control-erp-financial: Full P&L, P&L Summary, Revenue by Product Line | **Replaced** |
| 2 | FLS Balance Sheet.rpt | control-erp-financial: Balance Sheet Key Accounts | **Replaced** |
| 3 | A_R Detail.rpt | control-erp-financial: AR Snapshot, AR Aging Buckets, AR by Payment Terms + references/financial-analysis.md | **Replaced** |
| 4 | AR Report - Summary.rpt | control-erp-financial: AR Aging Buckets (summary view) | **Replaced** |
| 5 | A_P Aging Detail.rpt | control-erp-financial: AP Snapshot, AP Aging | **Replaced** |

#### WIP Reports (4 reports)

| # | Report | Skill Coverage | Status |
|---|--------|---------------|--------|
| 6 | WIP Summary - Order Level.rpt | control-erp-production: Order-Level WIP by Station (6a) | **Replaced** |
| 7 | WIP By Line Item Station.rpt | control-erp-production: LI-Level WIP by Station (6b) | **Replaced** |
| 8 | WIP By Machine.rpt | control-erp-production: Order-Level WIP by Station (6a) grouped by station | **Replaced** |
| 9 | WIP By Calendar Month Due.rpt | control-erp-production: Stage-Level Summary (6d) can be modified with DueDate grouping | **Partially covered** |

#### Sales Reports (14 reports)

| # | Report | Skill Coverage | Status |
|---|--------|---------------|--------|
| 10 | Orders Placed Between.rpt | control-erp-sales: Template 1 (Total Revenue for Period) | **Replaced** |
| 11 | Sales - By Product.rpt | control-erp-sales: Template 3 (DyeSub Category) + Template 8 (Full Reconciliation) | **Replaced** |
| 12 | Sales - By Product Category.rpt | control-erp-sales: Template 3 (FP_ProductDescription) | **Partially covered** (skill uses variable-based grouping, report uses Product entity/SelectionList) |
| 13 | Sales - By Customer (Volume).rpt | control-erp-customers: Top Customers by Revenue | **Replaced** |
| 14 | Sales - By Customer (Frequency).rpt | control-erp-customers: Top Customers by Order Count | **Replaced** |
| 15 | Sales - By Sales Account.rpt | control-erp-sales: Template 7 (with Account-level grouping) | **Partially covered** (sales account concept differs from salesperson) |
| 16 | Sales - By All Salespeople.rpt | control-erp-sales: Template 7 (Revenue by Salesperson) | **Partially covered** (skill uses SalesPerson1ID only; report covers all 3 slots) |
| 17 | Sales - By Primary Saleperson.rpt | control-erp-sales: Template 7 (Revenue by Salesperson) | **Replaced** |
| 18 | Based On Sales by Price.rpt | control-erp-sales: can be constructed from Template 1 with ORDER BY | **Partially covered** |
| 19 | Based on Orders Placed by Price.rpt | control-erp-sales: can be constructed similarly | **Partially covered** |
| 20 | Sales Graph.rpt | control-erp-sales: Template 2 (Monthly Revenue Trend) | **Replaced** (data equivalent, not visual graph) |
| 21 | Sales Report.rpt | control-erp-sales: Template 1 + Template 8 combined | **Replaced** |
| 22 | Sales - Yearly Sales By Calendar Month.rpt | control-erp-sales: Template 9 (Year-over-Year Comparison) | **Replaced** |
| 23 | Sales - Yearly Sales By Salesperson.rpt | control-erp-sales: Template 7 with year grouping | **Partially covered** |

#### Estimate Reports (4 reports)

| # | Report | Skill Coverage | Status |
|---|--------|---------------|--------|
| 24 | Converted Estimates.rpt | control-erp-core: Estimate -> Order Linkage section describes the pattern | **Partially covered** (pattern documented, no ready-made template) |
| 25 | Pending Estimates.rpt | control-erp-core: TransactionType 2, StatusID 11 filtering | **Partially covered** (filter documented, no ready-made template) |
| 26 | Last Order Aging.rpt | control-erp-customers: Simple Dormancy query | **Replaced** |
| 27 | Est. vs Act. Cost Summary By Order.rpt | No skill covers estimated vs actual cost comparison | **Not yet covered** |

#### Inventory & Product Reports (6 reports)

| # | Report | Skill Coverage | Status |
|---|--------|---------------|--------|
| 28 | All Parts Listing.rpt | control-erp-inventory: Part Detail Card (individual) + broad overview | **Partially covered** (no full catalog dump template) |
| 29 | Catalog Listing.rpt | control-erp-inventory: CatalogItem queries via PO history | **Partially covered** |
| 30 | Inventory Listing.rpt | control-erp-inventory: Stock Level Check + broad overview | **Partially covered** |
| 31 | Inventory History.rpt | control-erp-inventory: Usage History for a Part | **Partially covered** (PartUsageCard vs InventoryLog) |
| 32 | Product Detail.rpt | No skill covers full product configuration | **Not yet covered** |
| 33 | Product Parameters.rpt | No skill covers product parameter definitions | **Not yet covered** |

#### Other Reports (3 reports)

| # | Report | Skill Coverage | Status |
|---|--------|---------------|--------|
| 34 | Last Contact Aging.rpt | control-erp-customers: Last Contact Activity (per-customer) | **Partially covered** (individual customer only, no company-wide aging) |
| 35 | Shipped By Date.rpt | No skill covers shipment tracking | **Not yet covered** |
| 36 | Work Order with Proof.rpt | No skill (requires formatted output with embedded proof images) | **Not yet covered** (run in Control) |

### Coverage Summary

| Status | Count | Percentage |
|--------|-------|-----------|
| Replaced | 17 | 47% |
| Partially covered | 14 | 39% |
| Not yet covered | 5 | 14% |
| **Total** | **36** | **100%** |

## Domain Boundary Map (for routing table)

### Existing NL Routes in Each Skill

| Skill | NL Entries | Primary Domain | Overlap Areas |
|-------|-----------|----------------|---------------|
| control-erp-core | 13 | Transaction types, status codes, query formula | Revenue queries (also in sales) |
| control-erp-sales | 19 | Revenue, products, DyeSub categories, reps | "Top customers" (also in customers), "Open AR" (also in financial) |
| control-erp-financial | 26 | GL, AR, AP, P&L, payments, cash flow | "Revenue by product" (also in sales), AR (also in sales) |
| control-erp-customers | 30 | Lookup, profile, ranking, segmentation, churn | "Top customers" (also in sales), "AR for customer" (also in financial) |
| control-erp-inventory | 18 | Stock, reorder, PO history, consumption | "Inventory value" (also in financial) |
| control-erp-production | 25 | Artwork, stations, WIP, dwell time, bottlenecks | "WIP" (also in sales as status filter) |

### Current Glossary Routing (37 entries)

The existing glossary routes to 4 destinations:
- control-erp-sales: 16 entries
- control-erp-financial: 12 entries
- control-erp-core: 4 entries
- control-erp-glossary: 5 entries (terminology lookups)

**Missing from current routing:**
- control-erp-customers: 0 entries (all customer queries currently route to sales)
- control-erp-inventory: 0 entries
- control-erp-production: 0 entries
- Report discovery queries: 0 entries

### Recommended Domain Boundaries

| Query Intent | Primary Skill | Rationale |
|-------------|---------------|-----------|
| Revenue, sales trends, product performance | control-erp-sales | Revenue is the core focus |
| GL entries, P&L, AR aging, AP aging, payments, cash flow | control-erp-financial | Accounting/financial focus |
| Customer lookup, profile, ranking, segmentation, churn | control-erp-customers | Customer-centric focus |
| Stock levels, reorder, PO history, parts, consumption | control-erp-inventory | Materials/purchasing focus |
| Artwork pipeline, station workload, dwell time, WIP detail | control-erp-production | Production workflow focus |
| Term definitions, report discovery, routing | control-erp-glossary | Routing layer |
| Transaction types, query rules, business rules | control-erp-core | Foundation reference |

### Overlap Resolution Rules

| Ambiguous Query | Resolution |
|----------------|------------|
| "top customers" | control-erp-customers (comprehensive ranking + Pareto + segmentation) |
| "customer revenue" / "[customer] sales" | control-erp-sales (revenue context) |
| "AR for [customer]" / "what does [customer] owe" | control-erp-financial OR control-erp-customers (both handle it; route to customers for profile context, financial for aging detail) |
| "inventory value" | control-erp-financial (GL-based, authoritative) with note that control-erp-inventory has part-level detail |
| "revenue by product" | control-erp-sales (TransHeader/TransDetail based) vs control-erp-financial (GL-based). Default: sales. Clarify if user wants GL perspective. |
| "WIP" (general) | control-erp-production (station-level detail) vs control-erp-sales (StatusID=1 filter). Default: production. |
| "orders placed" vs "sales" | control-erp-sales for both. "Orders placed" = OrderCreatedDate. "Sales" = SaleDate. |

## Existing Material Inventory

### Files to Read (Input)
- `output/report_summary.md` -- 36 reports cataloged with categories, purpose, primary tables
- `output/reports/*.md` -- 36 individual report analyses with inferred SQL and joins
- `output/report_join_patterns.md` -- all join patterns from Crystal Reports
- Each skill's NL routing table (6 skills)

### Files to Write (Output)
- `skills/control-erp-glossary/control-erp-glossary-SKILL.md` -- updated with reports catalog + expanded routing
- `skills/control-erp-glossary/gap-log.md` -- new file tracking uncovered domains

### Files NOT to Modify
- All other skill files remain unchanged
- Report analysis files in output/reports/ remain unchanged

## Sources

### Primary (HIGH confidence)
- `skills/control-erp-glossary/control-erp-glossary-SKILL.md` -- current glossary, 492 lines, read in full
- `skills/control-erp-core/control-erp-core-SKILL.md` -- core business rules, 348 lines, read in full
- `skills/control-erp-sales/control-erp-sales-SKILL.md` -- sales skill, 371 lines, read in full
- `skills/control-erp-financial/control-erp-financial-SKILL.md` -- financial skill, 889 lines, read in full
- `skills/control-erp-customers/control-erp-customers-SKILL.md` -- customer skill, 1076 lines, read in full
- `skills/control-erp-inventory/control-erp-inventory-SKILL.md` -- inventory skill, 663 lines, read in full
- `skills/control-erp-production/control-erp-production-SKILL.md` -- production skill, 1096 lines, read in full
- `output/report_summary.md` -- Crystal Reports catalog, 149 lines, read in full
- `output/report_join_patterns.md` -- join patterns, 630 lines, read in full
- `output/reports/FLS_Income_Statement.md` -- sample report analysis, read in full
- `output/reports/A_R_Detail.md` -- sample report analysis, read in full
- `.planning/phases/13-glossary-integration/13-CONTEXT.md` -- user decisions, read in full

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all inputs are existing files already read; no external dependencies
- Architecture: HIGH -- glossary structure is established; changes are additive
- Pitfalls: HIGH -- domain boundaries and routing overlaps identified from actual skill content
- Coverage assessment: MEDIUM -- report-to-skill mapping requires judgment on partial coverage
- Test queries: HIGH -- derived from actual NL tables in existing skills

**Research date:** 2026-02-09
**Valid until:** Indefinite (no external dependencies that could change)
