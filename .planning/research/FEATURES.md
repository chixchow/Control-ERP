# Feature Landscape: 5 New Domain Skills for Control ERP NL Interface

**Domain:** Natural language query interface for Cyrious Control ERP (sign/banner industry)
**Researched:** 2026-02-09
**Business context:** FLS Banners, ~$3M revenue, custom flag/banner/signage manufacturer
**Existing system:** Core skill, Sales skill, Financial foundation, Glossary, Wiki knowledge base

---

## How to Read This Document

Each domain section has four categories:

- **Table Stakes** -- Features users expect from a domain skill. Missing = feels broken.
- **Differentiators** -- Features that make the NL interface notably better than manual SQL. Not expected, but high-value.
- **Anti-Features** -- Things to deliberately NOT build, with rationale.
- **Dependencies** -- What existing skill content can be reused or must be extended.

Complexity ratings: **Low** (single table, simple query), **Med** (2-3 table joins, business logic), **High** (complex joins, multiple query patterns, edge cases).

---

## 1. FINANCIAL DOMAIN

The financial skill already has a strong foundation (AR/AP snapshots, P&L, GL architecture, payment posting patterns). This domain expansion deepens existing capabilities into areas users expect for day-to-day financial management.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| AR aging by customer | Every business needs to know who owes what and how old it is | Med | Already built in financial skill as aggregate buckets. Need per-customer drill-down with contact info and payment terms. Join: TransHeader + Account + PaymentTerms |
| AP aging by vendor | Must know what we owe and when it is due | Med | Already built as aggregate. Need vendor-level drill-down with DueDate-based aging (not SaleDate). TransHeader WHERE TransactionType = 8, StatusID = 6 |
| P&L for any date range | "How did we do last quarter?" is a day-one question | Med | Already built in financial skill. Ensure month/quarter/year shortcuts work. GL WHERE GLClassificationType IN (4000,4001,5001,5002) |
| Cash position | "How much cash do we have?" -- immediate need | Low | Already built. Sum GL for NodeIDs 90 (checking) and 10412 (MM). Need to include undeposited funds (91,92,93,543,10137,10528,10530,10531) as a separate line |
| Payment history for an order | "How was order #133xxx paid?" -- routine customer service | Med | Already addressed in financial skill via GL register. Need clean formatting: date, method, amount, check/CC last 4 |
| GL entries for an order | "Show me the GL for order #133xxx" | Med | Already addressed. Ledger WHERE TransHeaderID = (SELECT ID FROM TransHeader WHERE OrderNumber = X) |
| Revenue by product line (GL) | "What is our DyeSub revenue this month?" from GL perspective | Med | Already built. SUM(-Amount) WHERE GLClassificationType = 4000 grouped by AccountName |
| WIP balance | "What is in WIP right now?" -- production/finance crossover | Med | GL WHERE GLAccountID = 11, SUM(Amount) grouped by TransactionID. Already documented in financial skill |
| Expense breakdown | "What are our biggest expenses?" | Med | Already built. Full P&L filtered to GLClassificationType = 5002 |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Cash flow summary | "Cash in vs cash out this month" -- payments received vs bills paid, in a single view. No built-in Control report for this | High | Combine: (1) SUM payments received (Journal WHERE ClassTypeID = 20001) with (2) SUM bills paid (Journal WHERE ClassTypeID = 20009) by date range. Must handle undeposited vs deposited distinction |
| Historical AR as of a date | "What was our AR on Dec 31?" -- needed for period-end reporting. Control has a Crystal Report for this but it requires manual parameter entry | High | GL WHERE GLAccountID = 14 AND EntryDateTime <= @AsOfDate, grouped by TransactionID. Wiki provides exact SQL pattern. Already validated |
| GL integrity checks | "Is our GL in balance?" -- proactive health monitoring. Wiki provides 7+ integrity check queries that currently require manual SQL execution | High | Master GL Balance Integrity (AR, WIP, Built, AP, Customer Credit, Vendor Credit, Deposits). Wiki has exact SQL. This replaces running complex queries manually |
| Month-over-month expense trend | "Are our expenses going up?" -- trend detection that requires manual spreadsheet work today | Med | P&L query with YEAR/MONTH grouping, compare current period to prior periods. Derive from existing P&L template |
| Undeposited payments | "What payments haven't been deposited yet?" -- daily operations question | Low | Payment WHERE Undeposited = 1 AND IsActive = 1. Already documented in financial skill. Format: amount, method, order, date |
| Gross margin by time period | "What was our gross margin last quarter vs this quarter?" | Med | P&L Summary with two date ranges. Already have single-period template; need dual-period comparison wrapper |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Bank reconciliation | Read-only NL interface cannot mark entries as reconciled. Reconciliation requires write access and human judgment matching bank statements to GL entries | Show reconciliation status (Ledger.Reconciled flag) but do not attempt to perform reconciliation. Route to "use Control's reconciliation tool" |
| GL entry creation/modification | Writing to the Ledger requires CHAPI/SQLBridge and full transaction integrity. One wrong GL entry can cascade through the entire chart of accounts | Read-only queries only. Never generate INSERT/UPDATE statements for financial tables |
| Tax filing preparation | Tax calculations involve Avalara integration, state-specific rules, exemptions. Tax compliance is too high-stakes for an NL interface | Show tax amounts from TransTax and GL (GLClassificationType = 2005) for review, but never claim compliance readiness |
| Payroll queries | Payroll data is in Journal/Payment with ClassTypeID 35250/35300/35301. Payroll is sensitive, regulated, and the data model is complex (tax tables, deductions, comp codes). Separate domain entirely | Defer payroll to a future dedicated skill. Do not mix payroll with general financial queries |
| Budget vs actual | Control has no budget table. Would require external data import | Acknowledge limitation. Suggest: "Control does not store budgets. Compare to prior year as a proxy" |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Financial skill: GL architecture | **Reuse fully** | GL view vs Ledger, sign conventions, NodeID reference all stay |
| Financial skill: AR/AP snapshots | **Extend** | Existing aggregate queries need per-customer/vendor drill-down |
| Financial skill: P&L templates | **Extend** | Add period comparison, trend analysis wrappers |
| Financial skill: Payment posting | **Reuse fully** | TenderType, ClassTypeID, deposit workflow all documented |
| Core skill: StatusID, TransactionType | **Reuse** | All business rules stay in core |
| Wiki: GL integrity SQL | **New import** | 7+ integrity check queries from wiki extracts need to be encoded as skill patterns |

