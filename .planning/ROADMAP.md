# Roadmap: Project Cornerstone

## Milestones

- v1.0 Foundation + Sales (shipped 2026-02-09)
- v1.1 Read-Layer Build-Out (in progress)

## Phases

<details>
<summary>v1.0 Foundation + Sales (Phases 1-8) - SHIPPED 2026-02-09</summary>

See: .planning/milestones/v1.0-ROADMAP.md

8 phases, 15 plans, 35/35 requirements verified.

</details>

### v1.1 Read-Layer Build-Out (In Progress)

**Milestone Goal:** Expand the natural language interface from sales/financial foundation to all major business domains -- customer intelligence, inventory management, production workflow -- and integrate a Crystal Reports routing catalog. Every new skill validates against Control's built-in reports before shipping.

**Phase Numbering:** Integer phases (9, 10, 11, 12, 13). Decimal phases (e.g., 10.1) reserved for urgent insertions.

- [x] **Phase 9: Financial Depth** - Extend financial skill with AR aging, AP tracking, P&L breakdown, and cash flow
- [x] **Phase 10: Customer Intelligence** - New skill for customer lookup, segmentation, CLV, and churn detection
- [ ] **Phase 11: Inventory Management** - New skill for stock levels, reorder monitoring, and warehouse planning
- [ ] **Phase 12: Production Workflow** - New skill for artwork pipeline, station workload, and labor analysis
- [ ] **Phase 13: Glossary Integration + Reports Catalog** - Extend glossary with Crystal Reports catalog and routing for all new domains

## Parallelism

```
          +---> Phase 9 (Financial) ---+
          |                            |
START ----+                            +---> Phase 11 (Inventory) ---> Phase 12 (Production) ---> Phase 13 (Glossary)
          |                            |
          +---> Phase 10 (Customer) ---+
```

- **Wave 1** (parallel): Phases 9, 10 -- zero dependencies between Financial and Customer
- **Wave 2**: Phase 11 -- cross-references Financial GL cost accounts for inventory valuation
- **Wave 3**: Phase 12 -- soft dependency on Inventory for part usage context
- **Wave 4**: Phase 13 -- routes to ALL domain skills, must come last

## Phase Details

### Phase 9: Financial Depth

**Goal**: Users can answer AR/AP, P&L, and cash flow questions through natural language against the Control ERP -- extending the validated financial skill with analytical depth
**Depends on**: v1.0 Phase 5 (financial foundation already shipped)
**Requirements**: FIN-04, FIN-05, FIN-06, FIN-07
**Success Criteria** (what must be TRUE):
  1. User can ask "show me AR aging" and get current/30/60/90+ day buckets with customer breakdown that matches Control's A_R Detail report
  2. User can ask about vendor balances, outstanding payables, and payment history and get accurate AP data
  3. User can ask for P&L with product-line breakdown (revenue, COGS, gross margin by category) for any date range
  4. User can ask "what's our bank balance" or "show me cash flow" and get current balances plus cash in/out summary
  5. All financial queries use GL view (not Ledger table), correct sign conventions (SUM(-Amount) for revenue), and SaleDate for AR aging (not DueDate)
**Plans**: 2 plans

Plans:
- [x] 09-01-PLAN.md -- AR detail with customer breakdown + AP detail with vendor breakdown
- [x] 09-02-PLAN.md -- P&L analysis with comparison periods + cash flow and bank balance queries

**Critical pitfalls:**
- GL sign convention inversion: Revenue stored as negative credits. Must use SUM(-Amount) for GLClassificationType IN (4000, 4001).
- DueDate semantic overloading: AR aging MUST use SaleDate (invoice date), NOT DueDate (production deadline).
- GL view vs Ledger table: Must use GL view (excludes ~307K off-balance-sheet entries), not Ledger directly.

**Research flag:** Standard patterns. No phase-level research needed -- P&L, AR, AP, cash flow all use validated GL/Ledger patterns from v1.0.

### Phase 10: Customer Intelligence

**Goal**: Users can look up any customer, understand their value and behavior, and identify at-risk accounts -- through a new control-erp-customers skill
**Depends on**: v1.0 Phase 1 (core skill for TransactionType/StatusID/pricing rules)
**Requirements**: CUST-01, CUST-02, CUST-03
**Success Criteria** (what must be TRUE):
  1. User can search for a customer by name, number, or contact and get a complete profile (contacts, addresses, phone numbers, account flags, order history, revenue, AR balance)
  2. User can ask "who are our top customers" or "show customer segmentation" and get revenue ranking, order frequency, average order value, and lifetime value -- with results that cross-check against the validated sales skill top-10 output
  3. User can ask "which customers are at risk" or "who haven't ordered in 6 months" and get dormant customer detection with last order date and revenue at risk
  4. Customer revenue totals are internally consistent with core revenue baseline ($3,052,952.52)
**Plans**: 2 plans

Plans:
- [ ] 10-01-PLAN.md -- Create customer skill with profile lookup, smart search, and drill-down queries (CUST-01)
- [x] 10-02-PLAN.md -- Add ranking, RFM segmentation, and churn detection with personalized dormancy (CUST-02, CUST-03)

**Critical pitfalls:**
- Account table uses CompanyName, NOT AccountName.
- Must filter IsClient=1, IsActive=1 by default (Account has IsClient, IsProspect, IsActive flags).
- Must use core skill's TransactionType=1, SubTotalPrice, SaleDate patterns -- no independent implementations.

