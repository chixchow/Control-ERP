# FLS Banners ERP Query Pattern Reference

This document references validated query patterns from skill files. For actual SQL, load the referenced skill. No SQL is duplicated here — each entry points to the owning skill and template/section.

All patterns validated against FLS Banners 2025 financial data. Scorecard: 21/21 PASS.

For skill architecture, dependency graph, and order lifecycle, see `output/docs/skill-architecture.md`.

---

## 1. Revenue & Sales

Patterns answering: "What did we sell?"

### "Total sales for [year]"
- **Also:** "annual revenue," "how much did we sell," "yearly sales"
- **Skill:** control-erp-sales, Template 1
- **Returns:** Annual revenue total with order count, grouped by year
- **Benchmark:** $3,052,952.52 for 2025 (Test 1.1, 0.02% of known income $3,053,541.85)

### "Monthly sales breakdown"
- **Also:** "monthly revenue," "sales by month," "monthly trend"
- **Skill:** control-erp-sales, Template 2
- **Returns:** Revenue and order count per month for a given year
- **Benchmark:** 12 rows for 2025, monthly sum matches annual within $4 (Test 1.2)

### "Quarterly revenue"
- **Also:** "Q4 sales," "sales by quarter"
- **Skill:** control-erp-sales, Template 2 (filtered by quarter months)
- **Returns:** Revenue summed by quarter with growth percentage
- **Benchmark:** Q4 2025 = $592,789 (Test 1.3)

### "Compare this year to last year"
- **Also:** "year-over-year," "YoY comparison," "sales growth"
- **Skill:** control-erp-sales, Template 9
- **Returns:** Side-by-side monthly revenue for two years
- **Benchmark:** 2025 vs 2024 validated (Test 1.4)

### "Revenue by salesperson"
- **Also:** "sales by rep," "salesperson performance," "rep comparison"
- **Skill:** control-erp-sales, Template 7
- **Returns:** Revenue and order count per salesperson
- **Benchmark:** Sum = $3,052,952.52 (exact match Test 1.1). Josh Gregory 50.7%, Hervy Hodges 35.7% (Test 6.1)

---

## 2. Product Analysis

Patterns answering: "What products are selling?"

### "Sales by product"
- **Also:** "product breakdown," "revenue by product category," "what products are we selling"
- **Skill:** control-erp-sales, Template 3
- **Returns:** Revenue by DyeSub Print product category (FP_ProductDescription)
- **Benchmark:** DyeSub Print total $1,793,445 across 21 categories (Test 2.1)

### "DyeSub Print revenue"
- **Also:** "[category name] sales" (e.g., "Feather Flag sales," "Swing Flag revenue")
- **Skill:** control-erp-sales, Template 3 (filtered by category)
- **Returns:** Revenue for specific DyeSub category with order count
- **Benchmark:** Swing Flags $615,203, Feather Flags $485,666 (Test 2.4)

### "Table cover sales"
- **Also:** "DyeLux sales," "table throw revenue," "fitted table covers"
- **Skill:** control-erp-sales, Template 8 (filtered to table cover groups)
- **Returns:** Revenue by table cover type (DyeLux, Other Table Covers, Table Runners)
- **Benchmark:** Total table cover revenue $548,181 (Test 2.2)

### "Full product breakdown"
- **Also:** "where does our revenue come from," "product mix," "revenue reconciliation"
- **Skill:** control-erp-sales, Template 8
- **Returns:** All product groups with revenue (DyeSub categories + non-DyeSub groups)
- **Benchmark:** Detail-level total $3,045,878, 0.02% grand total variance (Test 2.3)

### "SKU-level detail"
- **Also:** "product detail," "sales by SKU," "specific product revenue"
- **Skill:** control-erp-sales, Template 4
- **Returns:** Revenue by FP_ProductID (individual SKU) within DyeSub Print
- **Benchmark:** Validated within DyeSub Print container architecture