---

## 2. CUSTOMER DOMAIN

Customer intelligence is a natural expansion. FLS has ~3,400 active client accounts. The sales skill already handles customer revenue ranking (Template 6) but does not cover lookup, profile, segmentation, or lifecycle analysis.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Customer lookup by name | "Find ABC Company" -- most basic CRM query | Low | Account WHERE CompanyName LIKE '%ABC%' AND IsActive = 1 AND IsClient = 1. Return: CompanyName, AccountNumber, primary contact, phone, email, city/state |
| Customer profile | "Tell me about [customer]" -- full customer snapshot | Med | Account + AccountContact (PrimaryContactID) + Address (BillingAddressID) + PhoneNumber (MainPhoneNumberID) + PaymentTerms + last order + total revenue + balance due. Multiple joins |
| Customer revenue (total and by period) | "How much has ABC Company spent with us?" | Med | Already in sales skill Template 6 (top customers). Extend to single-customer view with period breakdown |
| Customer order history | "Show me recent orders for [customer]" | Med | TransHeader WHERE AccountID = @ID AND TransactionType = 1, ordered by SaleDate DESC. Include OrderNumber, Description, SubTotalPrice, StatusText, SaleDate |
| Customer contact list | "Who are the contacts at [customer]?" | Low | AccountContact WHERE AccountID = @ID AND IsActive = 1. Return: name, title, email, phone |
| Customer open balances | "Does [customer] owe us anything?" | Med | TransHeader WHERE AccountID = @ID AND TransactionType = 1 AND BalanceDue > 0 AND StatusID != 9. Sum BalanceDue and list invoices |
| Top customers | "Who are our biggest customers?" | Med | Already in sales skill Template 6. Reuse directly |
| Customer payment terms | "What terms does [customer] have?" | Low | Account.PaymentTermsID -> PaymentTerms.TermsName, GracePeriod |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Customer lifetime value (CLV) | "What is the total lifetime value of [customer]?" -- helps prioritize accounts. Requires aggregating all historical orders | Med | Already in advanced_analytics.md. SUM(SubTotalPrice) WHERE TransactionType = 1 AND StatusID NOT IN (9), plus first/last order dates, order count, average order value |
| Dormant customer detection | "Which customers haven't ordered in 6 months?" -- proactive retention. No equivalent in Control UI without manual searching | Med | Already in advanced_analytics.md. MAX(SaleDate) < DATEADD(MONTH, -6, GETDATE()). Return: company, last order date, days since, previous revenue |
| Customer concentration analysis | "How dependent are we on our top 10?" -- risk analysis. Manual spreadsheet work today | Med | Already in advanced_analytics.md (Pareto). Top N customers with cumulative % of revenue |
| New customer acquisition trend | "How many new customers this year vs last year?" -- growth metric | Med | Already in advanced_analytics.md. MIN(SaleDate) per AccountID grouped by year/month |
| Customer credit balance | "Does [customer] have a credit with us?" -- customer service | Low | Account.CreditBalance. Simple field lookup. Also available from GL WHERE GLAccountID = 23 AND AccountID = @ID |
| Customer segmentation by product | "Which customers buy table covers?" -- cross-sell identification | High | TransHeader + TransDetail + TransDetailParam (VariableID = 11053 for DyeSub, Description LIKE for non-DyeSub). Requires sales skill product identification logic |
| Estimate conversion by customer | "What % of [customer]'s estimates convert?" -- sales pipeline insight | Med | TransHeader WHERE TransactionType = 2 AND AccountID = @ID. Count StatusID 13 (Converted) vs total. Already have estimate conversion analytics |
| Customer industry/origin breakdown | "Where do our customers come from?" | Med | Account.IndustryID -> MarketingListItem (MarketingListID = 10). Account.OriginID -> MarketingListItem (MarketingListID = 11). New query pattern needed |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Customer record creation/editing | Write operations require CHAPI and csp_ImportCompany. Contact management is a high-integrity workflow | Read-only lookups. Route creation requests to: "Create new customers in Control directly" |
| Email campaign execution | Control has macros and email activities but sending email requires SSLIP integration, not SQL queries | Show contact email lists for export. Do not attempt to send emails |
| Prospect pipeline management | Prospects use Company Stages (Element table, ClassTypeIDs 5511/5512) with stage change tracking. Managing pipeline requires writes to Account.StageID | Show current pipeline counts and stage distributions. Do not attempt stage changes |
| Division-level customer reporting | FLS has Banner and Apparel divisions. Division-split customer analysis is deferred per project charter | Acknowledge division exists (Account.DivisionID, TransHeader.DivisionID) but do not build division-specific customer reporting yet. Add "Division reporting deferred to future milestone" |
| Real-time customer interaction history | ContactActivity joins through Journal with complex ClassTypeIDs (21100, 21150, 21300, 21350). Activity data model is intricate and not well-validated | Defer to future. Focus on order-based customer intelligence which is well-understood and validated |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Sales skill: Template 6 (top customers) | **Reuse** | Customer revenue ranking already built |
| Sales skill: Product identification | **Reuse** | Container vs non-container logic needed for "which customers buy X" |
| Financial skill: AR by customer | **Extend** | AR drill-down adds customer context |
| Core skill: Account.CompanyName gotcha | **Reuse** | Field is CompanyName, not AccountName |
| Advanced analytics: CLV, dormant, Pareto | **Import** | Already have query templates, need to encode as skill patterns |
| Wiki: Company stages, CRM | **New import** | Stage definitions, marketing lists from wiki extract |