**Research flag:** Standard patterns. Account table fully documented, CRM patterns in wiki.

### Phase 11: Inventory Management

**Goal**: Users can check stock levels, identify reorder needs, and plan material usage -- through a new control-erp-inventory skill
**Depends on**: Phase 9 (cross-references Financial GL cost accounts for inventory valuation)
**Requirements**: INV-01, INV-02, INV-03
**Success Criteria** (what must be TRUE):
  1. User can ask "what's in stock" or "check inventory for [part]" and get current quantities (QuantityAvailable, not QuantityOnHand) with the distinction clearly explained when relevant
  2. User can ask "what needs reordering" and get parts below their reorder threshold with suggested quantities -- matching Control's Inventory Listing report
  3. User can ask about warehouse inventory, material consumption rates, and usage history -- with inventory valuation that approximates GL NodeID 10414 balance
  4. All inventory queries use the Inventory table (not Part table alone) with ClassTypeID 12200

**Critical pitfalls:**
- QuantityAvailable vs QuantityOnHand: OnHand - Reserved = Available. "Available" is what matters for new orders.
- Must query Inventory table with ClassTypeID 12200, not Part table quantities.
- FLS warehouse configuration needs live database discovery before writing query templates.

**Research flag:** HIGH -- FLS-specific warehouse configuration (number of warehouses, types, part-to-warehouse assignments) requires live discovery queries during planning.
**Plans**: 2 plans

Plans:
- [x] 11-01-PLAN.md -- Create inventory skill with table architecture, stock level queries (INV-01), and reorder monitoring (INV-02)
- [ ] 11-02-PLAN.md -- Add purchasing intelligence, material consumption, inventory valuation (INV-03), and NL routing

### Phase 12: Production Workflow

**Goal**: Users can track artwork approvals, monitor station workload, and analyze labor time -- through a new control-erp-production skill
**Depends on**: Phase 11 (soft -- part usage cards reference Part table)
**Requirements**: PROD-01, PROD-02, PROD-03
**Success Criteria** (what must be TRUE):
  1. User can ask "what artwork is pending approval" or "show me the artwork pipeline" and get accurate status counts, stuck items, and turnaround times that match Control's artwork workflow
  2. User can ask "what's the workload at [station]" or "which stations are backed up" and get WIP counts per station with bottleneck identification -- WIP count matching TransHeader WHERE StatusID IN (1,2)
  3. User can ask "how many hours did [employee] work" or "show labor by station" and get accurate time tracking from TimeCard tables without double-counting parent/detail records
  4. TimeCard queries use ClassTypeID 20050 for total hours and 20051 for station-level detail, never mixing the two

**Critical pitfalls:**
- TimeCard ClassTypeID double-counting: ClassTypeID 20050 = parent (clock in/out), 20051 = station detail. Mixing them doubles hours. This is the production domain's equivalent of the $1.3M TransDetailParam bug.
- Geography columns in Station table crash SELECT * queries -- must specify columns explicitly.
- Journal table is multi-purpose (5.18M rows) -- must filter JournalActivityType=45 for station-related entries.
- FLS station hierarchy and artwork status usage need live database discovery.

**Research flag:** HIGH -- FLS-specific station configuration (98 stations, department hierarchy, which statuses actively used) requires live discovery queries during planning.

### Phase 13: Glossary Integration + Reports Catalog

**Goal**: Users asking any business question get routed to the correct skill, and users asking about reports get guidance on which Crystal Report to run vs. when to use a custom query
**Depends on**: Phases 9, 10, 11, 12 (routes to ALL domain skills -- must come last)
**Requirements**: RPT-01
**Success Criteria** (what must be TRUE):
  1. User can ask "what report shows AR aging" or "is there a report for inventory" and get the specific Crystal Report name, its parameters, and guidance on when to use the report vs. a custom query
  2. Glossary routes inventory, production, customer, and financial-depth natural language queries to the correct domain skill with 90%+ accuracy on 20 test queries
  3. All 36 cataloged Crystal Reports are documented with purpose, category, and routing guidance in the glossary skill

**Critical pitfalls:**
- Report metadata is inferred from filenames (Crystal Reports binary format), not extracted from internal SQL. Acknowledge inference quality.
- NL routing ambiguity: "top customers" could route to sales, financial, or customers. Clear domain boundaries must be defined.

**Research flag:** Standard patterns. Static catalog from report_summary.md, straightforward routing table extension.

## Progress

**Execution Order:**
Phases execute in numeric order with parallelization: 9+10 (parallel) -> 11 -> 12 -> 13

| Phase | Milestone | Plans Complete | Status | Completed |
|-------|-----------|----------------|--------|-----------|
| 9. Financial Depth | v1.1 | 2/2 | Complete | 2026-02-09 |
| 10. Customer Intelligence | v1.1 | 2/2 | Complete | 2026-02-09 |
| 11. Inventory Management | v1.1 | 1/2 | In progress | - |
| 12. Production Workflow | v1.1 | 0/TBD | Not started | - |
| 13. Glossary Integration | v1.1 | 0/TBD | Not started | - |

---
*Roadmap created: 2026-02-09*
*Last updated: 2026-02-09 after Phase 11 Plan 01 completion*
