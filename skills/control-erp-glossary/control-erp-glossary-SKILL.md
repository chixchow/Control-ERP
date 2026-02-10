---
name: control-erp-glossary
description: Maps FLS Banners business language and Control ERP technical terminology to database entities, queries, and the right skill to consult. Use when users reference FLS product names (blankets, SEG, Fab Frame, DyeLux, table throw, step and repeat), Control-specific terms (CHAPI, SSLIP, CFL, ClassTypeID), or when you need to route a question to the correct skill.
---

# Control ERP Glossary — Business & Technical Term Translation

This glossary is a **routing and translation tool**, not a knowledge repository. It maps casual FLS business language to database query patterns and directs you to the right skill for detailed query templates. For all technical details, financial queries, or revenue analysis, consult the referenced skill files.

**How to use this glossary:**
1. User mentions a product term → Look up database identification pattern → Route to control-erp-sales
2. User mentions a technical term → Look up definition and database relationship → Route to owning skill
3. User asks a business question → Check routing table → Direct to correct skill and query approach

---

## FLS PRODUCT LINE TERMINOLOGY

### DyeSub Print Container Products

**Container product** — A product in Control that acts as a configurable shell housing many actual product lines through variables. DyeSub Print is the primary container at FLS, containing 20+ categories via the FP_ProductDescription variable. **Never report "DyeSub Print" as a product name; always drill into the specific category.**

**DyeSub Print** — The primary container product at FLS, housing 20+ product categories. Represents 58.7% of FLS revenue. NOT a product itself but a framework for configurable products.
- **Database identification:** Join to TransDetailParam WHERE VariableID = 11053 (FP_ProductDescription)
- **Revenue queries:** control-erp-sales Template 3

**FP_ProductDescription** — TransDetailParam variable (VariableID 11053) that identifies the actual product category within DyeSub Print containers.
- **Column:** ValueAsStr25
- **Usage:** GROUP BY for category-level reporting

**FP_ProductID** — TransDetailParam variable (VariableID 11052) that identifies the specific SKU within DyeSub Print containers.
- **Column:** ValueAsStr25
- **Usage:** Filter or GROUP BY for SKU-level detail

### DyeSub Print Categories (20 categories)

**Swing Flags** — Large outdoor promotional flags mounted on tall poles, highest DyeSub revenue category.
- **Aliases:** swing flag, swooper
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Swing Flags'
- **Revenue:** control-erp-sales Template 3

**Feather Flags** — Teardrop-shaped promotional flags with curved top, second-highest DyeSub category.
- **Aliases:** feather flag, feather banner
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Feather Flags'
- **Revenue:** control-erp-sales Template 3

**Banners** — General-purpose fabric or vinyl banners.
- **Aliases:** banner, vinyl banner, fabric banner
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Banners'
- **Revenue:** control-erp-sales Template 3

**SEG** — Silicone Edge Graphics, fabric prints with silicone strip edge for frame mounting.
- **Aliases:** silicone edge graphics, SEG frame, seg graphic
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'SEG'
- **Revenue:** control-erp-sales Template 3

**Tear Drop Flags** — Teardrop-shaped flags similar to feather flags.
- **Aliases:** teardrop, tear drop
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Tear Drop Flags'
- **Revenue:** control-erp-sales Template 3

**Banner Stands** — Retractable or roll-up banner display hardware.
- **Aliases:** banner stand, retractable banner, roll-up banner
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Banner Stands'
- **Revenue:** control-erp-sales Template 3

**Fab Frames** — Fabric tension frames for trade show displays.
- **Aliases:** fab frame, fabric frame, tension frame
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Fab Frames'
- **Revenue:** control-erp-sales Template 3

**Custom Flag** — Custom-designed flags for specific customer applications.
- **Aliases:** custom flags
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Custom Flag'
- **Revenue:** control-erp-sales Template 3

**Golf Flag 14x20** — Standard golf course flags (14" x 20" size).
- **Aliases:** golf flag, course flag
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Golf Flag 14x20'
- **Revenue:** control-erp-sales Template 3

**Table Runner** — Narrow decorative fabric strips running along table centers.
- **Aliases:** table runner
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Table Runner'
- **Revenue:** control-erp-sales Template 3

