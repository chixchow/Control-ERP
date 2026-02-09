# Domain Pitfalls

**Project:** Cornerstone v1.1 -- Read-Layer Domain Expansion
**Domains:** Financial (AR/AP/P&L/Cash), Customer Intelligence, Inventory, Production, Reports Catalog
**Researched:** 2026-02-09
**Calibration:** v1.0 TransDetailParam IsActive bug = $1.3M revenue discrepancy. All severity ratings use this as the "Critical" reference point.

---

## Critical Pitfalls

Mistakes that produce wrong numbers, break trust, or require rewrites. Each of these has a real analog in v1.0 or a demonstrated data pattern in the FLS database.

---

### Pitfall C1: GL Sign Convention Inversion (Financial Domain)

**What goes wrong:** Revenue queries return negative numbers (or expenses return negative), producing nonsensical P&L reports. Revenue accounts store amounts as negative (credits) while expense accounts store amounts as positive (debits). A skill that does `SUM(Amount)` on revenue accounts gets the wrong sign.

**Why it happens:** Standard double-entry accounting uses debit-positive/credit-negative conventions. Most developers expect revenue to be positive. The GL view and Ledger table both follow this convention, and there is no flag on the row indicating "this should be negated for display."

**Consequences:** P&L shows negative revenue or negative expenses. If uncaught, the skill returns "Revenue: -$3,053,541" to the user, which is obviously wrong. More subtly, if gross margin is calculated as `Revenue - COGS` and both signs are wrong, the margin could look correct but the components are inverted.

**Prevention:**
- Hardcode the sign convention rule into the financial skill: `SUM(-Amount)` for GLClassificationType IN (4000, 4001) and `SUM(Amount)` for GLClassificationType IN (5001, 5002)
- Add a validation check: GL-based revenue for 2025 must approximately match the known $3,053,541.85 benchmark
- The existing `control-erp-financial` skill already documents this correctly; the pitfall is in failing to propagate this rule to every query pattern

**Detection:** Any P&L query that returns negative revenue or negative expenses.

**Severity:** HIGH -- wrong sign on $3M of revenue is immediately visible, but the subtler case of wrong-sign COGS producing an incorrect margin percentage would be harder to detect.

**Domain/Phase:** Financial skill, earliest phase. Must be baked into every GL query template.

---

### Pitfall C2: GL View vs Ledger Table Confusion (Financial Domain)

**What goes wrong:** Financial queries use the `Ledger` table directly instead of the `GL` view, including ~307K off-balance-sheet entries that distort financial totals. Or conversely, production cost queries use the `GL` view and miss off-balance-sheet cost entries.

**Why it happens:** `GL` is a VIEW defined as `SELECT * FROM Ledger WHERE OffBalanceSheet = 0`. The name "GL" is intuitive for financial queries, but developers who inspect the schema see `Ledger` as the "real" table with 2.75M rows and might query it directly, not realizing ~11% of entries are off-balance-sheet cost accounting entries that should be excluded from financial reports.

**Consequences:**
- Financial reports inflated by off-balance-sheet entries (expense double-counting for parts expensed at purchase)
- Production cost analysis that misses off-balance-sheet entries (understating true production costs by the amount tracked in NodeIDs 60/61)

**Prevention:**
- Rule: "Use `GL` view for all financial queries. Use `Ledger` table only when explicitly analyzing production costs including non-accrual materials."
- The skill should never reference `FROM Ledger` in a standard financial query template
- Production cost queries should explicitly note when they include `OffBalanceSheet = 1` entries and explain why

**Detection:** Compare `SELECT COUNT(*) FROM GL` vs `SELECT COUNT(*) FROM Ledger`. If the skill is using Ledger and the user asked a financial question, it is wrong.

**Severity:** HIGH -- ~11% data distortion. Off-balance-sheet entries for parts expensed at purchase would inflate COGS/expense figures, producing wrong profit margins.

**Domain/Phase:** Financial skill. Must be established in the query architecture before any GL query patterns are written.

---

### Pitfall C3: DueDate Semantic Overloading (Financial Domain -- AR/AP)

**What goes wrong:** AR aging uses `TransHeader.DueDate` as the payment deadline, producing completely wrong aging buckets. A $50K order due to ship next week shows as "current" in AR aging when it actually has a 90-day-old unpaid invoice.

**Why it happens:** `TransHeader.DueDate` means **production/shipping deadline** for Type 1 orders but **vendor payment deadline** for Type 8 bills. There is no dedicated "payment due date" field on TransHeader. The actual payment due date for AR is calculated at runtime as `SaleDate + PaymentTerms.GracePeriod`. This is the exact same class of bug as v1.0's "use SaleDate not OrderCreatedDate" -- a field that sounds right but means something different than expected.

**Consequences:** AR aging buckets are completely wrong. Orders that shipped yesterday (DueDate = yesterday) but were invoiced 60 days ago would appear "current" instead of "31-60 days." Users lose trust in the entire AR skill.