---

## 3. INVENTORY DOMAIN

FLS uses Control's inventory module for tracking raw materials (fabric, vinyl, ink, hardware). The sign industry has specific inventory challenges: large format media rolls, hardware components, and subcontract materials. Control uses a complex inventory model with multiple tracking modes.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Current stock levels | "How much [material] do we have?" | Med | Inventory WHERE PartID = @ID AND ClassTypeID = 12200. Return: QuantityOnHand, QuantityAvailable, QuantityReserved, QuantityOnOrder, WarehouseID. Must understand Inventory formula: OnHand = Billed + ReceivedOnly, Available = OnHand - Reserved |
| Inventory listing | "Show me all inventory" | Med | Inventory JOIN Part JOIN Warehouse WHERE Part.TrackInventory = 1. Filter IsActive = 1 on both tables |
| Inventory value | "What is our total inventory value?" | Med | SUM(Inventory.QuantityBilled * Inventory.AverageCost) WHERE Part.TrackInventory = 1 AND Part.AccrueCosts = 1. Wiki provides exact SQL. Also cross-reference with GL NodeID 10414 |
| Low stock / reorder alerts | "What parts need reordering?" | Med | Inventory WHERE QuantityAvailable <= Part.ReOrderPoint (if field exists) or where QuantityAvailable <= 0 and QuantityOnOrder = 0. Need to verify ReOrderPoint field availability |
| Parts listing | "Show me all our materials" or "List all parts" | Low | Part WHERE IsActive = 1. Filter by Part.PartType (0=Material, 1=Labor, 2=Equipment, 3=Outsource, 4=Other, 5=Freight). Return: ItemName, SKU, UnitCost, PartType |
| Inventory by warehouse | "What's in warehouse X?" | Med | Inventory JOIN Warehouse WHERE WarehouseID = @ID. Group by Part. FLS may have Standard, Kanban, and Stockroom warehouse types |
| Part cost lookup | "How much does [material] cost?" | Low | Part.UnitCost for standard cost; Inventory.AverageCost for weighted average |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Inventory turnover | "Which materials move fastest/slowest?" -- helps optimize purchasing | High | Already in advanced_analytics.md. Annual usage (SUM TransPart.Quantity) / Current stock. Requires TransPart + Inventory + Part joins |
| Slow-moving inventory | "What inventory hasn't moved in 90+ days?" -- capital management | High | Already in advanced_analytics.md. MAX(InventoryLog date) per part, filter > 90 days. Flags dead stock |
| Inventory valuation vs GL reconciliation | "Does our inventory value match the GL?" -- audit support | High | Wiki provides exact SQL: compare SUM(Inventory.QuantityBilled * AverageCost) by AssetAccountID vs GL WHERE GLClassificationType = 1003. Critical for month-end close |
| Parts consumption trend | "How much vinyl have we used monthly?" -- forecasting | Med | Already in advanced_analytics.md. TransPart grouped by YEAR/MONTH/PartID. Shows usage patterns |
| Open purchase orders | "What POs are outstanding?" | Med | TransHeader WHERE TransactionType = 7 AND StatusID IN (25,26,27,31) -- Requested, Approved, Ordered, Received. Join Account for vendor name |
| Part usage by order | "What materials went into order #133xxx?" | Med | TransPart WHERE TransHeaderID = (Order's ID). Join Part for names. Show EstimatedValue vs ActualValue |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Inventory adjustments | Write operation. Adjustments require Journal entries, GL postings, and InventoryLog records. Integrity-critical | Show current levels and discrepancies. Route adjustments to: "Use Control's inventory adjustment screen" |
| Purchase order creation | Requires csp_Import or full TransHeader creation with TransactionType = 7. Multi-step process | Show what needs reordering. Route PO creation to Control |
| Barcode scanning integration | Requires ProFITS module and hardware integration | Out of scope for NL interface |
| Warehouse transfer operations | Requires multi-step GL entries (production warehouse inventory account transfers) | Show transfer history from InventoryLog. Do not attempt transfers |
| Kanban/production warehouse management | Complex warehouse type logic (Standard vs Kanban vs Stockroom vs Group) with different rules per type | Show inventory by warehouse type but do not manage production floor inventory workflows |
| Display unit conversion | Parts have DisplayToUnitRatio for converting between inventory units and display units. Complex and varies per part | Show quantities in native inventory units. Note display unit if available but do not attempt conversion math |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Financial skill: Inventory value from GL | **Reuse** | Balance sheet query for NodeID 10414 already built |
| Core skill: GoodsItemClassTypeID | **Reuse** | 49=Product, 30=Part polymorphic pattern |
| Advanced analytics: turnover, slow-moving | **Import** | Query templates exist, need encoding as skill patterns |
| Wiki: Inventory formulas and tracking modes | **New import** | QuantityBilled + QuantityReceived = OnHand; three tracking modes |
| Wiki: Inventory valuation SQL | **New import** | Summary and detailed valuation queries from wiki |

---

## 4. PRODUCTION DOMAIN

Production is the operational heart of a sign shop. FLS uses Control's station-based workflow, artwork approval pipeline, and time clock system. This domain covers artwork status, station workload, and labor tracking -- queries that production managers ask daily.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Artwork status | "What proofs are pending approval?" | Med | ArtworkGroup WHERE StatusID < 7 (not Produced). Join TransHeader for order info. Status values: In Design, Pending Approval, Approved, Production Ready, Produced (7), Rejected |
| Artwork pipeline count | "How many artworks in each status?" | Med | COUNT(*) GROUP BY StatusID from ArtworkGroup WHERE IsActive = 1. Map StatusID to names |
| WIP orders by station | "What's at [station]?" | Med | TransHeader WHERE StatusID = 1 AND StationID = @StationID. Or TransDetail for line-item-level station tracking |
| Station workload | "Which stations have the most work?" | Med | COUNT orders or line items per station for WIP orders. TransHeader.StationID or TransDetail.StationID -> Station.StationName |
| Employee time today/this week | "How many hours has [employee] worked?" | Med | TimeCard JOIN Journal WHERE ClassTypeID = 20051 AND StartDateTime >= @Date. SUM(StraightTime + OverTime + DoubleTime + ShiftDiffTime). Wiki provides exact SQL |
| Orders due today/this week | "What is due today?" | Low | TransHeader WHERE StatusID IN (1,2) AND DueDate BETWEEN @Start AND @End. DueDate is production/ship deadline on Type 1 orders |
| Time clock status | "Who is clocked in right now?" | Med | TimeCard JOIN Journal WHERE ClassTypeID = 20050 (parent clock records). The most recent record per employee with EndDateTime IS NULL = currently clocked in. Or use TimeClockStatus table |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Artwork turnaround time | "How long does proof approval take on average?" -- bottleneck detection | High | Already in advanced_analytics.md. ArtworkGroupStatusHistory: time from first status to Approved status. Complex CTE query but high value |
| Station utilization | "Which stations are underutilized?" -- resource planning | High | Already in advanced_analytics.md. SUM hours per station / available hours. Requires TimeCard with ClassTypeID 20051 |
| Estimated vs actual cost comparison | "Are we estimating correctly?" -- profitability control | High | TransPart.EstimatedValue vs ActualValue per order. Crystal Report exists ("Est. vs Act. Cost Summary By Order.rpt"). Encode equivalent query |
| Artwork bottleneck detection | "What artworks have been stuck in Pending Approval for more than 3 days?" -- operational alert | Med | ArtworkGroup WHERE StatusID = [Pending Approval] AND StatusDT < DATEADD(DAY, -3, GETDATE()). High operational value for production manager |
| Employee productivity | "Hours per employee this week by station" -- labor management | Med | Already in advanced_analytics.md. TimeCard grouped by EmployeeID, StationID, date. Journal.StartDateTime/EndDateTime |
| Order cycle time | "How long from order creation to shipping?" -- process improvement | High | DATEDIFF between TransHeader.OrderCreatedDate and (Shipments.ShipDate or ClosedDate). Requires TransHeader + Shipments join |
| Parts usage vs estimate | "Did we use more material than estimated on order #X?" -- waste tracking | Med | TransPart WHERE TransHeaderID = @ID. Compare EstimatedValue (estimated qty) vs ActualValue (actual qty). Already referenced in wiki |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Station changes | Changing order or line item stations requires CHAPI notification, Journal entry creation, macro trigger processing | Show current station assignments. Route changes to: "Use Control's station change feature" |
| Time clock in/out | Requires Journal + TimeCard inserts with correct ClassTypeIDs (20050/20051), Station validation, lunch deduction rules | Show time clock data. Route clock operations to: "Use Control's time clock or Production Terminal" |
| Artwork status changes | Status transitions trigger CHAPI refresh, email notifications, and macro processing. Complex state machine | Show artwork status. Route changes to: "Use Control or CloudApp for artwork approval" |
| Production scheduling | Control does not have a scheduling engine. Stations are queues, not scheduled resources | Show workload by station and due dates. Do not attempt to schedule or prioritize |
| Part usage card creation | Usage cards (PartUsageCard) are created automatically at configurable status transitions or manually via Production Terminal. Write-heavy workflow | Show existing usage cards and their data. Route creation to Control |
| JDF export | JDF (Job Definition Format) is a specialized print industry format requiring XML generation. Out of scope | Not a query feature. Ignore |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| Financial skill: WIP balance | **Reuse** | GL-based WIP value (NodeID 11) already built |
| Core skill: StatusID reference | **Reuse** | Order status flow documented in core |
| Advanced analytics: station utilization, artwork | **Import** | Templates exist but may need validation against actual FLS data |
| Wiki: Artwork workflow | **New import** | Full state machine, status values, table relationships from wiki extract |
| Wiki: TimeCard SQL patterns | **New import** | Detail time card queries, ClassTypeID 20050/20051 patterns |
| Wiki: Station time tracking | **New import** | Journal.JournalActivityType = 45 for station changes |

---

## 5. REPORTS DOMAIN

This domain is unique: instead of querying data, it serves as a routing guide to Control's 36 Crystal Reports. Users often know what information they want but not which report to run. The NL interface can map intent to the right Crystal Report, explain its parameters, and in some cases replicate the report's output directly.

### Table Stakes

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Report catalog lookup | "What reports do we have?" or "Is there a report for X?" | Low | Static catalog from report_summary.md. 36 reports in 6 categories: Financial (5), WIP (4), Sales (14), Estimates (4), Inventory (6), Other (3) |
| Report recommendation | "I need to see sales by product" -> "Run the 'Sales - By Product.rpt' report" | Low | Intent-to-report mapping. Match user question to report name, purpose, and category. Static routing table |
| Report parameter guidance | "What parameters does the AR report need?" -> "Date range, optional Division" | Low | Static parameter reference. Most reports accept: Start/End Date, DivisionID, optional entity filters |
| Report description | "What does the WIP By Machine report show?" | Low | Descriptions from report_summary.md. One-liner purpose + primary tables + key fields |
| SQL equivalent for common reports | "Show me what the Sales By Product report shows" -> actual query | Med | For the 14 sales reports and 5 financial reports, provide SQL equivalents that produce similar output. Many already exist in sales/financial skills |

### Differentiators

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Report output replication | "Run the AR aging report for me" -> produce the actual data, not just a recommendation | High | For reports where we have validated SQL equivalents (AR, AP, Sales, WIP, P&L), actually execute the query and format output. Eliminates need to launch Crystal Reports for routine queries |
| Cross-report intelligence | "Which reports show customer data?" -> intelligent catalog search across descriptions, tables, and parameters | Low | Metadata search across the 36-report catalog. Group by entity: customer-related, product-related, financial, etc |
| Report gap identification | "I want to see X but there's no report" -> suggest a custom query or acknowledge the gap | Med | When user intent does not match any Crystal Report, fall back to ad-hoc query capabilities from other skills. "There's no Crystal Report for this, but I can query it directly" |
| Parameter value suggestions | "What divisions/stations/products can I filter by?" -> query reference tables to show valid parameter values | Med | Query Station, DivisionData, Product, Employee tables to show valid filter options. Dynamic lookups |

### Anti-Features

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| Crystal Report execution | The NL interface cannot launch Crystal Reports or invoke the Crystal runtime. .rpt files are binary OLE documents requiring SAP Crystal Reports SDK | Provide SQL equivalents for common reports. For others, describe what the report shows and how to run it in Control: "Open Control > Reports > [Report Name]" |
| Report modification/creation | Crystal Reports design requires the SAP designer application and knowledge of the proprietary format | Suggest SQL-based alternatives for custom reporting needs |
| Report scheduling | Control handles report scheduling through macros and SSLIP. Not a query operation | Route to: "Set up report scheduling in Control's macro system" |
| Crystal Report PDF generation | Requires Crystal Reports runtime and printer/export integration | Out of scope. The NL interface produces query results, not formatted documents |

### Dependencies on Existing Skills

| Existing Content | Reuse? | Notes |
|-----------------|--------|-------|
| report_summary.md (36 reports) | **Import** | Full catalog with categories, purposes, primary tables |
| report_join_patterns.md | **Import** | SQL equivalents for each report category |
| Sales skill: 9 query templates | **Reuse** | Many sales report equivalents already built |
| Financial skill: AR, AP, P&L | **Reuse** | Financial report equivalents already built |
| Glossary: NL routing table | **Extend** | Add report-specific routing entries |

---

## CROSS-DOMAIN FEATURE DEPENDENCIES

Some features span multiple domains. Document once, reference from each domain.

### Order-Centric Queries Touch Everything

An order (#133xxx) connects to:
- **Financial:** GL entries, payments, AR balance (TransHeader -> Ledger, Payment, Journal)
- **Customer:** Company, contact, billing address (TransHeader -> Account, AccountContact, Address)
- **Inventory:** Parts consumed, materials reserved (TransHeader -> TransPart -> Part -> Inventory)
- **Production:** Station, artwork status, time cards (TransHeader -> Station, ArtworkGroup, TimeCard via Journal)
- **Reports:** Appears in WIP, Sales, AR, Shipping reports

**Implication:** An "order profile" query that synthesizes all five domains into one view would be a powerful cross-domain differentiator, but should be a LATER feature after individual domains are solid.

### Shared Business Logic

These rules apply across all 5 domains and are already encoded in the core skill:

| Rule | Domains Affected | Source |
|------|-----------------|--------|
| Use SubTotalPrice for revenue | Financial, Customer, Sales | Core skill |
| Use SaleDate for date filtering | All 5 | Core skill |
| Never filter IsActive on TransDetailParam | Customer (product analysis), Inventory (parts) | Core skill + MEMORY.md |
| TransactionType filtering | All 5 | Core skill |
| StatusID meanings vary by TransactionType | All 5 | Core skill |
| COALESCE(DivisionID, 10) for division | All 5 | Core skill |
| Account.CompanyName not AccountName | Customer, Financial | Core skill + MEMORY.md |

---

## MVP RECOMMENDATION

For MVP of each domain skill, prioritize table stakes features. The ordering below reflects implementation dependencies:

### Phase 1: Customer + Reports (lowest dependency, highest user value)
1. **Customer lookup and profile** -- immediately useful for daily operations
2. **Report catalog and routing** -- helps users find what they need
3. **Customer revenue and order history** -- extends existing sales skill
4. **Report SQL equivalents** -- leverages already-built financial/sales queries

### Phase 2: Financial expansion (extends existing foundation)
5. **AR/AP per-customer/vendor drill-down** -- extends existing aggregate queries
6. **Cash flow summary** -- new high-value query
7. **GL integrity checks** -- imports wiki SQL patterns
8. **Historical AR** -- imports wiki SQL pattern

### Phase 3: Inventory (new domain, well-documented)
9. **Stock levels and inventory listing** -- straightforward queries
10. **Inventory valuation + GL reconciliation** -- wiki provides exact SQL
11. **Low stock alerts and PO tracking** -- operational value

### Phase 4: Production (most complex, least validated)
12. **Artwork pipeline and status** -- well-documented state machine
13. **Station workload and WIP by station** -- medium complexity
14. **Time tracking queries** -- requires validation against FLS data
15. **Estimated vs actual cost** -- high value but complex joins

### Defer to Post-MVP
- Division-level reporting (all domains)
- Customer segmentation by product (requires sales skill integration)
- Cross-domain order profile
- Real-time customer interaction history (ContactActivity complexity)
- Payroll queries (separate domain entirely)

---

## COMPLEXITY SUMMARY

| Domain | Table Stakes | Differentiators | Anti-Features | Total Features |
|--------|-------------|-----------------|---------------|----------------|
| Financial | 9 (4 already built) | 6 | 5 | 15 |
| Customer | 8 (1 already built) | 8 | 5 | 16 |
| Inventory | 7 | 6 | 6 | 13 |
| Production | 7 | 7 | 6 | 14 |
| Reports | 5 | 4 | 4 | 9 |
| **Totals** | **36** | **31** | **26** | **67** |

**Key insight:** Financial domain has the most existing foundation (4 of 9 table stakes already built). Customer domain has the most new features to build. Production domain has the highest implementation complexity. Reports domain has the lowest complexity and highest leverage (static catalog + reusing existing query templates).

---

## SOURCES

- `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` -- Existing financial skill (validated)
- `/Users/cain/projects/control-db-map/skills/control-erp-glossary/control-erp-glossary-SKILL.md` -- Existing glossary (validated)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/orders_accounting_knowledge.md` -- Wiki: GL lifecycle, payments, billing
- `/Users/cain/projects/control-db-map/output/wiki/extracts/production_inventory_knowledge.md` -- Wiki: Artwork, stations, inventory, shipping
- `/Users/cain/projects/control-db-map/output/wiki/extracts/crm_payroll_system_knowledge.md` -- Wiki: CRM, payroll, contacts, reporting
- `/Users/cain/projects/control-db-map/output/wiki/extracts/sql_queries_reference.md` -- Wiki: 40+ utility SQL queries
- `/Users/cain/projects/control-db-map/output/wiki/extracts/database_integration_knowledge.md` -- Wiki: ClassTypeIDs, SQLBridge, CHAPI
- `/Users/cain/projects/control-db-map/output/skill/references/advanced_analytics.md` -- Existing analytics queries (partially validated)
- `/Users/cain/projects/control-db-map/output/report_summary.md` -- Crystal Reports catalog (36 reports)
- `/Users/cain/projects/control-db-map/output/report_join_patterns.md` -- Inferred report SQL patterns

**Confidence:** HIGH for Financial and Customer domains (well-documented, existing skill foundation). MEDIUM for Inventory and Reports (wiki-documented but not yet validated against FLS data). MEDIUM for Production (wiki-documented but TimeCard/ArtworkGroup queries need FLS data validation).