**Pillow Case Frames** — Fabric sleeve displays slipped over metal frames.
- **Aliases:** pillowcase, pillow case
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Pillow Case Frames'
- **Revenue:** control-erp-sales Template 3

**Pop Up Banners** — Spring-loaded popup display banners.
- **Aliases:** popup banner, pop-up
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Pop Up Banners'
- **Revenue:** control-erp-sales Template 3

**Backdrop** — Large-format photo backdrop banners for events.
- **Aliases:** step and repeat, photo backdrop, backdrop banner
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Backdrop'
- **Revenue:** control-erp-sales Template 3

**Golf Flag 5x8** — Smaller golf flags (5" x 8" size) for putting greens.
- **Aliases:** putting green flag
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Golf Flag 5x8'
- **Revenue:** control-erp-sales Template 3

**By The Yard** — Fabric sold by linear yard measurement.
- **Aliases:** fabric by the yard
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'By The Yard'
- **Revenue:** control-erp-sales Template 3

**Golf Cart Flag** — Small flags for golf cart mounting.
- **Aliases:** cart flag
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Golf Cart Flag'
- **Revenue:** control-erp-sales Template 3

**Bell Covers** — Custom fabric covers for musical instrument bells.
- **Aliases:** bell cover, instrument cover
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Bell Covers'
- **Revenue:** control-erp-sales Template 3

**Swoopper Flags** — Variant spelling of swooper flags (same as Swing Flags).
- **Aliases:** swooper, swoopper
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Swoopper Flags'
- **Revenue:** control-erp-sales Template 3

**Trombone cover** — Fabric covers specifically for trombone bells.
- **Aliases:** trombone
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Trombone cover'
- **Revenue:** control-erp-sales Template 3

**Printed Dyesub Paper** — Dye sublimation transfer paper for heat press applications.
- **Aliases:** dyesub paper, transfer paper
- **Container:** DyeSub Print
- **Database identification:** FP_ProductDescription = 'Printed Dyesub Paper'
- **Revenue:** control-erp-sales Template 3

### Non-DyeSub Product Groups (6 groups)

**DyeLux Table Cover** — Full-print fitted table covers (premium product). Note: TC_ variables (TC_FabricCategory, TC_ProductName) are NOT saved to TransDetailParam, so variable-based queries do not work.
- **Aliases:** dyelux, full print table cover, fitted table cover
- **Container:** Non-DyeSub standalone
- **Database identification:** Description LIKE '%Dyelux%Table Cover%' OR '%FULL%Table Cover%'
- **Revenue:** control-erp-sales Template 8

**Table Throw** — Simple drape-style table covers (not fitted).
- **Aliases:** table throw, table drape
- **Container:** Non-DyeSub standalone
- **Database identification:** Description LIKE '%Table Throw%'
- **Revenue:** control-erp-sales Template 8

**Table Cover** — Generic table covers not classified as DyeLux or Table Throw.
- **Aliases:** table cover
- **Container:** Non-DyeSub standalone
- **Database identification:** Description LIKE '%Table Cover%'
- **Revenue:** control-erp-sales Template 8

**Garments/Apparel** — Embroidered or screen-printed clothing items (Apparel division).
- **Aliases:** garment, embroidered, t-shirt, apparel, clothing
- **Container:** Non-DyeSub standalone
- **Database identification:** Description LIKE '%Garment%' OR '%Embroidered%'
- **Revenue:** control-erp-sales Template 8

**Tent/Pop-Up** — Pop-up tents and canopy structures.
- **Aliases:** tent, pop-up tent, canopy
- **Container:** Non-DyeSub standalone
- **Database identification:** Description LIKE '%Tent%' OR '%Pop%Up%'
- **Revenue:** control-erp-sales Template 8

**Design/Artwork** — Design services and artwork fees.
- **Aliases:** design fee, artwork, art charge
- **Container:** Non-DyeSub standalone
- **Database identification:** Description LIKE '%Design%' OR '%Artwork%'
- **Revenue:** control-erp-sales Template 8

**Blanket** — Promotional fleece or knit blankets, typically custom-printed with logos.
- **Aliases:** throw blanket, promotional blanket, fleece blanket
- **Container:** Non-DyeSub standalone
- **Database identification:** Description LIKE '%Blanket%'
- **Revenue:** control-erp-sales Template 8

---

## CONTROL TECHNICAL TERMINOLOGY