**Prevention:**
- AR aging MUST use `SaleDate` as the aging anchor (invoice date), NOT `DueDate`
- AP aging MUST use `DueDate` as the aging anchor (vendor payment deadline) -- this IS correct for bills
- Document the semantic difference prominently: "DueDate = ship date for orders, payment date for bills"
- The existing `control-erp-financial` skill already implements this correctly; the risk is in new query patterns or customer intelligence queries that cross-reference AR data

**Detection:** Validate AR aging against Control's built-in "A_R Detail.rpt" report. If buckets don't match, DueDate is likely being misused.

**Severity:** CRITICAL -- This is the v1.0 SaleDate/OrderCreatedDate bug pattern in a new domain. Wrong AR aging would produce incorrect collections prioritization.

**Domain/Phase:** Financial skill (AR/AP section). Must be established in the first AR query template.

---

### Pitfall C4: Inventory Quantity Field Confusion (Inventory Domain)

**What goes wrong:** The skill reports "available inventory" using `QuantityOnHand` instead of `QuantityAvailable`, overstating what can actually be allocated to new orders. Or it reports raw `QuantityBilled` instead of `QuantityOnHand`, missing received-but-not-yet-billed stock.

**Why it happens:** Control's inventory model has a specific quantity chain:
```
QuantityBilled + QuantityReceivedOnly = QuantityOnHand
QuantityOnHand - QuantityReserved = QuantityAvailable
QuantityAvailable + QuantityOnOrder = QuantityExpected
```
Each field answers a different question. A developer unfamiliar with this chain will pick the wrong field. `QuantityOnHand` sounds like "what's available" but it includes stock reserved for existing orders.

**Consequences:** If the skill reports `QuantityOnHand` as available stock, users might commit inventory that is already reserved for in-progress orders. This is operationally dangerous for a custom flag/banner shop where materials are cut-to-order and cannot be easily reallocated.

**Prevention:**
- Document the quantity chain in the skill with explicit guidance: "When user asks 'how much X do we have,' use `QuantityAvailable`. When user asks 'what's on the shelf,' use `QuantityOnHand`. When user asks 'what's coming,' use `QuantityExpected`."
- Natural language mapping: "available" -> `QuantityAvailable`, "on hand" / "in stock" -> `QuantityOnHand`, "on order" -> `QuantityOnOrder`, "reserved" -> `QuantityReserved`
- Validate against Control's "Inventory Listing.rpt" report

**Detection:** Compare skill output for "how much vinyl do we have" against the Inventory screen in Control. If reserved quantity is non-trivial, the difference will be visible.

**Severity:** HIGH -- For a $3M signage business, committing reserved inventory to new orders could delay in-progress production and damage customer relationships.

**Domain/Phase:** Inventory skill. Must be established before any inventory quantity query patterns.

---

### Pitfall C5: Inventory Table vs Part Table Quantity Duplication (Inventory Domain)

**What goes wrong:** The skill queries `Part.QuantityOnHand` / `Part.QuantityAvailable` instead of `Inventory.QuantityOnHand` / `Inventory.QuantityAvailable`, getting aggregate quantities instead of warehouse-specific quantities. Or worse, it confuses the two and double-counts.

**Why it happens:** Both `Part` (7,684 rows) and `Inventory` (27,268 rows) have quantity fields with identical names: `QuantityOnHand`, `QuantityReserved`, `QuantityAvailable`, `QuantityOnOrder`. The `Part` table has aggregate quantities across all warehouses. The `Inventory` table has per-warehouse/per-division quantities (Part:Inventory is 1:many via `Inventory.PartID`). Also, `Inventory.ClassTypeID = 12200` for warehouse-level records vs group/summary records.

**Consequences:** Queries that join Part and Inventory without proper filtering double-count quantities. Queries that use Part quantities instead of Inventory miss warehouse-level detail. For FLS with multiple warehouses (10, 11 for Company division; 10000, 10001 for Apparel), this produces wrong stock levels per location.

**Prevention:**
- Rule: "Always query `Inventory` table for quantity questions, filtering `ClassTypeID = 12200` for warehouse-level records. Never use `Part.QuantityXxx` fields for inventory reporting."
- If a user asks "how much X across all warehouses," use `SELECT SUM(QuantityAvailable) FROM Inventory WHERE PartID = @PartID AND ClassTypeID = 12200`
- If a user asks "how much X in [warehouse]," add `AND WarehouseID = @WarehouseID`

**Detection:** Compare `Part.QuantityOnHand` for a given part with `SUM(Inventory.QuantityOnHand) WHERE PartID = Part.ID AND ClassTypeID = 12200`. If they differ (due to group records or stale aggregates), the Part-level query is wrong.

**Severity:** HIGH -- Wrong inventory levels affect reorder decisions and order fulfillment.

**Domain/Phase:** Inventory skill. Must be established in the data model documentation.