### "Revenue by product line (GL-based)"
- **Also:** "product revenue from GL," "accounting view of product sales"
- **Skill:** control-erp-financial, Revenue by Product Line section
- **Returns:** Revenue by GLAccount product line (matches accountant's view)
- **Benchmark:** All-time revenue $62.45M across product accounts

---

## 3. Customer Intelligence

Patterns answering: "Who are our customers?"

### "Top customers"
- **Also:** "biggest accounts," "top N customers by revenue," "best customers"
- **Skill:** control-erp-sales, Template 6
- **Returns:** Top N customers by revenue with order count and last sale date
- **Benchmark:** Top 10 = $1,506,214 (49.3% of total). FLASH Visual Media #1 (Test 3.1)

### "Customer [name] history"
- **Also:** "what did [customer] buy," "[customer] sales," "customer detail"
- **Skill:** control-erp-sales, Template 5
- **Returns:** Customer's revenue by product line with order count
- **Benchmark:** PAC BannerWorks detail validated (Test 3.2)

### "Order count"
- **Also:** "how many orders," "total orders in [period]"
- **Skill:** control-erp-sales, Template 1 (COUNT column)
- **Returns:** Number of Type 1 orders with SaleDate in period
- **Benchmark:** 4,172 orders in 2025 (Test 3.3)

### "New customers"
- **Also:** "first-time buyers," "customer acquisition"
- **Skill:** control-erp-core, Account.CreatedDate + TransHeader date filtering
- **Returns:** Customers with first order in given period

---

## 4. Estimates & Pipeline

Patterns answering: "What's in the pipeline?"

### "Estimate conversion rate"
- **Also:** "quote to close rate," "win rate," "how many estimates convert"
- **Skill:** control-erp-core, TransactionType Reference (Type 2 StatusID distribution)
- **Returns:** Percentage of estimates that converted (StatusID 13) vs lost (StatusID 12)
- **Benchmark:** 56.1% converted, 41.3% lost (Test 4.1)

### "Pending estimates"
- **Also:** "open quotes," "active estimates," "outstanding quotes"
- **Skill:** control-erp-core, Type 2 + StatusID 11
- **Returns:** Count and value of estimates still pending
- **Benchmark:** 326 pending estimates (Test 4.2)

### "Pipeline value"
- **Also:** "estimate value," "what's in the pipeline," "potential revenue"
- **Skill:** control-erp-core, Type 2 + StatusID 11 + SUM(SubTotalPrice)
- **Returns:** Total SubTotalPrice of pending estimates
- **Benchmark:** $1,635,895 pipeline, avg $5,018 per estimate (Test 4.3)

### "Lost quotes in [period]"
- **Also:** "lost estimates," "quotes we didn't win"
- **Skill:** control-erp-core, Type 2 + StatusID 12 + EstimateCreatedDate filter
- **Returns:** Lost estimates in period with total value
- **Benchmark:** 244 lost in 2025, $3,268,561 value (Test 4.4)
- **CRITICAL:** Must use `EstimateCreatedDate` for date filtering. Both SaleDate and OrderCreatedDate are always NULL on Type 2.

---

## 5. Financial

Patterns answering: "What's our financial position?"

### "What's our AR?"
- **Also:** "open invoices," "accounts receivable," "who owes us money"
- **Skill:** control-erp-financial, AR Snapshot section
- **Returns:** Open invoices with customer, amount, sale date, payment terms
- **Benchmark:** ~$80,899 AR (119 invoices, 30 customers). 93% current.

### "AR aging"
- **Also:** "past due invoices," "overdue accounts," "aging report"
- **Skill:** control-erp-financial, AR Aging Buckets section
- **Returns:** AR balance grouped by 0-30, 31-60, 61-90, 90+ day buckets
- **Benchmark:** Validated against FLS AR report

### "What do we owe?"
- **Also:** "AP balance," "accounts payable," "open bills," "vendor balances"
- **Skill:** control-erp-financial, AP Snapshot section
- **Returns:** Open vendor bills with amount and due date
- **Benchmark:** ~$125,319 AP (66 bills, 35 vendors). 96% current.

### "AP aging"
- **Also:** "overdue bills," "vendor aging"
- **Skill:** control-erp-financial, AP Aging section
- **Returns:** AP balance grouped by aging buckets (uses DueDate, not SaleDate)

### "P&L"
- **Also:** "profit and loss," "income statement," "net income"
- **Skill:** control-erp-financial, P&L Summary section
- **Returns:** Revenue, COGS, gross profit, operating expenses, net income from GL
- **Benchmark:** Gross margin ~82% in recent periods

### "Gross margin"
- **Also:** "what's our margin," "profitability"
- **Skill:** control-erp-financial, P&L Summary (GrossProfit / TotalRevenue)
- **Returns:** Gross margin percentage derived from GL entries

### "Revenue by product from accounting"
- **Also:** "GL revenue," "revenue by GL account"
- **Skill:** control-erp-financial, Revenue by Product Line section
- **Returns:** Revenue grouped by GLAccount (GLClassificationType 4000)

### "Cash position"
- **Also:** "how much cash do we have," "bank balance"
- **Skill:** control-erp-financial, Balance Sheet Key Accounts section
- **Returns:** Cash-Checking (NodeID 90) and Cash-MM (10412) balances

### "Expenses breakdown"
- **Also:** "operating expenses," "where is our money going"
- **Skill:** control-erp-financial, Full P&L (filtered to GLClassificationType 5002)
- **Returns:** All expense accounts with amounts

### "COGS"
- **Also:** "cost of goods sold," "direct costs"
- **Skill:** control-erp-financial, Full P&L (filtered to GLClassificationType 5001)
- **Returns:** All COGS accounts (purchases, freight, subcontract)

### "GL entries for order [number]"
- **Also:** "what happened in GL for this order," "order GL history"
- **Skill:** control-erp-financial, Ledger WHERE TransHeaderID section
- **Returns:** All GL entries linked to a specific order

### "How did [customer] pay?"
- **Also:** "payment history," "payment method for order"
- **Skill:** control-erp-financial, Payment Posting Patterns section
- **Returns:** Payment records with tender type, amount, bank account

### "Undeposited payments"
- **Also:** "pending deposits," "what hasn't been deposited"
- **Skill:** control-erp-financial, Deposit Workflow section
- **Returns:** Payments where Undeposited = 1 with bank account details

### "Inventory value"
- **Also:** "what's our inventory worth"
- **Skill:** control-erp-financial, Balance Sheet Key Accounts (NodeID 10414)
- **Returns:** Current inventory asset balance from GL

### "WIP balance"
- **Also:** "orders in production value," "work in progress total"
- **Skill:** control-erp-financial, Balance Sheet Key Accounts (NodeID 11)
- **Returns:** Current WIP asset balance from GL

---

## 6. Terminology & Routing

Patterns needing term translation before querying.

### "[FLS product name]"
- **Also:** "blanket orders," "SEG orders," "Fab Frame sales," "step and repeat"
- **Skill:** control-erp-glossary, FLS Product Line Terminology section
- **Returns:** Routes term to database identification pattern, then to control-erp-sales
- **Examples:** "blanket" maps to `Description LIKE '%Blanket%'`; "step and repeat" maps to `FP_ProductDescription = 'Backdrop'`

### "[Control technical term]"
- **Also:** "what is CHAPI," "what is ClassTypeID," "what is SSLIP," "what is CFL"
- **Skill:** control-erp-glossary, Control Technical Terminology section
- **Returns:** Definition, database relationship, and route to relevant skill

### "What does [StatusID/TransactionType] mean?"
- **Also:** "what is StatusID 13," "what is Type 7"
- **Skill:** control-erp-glossary, Order Lifecycle and Workflow Terminology section
- **Returns:** Definition with cross-reference to core skill StatusID/TransactionType tables

---

## 7. Web Orders

Patterns about online/imported orders.

### "Web order volume"
- **Also:** "website orders," "online orders," "CHAPI imports"
- **Skill:** control-erp-core, Web Import Identification section
- **Returns:** Deduplicated web order count and revenue (Import_Order_Number UDF)
- **Benchmark:** 429 orders, $150,750 revenue in 2025 (Test 5.1)
- **CRITICAL:** Deduplication mandatory. Raw count 467 includes 38 clones.

### "Web order percentage"
- **Also:** "what percent of sales are online," "web vs in-house"
- **Skill:** control-erp-core, Web Import Identification section
- **Returns:** Web orders as percentage of total revenue and order count
- **Benchmark:** 4.94% of revenue, 10.28% of order count. Avg $351/order vs $732 overall (Test 5.2)

---

## 8. Not Yet Available (Milestone 2)

These question types are not yet covered by validated skill templates:

- **Inventory levels and reorder points** — Schema documented (Inventory, Part, Warehouse tables) but no validated query templates
- **Artwork pipeline status** — ArtworkItem, ArtworkGroup tables documented but no workflow query templates
- **Station scheduling and production workload** — Station, ActivityTimeSpan tables documented but no production analytics
- **Employee/payroll/PTO** — Employee, Payroll, TimeCard tables documented but no HR query templates
- **Time tracking and labor costs** — TimeCard ClassTypeID patterns identified (20050/20051) but no validated templates
- **Dashboards and forecasting** — Advanced analytics reference exists (output/skill/references/advanced_analytics.md) but patterns are not validated
- **Commission calculations** — CommissionPlan, CommissionRate tables documented but no query templates
- **Service ticket analysis** — ServiceTicketPriority, ServiceContractType tables documented but minimal FLS data

---

*All patterns reference skills validated against FLS Banners 2025 data. Revenue: $3,052,952.52 (99.98% of known income). Product: $3 variance from Control report. Scorecard: 21/21 PASS.*