### Core System Components (Required Terms)

**CHAPI (Cyrious Host Application Programming Interface)** — The server-side HTTP service that coordinates database writes between SQL Bridge, SSLIP, and all connected Control clients. Runs on port 12556.
- **Why it matters:** All SQL Bridge stored procedures must call CHAPI functions to notify the system of changes.
- **Database functions:** csf_chapi_nextid(), csf_chapi_nextnumber(), csf_chapi_lock(), csf_chapi_unlock(), csf_chapi_refresh(), csf_chapi_recompute()
- **See:** output/wiki/extracts/database_integration_knowledge.md Section 1

**SSLIP (Server-Side Logic and Integration Platform)** — The background server process that handles recompute operations, macro execution, Production Terminal, and WebView hosting.
- **Why it matters:** After SQL Bridge inserts/updates, SSLIP must be notified via CHAPI to recalculate pricing and execute automation rules.
- **Key roles:** Pricing recalculation, macro execution, Production Terminal web interface, WebView customer portal
- **See:** output/wiki/extracts/database_integration_knowledge.md Section 6

**CFL (Cyrious Formula Language)** — The expression language used in PricingPlan formulas, Variable formulas, and IOTemplate fields throughout Control.
- **Why it matters:** Product pricing, variable defaults, workflow automation all depend on CFL expressions.
- **Where used:** PricingPlan formulas, Variable.DefaultFormula, IOTemplate fields, Macro conditions
- **See:** output/wiki/extracts/cfl_formula_language_knowledge.md

**SQLBridge** — The collection of SQL Server stored procedures and functions that safely insert/update Control database records and notify CHAPI of changes.
- **Why it matters:** Direct SQL inserts bypass Control's business logic. SQLBridge procedures ensure data integrity and system notification.
- **Key procedures:** csf_chapi_* functions, csp_Import* procedures
- **See:** output/wiki/extracts/database_integration_knowledge.md Section 1

**ClassTypeID** — A polymorphic type identifier integer used throughout Control to identify what type of entity a record represents. Nearly every major table has a ClassTypeID column.
- **Why it matters:** Many tables (Journal, PricingElement, CustomerGoodsItem) store multiple entity types distinguished only by ClassTypeID. Filtering by the wrong ClassTypeID returns incorrect results.
- **Database column:** ClassTypeID (int) on nearly every table
- **Key values:** 10000=TransHeader, 10100=TransDetail, 2000=Account, 12000=Product, 12014=Part, 8900=Ledger
- **See:** output/wiki/extracts/database_integration_knowledge.md Section 2 (350+ mappings)

**GoodsItemClassTypeID** — A specific ClassTypeID column on TransDetail that identifies whether the line item references a Product (catalog item) or a Part.
- **Why it matters:** TransDetail.GoodsItemID is polymorphic. Without checking GoodsItemClassTypeID, you cannot tell if GoodsItemID points to a Product or a Part.
- **Database column:** TransDetail.GoodsItemClassTypeID
- **Key values:** 12000=Product (catalog item), 49=Product (direct), 30=Part
- **See:** control-erp-core TransDetail section

### Database Fields and Patterns

**SeqID** — A version tracking integer on every Control table that increments with each update. History tables use (ID, SeqID) composite keys for audit trails.
- **Why it matters:** SQL Bridge updates must increment SeqID. Control clients detect changes by comparing SeqID values.
- **Database pattern:** Every table has a SeqID column; every *History table uses (ID, SeqID) composite key
- **See:** control-erp-core History Tables section

**TransactionType** — An integer on TransHeader identifying the transaction category (1=Order, 2=Estimate, 7=PO, 8=Bill, etc.).
- **Why it matters:** Revenue queries must filter to TransactionType = 1. Including other types produces incorrect totals.
- **Database column:** TransHeader.TransactionType
- **See:** control-erp-core TransactionType Reference table

**StatusID** — An integer on TransHeader identifying the transaction lifecycle status. StatusID meanings vary by TransactionType.
- **Why it matters:** Revenue recognition occurs when StatusID transitions to 3 (Sale) or 4 (Closed) for Type 1 orders.
- **Database column:** TransHeader.StatusID
- **See:** control-erp-core StatusID Reference section