---

### Pitfall C6: TimeCard ClassTypeID Filtering Omission (Production Domain)

**What goes wrong:** Production time queries return inflated hours because they count both parent clock-in/out records (ClassTypeID 20050) AND station time detail records (ClassTypeID 20051) when they should only count one or the other.

**Why it happens:** The TimeCard table stores two types of records using ClassTypeID as a discriminator: 20050 = clock in/out parent record (top-level time card entry) and 20051 = station time detail (time spent at a specific station). If you query `SUM(StraightTime)` from TimeCard without filtering ClassTypeID, you sum the parent totals AND the child details, roughly doubling the actual hours.

**Consequences:** Employee hours doubled. Labor cost reports show 2x the actual cost. Production efficiency metrics are halved. This is analogous to the v1.0 TransHeader/TransDetail SubTotalPrice gap but worse -- it is not a small $7K gap but a potential 2x multiplier.

**Prevention:**
- Rule: "For total employee hours worked, use `ClassTypeID = 20050` (parent records). For time-by-station breakdown, use `ClassTypeID = 20051` (detail records). Never mix them in the same SUM."
- Natural language mapping: "how many hours did X work" -> ClassTypeID 20050. "how long was X at station Y" -> ClassTypeID 20051.
- The TimeCard schema file already documents this: "ClassTypeID 20050 = Clock in/out parent record, ClassTypeID 20051 = Station time detail record."

**Detection:** Compare `SUM(StraightTime) WHERE ClassTypeID = 20050 AND EmployeeID = @ID` with `SUM(StraightTime) WHERE ClassTypeID = 20051 AND EmployeeID = @ID`. If they are approximately equal, you found the double-counting pattern.

**Severity:** CRITICAL -- 2x labor hours means 2x labor cost, producing dramatically wrong production cost reports. This is the production domain's equivalent of the $1.3M TransDetailParam bug.

**Domain/Phase:** Production skill. Must be the first rule established for TimeCard queries.

---

### Pitfall C7: TimeCard Geography Column Query Failures (Production Domain)

**What goes wrong:** Queries against the TimeCard table fail with a SQL Server error because `SELECT *` or column inclusion of `LatLonStart` / `LatLonEnd` (geography/spatial data type) causes the MCP MSSQL tools to crash.

**Why it happens:** TimeCard has two geography columns: `LatLonStart` and `LatLonEnd`. The MCP MSSQL tools cannot serialize geography data types. This was a known v1.0 issue that forced explicit column selection for any table with spatial data.

**Consequences:** Skill queries that use `SELECT * FROM TimeCard` or include these columns will fail silently or with cryptic errors. The user gets no results when asking about employee hours.

**Prevention:**
- Rule: "Never use SELECT * on TimeCard. Always explicitly list columns, excluding LatLonStart and LatLonEnd."
- Template all TimeCard queries with explicit column lists
- This rule already exists in CLAUDE.md ("Skip geography/spatial columns - they cause errors") and in the TimeCard schema notes

**Detection:** Any TimeCard query that returns an error mentioning geography or spatial types.

**Severity:** MEDIUM -- Causes query failures, not wrong data. But it blocks all production time reporting until fixed.

**Domain/Phase:** Production skill. Must be documented in the first TimeCard query template.

---

### Pitfall C8: Journal Multi-Purpose Table Conflation (Production + Financial Domains)

**What goes wrong:** A production query for station time tracking accidentally pulls payment records, closeout records, or CRM activity records from the Journal table. Or a financial query for payments accidentally includes production station changes.

**Why it happens:** The Journal table (5.18M rows) is a mega-table used for at least 6 different purposes:
- Payments (ClassTypeID 20000-20038) -- splits with Payment table
- Closeouts (ClassTypeID 8911-8919)
- Station time tracking (JournalActivityType = 45)
- Contact activities (various ClassTypeIDs)
- Email activities
- Macro activities

Without proper ClassTypeID or JournalActivityType filtering, a query against Journal will conflate unrelated records. The Payment table has `Payment.ID = Journal.ID` (split table pattern), so joining Journal and Payment requires understanding they share the same ID space.

**Consequences:** Production station time queries inflated by non-station Journal entries. Payment totals contaminated by non-payment Journal records. The 5.18M-row table makes unfiltered queries slow and wrong simultaneously.

**Prevention:**
- Rule: "Always filter Journal by ClassTypeID or JournalActivityType. Never query Journal unfiltered."
- Station time: `WHERE JournalActivityType = 45`
- Payments: `WHERE ClassTypeID IN (20000, 20001, 20002, 20003, 20004, 20005, 20006, 20007, 20009)` or use the Payment table directly
- Closeouts: `WHERE ClassTypeID IN (8911, 8912, 8913, 8914, 8915, 8916, 8917, 8918, 8919)`
- Document the Journal-Payment split table pattern prominently