**SubTotalPrice** — The pre-tax revenue amount on TransHeader and TransDetail. This is the field used for revenue reporting, NOT TotalPrice.
- **Why it matters:** TotalPrice includes tax. Using TotalPrice in revenue queries overstates income.
- **Database column:** TransHeader.SubTotalPrice, TransDetail.SubTotalPrice
- **See:** control-erp-core Pricing Fields Reference

**SaleDate** — The date revenue is recognized on Type 1 orders. Always NULL on Type 2 estimates.
- **Why it matters:** Revenue queries must filter by SaleDate, NOT OrderCreatedDate. OrderCreatedDate reflects order entry, not sale.
- **Database column:** TransHeader.SaleDate
- **See:** control-erp-core Date Filtering Patterns

**IsActive** — Standard boolean flag (bit) on most tables indicating whether a record is active. **CRITICAL EXCEPTION:** Do NOT filter TransDetailParam by IsActive. Control sets IsActive=false after product config, but data is still valid. Filtering IsActive on TransDetailParam suppresses ~75% of valid records.
- **Why it matters:** Standard filter for active records, but TransDetailParam is a documented exception.
- **Database column:** IsActive (bit) on most tables
- **See:** control-erp-core TransDetailParam query rules

**Import_Order_Number** — A user-defined field (UDF) on TransHeaderUserField that identifies web-imported orders from CHAPI integrations.
- **Why it matters:** Web orders are Type 1 orders (not Type 2), identified by this UDF. Deduplication is mandatory as cloned orders share Import_Order_Number values.
- **Database column:** TransHeaderUserField.Import_Order_Number
- **See:** control-erp-core Web Import Identification section

### Account and Customer Concepts

**UDF (User-Defined Field)** — Custom fields added to Account, Contact, TransHeader, Product, or Part entities via the UserFieldDef and UserFieldLayout tables.
- **Why it matters:** Critical business data (like Import_Order_Number) often lives in UDF tables, not core schema.
- **Database tables:** AccountUserField, TransHeaderUserField, EmployeeUserField
- **See:** output/wiki/extracts/udfs_custom_fields_knowledge.md (if exists)

**Division** — A multi-company context separating Company (Banners/Signs) from Apparel divisions. Each has its own warehouses and product lines.
- **Why it matters:** Revenue and inventory queries should be division-aware. Mixing divisions produces confusing results.
- **Database table:** Division
- **See:** control-erp-core Multi-Division Context section

### Pricing and Product Concepts

**Pricing Plan** — A set of pricing formulas (PreDiscountPrice, BuiltInDiscount, DefaultDiscount) that vary by customer group. Different customers see different pricing based on assigned plan.
- **Why it matters:** Querying product pricing without context of which plan is active produces generic/incorrect prices.
- **Database table:** PricingPlan
- **See:** output/wiki/extracts/products_pricing_knowledge.md Section 7

**Pricing Level** — A percentage adjustment applied invisibly to pricing (not shown to customer). Used for volume discounts or dealer pricing tiers.
- **Why it matters:** The visible price differs from the calculated price when Pricing Levels are active.
- **Database table:** PricingLevel
- **See:** output/wiki/extracts/products_pricing_knowledge.md Section 8

**Promotion** — A visible discount applied to orders, shown on invoices and estimates.
- **Why it matters:** Discounts can come from Promotions, Pricing Plans, or manual adjustments. Promotion-based discounts are tracked separately.
- **Database table:** Promotion
- **See:** output/wiki/extracts/products_pricing_knowledge.md Section 16

**Variable** — A configurable product attribute (size, color, quantity, etc.) stored per line item in TransDetailParam.
- **Why it matters:** Container products like DyeSub Print are defined entirely by variables. Without querying TransDetailParam, you cannot identify what was actually sold.
- **Database tables:** Variable, TransDetailParam
- **See:** control-erp-sales Product Architecture section

**Modifier** — A self-contained pricing add-on (lamination, grommets, rush fee, etc.) that modifies a product's price and cost.
- **Why it matters:** Line-item pricing includes base product + modifiers. Ignoring TransMod produces incomplete pricing.
- **Database table:** TransMod
- **See:** output/wiki/extracts/products_pricing_knowledge.md Section 5

**CustomerGoodsItem** — The actual table name for Products (ClassTypeID 12000) and Modifiers (ClassTypeID 12010). "Product" is the UI term, "CustomerGoodsItem" is the database table.
- **Why it matters:** Schema documentation and SQL queries reference CustomerGoodsItem, not Product.
- **Database table:** CustomerGoodsItem
- **See:** output/schemas/Product.md

**PricingElement** — A general-purpose table storing categories, families, plan types, and other pricing-related groupings identified by ClassTypeID.
- **Why it matters:** Pricing hierarchy (families, categories) is not stored in dedicated tables but as PricingElement records with different ClassTypeIDs.
- **Database table:** PricingElement
- **See:** output/wiki/extracts/products_pricing_knowledge.md Section 9

### Production and Workflow Concepts

**Station** — A production workflow stage (printing, cutting, finishing, shipping) tracked in the Station table with ClassTypeID 26100.
- **Why it matters:** Time tracking, production costs, and workflow reporting depend on station tracking.
- **Database table:** Station
- **See:** output/schemas/Station.md (if exists)

**Macro** — An automation rule (RuleMacro) triggered by events (status change, payment received, etc.) to execute actions (send email, update field, create task).
- **Why it matters:** Business logic embedded in macros affects order workflow. Understanding macro triggers explains why fields auto-populate.
- **Database table:** RuleMacro
- **See:** output/wiki/extracts/macros_automation_knowledge.md

**Variation** — An alternate pricing configuration within a single estimate, allowing customers to compare multiple options on one estimate document.
- **Why it matters:** Estimate revenue calculations typically use only the active/first variation. Including all variations overstates pipeline.
- **Database table:** TransVariation
- **See:** output/wiki/extracts/orders_accounting_knowledge.md Section 1

### Financial Concepts

**GL View** — A SQL view filtering the Ledger table to on-balance-sheet entries only (WHERE OffBalanceSheet = 0). This is what "GL" refers to in Control documentation.
- **Why it matters:** Financial reports use the GL view. Cost analysis uses the full Ledger table. Querying the wrong source produces incomplete results.
- **Database view:** GL (view on Ledger table)
- **See:** control-erp-financial GL/Ledger Architecture section

**Ledger** — The actual general ledger table containing all GL entries (on-balance-sheet and off-balance-sheet).
- **Why it matters:** The GL view excludes off-balance-sheet entries. For complete cost tracking, query Ledger directly.
- **Database table:** Ledger
- **See:** control-erp-financial GL/Ledger Architecture section

**Journal** — An activity/event log table tracking all system events (order created, status changed, payment received, etc.) linked via JournalID to other tables.
- **Why it matters:** Audit trails and payment history are stored in Journal, not in the transaction tables themselves.
- **Database table:** Journal
- **See:** control-erp-financial Payment Posting Patterns section

**Payment** — A split table with Journal (Payment.ID = Journal.ID) storing payment financial details (Amount, BankAccountID, PaymentMethodID).
- **Why it matters:** Payment records exist in both Journal and Payment tables. Querying only one table produces incomplete data.
- **Database table:** Payment
- **See:** control-erp-financial Payment Posting Patterns section

**Closeout** — A period-locking mechanism preventing backdated GL entries. The most recent Closeout of each type determines the locked period boundary.
- **Why it matters:** GL entries before the last closeout date cannot be modified without reopening the period.
- **Database table:** Closeout
- **See:** control-erp-financial Closeout section

### System and Integration Concepts

**History Tables** — Append-only audit tables (*History suffix) with (ID, SeqID) composite keys preserving every version of a record.
- **Why it matters:** Current state is in the base table. Full audit trail is in the *History table. Price change analysis requires History tables.
- **Database tables:** TransHeaderHistory, TransDetailHistory, LedgerHistory, etc.
- **See:** control-erp-core History Tables section

**Polymorphic Pattern** — A database design pattern using ParentID + ParentClassTypeID columns to link records to different parent types.
- **Why it matters:** Understanding ClassTypeID filtering is essential. Filtering by the wrong ParentClassTypeID returns no results.
- **Database pattern:** ParentID + ParentClassTypeID columns throughout schema
- **See:** control-erp-core Schema Quick Reference section

**CRIX (Cyrious Report Integration eXchange)** — A reporting subsystem within Control for generating Crystal Reports.
- **Why it matters:** Mentions of "CRIX" in documentation refer to the reporting engine, not a separate product.
- **See:** SSLIP documentation