**Detection:** Query `SELECT ClassTypeID, COUNT(*) FROM Journal GROUP BY ClassTypeID` -- if you see dozens of different ClassTypeIDs, you know unfiltered queries will mix unrelated data.

**Severity:** HIGH -- Produces wrong numbers in both production and financial domains.

**Domain/Phase:** Both Production and Financial skills. Must be documented in each skill's Journal query section.

---

### Pitfall C9: Account.CompanyName vs "AccountName" Confusion (Customer Domain)

**What goes wrong:** Customer queries fail because the skill uses `Account.AccountName` (which does not exist) or joins on `Account.Name`. The actual field is `Account.CompanyName`. A second trap: `GLAccount.AccountName` exists (for chart of accounts) and could be confused with customer account names.

**Why it happens:** Every other ERP system uses "AccountName" for customers. Control uses `CompanyName`. This was a known v1.0 gotcha documented in MEMORY.md. The risk in v1.1 is that a new Customer Intelligence skill re-introduces this error, especially if the developer consults generic ERP patterns rather than the existing schema documentation.

**Consequences:** SQL errors (column not found) or, worse, a join to GLAccount.AccountName producing chart-of-accounts names instead of customer names.

**Prevention:**
- Already documented in MEMORY.md: "Account table uses CompanyName, NOT AccountName"
- The Customer Intelligence skill MUST reference the Account schema and use `CompanyName` consistently
- Test with a simple query: `SELECT TOP 1 CompanyName FROM Account WHERE IsClient = 1`

**Detection:** Any SQL error mentioning "Invalid column name 'AccountName'" on the Account table.

**Severity:** MEDIUM -- Causes query failures, not wrong data. Easy to fix once detected.

**Domain/Phase:** Customer Intelligence skill. Must be established in the Account table documentation.

---

### Pitfall C10: IsClient/IsProspect/IsActive Triple-Flag Filtering (Customer Domain)

**What goes wrong:** Customer intelligence queries include all 54,719 Account records (prospects, vendors, inactive, personal accounts) when they should only include active clients. Or they exclude prospects when the user explicitly asks about prospects.

**Why it happens:** The Account table has multiple boolean classification flags:
- `IsClient` = 1 for customers who have placed orders
- `IsProspect` = 1 for leads who haven't ordered yet
- `IsVendor` = implicit (accounts used as vendors on Type 7/8/9 transactions)
- `IsActive` = 1 for active accounts
- `IsPersonal` = personal accounts

A query for "top customers" that omits `IsClient = 1 AND IsActive = 1` will include prospects, vendors, and inactive accounts. Conversely, a query for "how many prospects do we have" that filters `IsClient = 1` returns zero results.

**Consequences:** Customer metrics (count, revenue per customer, CLV) are distorted. "We have 54,719 customers" vs the actual ~X active clients is a credibility-destroying error.

**Prevention:**
- Default filters for customer queries: `WHERE IsClient = 1 AND IsActive = 1`
- Default filters for prospect queries: `WHERE IsProspect = 1 AND IsActive = 1`
- Default filters for vendor queries: Join to TransHeader WHERE TransactionType IN (7, 8, 9)
- Natural language mapping: "customers" / "clients" -> `IsClient = 1`. "prospects" / "leads" -> `IsProspect = 1`. "all accounts" -> no IsClient/IsProspect filter.
- Always include `IsActive = 1` unless user explicitly asks about inactive/churned accounts

**Detection:** `SELECT COUNT(*) FROM Account WHERE IsClient = 1 AND IsActive = 1` should return a number much smaller than 54,719.

**Severity:** HIGH -- Wrong customer counts undermine all customer intelligence metrics.

**Domain/Phase:** Customer Intelligence skill. First filter rule to establish.

---

### Pitfall C11: Artwork Status Integer Magic Numbers (Production Domain)

**What goes wrong:** Production artwork tracking queries use wrong StatusID values because the artwork status codes are not the same as order status codes. A query filtering `ArtworkGroup.StatusID = 3` (thinking "Sale" from Type 1 orders) gets no results because artwork uses a completely different status system.

**Why it happens:** ArtworkGroup.StatusID uses its own status codes (linked to `_ArtworkStatus` lookup table):
- In Design -> Pending Approval -> Approved -> Production Ready -> Produced (StatusID = 7)
- Rejected is a separate status

These are NOT the same as TransHeader StatusIDs (0-4, 9 for Type 1). A developer who has internalized the order status codes will incorrectly apply them to artwork.

**Consequences:** Artwork pipeline queries return wrong counts. "How many proofs are pending approval" returns zero or wrong numbers. Production bottleneck analysis misidentifies where artwork is stuck.

**Prevention:**
- Document artwork statuses separately from order statuses
- The wiki extract shows `StatusID < 7 = not yet produced, StatusID = 7 = Produced`
- Join to `_ArtworkStatus` lookup table when displaying status names
- Validate against ArtworkGroup data: `SELECT StatusID, COUNT(*) FROM ArtworkGroup WHERE IsActive = 1 GROUP BY StatusID`