**Production Terminal** — A web-based production workflow interface served by SSLIP for shop floor tracking.
- **Why it matters:** Station time tracking and workflow updates happen through Production Terminal, not the main Control client.
- **See:** SSLIP documentation

**WebView** — A customer-facing web portal served by SSLIP for order status, proofing, and self-service.
- **Why it matters:** Web-visible data (estimates, proofs, order status) flows through WebView integration.
- **See:** SSLIP documentation

---

## ORDER LIFECYCLE AND WORKFLOW TERMINOLOGY

**Estimate / Quote** — A Type 2 transaction representing a pre-sale proposal for a customer. Estimates never contribute to revenue. 56% convert to orders, 41% are lost.
- **StatusIDs:** 11=Pending, 12=Lost, 13=Converted, 14=Voided
- **See:** control-erp-core TransactionType Reference

**Order / Sale** — A Type 1 transaction representing the primary revenue record. Orders progress through WIP → Sale → Closed lifecycle.
- **StatusIDs:** 0=New, 1=WIP, 2=Built, 3=Sale, 4=Closed, 9=Voided
- **See:** control-erp-core TransactionType Reference

**WIP (Work in Progress)** — StatusID 1 on Type 1 orders, indicating production is underway. No revenue recognized yet.
- **See:** control-erp-core StatusID Reference

**Built** — StatusID 2 on Type 1 orders, indicating production is complete but order not yet invoiced. Optional intermediate status.
- **See:** control-erp-core StatusID Reference

**Sale (status)** — StatusID 3 on Type 1 orders, indicating order has been invoiced and revenue is recognized. SaleDate is set at this transition.
- **See:** control-erp-core StatusID Reference

**Closed** — StatusID 4 on Type 1 orders, indicating order is fully paid (BalanceDue = 0) and complete.
- **See:** control-erp-core StatusID Reference

**Voided** — StatusID 9 indicating a cancelled transaction. Always exclude from revenue and financial queries.
- **See:** control-erp-core StatusID Reference

**Converted** — StatusID 13 on Type 2 estimates, indicating the estimate successfully became a Type 1 order. Linked via EstimateNumber.
- **See:** control-erp-core Estimate → Order Linkage section

**Lost** — StatusID 12 on Type 2 estimates, indicating the customer chose not to proceed.
- **See:** control-erp-core StatusID Reference

**Pending** — StatusID 11 on Type 2 estimates, indicating an active estimate awaiting customer decision.
- **See:** control-erp-core StatusID Reference

**PO / Purchase Order** — A Type 7 transaction representing vendor purchasing documents in the vendor chain (PO → Bill → Receiving).
- **StatusIDs:** 6=Open, 25=Requested, 26=Approved, 27=Ordered, 28=Closed, 29=Cancelled, 30=Rejected, 31=Received
- **See:** control-erp-core TransactionType Reference

**Bill** — A Type 8 transaction representing vendor invoices/bills for accounts payable tracking.
- **StatusIDs:** 4=Closed, 6=Open, 9=Voided
- **See:** control-erp-financial AP section

**AR / Accounts Receivable** — Unpaid customer invoices (Type 1 orders with BalanceDue > 0 and StatusID = 3).
- **See:** control-erp-financial Accounts Receivable section

**AP / Accounts Payable** — Unpaid vendor bills (Type 8 bills with BalanceDue > 0 and StatusID = 6).
- **See:** control-erp-financial Accounts Payable section

**P&L / Profit and Loss** — Income statement showing revenue minus COGS and expenses. Generated from GL entries with GLClassificationType 4000/4001 (income) and 5001/5002 (expenses).
- **See:** control-erp-financial P&L from GL section

**COGS (Cost of Goods Sold)** — Expense category for direct product costs (materials, freight, subcontract). GLClassificationType 5001.
- **See:** control-erp-financial P&L from GL section

---

## CRYSTAL REPORTS CATALOG

Crystal Reports are the legacy reporting engine at FLS Banners, running for 15+ years with battle-tested SQL. Skills are their modern replacement. When a report is "Replaced by" a skill, use the skill's query instead of running the Crystal Report. When a report is "Not yet covered", the Crystal Report is the only option—run it in Control.

**LLM answers first.** Only recommend running the Crystal Report in Control when:
- The skill doesn't cover that report's data domain
- The report has complex formatting (invoices, work orders with proof images) that a query can't reproduce
- The user specifically asks to run a Crystal Report

> **Note:** Report details were inferred from filenames and database schema analysis, not extracted from binary .rpt files. Actual report definitions may differ.

### Financial Reports

| Report | Purpose | Coverage |
|--------|---------|----------|
| FLS Income Statement.rpt | P&L showing revenue, COGS, expenses by GL hierarchy | Replaced by control-erp-financial (Full P&L, P&L Summary) |
| FLS Balance Sheet.rpt | Assets, liabilities, equity snapshot | Replaced by control-erp-financial (Balance Sheet Key Accounts) |
| A_R Detail.rpt | AR aging with invoice-level detail per customer | Replaced by control-erp-financial (AR Aging Buckets, AR Snapshot) |
| AR Report - Summary.rpt | AR summary totals by customer | Replaced by control-erp-financial (AR Aging summary) |
| A_P Aging Detail.rpt | AP aging with vendor invoice detail | Replaced by control-erp-financial (AP Snapshot, AP Aging) |

### WIP Reports

| Report | Purpose | Coverage |
|--------|---------|----------|
| WIP Summary - Order Level.rpt | WIP summary aggregated at order level | Replaced by control-erp-production (Order-Level WIP) |
| WIP By Line Item Station.rpt | WIP detail by station per line item | Replaced by control-erp-production (LI-Level WIP) |
| WIP By Machine.rpt | WIP grouped by production station/machine | Replaced by control-erp-production (WIP by Station) |
| WIP By Calendar Month Due.rpt | WIP aggregated by due date calendar month | Partially covered by control-erp-production |

### Sales Reports

| Report | Purpose | Coverage |
|--------|---------|----------|
| Orders Placed Between.rpt | Orders placed within date range | Replaced by control-erp-sales (Template 1) |
| Sales - By Product.rpt | Revenue breakdown by product category | Replaced by control-erp-sales (Template 3 + 8) |
| Sales - By Product Category.rpt | Revenue aggregated by product category groups | Partially covered by control-erp-sales |
| Sales - By Customer (Volume).rpt | Top customers ranked by total revenue | Replaced by control-erp-customers (Top Customers by Revenue) |
| Sales - By Customer (Frequency).rpt | Top customers ranked by order count | Replaced by control-erp-customers (Top Customers by Order Count) |
| Sales - By Sales Account.rpt | Revenue grouped by sales account | Partially covered by control-erp-sales |
| Sales - By All Salespeople.rpt | Revenue for all salesperson slots | Partially covered by control-erp-sales |
| Sales - By Primary Saleperson.rpt | Revenue by primary salesperson | Replaced by control-erp-sales (Template 7) |
| Based On Sales by Price.rpt | Sales sorted/grouped by price point | Partially covered by control-erp-sales |
| Based on Orders Placed by Price.rpt | Orders sorted/grouped by price point | Partially covered by control-erp-sales |
| Sales Graph.rpt | Visual sales trend graph | Replaced by control-erp-sales (Template 2) |
| Sales Report.rpt | General sales summary report | Replaced by control-erp-sales (Template 1 + 8) |
| Sales - Yearly Sales By Calendar Month.rpt | Year-over-year monthly comparison | Replaced by control-erp-sales (Template 9) |
| Sales - Yearly Sales By Salesperson.rpt | Salesperson performance year-over-year | Partially covered by control-erp-sales |

### Estimate Reports

| Report | Purpose | Coverage |
|--------|---------|----------|
| Converted Estimates.rpt | Estimates that became orders | Partially covered by control-erp-core |
| Pending Estimates.rpt | Active estimates awaiting decision | Partially covered by control-erp-core |
| Last Order Aging.rpt | Dormant customer detection by last order date | Replaced by control-erp-customers (Dormancy) |
| Est. vs Act. Cost Summary By Order.rpt | Estimated vs actual cost variance analysis | Not yet covered |

### Inventory & Product Reports