**Detection:** If a query for "pending approval" artwork returns zero results, the StatusID filter is likely wrong.

**Severity:** MEDIUM -- Wrong production pipeline visibility. Operationally inconvenient but not financially damaging.

**Domain/Phase:** Production skill (artwork subsection). Must be documented before artwork query patterns.

---

## Moderate Pitfalls

Mistakes that cause delays, wrong-but-not-catastrophic results, or technical debt.

---

### Pitfall M1: AR Report Includes WIP/Built Orders (Financial Domain)

**What goes wrong:** The skill's AR total does not match Control's built-in AR report because the skill only includes StatusID >= 3 (Sale/Closed), while Control's AR report also shows WIP/Built orders with balances for visibility (even though they haven't posted to the GL AR account).

**Why it happens:** Control's AR report is a management view that includes orders in WIP/Built status with non-zero `BalanceDue`. The GL AR account (NodeID 14) only contains entries for orders that have reached Sale status. These are two different definitions of "AR" and both are used at FLS.

**Prevention:**
- Document both definitions: "Financial AR = GL NodeID 14 balance. Management AR = TransHeader with BalanceDue > 0 across all active statuses."
- Default to management AR (TransHeader-based, matching Control's report) for user queries about "AR" or "what's owed to us"
- Use GL AR only when reconciling to the general ledger or generating trial balance
- The existing `control-erp-financial` skill already handles this correctly; propagate the pattern

**Detection:** Compare skill AR total with Control's "AR Report - Summary.rpt." If the skill is lower, it is missing WIP/Built orders.

**Severity:** MEDIUM -- Understated AR by the value of WIP/Built orders with deposits.

**Domain/Phase:** Financial skill (AR section).

---

### Pitfall M2: Payment/Journal Split Table Architecture (Financial Domain)

**What goes wrong:** Payment queries miss data because the developer queries `Payment` alone (286K rows) without realizing that payment amounts, dates, and transaction links live in the `Journal` table (where `Payment.ID = Journal.ID`). Or the developer queries `Journal` for payments but does not filter by payment ClassTypeIDs.

**Why it happens:** Control uses a split table pattern: `Payment` stores payment-specific fields (TenderType, BankAccountID, card details) while `Journal` stores the common fields (DetailAmount, StartDateTime, TransactionID, AccountID, IsVoided). They share the same primary key. This is unusual in SQL databases.

**Prevention:**
- Document the split table pattern: "Payment.ID = Journal.ID. To get payment amount, join: `Payment p INNER JOIN Journal j ON p.ID = j.ID`"
- For payment amount, use `Journal.DetailAmount` (the Payment table does not have an Amount column -- it is on Journal)
- Always filter `Journal.IsVoided = 0` to exclude voided payments
- Filter `Journal.ClassTypeID IN (20001, 20009)` for individual order/bill payments

**Detection:** If `SELECT Amount FROM Payment` fails (no Amount column), the split table pattern was missed.

**Severity:** MEDIUM -- Blocks payment reporting until understood, but schema inspection reveals the issue quickly.

**Domain/Phase:** Financial skill (payments section).

---

### Pitfall M3: TransHeader Polymorphism in Customer Intelligence (Customer Domain)

**What goes wrong:** Customer metrics queries (revenue per customer, order frequency) accidentally include vendor transactions (Type 7/8/9), estimates (Type 2), or service tickets (Type 6) in customer spending calculations.

**Why it happens:** TransHeader stores ALL transaction types in one table. `Account` serves as both customer and vendor. A query like `SELECT AccountID, SUM(SubTotalPrice) FROM TransHeader GROUP BY AccountID` conflates customer sales, vendor purchases, estimates, and service tickets -- all linked to the same Account records.

**Prevention:**
- Customer revenue queries MUST include `TransactionType = 1 AND SaleDate IS NOT NULL` (from v1.0 core skill)
- Customer order count MUST filter `TransactionType = 1`
- Vendor spending queries MUST filter `TransactionType = 8`
- The `control-erp-core` skill's standard filters must be applied in every customer intelligence query

**Detection:** If top customers include vendor names (material suppliers), the TransactionType filter is missing.

**Severity:** MEDIUM -- Inflated customer metrics. A vendor with $500K in bills would appear as a "top customer."

**Domain/Phase:** Customer Intelligence skill. Apply core skill filters to every customer query.

---

### Pitfall M4: PartType Classification Ignorance (Inventory Domain)

**What goes wrong:** Inventory queries report labor hours and equipment time as "material inventory" because the skill treats all Part records equally without filtering by `Part.PartType`.

**Why it happens:** Control's Part table stores 6 types of items: Labor, Material, Equipment, Outsource, Freight, Other. Only Material parts have physical inventory. Labor and Equipment parts track time, not stock. If the skill queries `SELECT ItemName, QuantityOnHand FROM Part WHERE TrackInventory = 1`, it includes labor parts.

**Prevention:**
- For physical inventory queries, filter by `Part.PartType` that corresponds to Material
- For cost analysis queries, include all part types but categorize them
- Document the PartType reference: discover actual values with `SELECT PartType, COUNT(*) FROM Part GROUP BY PartType`
- Natural language mapping: "inventory" / "stock" / "materials" -> Material parts only. "costs" / "expenses" -> all part types.

**Detection:** If inventory listings include items like "Design Labor" or "Press Time," the PartType filter is missing.

**Severity:** MEDIUM -- Confusing results but not financially wrong (labor parts typically have 0 QuantityOnHand anyway).

**Domain/Phase:** Inventory skill.

---

### Pitfall M5: Production Station Hierarchy Confusion (Production Domain)

**What goes wrong:** Production queries count station time at both department level AND workstation level, or fail to aggregate workstation times up to department totals.

**Why it happens:** Stations have a hierarchical structure: Department (parent) > Workstation (child). `Station.ParentID` links workstations to departments. `Station.DepartmentID` provides a shortcut to the top-level department. When querying "time in Production department," the skill needs to include all workstations under that department, not just the department station itself.

**Prevention:**
- For department-level rollups: `WHERE Station.DepartmentID = @DepartmentStationID`
- For specific workstation queries: `WHERE StationID = @WorkstationID`
- Station hierarchy: `SELECT ID, StationName, ParentID, DepartmentID FROM Station WHERE IsActive = 1`
- Use `Station.ShowOnTimeClock = 1` to filter to employee-clockable stations

**Detection:** If production time for a department is 0 but individual workstations show time, the query is filtering to the parent station ID only.

**Severity:** MEDIUM -- Wrong production efficiency metrics per department.

**Domain/Phase:** Production skill (station tracking section).

---

### Pitfall M6: Report Catalog Metadata Limitations (Reports Domain)

**What goes wrong:** The reports catalog skill promises to tell users what a Crystal Report does, but the .rpt files are encrypted binary (SAP Crystal Reports proprietary OLE compound format) that cannot be parsed without the Crystal Reports SDK. The skill ends up with inferred metadata only.

**Why it happens:** The report_summary.md explicitly notes: "SQL queries and field references were inferred from report filenames and database schema knowledge. The .rpt files use SAP Crystal Reports' proprietary encrypted OLE compound document format and cannot be parsed without the Crystal Reports SDK or designer application." The skill cannot deliver "what tables does this report use" with certainty.

**Prevention:**
- Be honest about confidence levels: report metadata is INFERRED, not extracted
- The CrystalSystemReports, Report, ReportTemplate, and ReportElement tables in the database may contain some report metadata (report names, categories, parameters) that can supplement filename-based inference
- Focus the reports skill on "which report answers what question" guidance rather than technical report internals
- Cross-reference with wiki knowledge extracts for report documentation

**Detection:** If a user asks "what SQL does the Sales by Product report use" and the skill provides a confident answer, it is likely fabricated.

**Severity:** LOW -- Users asking about reports usually want guidance on which report to run, not the internal SQL. But overconfident answers about report internals would erode trust.

**Domain/Phase:** Reports Catalog skill. Set expectations correctly in the skill description.

---

### Pitfall M7: PartUsageCard vs TransPart Confusion (Production + Inventory Domains)

**What goes wrong:** Production cost queries use `TransPart` (estimated parts on orders, 2.5M rows) when they should use `PartUsageCard` (actual usage, 287K rows), or vice versa. Estimated costs diverge from actual costs, especially for custom work where material waste varies.

**Why it happens:** TransPart represents the ESTIMATED parts for an order (what the pricing formula calculated). PartUsageCard represents ACTUAL consumption (created when parts are used, time is clocked, or orders reach configured status). For a custom signage business, actual fabric usage frequently differs from estimates due to waste, reprints, or substitutions.

**Prevention:**
- For "what was the cost of this order" (actual): Use `PartUsageCard WHERE TransHeaderID = @ID`
- For "what should this order cost" (estimated): Use `TransPart WHERE TransHeaderID = @ID`
- For "estimated vs actual" comparison: Join both on TransHeaderID/TransDetailID
- Natural language mapping: "actual cost" / "real cost" -> PartUsageCard. "estimated cost" / "projected cost" -> TransPart.

**Detection:** If production costs look too uniform (exactly matching pricing formulas), the skill is probably using TransPart instead of PartUsageCard.

**Severity:** MEDIUM -- Wrong cost analysis, especially for variance reporting.

**Domain/Phase:** Production and Inventory skills.

---

## Minor Pitfalls

Mistakes that cause annoyance but are quickly fixable.

---

### Pitfall m1: Account.PrimaryContactID Join Failure (Customer Domain)

**What goes wrong:** Customer queries that join `Account.PrimaryContactID = AccountContact.ID` return NULL contact names for accounts without a primary contact set. If the join is INNER instead of LEFT, these accounts are silently excluded.

**Prevention:** Always use `LEFT JOIN AccountContact ON Account.PrimaryContactID = AccountContact.ID` for customer reports. Count NULLs to quantify the gap.

**Severity:** LOW -- Missing contact names, not wrong data.

---

### Pitfall m2: GoodsItemClassTypeID Polymorphic Link (Inventory + Production)

**What goes wrong:** TransDetail.GoodsItemID is a polymorphic foreign key. Without checking `GoodsItemClassTypeID`, the skill joins to the wrong table (Product when it should be Part, or vice versa).

**Prevention:** Rule: "When GoodsItemClassTypeID = 49 or 12000, join to Product. When GoodsItemClassTypeID = 30, join to Part. Always include ClassTypeID in the join condition."

**Severity:** LOW -- Wrong product/part names in drill-down reports.

---

### Pitfall m3: ClassTypeID 8000 vs 8001 in GLAccount Queries (Financial Domain)

**What goes wrong:** GL account queries include category/folder nodes (ClassTypeID 8000) alongside leaf accounts (ClassTypeID 8001), producing duplicate or hierarchy-contaminated results.

**Prevention:** Filter `GLAccount.ClassTypeID = 8001` for actual accounts. Use `ClassTypeID = 8000` only when navigating the chart of accounts hierarchy. This is the same pattern as v1.0's "IsCategory in GL exports is derived from ClassTypeID: 8000=category/folder, 8001=leaf account."

**Severity:** LOW -- Extra rows in results, not wrong totals (category nodes typically have no direct Ledger entries).

---

### Pitfall m4: Closeout Period Locking Ignorance (Financial Domain)

**What goes wrong:** Financial queries for historical periods return results that seem to change over time because GL periods have been reopened and entries modified after the closeout boundary was moved.

**Prevention:** For historical financial queries, note the last closeout dates. If querying a period that has been closed and potentially reopened, flag the result as potentially modified. Query: `SELECT CloseoutType, MAX(EndDate) AS LastClosedThrough FROM Closeout WHERE IsActive = 1 GROUP BY CloseoutType`

**Severity:** LOW -- Rare edge case, only matters for auditing and month-over-month comparisons.

---

### Pitfall m5: Ledger.EntryDateTime vs TransHeader.SaleDate for Revenue Timing (Financial Domain)

**What goes wrong:** GL-based revenue (using `Ledger.EntryDateTime`) and order-based revenue (using `TransHeader.SaleDate`) disagree for the same period because of batch posting. Orders may be marked Sale at 3:59 PM but GL entries post with a slightly different timestamp.

**Prevention:** Acknowledge both approaches in the skill and note that minor timing differences (<$1K) are expected. For reconciliation, use the core skill's validated revenue formula as the source of truth.

**Severity:** LOW -- Small discrepancies that are expected, not bugs.

---

## Integration Pitfalls

Mistakes specific to adding new skills alongside existing v1.0 skills.

---

### Pitfall I1: Contradicting Core Skill Business Rules

**What goes wrong:** A new domain skill (Financial, Customer, etc.) hardcodes business rules that contradict the `control-erp-core` skill. For example, the Financial skill uses `TotalPrice` instead of `SubTotalPrice` for revenue, or the Customer skill uses `OrderCreatedDate` instead of `SaleDate`.

**Why it happens:** Each skill is written independently. Without explicit cross-referencing, a new skill author may re-derive business rules and get them wrong (exactly as the original v1.0 skill files did before validation).

**Prevention:**
- All new skills MUST declare dependency on `control-erp-core` and defer to it for: TransactionType mappings, price field selection, date field selection, standard filters, StatusID definitions
- No new skill should redefine any rule that exists in `control-erp-core`
- Validation standard: every query pattern in a new skill must produce results consistent with the core skill's validated $3,053,541.85 revenue benchmark

**Severity:** CRITICAL -- This is how v1.0 started with wrong TransactionType mappings across multiple skill files.

**Domain/Phase:** All domains. Must be the first rule in every new skill file.

---

### Pitfall I2: Natural Language Routing Ambiguity Between Skills

**What goes wrong:** A user asks "what are our top customers" and the system cannot determine whether to use the Customer Intelligence skill (customer ranking by revenue), the Financial skill (AR by customer), or the Sales skill (sales by customer). Each produces slightly different results using different query approaches.

**Why it happens:** Multiple skills cover overlapping concepts. "Top customers" could mean highest revenue (sales skill), highest AR balance (financial skill), most orders (customer skill), or highest CLV (customer intelligence skill).

**Prevention:**
- Define clear domain boundaries in each skill's description
- Revenue/sales questions -> core sales skill
- Customer profiling/segmentation -> customer intelligence skill
- AR/AP/payment questions -> financial skill
- Inventory questions -> inventory skill
- Production/artwork questions -> production skill
- When ambiguous, prefer the more specific skill over the more general one
- Test with ambiguous prompts during validation

**Severity:** MEDIUM -- Wrong skill selection produces confusing (not wrong) answers.

**Domain/Phase:** All domains. Define routing rules in the skill descriptions.

---

### Pitfall I3: Duplicated Query Patterns Across Skills

**What goes wrong:** The Financial skill and the Customer Intelligence skill both implement "revenue by customer" queries but with slightly different logic (one uses TransHeader.SubTotalPrice, the other uses GL.Amount). They produce different numbers for the same question.

**Prevention:**
- Identify overlapping query patterns during skill design
- Designate one skill as authoritative for each query type
- Cross-reference: "For customer revenue analysis, see control-erp-core revenue formula"
- Use the core skill's validated patterns as the single source of truth for revenue calculations

**Severity:** MEDIUM -- Conflicting answers destroy user trust.

**Domain/Phase:** All domains. Identify overlaps during skill design phase.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation | Severity |
|-------------|---------------|------------|----------|
| Financial: AR aging | C3 (DueDate semantic overloading) | Use SaleDate for AR aging, DueDate for AP aging | CRITICAL |
| Financial: P&L | C1 (GL sign inversion) | Hardcode SUM(-Amount) for revenue, SUM(Amount) for expenses | HIGH |
| Financial: GL queries | C2 (GL view vs Ledger table) | Always use GL view; Ledger only for production costs | HIGH |
| Financial: Payments | M2 (Payment/Journal split table) | Document Payment.ID = Journal.ID pattern | MEDIUM |
| Financial: GL accounts | m3 (ClassTypeID 8000 vs 8001) | Filter ClassTypeID = 8001 for leaf accounts | LOW |
| Customer: Account names | C9 (CompanyName not AccountName) | Use Account.CompanyName consistently | MEDIUM |
| Customer: Filtering | C10 (IsClient/IsProspect/IsActive flags) | Default to IsClient = 1 AND IsActive = 1 | HIGH |
| Customer: Revenue | M3 (TransHeader polymorphism) | Always filter TransactionType = 1 | MEDIUM |
| Inventory: Quantities | C4 (Quantity field confusion) | Use QuantityAvailable, not QuantityOnHand, for "available" | HIGH |
| Inventory: Table choice | C5 (Inventory vs Part quantities) | Use Inventory table, not Part, for quantities | HIGH |
| Inventory: Part types | M4 (PartType classification) | Filter to material parts for physical inventory | MEDIUM |
| Production: TimeCard | C6 (ClassTypeID 20050 vs 20051) | Filter by ClassTypeID; never mix parent and detail | CRITICAL |
| Production: TimeCard | C7 (Geography columns) | Never SELECT *; exclude LatLonStart/LatLonEnd | MEDIUM |
| Production: Station time | C8 (Journal multi-purpose) | Always filter JournalActivityType = 45 for stations | HIGH |
| Production: Artwork | C11 (Artwork status codes) | Use artwork-specific StatusIDs, not order StatusIDs | MEDIUM |
| Production: Costs | M7 (PartUsageCard vs TransPart) | PartUsageCard = actual, TransPart = estimated | MEDIUM |
| Reports: Metadata | M6 (Binary .rpt files) | Acknowledge inferred metadata; focus on guidance | LOW |
| All domains: Integration | I1 (Contradicting core rules) | All skills defer to control-erp-core for business rules | CRITICAL |
| All domains: Routing | I2 (NL routing ambiguity) | Define clear skill domain boundaries | MEDIUM |
| All domains: Duplication | I3 (Duplicated query patterns) | Designate one authoritative skill per query type | MEDIUM |

---

## Sources

- `/Users/cain/projects/control-db-map/skills/control-erp-core/control-erp-core-SKILL.md` -- Validated v1.0 business rules (HIGH confidence)
- `/Users/cain/projects/control-db-map/skills/control-erp-financial/control-erp-financial-SKILL.md` -- Financial domain patterns (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/schemas/*.md` -- Database schema documentation (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/orders_accounting_knowledge.md` -- GL lifecycle, pricing mechanics (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/production_inventory_knowledge.md` -- Inventory formula, artwork statuses, stations (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/crm_payroll_system_knowledge.md` -- Account flags, CRM patterns (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/database_integration_knowledge.md` -- ClassTypeID mappings, Journal patterns (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/sql_queries_reference.md` -- GL integrity checks, AR/AP validation SQL (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/skill/references/relationships.md` -- FK and implicit relationships (HIGH confidence)
- `/Users/cain/projects/control-db-map/validation/control-erp-validation-results.md` -- v1.0 validation findings (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/report_summary.md` -- Crystal Reports metadata limitations (MEDIUM confidence, inferred metadata)
- `/Users/cain/projects/control-db-map/CLAUDE.md` -- Project instructions, known gotchas (HIGH confidence)