| Report | Purpose | Coverage |
|--------|---------|----------|
| All Parts Listing.rpt | Complete parts catalog dump | Partially covered by control-erp-inventory |
| Catalog Listing.rpt | Catalog item listing | Partially covered by control-erp-inventory |
| Inventory Listing.rpt | Current inventory levels for all parts | Partially covered by control-erp-inventory |
| Inventory History.rpt | Historical inventory movement tracking | Partially covered by control-erp-inventory |
| Product Detail.rpt | Full product configuration and pricing | Not yet covered |
| Product Parameters.rpt | Product parameter/variable definitions | Not yet covered |

### Other Reports

| Report | Purpose | Coverage |
|--------|---------|----------|
| Last Contact Aging.rpt | Customer engagement tracking by last contact | Partially covered by control-erp-customers |
| Shipped By Date.rpt | Shipment tracking by date range | Not yet covered |
| Work Order with Proof.rpt | Formatted work order with embedded proof images | Not yet covered (run in Control) |

### Coverage Summary

| Status | Count | % |
|--------|-------|---|
| Replaced by skill | 17 | 47% |
| Partially covered | 14 | 39% |
| Not yet covered | 5 | 14% |

---

## NATURAL LANGUAGE ROUTING TABLE

| User Says | Route To | Key Pattern/Section |
|-----------|----------|-------------------|
| "find blanket orders" | control-erp-sales | Description LIKE '%Blanket%' pattern |
| "what are SEG orders" / "SEG revenue" | control-erp-sales | FP_ProductDescription = 'SEG' via Template 3 |
| "how much did we sell in feather flags" | control-erp-sales | Template 3 filtered by FP_ProductDescription |
| "sales by product" / "product breakdown" | control-erp-sales | Template 3 (category level) or Template 4 (SKU level) |
| "DyeLux table cover sales" | control-erp-sales | Template 8 with Description LIKE pattern |
| "step and repeat revenue" | control-erp-sales | FP_ProductDescription = 'Backdrop' |
| "Fab Frame orders" | control-erp-sales | FP_ProductDescription = 'Fab Frames' |
| "table throw sales" | control-erp-sales | Description LIKE '%Table Throw%' |
| "top customers" / "customer revenue" | control-erp-sales | Template 6 |
| "sales by rep" / "salesperson performance" | control-erp-sales | Template 7 |
| "what were our sales in [period]" | control-erp-sales | Template 1 or 2 |
| "monthly revenue trend" | control-erp-sales | Template 2 |
| "compare this year to last year" | control-erp-sales | Template 9 |
| "What is CHAPI?" | control-erp-glossary | See CHAPI definition above |
| "What is a SSLIP?" | control-erp-glossary | See SSLIP definition above |
| "what is a ClassTypeID" | control-erp-glossary | See ClassTypeID definition above |
| "what TransactionType is an estimate" | control-erp-core | TransactionType Reference table |
| "order lifecycle" / "status flow" | control-erp-core | StatusID Reference section |
| "revenue query formula" | control-erp-core | CRITICAL: Revenue Query Formula section |
| "AR aging" / "past due invoices" | control-erp-financial | AR Aging Buckets query |
| "what's our AR" / "open invoices" | control-erp-financial | AR Snapshot query |
| "AP aging" / "what's overdue" | control-erp-financial | AP Aging query |
| "what do we owe" / "open bills" | control-erp-financial | AP Snapshot query |
| "P&L" / "profit and loss" | control-erp-financial | P&L Summary or Full P&L query |
| "revenue by product" (GL-based) | control-erp-financial | Revenue by Product Line query |
| "gross margin" | control-erp-financial | P&L Summary → GrossProfit/TotalRevenue |
| "cash position" / "how much cash" | control-erp-financial | Balance Sheet Key Accounts |
| "COGS" / "cost of goods" | control-erp-financial | Full P&L filtered to GLClass 5001 |
| "expenses breakdown" | control-erp-financial | Full P&L filtered to GLClass 5002 |
| "inventory value" | control-erp-financial | Balance Sheet → NodeID 10414 |
| "WIP balance" / "orders in production value" | control-erp-financial | Balance Sheet → NodeID 11 |
| "GL entries for order #NNNNN" | control-erp-financial | Ledger WHERE TransHeaderID query |
| "how did [customer] pay" / "payment history" | control-erp-financial | GL register query on NodeID 24 or 14 |

---

*This glossary routes FLS business language to database patterns and the correct skill for detailed queries. It does not duplicate query templates, revenue figures, or business rules — those live in the referenced skills where they can be maintained and validated.*
