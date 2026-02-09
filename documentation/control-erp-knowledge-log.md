# Control ERP Knowledge Log

**Purpose:** Living document capturing every validated discovery about the StoreData database. Every exploration session adds to this log. Nothing learned gets lost.

**Rule:** No assumption goes undocumented. If we queried it and confirmed it, it goes here with the date, the query, and the finding.

---

## How This Document Works

1. **Every discovery session** adds entries under the relevant section
2. **Each entry** includes: date, what we learned, how we confirmed it, and any caveats
3. **Corrections** to previous entries are noted (not deleted) so we can track our evolving understanding
4. **Open questions** are logged at the bottom until resolved, then moved to the appropriate section
5. This document feeds directly into skill updates — if it's validated here, it belongs in a skill

### Entry Template

Use this format for all new entries:

| Date | Finding | Method | Source |
|------|---------|--------|--------|
| YYYY-MM-DD | What was discovered or confirmed | SQL query, wiki page, user confirmation, or Control report comparison | File or URL where evidence exists |

---

## 1. TRANSACTION TYPES

### TransactionType Reference (Validated 2025-02-07)

| Type | Identity | Confirmed By | Notes |
|------|----------|-------------|-------|
| **1** | **Order/Sale** | Query: GROUP BY TransactionType on 2025 data. 4,241 records, $3,057,372.81 TotalPrice, $3,045,917.12 SubTotalPrice | Primary revenue record. Uses OrderNumber + InvoiceNumber (same value). Has SaleDate. StatusIDs: 0=New, 1=WIP, 2=Built, 3=Sale, 4=Closed, 9=Voided |
| **2** | **Estimate/Quote** | Investigation 2025-02-07: 71,200 total records since 2003. 655 in 2025. Confirmed via StatusText values and EstimateNumber linkage to Type 1 orders. | Lifecycle: Pending(11) → Converted(13, 56%) or Lost(12, 41%) or Voided(14, 2%). Converted estimates link to Type 1 orders via shared EstimateNumber. SaleDate always NULL. Prices may change between estimate and order (scope changes, negotiation). |
| **3** | **Recurring Order** | Wiki: sql_structure_-_transheader_table | StatusID 21. Zero records in 2025 FLS data. |
| **4** | **Credit Memo** | Wiki: sql_structure_-_transheader_table | StatusID 20 (Credit Memo), 9 (Voided). Zero records in 2025 FLS data. |
| **5** | **UNUSED** | Query: Zero records exist in 2025 data | Wiki shows "?" — undefined |
| **6** | **Service Ticket** | Query: 1 record in 2025, $0.00 | Rarely used at FLS |
| **7** | **Vendor Purchase Order** | Query: 616 records, $703,333.25. All StatusID=28 (Closed) | Uses PurchaseOrderNumber |
| **8** | **Vendor Bill (Payment)** | Query: 1,642 records, $2,139,193.80. All StatusID=4 (Completed) | Uses PurchaseOrderNumber |
| **9** | **Vendor Bill (Received)** | Query: 239 records, $437,535.07. All StatusID=4 (Completed) | Uses PurchaseOrderNumber. Wiki calls this "Receiving Document" |
| **10** | **Vendor Credit Memo** | Wiki: sql_structure_-_transheader_table | StatusID 24. Not yet validated in FLS data. |

### Previous Incorrect Definitions (for reference)
- SKILL.md said: 1=Order, 2=Estimate, 3=PO, 6=Invoice — **WRONG**
- Cheat Sheet said: 1=Estimate, 3=Order, 4=Invoice, 6=Service Ticket — **WRONG**
- Schema Reference said: 1=Order, 2=Estimate, 3=PO, 4=Service Order, 6=Invoice — **WRONG**

### Vendor Purchasing Chain
Confirmed 2025-02-07: The vendor workflow follows Type 7→8→9
- Type 7: Vendor PO created (StatusID 28 = Closed)
- Type 8: Vendor Bill Payment (StatusID 4 = Completed)
- Type 9: Vendor Bill Received (StatusID 4 = Completed)

---

## 2. STATUS IDs

### DEFINITIVE StatusID Reference (from Control Wiki — sql_structure_-_transheader_table)

**Type 1 Orders / Service Tickets:**
| StatusID | Meaning | Revenue? | Notes |
|----------|---------|----------|-------|
| 0 | New | No | Just created |
| 1 | WIP | No (unless SaleDate exists, edge case) | Work in progress |
| 2 | Built | No | Production complete, not yet invoiced |
| 3 | Sale | **Yes** | Invoiced — SaleDate set here. Revenue recognition point. |
| 4 | Closed | **Yes** | Paid in full (BalanceDue = 0). Prepaid orders auto-close. |
| 9 | Voided | **NO — always exclude** | |

**Type 2 Estimates:**
| StatusID | Meaning | Count (all-time) | % |
|----------|---------|-----------------|---|
| 11 | Pending | 326 (current) | <1% |
| 12 | Lost | 29,437 | 41% |
| 13 | Converted | 39,963 | 56% |
| 14 | Voided | 1,473 | 2% |

**Type 4 Credit Memo:** StatusID 20 = Credit Memo, 9 = Voided

**Type 7 Purchase Orders:**
| StatusID | Meaning |
|----------|---------|
| 25 | Requested |
| 26 | Approved |
| 27 | Ordered |
| 28 | Closed |
| 29 | Cancelled |
| 30 | Rejected |
| 31 | Received |
| 6 | Open |

**Type 8 Bill / Type 9 Receiving Document:** StatusID 4 = Closed, 6 = Open, 9 = Voided

**Type 10 Vendor Credit Memo:** StatusID 24 = Vendor Credit Memo

**Type 3 Recurring Order:** StatusID 21 = Recurring Order

> **Note:** StatusID 1600/1700 from the original "Cheat Sheet" do NOT appear in the official wiki documentation. They may be deprecated or incorrect. Use the wiki reference above as authoritative.

---

## 3. PRICING FIELDS

### Revenue Field Selection (Validated 2025-02-07)

| Field | Value (2025 Type 1) | Matches "Income"? | Notes |
|-------|--------------------|--------------------|-------|
| `TransHeader.SubTotalPrice` | $3,052,952.52 | **YES — 99.98% match** | Pre-tax total. USE THIS for revenue. |
| `TransHeader.TotalPrice` | $3,064,581.48 | No (~$11K over) | Includes tax. |
| Known actual income | $3,053,541.85 | Target | Per FLS accounting |
| Variance | -$589.33 | 0.02% | Acceptable — may be rounding, timing, web orders |

### Header vs Detail Gap (Validated 2025-02-07)
- `SUM(TransHeader.SubTotalPrice)`: $3,052,952.52
- `SUM(TransDetail.SubTotalPrice)`: $3,045,889.31
- **Gap: $7,063.21**
- Cause: Header-level discounts/adjustments not distributed to line items, rounding, order-level pricing overrides
- **Rule: Always use TransHeader.SubTotalPrice for revenue totals**

### Date Field for Revenue (Validated 2025-02-07)
- **Use `SaleDate`** for revenue queries on Type 1 — reflects when sale was finalized
- **NOT `OrderCreatedDate`** — reflects when order was entered
- `SaleDate` can be NULL on Type 1 (new/unsold orders) — always filter `SaleDate IS NOT NULL`
- Type 2 records: `SaleDate` is ALWAYS NULL

### Date Field for Estimates (Corrected 2025-02-07)
- **Use `EstimateCreatedDate`** for Type 2 estimate date filtering
- **NOT `OrderCreatedDate`** — this is NULL for all Type 2 records
- `EstimateCreatedDate` is populated on all Type 2 records

---

## 4. PRODUCT ARCHITECTURE

### Container Products (Validated 2025-02-07)

#### DyeSub Print (Primary Container — ~$464K of $3.05M in 2025)
- NOT a product — it's a configurable framework
- **Never report "DyeSub Print" as a product name**
- Houses majority of printed product lines through variables

| Variable | Correct Name | VariableID | Column | Records | Purpose |
|----------|-------------|-----------|--------|---------|---------|
| Product Category | `FP_ProductDescription` | **11053** | ValueAsStr25 | 13,268 | Category grouping (Feather Flags, Swing Flags, etc.) |
| Product SKU | `FP_ProductID` | **11052** | ValueAsStr25 | 13,268 | Specific product identifier |

> ⚠️ **CORRECTION:** Originally documented as `FP_ProductionDescription` — the correct name is `FP_ProductDescription` (no "ion"). Confirmed 2025-02-07.

**Template Variable IDs (unused in transactions — don't query these):**
- FP_ProductDescription: 13174, 13490, 13547 (0 records each)
- FP_ProductID: 13548 (0 records)

#### DyeLux-Full Print Table Cover (~$479K in 2025)

| Variable | VariableID | Records in TransDetailParam | Status |
|----------|-----------|---------------------------|--------|
| TC_FabricCategory | 13096 | **0** | NOT SAVED to transactions |
| TC_ProductName | 10591, 13099 | **0** | NOT SAVED to transactions |
| TC_FabricFR | 13123 | 11,941 | Fire Retardant flag — IS saved |

**How to identify Table Cover products (since TC_ variables don't persist):**
```sql
-- DyeLux Full Print
WHERE td.Description LIKE '%Dyelux%Table Cover%' OR td.Description LIKE '%FULL%Table Cover%'
-- All table covers
WHERE td.Description LIKE '%Table Cover%'
-- By GoodsItemID
WHERE td.GoodsItemID = 10026
```

### DyeSub Print Category Revenue (2025 — CORRECTED 2025-02-07, validated vs Control report)

| Category | Orders | Revenue |
|----------|--------|---------|
| Swing Flags | 490 | $615,203 |
| Feather Flags | 544 | $485,666 |
| Banners | 296 | $259,212 |
| SEG | 91 | $69,728 |
| Tear Drop Flags | 61 | $53,041 |
| Banner Stands | 116 | $45,505 |
| Fab Frames | 65 | $40,474 |
| Custom Flag | 101 | $39,209 |
| Golf Flag 14x20 | 151 | $38,821 |
| Table Runner | 112 | $35,733 |
| Pillow Case Frames | 60 | $33,553 |
| Pop Up Banners | 16 | $28,841 |
| Printed Dyesub Paper | 11 | $20,619 |
| Golf Flag 5x8 | 102 | $13,485 |
| Backdrop | 20 | $5,823 |
| Trombone cover | 6 | $2,480 |
| By The Yard | 5 | $1,690 |
| Golf Cart Flag | 7 | $1,408 |
| Bell Covers | 4 | $943 |
| Swoopper Flags | 3 | $843 |
| CUSTOM BURGEE | 2 | $0 |
| **TOTAL** | — | **$1,793,445** |

**DyeSub Print = 58.7% of total FLS revenue ($3,052,953)**
Validated against Control "Sales by Product" report: $1,793,442 (variance: $3)

### Non-Container Product Groups (2025 — CORRECTED 2025-02-07)

| Product Group | Pattern Used | Revenue |
|---------------|------------|---------|
| DyeLux Full Print Table Covers | `%Dyelux%Table Cover%` / `%FULL%Table Cover%` | $479,184 |
| Other Products/Services | Everything else | $256,563 |
| Garments/Apparel | `%Garment%` / `%Embroidered%` | $224,430 |
| Design/Artwork | `%Design%` / `%Artwork%` | $73,804 |
| Other Table Covers | `%Table Cover%` (non-DyeLux) | $68,997 |
| Shipping | `%Shipment%` / `%Shipping%` / `%Freight%` | $65,714 |
| Tents/Pop-Ups | `%Tent%` / `%Pop%Up%` | $61,191 |
| Banners (accessories/hardware) | `%Banner%` (non-DyeSub) | $13,184 |
| SEG (accessories/hardware) | `%SEG%` (non-DyeSub) | $5,695 |
| Flags (accessories/hardware) | `%Flag%` (non-DyeSub) | $4,852 |

**Note:** With corrected TransDetailParam queries, DyeSub Print properly captures most flag, banner, and SEG products. The hardware/accessories groups dropped dramatically (Flags: $871K → $4.9K) because those line items are actually DyeSub Print products that were previously excluded by the incorrect IsActive/ParentClassTypeID filters.

---

## 5. SCHEMA & JOIN PATTERNS

### TransDetailParam → Variable (Validated 2025-02-07, CORRECTED 2025-02-07)

| Column | Links To | Notes |
|--------|---------|-------|
| `TransDetailParam.ParentID` | `TransDetail.ID` | FK to line item |
| `TransDetailParam.VariableID` | `Variable.ID` | FK to variable definition |
| `TransDetailParam.TransHeaderID` | `TransHeader.ID` | Denormalized FK (convenience) |

**⚠️ CRITICAL CORRECTION (2025-02-07):** Do NOT filter TransDetailParam by `IsActive = 1` or `ParentClassTypeID = 10100`. These filters exclude ~75% of valid parameter records. The original validation query included these filters, producing DyeSub Print revenue of $464K. The correct value (without filters) is **$1,793,445** — validated to within $3 of Control's "Sales by Product" report.

**Correct JOIN pattern:**
```sql
INNER JOIN TransDetailParam tdp ON tdp.ParentID = td.ID
    AND tdp.VariableID = @VariableID  -- Always filter by specific VariableID
-- Do NOT add: AND tdp.IsActive = 1
-- Do NOT add: AND tdp.ParentClassTypeID = 10100
```

### Value Storage in TransDetailParam (Validated 2025-02-07)

| ValueType | Column | Use |
|-----------|--------|-----|
| 256 | `ValueAsStr25` (nvarchar 25) | FP_ and TC_ variables use this |
| 256 | `ValueAsString` (text) | Longer string values |
| 11 | `ValueAsFloat` | Numeric values |
| — | `ValueAsInteger` | Integer values |
| — | `ValueAsDateTime` | Date values |

### ClassTypeID Reference (Validated 2025-02-07)

| ClassTypeID | Entity |
|------------|--------|
| 10000 | TransHeader |
| 10100 | TransDetail |
| 10150 | TransDetailParam |
| 10200 | TransMod |
| 10300 | TransPart |

---

## 6. USER DEFINED FIELDS (UDFs)

### Web Import UDFs (Validated 2025-02-07)

| UDF Name | Purpose | Storage | Date Confirmed |
|----------|---------|--------|---------------|
| Import_Order_Number | Identifies web-imported orders (via CHAPI). Non-null = web import. | TransHeaderUserField | 2025-02-07 |
| Import_Order_Date | Original order date from website | TransHeaderUserField | 2025-02-07 |
| Import_Estimate_Number | Website estimate/cart number | TransHeaderUserField | 2025-02-07 |
| Import_Order_Amount | Order amount from website (before tax/shipping) | TransHeaderUserField | 2025-02-07 |
| Import_Tax_Amount | Tax amount from website | TransHeaderUserField | 2025-02-07 |
| Import_Shipping_Amount | Shipping amount from website | TransHeaderUserField | 2025-02-07 |
| Import_Total_Amount | Total amount from website | TransHeaderUserField | 2025-02-07 |

**Note:** These UDF fields store the website's own pricing breakdown separately from Control's order pricing. Useful for reconciling web vs Control amounts.

### Web Import Statistics (Corrected 2025-02-07 — deduplicated)

**⚠️ CRITICAL DATA QUALITY RULE:** Duplicate `Import_Order_Number` values = cloned orders in Control, NOT separate web orders. Only the first order (lowest OrderNumber) per Import_Order_Number is a true web import. Control was fixed on Feb 7, 2026 to prevent future clones, but historical data requires deduplication.

**Dedup query pattern:**
```sql
-- Use ROW_NUMBER to identify true web imports (first order per Import_Order_Number)
WITH WebOrders AS (
    SELECT th.*, uf.Import_Order_Number,
        ROW_NUMBER() OVER (PARTITION BY uf.Import_Order_Number ORDER BY th.OrderNumber ASC) AS rn
    FROM TransHeader th
    INNER JOIN TransHeaderUserField uf ON th.ID = uf.TransHeaderID
    WHERE uf.Import_Order_Number IS NOT NULL AND uf.Import_Order_Number != ''
        AND th.TransactionType = 1 AND th.IsActive = 1
)
SELECT * FROM WebOrders WHERE rn = 1  -- Only first order per Import_Order_Number
```

| Metric | Raw (before dedup) | Deduplicated (correct) |
|--------|-------------------|----------------------|
| 2025 Web orders | 467 | **429** |
| 2025 Web revenue | $172,164 | **$150,750** |
| % of revenue | 5.64% | **4.94%** |
| % of order count | 11.19% | **10.28%** |
| Clones excluded | — | 38 orders / $21,414 |

- Web imports = ~4.9% of 2025 revenue, ~10.3% of order count
- Predominantly small B2C orders (table covers, flags, dyesub prints)
- Started in 2023, steady growth through 2025

*More UDFs to be documented as discovered*

---

## 7. BUSINESS WORKFLOW RULES

### Estimate → Order Lifecycle (Validated 2025-02-07)
```
Type 2 Estimate Created (StatusID 11 = Pending)
    │
    ├──→ StatusID 13 = Converted (56%) → Creates Type 1 Order (linked via EstimateNumber)
    │        └── Prices may change during conversion (scope changes, negotiation)
    │
    ├──→ StatusID 12 = Lost (41%) → Customer did not proceed
    │
    └──→ StatusID 14 = Voided (2%) → Estimate cancelled
```

**Key linkage:** Type 2 and Type 1 share `EstimateNumber`. When an estimate converts, a Type 1 order is created with the same EstimateNumber. The Type 1 order gets its own `OrderNumber` and `InvoiceNumber`.

**Revenue impact:** Converted estimate revenue is already captured in Type 1 totals. Type 2 SubTotalPrice represents *quoted* value, not *sold* value. Never include Type 2 in revenue calculations.

**Quote-to-close analysis:** Compare Type 2 SubTotalPrice (quoted) vs matched Type 1 SubTotalPrice (sold) to measure scope change / price negotiation impact.

### Estimate Conversion Rate
- Historical (all time): 56% convert, 41% lost, 2% voided
- 71,200 total estimates since November 2003

### Order Lifecycle — Type 1 (Validated 2025-02-07, confirmed by user)

```
Type 1 Order Created (StatusID 1 = WIP / Work in Process)
    │
    │   [Order is processed, produced, may receive deposit]
    │   [Many orders are prepaid — deposit = full order value at placement]
    │
    ├──→ StatusID 2 = Built (rarely used — indicates production complete)
    │
    ├──→ StatusID 3 = Sale (order shipped/fulfilled, customer invoiced)
    │        │   SaleDate is set at this point — THIS is when revenue is recorded
    │        │
    │        ├──→ StatusID 4 = Closed (automatically, if no balance due)
    │        │        Prepaid orders go directly from Sales → Closed
    │        │        because deposit covered the full amount
    │        │
    │        └──→ Remains at Sales status (if balance due > 0)
    │                 Moves to Closed when payment received in full
    │
    └──→ StatusID 9 = Voided (order cancelled)
```

**Key Business Rules:**
1. **SaleDate** = when order is marked as "Sales" (shipped/invoiced). This is the revenue recognition date. Always use this for revenue reporting.
2. **Sales vs Closed distinction:**
   - **Sales** (StatusID 3) = invoiced but may still have balance due
   - **Closed** = fully paid, no balance due
   - Prepaid orders skip the "unpaid Sales" state — they go directly to Closed because the deposit already covered the full amount
3. **BalanceDue** differentiates: Sales with BalanceDue > 0 = awaiting payment. Closed = BalanceDue = 0.
4. **Most orders are prepaid** — deposit equal to order value is collected at placement. These auto-close on sale.
5. **Revenue timing** — revenue is recorded at SaleDate (when marked Sales), NOT when Closed. An order in Sales status with a balance due is still recognized revenue.

### Order Payment Patterns
- **Prepaid (majority):** Order placed → Deposit = full value → Produced → Marked Sales → Auto-Closed (BalanceDue = $0)
- **Balance due:** Order placed → Partial/no deposit → Produced → Marked Sales (BalanceDue > $0) → Payment received → Closed
- **Revenue recognition** happens at Sales status regardless of payment status

### StatusID for Type 1 Orders (Updated 2025-02-07)

| StatusID | StatusText | Meaning | Include in Revenue? | BalanceDue |
|----------|-----------|---------|-------------------|------------|
| 0 | New | Just created | No | Varies |
| 1 | WIP | Work in Process — order entered, being produced | Only if SaleDate exists (unlikely) | Varies |
| 2 | Built | Production complete (rarely used) | Only if SaleDate exists | Varies |
| 3 | Sale | Shipped/invoiced, SaleDate set | **YES** | May be > $0 |
| 4 | Closed | Fully paid | **YES** | $0 |
| 9 | Voided | Cancelled | **NO** | — |

### Web Import Workflow (Confirmed 2025-02-07 by user + validated by query, dedup rule added same day)
- FLS does NOT use Control's built-in web sales feature
- Website orders are imported via CHAPI as Type 1 orders
- Identifiable by `Import_Order_Number` UDF having a non-null/non-empty value
- **DEDUP REQUIRED:** Duplicate Import_Order_Number = cloned order in Control, NOT a second web order. Only the first order (lowest OrderNumber) per Import_Order_Number is a true web import.
- Control was fixed Feb 7, 2026 to prevent future clones, but historical data still requires dedup
- Additional UDFs capture website pricing breakdown (order, tax, shipping, total amounts)
- Products: mostly DyeLux table covers, flags, dyesub prints
- Profile: small B2C orders averaging ~$351 in 2025 (deduplicated)

### Vendor Purchasing Workflow (Validated 2025-02-07)
- Type 7: Vendor PO created → StatusID 28 (Closed)
- Type 8: Vendor Bill Payment → StatusID 4 (Completed)
- Type 9: Vendor Bill Received → StatusID 4 (Completed)

---

## 8. GENERAL LEDGER / ACCOUNTING

### GL Transaction Flow (Documented 2026-02-07 from user-provided screenshots)

Control uses full double-entry accounting. GL entries are created automatically at each stage of the order lifecycle. Confirmed via screenshots of orders 133819 (prepaid) and 133841 (unpaid/AR).

**Stage 1: New Order Created**
Per line item:
- Debit **WIP** (line item amount)
- Credit **Orders Due** (line item amount)

**Stage 2: Deposit/Prepayment Received**
At order level:
- Debit **Undeposited MCVisa** (or other payment method account) — payment amount
- Credit **Order Prepayments** — payment amount

**Stage 3a: Marked Sales — WITH prepayment covering total (auto-closes)**
Per line item:
- Debit **Orders Due** / Credit **WIP** — reverses Stage 1
- Debit **Order Prepayments** — applies prepayment (order level)
- Credit **[Revenue Account]** — recognizes revenue (e.g., "Feather Flags", "Shipping")
- Debit **Cost of Goods Sold** / Credit **Inventory** — recognizes COGS

**Stage 3b: Marked Sales — WITHOUT deposit (generates AR)**
Per line item:
- Debit **Orders Due** / Credit **WIP** — reverses Stage 1
- Debit **Accounts Receivable** — creates AR balance (order level)
- Credit **[Revenue Account]** — recognizes revenue
- Debit **Cost of Goods Sold** / Credit **Inventory** — recognizes COGS

**Key difference between 3a and 3b:** Prepaid orders debit Order Prepayments (clearing the prepayment liability). Unpaid orders debit Accounts Receivable (creating AR). Revenue is recognized in both cases at the Sales status transition.

### GL Account Names (Observed from screenshots, mapped to GLClassificationType)

| Account Name | GLClassificationType | Code | When Used |
|-------------|---------------------|------|-----------|
| WIP | CurrentAsset | 1002 | Order creation → reversed at Sales |
| Inventory | Inventory | 1003 | Reduced at Sales (COGS recognition) |
| Undeposited MCVisa | UndepositedFunds | 1007 | Payment received (credit card) |
| Accounts Receivable | AccountsReceivable | 1001 | Sales without deposit |
| Orders Due | CurrentLiability | 2002 | Order creation → reversed at Sales |
| Order Prepayments | CurrentLiability | 2002 | Deposit received → applied at Sales |
| Cost of Goods Sold | CostGoodsSold | 5001 | Sales (inventory → expense) |
| Feather Flags | Income | 4000 | Product-specific revenue account |
| Dyelux Flag | Income | 4000 | Product-specific revenue account |
| Shipping | Income | 4000 | Shipping charges revenue |

### Database Architecture (Documented 2026-02-07 from Control Wiki)

**⚠️ CRITICAL: "GL" is a VIEW, not a table.** The actual data lives in the **Ledger** table.

```sql
-- GL View definition (filters to on-balance sheet only)
CREATE VIEW GL AS
SELECT * FROM Ledger WHERE OffBalanceSheet = 0
```

- **GL view** = on-balance sheet entries only → use for **financial reports**
- **Ledger table** = all entries including off-balance sheet → use for **cost analysis**
- `Ledger.OffBalanceSheet` = 0 (real financial entry) or 1 (cost tracking only)

**Off-balance sheet entries** are created when parts are used on orders but their costs are expensed at purchase (not accrued). This prevents double-counting expenses. Parts with "Accrue Costs" enabled generate on-balance sheet entries; parts without generate off-balance sheet entries.

### GL/Ledger Table Structure (from wiki: sql_structure_-_gl_table)

Key columns:
| Column | Type | Purpose |
|--------|------|---------|
| GL.ID | int | Primary key |
| GL.Amount | decimal(18,4) | Transaction amount |
| GL.GLAccountID | int | FK → GLAccount.ID |
| GL.GLClassificationType | int | Account classification code |
| GL.GLClassTypeName | varchar(50) | Account classification name |
| GL.JournalID | int | FK → Journal.ID (batch entry) |
| GL.TransactionID | int | FK → TransHeader.ID |
| GL.TransactionClassTypeID | int | ClassType of linked transaction |
| GL.TransDetailID | int | FK → TransDetail.ID (line item) |
| GL.TransDetailClassTypeID | int | ClassType of linked line item |
| GL.AccountID | int | FK → Account.ID (customer) |
| GL.DivisionID | int | FK → EmployeeGroup.ID |
| GL.Entry | datetime | Entry date |
| GL.Reconciled | bit | Whether reconciled |
| GL.IsTaxable | bit | Taxable flag |
| GL.ClassTypeID | int | Always 8900 (GL Entry) |
| GL.EntryType | int | 0, 1 (manual), or 2 |

### GLClassificationType Reference (from wiki)

**Assets (1000s):**
1000 BankAccount, 1001 AccountsReceivable, 1002 CurrentAsset, 1003 Inventory, 1004 FixedAsset, 1005 OtherAsset, 1006 Depreciation, 1007 UndepositedFunds

**Liabilities (2000s):**
2000 AccountsPayable, 2001 CreditCardAccount, 2002 CurrentLiability, 2003 LongTermLiability, 2004 OtherLiability, 2005 SalesTaxLiability, 2006 OtherTaxLiability, 2007 RoyaltyLiability, 2008 CommissionLiability

**Equity (3000s):** 3000 Equity

**Income (4000s):** 4000 Income, 4001 OtherIncome

**Expenses (5000s):** 5000 Wage, 5001 CostGoodsSold, 5002 Expense, 5003 OtherExpense

**Category (8000s):** 8000 AccountGroup, 8001 AssignedExpenseGroup

### Payment & Journal Architecture (from wiki)

**Journal** is the master activity tracker. **Payment is a split table** — `Payment.ID = Journal.ID` (shared primary key).

**Payment ClassTypeIDs:**
| ClassTypeID | Meaning |
|-------------|---------|
| 20000 | Master Payment |
| 20001 | Order Payment |
| 20002 | Over Payment |
| 20003 | Change Return |
| 20004 | Order Refund |
| 20005 | Credit Refund |
| 20006 | Order Credit Memo |
| 20007 | Failed Payment |
| 20008 | Tip Recorded |
| 20009 | Bill Payment |
| 20037 | Master Bill Payment |

**Key Payment fields:** PaymentDate, TenderType, BankAccountID→GLAccount, PaymentAccountID, Undeposited (bit)

**Journal ActivityType codes (order lifecycle):**
1=Created, 2=Edited, 3=Estimate Converted, 4=Estimate Lost, 5=Order Marked WIP, 6=Order Marked Built, 7=Order Marked Sale, 8=Order Marked Closed, 9=Order Voided

**Journal JournalActivityType codes:**
2/4=Payment, 13=Close Out, 32=Purchase Order, 36=Bill, 37=Receiving Document, 38=Deposit Made, 39=Account Reconciliation, 40=Journal Entry, 43=Bill Payment

### TransHeader Financial Fields (from wiki — confirmed, with corrections)

| Field | Type | Purpose |
|-------|------|---------|
| **DueDate** | datetime | **⚠️ This is the PRODUCTION/SHIPPING due date (when order is due to customer), NOT the payment due date.** |
| IsFirmDueDate | bit | Whether production due date is firm |
| BalanceDue | decimal(18,4) | Outstanding balance |
| PaymentTotal | decimal(18,4) | Total payments received |
| FinanceChargeAmount | decimal(18,4) | Finance charges applied |
| LastFinanceChargeDate | datetime | Last finance charge date |
| WriteOffAmount | decimal(18,4) | Written off amount |
| POPaymentTermsID | int | FK → PaymentTerms.ID |
| ClosedDate | datetime | When order was closed |
| BuiltDate | datetime | When order was marked built |
| SaleDate | datetime | When order was marked sale (invoice date) |
| VoidedDate | datetime | When order was voided |
| UseProgressBilling | bit | Progress billing flag |
| UsePaymentPlan | bit | Payment plan flag |
| ScheduledPaymentPlanID | int | FK to payment plan |
| CreditMemoAmount | float | Credit memo amount |

**Payment Due Date Theory (unconfirmed, logically sound):**
The AR report's aging "Due Date" column is likely **calculated at report runtime** as `SaleDate + PaymentTerms.DaysUntilDue` (or similar). Evidence:
- Different customers have different payment terms (Account.PaymentTermsID)
- Storing a calculated payment due date on each order would create data integrity issues if terms change
- The AR report shows a "DueDate" column and "Days Late" — these are likely computed from SaleDate + terms, not from TransHeader.DueDate
- TransHeader.DueDate appears in the AR report as a separate concept (production timing)

**⚠️ This needs validation** — either via the PaymentTerms table structure or by comparing AR report dates against TransHeader.DueDate values for the same orders.

### Account Payment/Credit Fields (from wiki — confirmed)

| Field | Type | Purpose |
|-------|------|---------|
| HasCreditAccount | bit | **This is the `*` prefix in the AR report** |
| CreditBalance | decimal(18,4) | Current credit balance |
| CreditLimit | decimal(18,4) | Credit limit |
| CreditApprovalDate | datetime | When credit was approved |
| PaymentTermsID | int | FK → PaymentTerms.ID (AR terms) |
| VendorPaymentTermsID | int | FK → PaymentTerms.ID (AP terms) |
| VendorCreditBalance | float | Credit balance with vendor |
| Is1099Vendor | bit | 1099 vendor flag |

### Wiki Source

All GL/Payment/Account table documentation sourced from: https://control.cyriouswiki.com/
- GL table: sql_structure_-_gl_table
- Payment table: sql_structure_-_payment_table
- Journal table: sql_structure_-_journal_table
- TransHeader: sql_structure_-_transheader_table
- Account: sql_structure_-_account_table
- Off-balance sheet: off-balance_sheet_gl_entries

**~130+ table documentation pages available** on the wiki for future reference.

---

## 9. ACCOUNTS RECEIVABLE

### AR Report Structure (Documented 2026-02-07 from A_R_Detail.xls)

**Report: "A/R Detail"** — Control's standard AR aging report.

**Current snapshot (as of 2026-02-07):**
- 30 customers with open balances
- 119 open invoices (121 per report — minor count difference)
- **Total AR: $80,899**

**AR by Payment Terms:**

| Terms | Open Invoices | AR Balance |
|-------|--------------|------------|
| Net 30 | 81 | $46,085 |
| Credit Card | 8 | $19,438 |
| Due Upon Receipt | 20 | $9,903 |
| Net 45 | 1 | $4,200 |
| 50%/50% Net 30 | 4 | $943 |
| Cash Customer | 3 | $136 |
| Net 10 | 1 | $128 |
| Net 15 | 1 | $67 |
| **Total** | **119** | **$80,899** |

**AR Aging (from SaleDate):**

| Bucket | Invoices | AR Balance | % of Total |
|--------|----------|------------|-----------|
| 0-30 days | 105 | $75,462 | 93.3% |
| 31-60 days | 10 | $5,258 | 6.5% |
| 61-90 days | 1 | $35 | <0.1% |
| 90+ days | 3 | $144 | 0.2% |

**Key observations:**
- AR is very healthy — 93% current, only $5.4K past 30 days
- Net 30 dominates (57% of AR balance, 68% of invoices)
- Credit Card AR ($19.4K, 8 invoices, oldest from Sept 2025) — **investigated, not a problem.** 7 of 8 invoices are less than 2 weeks old, just awaiting first charge. The one Sept item (Order 132444, Knights of Columbus) is a $4.13 residual — sales tax not covered by a prepayment. The $12,500 Signworks order (64% of CC AR) was sold Feb 4, just 3 days prior. These accounts are on "Credit Card" payment terms but haven't been charged yet — not failed/declined transactions.
- 90+ day items ($144) are straggler small amounts
- Aligns with predominantly prepaid business model — AR is small relative to $3M+ revenue

**Payment Terms table confirmed (2026-02-07):**
- 21 active terms in PaymentTerms table
- Key fields: TermsName, GracePeriod (days), InterestRate, FeesBasedOnSaleDate
- Aging statement text buckets built into terms (0-30, 31-60, 61-90, 90+)
- Early payment discount support
- `FeesBasedOnSaleDate` field confirms AR aging calculations reference SaleDate

**DueDate clarification (user-confirmed 2026-02-07):**
- TransHeader.DueDate = **production/shipping due date** (when order is due to customer)
- Payment due date for AR aging = calculated from `SaleDate + PaymentTerms.GracePeriod`
- Evidence: Orders 133508 (SaleDate=Jan 7, DueDate=Jan 7) and 133593 (SaleDate=Jan 16, DueDate=Jan 20) — DueDate tracks shipping, not payment

**Important AR report behaviors:**
1. **Aging likely uses calculated payment due date**, NOT TransHeader.DueDate (which is production/shipping due date). Payment due date is probably computed from SaleDate + customer's PaymentTerms at report runtime. The AR report's "Due Date" column and "Days Late" values are calculated, not stored on TransHeader.
2. **WIP/Built orders are included** in the AR report — these aren't technically AR yet (not invoiced), but the report shows them for visibility into upcoming receivables.
3. **`*` prefix on customer name** = charge account customer (Account.HasCreditAccount = 1). These customers have payment terms and don't prepay.
4. **Bold/Italic lines** = Progress Bill Orders
5. **Credit Balance** shown per customer (some may have overpayments)

**Report columns:**
Invoice | Description | SaleDate | DueDate | Age | DaysLate | OrderedBy | OrderTotal | WIP/Built | Current | 0-30 | 31-60 | 61-90 | 90+ | BalanceDue

**Business insight:** AR is very healthy — $80.9K total with only $329 past due. The bulk ($35.2K) is WIP/Built not yet invoiced, and $45.4K is current. This aligns with the predominantly prepaid business model — most customers pay upfront (deposits), so AR is small relative to $3M+ revenue.

---

### AP Report Structure (Documented 2026-02-07 from A_P_Aging_Detail.xls)

**Report: "A/P Aging Detail"** — Control's standard AP aging report (SYSTEM\\AP Reports_APAgingReport).

**Current snapshot (as of 2026-02-07):**
- 35 vendors with open balances
- 66 open bills
- **Total AP: $125,318.74**
- Total bill amounts: $254,079.48 (includes paid portions)

**AP Aging:**

| Bucket | Amount | % of Total |
|--------|--------|-----------|
| Current | $69,359 | 55.3% |
| 0-30 days | $8,876 | 7.1% |
| 31-60 days | $513 | 0.4% |
| 61-90 days | $10 | <0.1% |
| 90+ days | $46,562 | 37.2% |
| **Total** | **$125,319** | |

**Top vendors by AP balance:**

| Vendor | AP Balance | Aging |
|--------|-----------|-------|
| Mark Sales, Inc. | $46,562 | 100% in 90+ ⚠️ |
| Electronics for Imaging (EFI) | $13,260 | Current |
| American Express Credit Card | $12,554 | Current |
| Pritchard Paper Products | $12,424 | Current |
| Jeanquart Investments, LLC | $9,150 | Current |
| Sentry Insurance | $3,094 | Current |
| SanMar | $3,031 | Mostly current |

**Key observations:**
- AP is healthy excluding Mark Sales — 96% of remaining AP is current
- Mark Sales $46,562 (90+ days) is 37% of total AP — needs investigation (disputed? payment arrangement? stale?)
- AmEx CC bill ($12,554 current) — likely a monthly statement
- Jeanquart Investments and Goettelman Marital QTip Trust ($9,150 + $3,000) — look like rent/lease payments
- Bill numbering: 30xxx range (vs order numbering 133xxx) — TransactionType 8 bills use separate sequence

**GL Entry Pattern for Bills (from screenshot, Order 30938):**
- Debit Freight: $140.85 (expense account)
- Debit Cost of Goods Sold: $1,887.50 (expense account)
- Credit Accounts Payable: $2,028.35 (liability)
- Total debits = total credits (balanced)
- Pattern: Expenses recognized immediately at bill creation, AP liability created
- Mirror of AR: AR debits receivable/credits revenue; AP debits expense/credits payable

**AP Report columns:** Bill Number, Reference Number, Due Date, Bill Date, Invoice Date, Bill Amount, Current, 0-30, 31-60, 61-90, Over 90, Total, Age

**AP Aging calculation:** Uses "Age" column (days) — appears to be DATEDIFF from Due Date to report date. Negative age = not yet due (current).

---

### GLAccount NodeID Mapping (Documented 2026-02-07 from GL_SQL_Export.html)

**Source:** GL SQL Export — balance sheet accounts from GLAccount table with live balances.

**Table structure:** GLAccount fields include FormattedName, Balance, NodeName, ExportName, ExportNumber, IsActive, IsCategory (1=folder, 0=leaf), NodeID, NodeClassTypeID (8000=category, 8001=account), FormattedPath (hierarchy), GLClassificationType.

**172 total rows:** 29 categories (folders), 143 leaf accounts, 32 with non-zero balances.

**⚠️ This export contains BALANCE SHEET accounts only** (GLClassificationType 1xxx/2xxx/3xxx). Revenue (4xxx) and Expense (5xxx) accounts are NOT included — need separate query or export to get those NodeIDs.

**Key System Account NodeIDs (for Ledger/GL queries):**

| NodeID | Account | GLClass | Balance | Notes |
|--------|---------|---------|---------|-------|
| 11 | WIP | 1002 | $157,058.45 | Presale Asset — debit on new orders |
| 12 | Built | 1002 | $0.00 | Presale Asset — intermediate status |
| 14 | Accounts Receivable | 1002 | $59,446.91 | Debit on unpaid sales |
| 21 | Orders Due | 2002 | -$157,252.06 | Presale Liability — credit on new orders |
| 22 | Accounts Payable | 2000 | -$60,142.84 | Credit on bills |
| 23 | Customer Credit Balance | 2002 | -$2,163.75 | Customer overpayments |
| 24 | Order Prepayments | 2002 | -$31,910.04 | Deposits received before sale |
| 25 | Vendor Credit | 1002 | $0.00 | |
| 33 | Inventory Adjustment | 1002 | $0.00 | |
| 34 | Cost of Built - FGI | 1002 | $0.00 | Finished goods intermediate |
| 35 | Warehouse Transfer | 1002 | $0.00 | |
| 89 | Customer Credit Transfer | 1002 | $0.00 | |
| 90 | Cash-Checking | 1000 | $99,167.87 | Primary bank account |
| 91 | Undeposited Cash | 1007 | -$13.09 | |
| 92 | Undeposited MCVisa | 1007 | $0.00 | MC/Visa payments pending deposit |
| 93 | Undeposited Checks | 1007 | $447.02 | |
| 543 | Undeposited Discover | 1007 | $0.00 | |
| 10137 | Undeposited Amex | 1007 | $0.00 | |
| 10412 | Cash-MM (Money Market) | 1000 | $646,849.82 | |
| 10414 | Inventory | 1003 | $135,094.00 | Main inventory account |

**GLClassificationType Reference (balance sheet):**

| Code | Category | Count | Description |
|------|----------|-------|-------------|
| 1000 | Cash/Bank | 5 | Bank accounts, cash |
| 1002 | Other Current Assets | 24 | AR, presale assets, transfers |
| 1003 | Inventory | 5 | Raw materials, finished goods |
| 1004 | Fixed/Other Assets | 25 | Equipment, vehicles, notes receivable |
| 1005 | Presale Assets | 5 | Deposits, NR non-current |
| 1007 | Undeposited Funds | 13 | Payment gateway holding accounts |
| 2000 | Accounts Payable | 3 | AP, AP contra, US Bank CC |
| 2001 | Credit Cards | 3 | Amex, Capital One, Chase Ink |
| 2002 | Current Liabilities | 37 | Order prepayments, payroll, taxes, presale liabilities |
| 2003 | Long-term Liabilities | 11 | Notes payable, equipment/vehicle payable |
| 2005 | Presale Liabilities | 4 | Sales tax (Door, WI) |
| 3000 | Equity | 7 | Common stock, retained earnings |
| 8000 | Category/Folder | 29 | Not actual accounts — grouping nodes |

**Key balance sheet observations:**
- WIP ($157,058) ≈ Orders Due (-$157,252) — double-entry balanced within $194
- AR GL balance ($59,447) < AR report ($80,899) — difference is WIP/Built orders in AR report not yet posted to AR account
- AP GL balance ($60,143) < AP report ($125,319) — likely includes Mark Sales $46,562 and bills in various states
- Cash position: $99K checking + $647K money market = $746K liquid
- Inventory: $135K
- Retained Earnings: $13.36M (cumulative since inception)

**✅ RESOLVED:** Revenue and Expense account NodeIDs now documented from Trial Balance Export (see below).

---

### Payment Posting Patterns (Documented 2026-02-07 from Account Registers)

**Source:** GL Account Registers for Order Prepayments (NodeID 24) and Accounts Receivable (NodeID 14), January 2026.

**Key discovery: Orders flow through ONE account, not both.**
- 285 orders touched Order Prepayments
- 115 orders touched Accounts Receivable
- Only **1 order** appeared in both (Order 133696 — partial prepay/edit edge case)
- This confirms prepaid and credit orders follow completely separate GL paths

**PATH A: Prepaid Orders (Order Prepayments account)**

| Step | GL Entry | Description |
|------|----------|-------------|
| 1. Payment received | Credit Order Prepayments | "Order Payment (Check/Cash/Capture $X) for #NNNNN" |
| 2. Order sold | Debit Order Prepayments | "Order Status Marked Sale" |
| Net effect | $0 | Prepayment consumed at sale |

- 176 sale entries, 168+ payment entries in January
- Payment types observed: Check (88), Cash/ACH (80), Capture/CC (~25)
- "Capture" = credit card authorization capture (Authorize.net)
- "Credit Applied" = customer credit balance applied to order (3 instances)
- GL ID field = Ledger.ID (sequential, used for ordering)

**PATH B: Unpaid/Credit Orders (Accounts Receivable account)**

| Step | GL Entry | Description |
|------|----------|-------------|
| 1. Order sold | Debit AR | "Order Status Marked Sale" |
| 2. Payment received | Credit AR | "Order Payment (Capture $X) for #NNNNN" |
| Net effect | $0 | Receivable cleared |

- 109 sale entries, ~110 payment entries in January
- Payments are mostly CC captures (79) — these are charge-account customers whose cards are charged after shipment
- Some generic "Order Payment for #NNNNN" entries (no payment type in description)
- Sale creates receivable FIRST, then payment clears it (opposite of prepaid)

**Payment Description Format:**
```
Order Payment ({Type} {Amount}) for #{OrderNumber} - {CompanyName}
```
Types observed:
- `Check {amount}` — with check No. in the No. column
- `Cash {amount}` — ACH payments show "ACH" in No. column
- `Capture {amount}` — CC authorization capture
- `Credit Applied {amount}` — customer credit balance
- `MC/Visa {amount}`, `Amex {amount}`, `Discover {amount}` — card brand specific
- `Inksoft {amount}` — web platform payment
- `Shopify {amount}` — web platform payment
- Generic (no type) — some entries just say "Order Payment for #NNNNN"

**Other register entries observed:**
- "Order Edited" — 4 in Prepayments, 4 in AR (price adjustments create GL corrections)
- "Voided Payment on Order NNNNN" — payment reversal
- "Finance Charge" — late fees on AR accounts

**GL ID (Ledger.ID) observations:**
- Sequential numbering (2840xxx–2843xxx range in January)
- Multiple GL entries share timestamps (batch processing at "3:59 PM" — likely end-of-day sale batch)
- Each GL entry has a unique ID even within the same transaction

**Running Balance confirms account behavior:**
- Order Prepayments: balance fluctuates as payments come in (credits) and sales go out (debits)
- AR: balance fluctuates as sales create receivables (debits) and payments clear them (credits)

---

### Revenue & Expense Account NodeIDs (Documented 2026-02-07 from Trial Balance Export)

**Source:** Modified Trial Balance SQL export with GLClassificationType field added. 251 total accounts (29 categories, 222 leaf). All NodeIDs confirmed.

**GLClassificationType 4000 = Product Income (51 accounts):**

*Fabric/Dye Sub Printing:*

| NodeID | Account Name |
|--------|-------------|
| 10116 | DyeSub |
| 10120 | DyeSub Table Covers |
| 10126 | Dyelux Flag |
| 10127 | Dyelux Golf Flag |
| 10533 | Dyesub Displays |
| 10258 | Feather Flags |
| 10535 | Printed Fabric |
| 10534 | SEG |
| 10537 | Printed Paper |
| 10132 | Solvent |
| 10536 | Face Masks |

*Displays & Accessories:*

| NodeID | Account Name |
|--------|-------------|
| 10257 | Pop-Up Tents |
| 10122 | Carry Bags |
| 10123 | Tri Fold Displays |
| 10124 | Roll Up Banners |
| 10125 | Banner Stand |
| 10128 | Golf Pin |
| 10129 | Putting Cup |
| 10130 | Flag Swivel |
| 10039 | Stakes and Posts |

*Traditional Signs:*

| NodeID | Account Name |
|--------|-------------|
| 6043 | Artwork-Design |
| 6052 | Cut Vinyl Signs |
| 6068 | Digital Prints |
| 10021 | Acrylics and Plastics |
| 6070 | Post-Frames-Hardware |

*Screen Printing:*

| NodeID | Account Name |
|--------|-------------|
| 6053 | Screenprinting |
| 10086 | SP Banners |
| 10114 | SP Flags |
| 10115 | SP Table Covers |

*Garments:*

| NodeID | Account Name |
|--------|-------------|
| 10099 | Embroidery |
| 10100 | Screenprint (garment) |

*Subcontract:*

| NodeID | Account Name |
|--------|-------------|
| 10255 | Sub Con Screenprint |
| 10119 | SubCon Flag |
| 10037 | SubCon Sublimation |
| 10131 | SubCon Table Cover |
| 10118 | SubCon Vinyl |

*Services & Other:*

| NodeID | Account Name |
|--------|-------------|
| 6067 | Service |
| 10040 | Services |
| 10036 | Installation |
| 6029 | Outsource |
| 6072 | Textile |
| 6 | Promotion |
| 10439 | Internet Sales |

*Shipping:*

| NodeID | Account Name |
|--------|-------------|
| 400 | Shipping Sales |
| 10038 | Shipping |

*Adjustments (contra-revenue):*

| NodeID | Account Name |
|--------|-------------|
| 10317 | Discounts Given |
| 53 | Credit Memo |
| 41 | Finance Charges |
| 45 | Unclassified Sales Income |
| 10177 | Vendor Discounts |
| 10440 | Write Off |

**GLClassificationType 4001 = Other Income (7 accounts):**

| NodeID | Account Name |
|--------|-------------|
| 10314 | Finance Charge Income |
| 10315 | Freight Reimbursement |
| 10498 | Interest Income |
| 10499 | Miscellaneous Income |
| 10313 | Other Income |
| 10316 | Returns and Allowances |
| 10539 | WI Grant Income |

**GLClassificationType 5001 = COGS (9 accounts):**

| NodeID | Account Name |
|--------|-------------|
| 10178 | Cost of Goods Sold |
| 10182 | Contract Purchases |
| 10183 | Credit Card Charge Purchases |
| 10184 | Dye Sub Purchases |
| 10185 | Fabric Purchases |
| 10186 | Garment Purchases |
| 10180 | Purchases |
| 10527 | Subcontract Purchases |
| 10493 | Freight (COGS) |

**GLClassificationType 5002 = Operating Expenses (92 accounts):** Major categories include Payroll (~20 accounts by department), Insurance (4), Taxes (4), Supplies (6), Professional Fees (3), plus standard operating expenses.

**Key GL query patterns enabled:**

```sql
-- Revenue by product line (credits are negative in GL, so negate)
SELECT ga.AccountName, SUM(-l.Amount) AS Revenue
FROM Ledger l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 4000
  AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY ga.AccountName
ORDER BY Revenue DESC

-- COGS total
SELECT ga.AccountName, SUM(l.Amount) AS COGS
FROM Ledger l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 5001
  AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY ga.AccountName

-- Gross margin = Revenue (4000 + 4001) minus COGS (5001)
-- Net income = Gross margin minus Expenses (5002)
```

**Note on Trial Balance export:** Two versions received:
1. First version: zero ChangeAmount (date range issue) — used for NodeID mapping only
2. Second version: non-zero balances with period activity and cumulative totals

**Trial Balance Period Activity (likely ~1 month based on $148K revenue vs $3M annual):**

| Line | Amount |
|------|--------|
| Product Revenue | $147,836 |
| Other Income | $2,746 |
| **Total Revenue** | **$150,582** |
| COGS | $26,876 |
| **Gross Profit** | **$123,706** (82.1% margin) |
| Operating Expenses | $120,781 |
| **Net Income** | **$2,925** |

Top revenue lines (period): DyeSub ($80.4K), DyeSub Table Covers ($26.2K), Dyesub Displays ($7.7K), Shipping ($5.4K), Banner Stand ($5.2K), Feather Flags ($5.2K)

Top COGS: Freight ($13.4K), Cost of Goods Sold ($10.7K), Garment Purchases ($2.8K)

Top expenses: Manufacturing Wages ($30.5K), Health Insurance ($19.1K), Officer Wages ($12.3K), Rent-Other ($9.2K), Office Wages ($9.0K)

**ALL-TIME CUMULATIVE REVENUE BY PRODUCT ($62.45M total):**

| Product | Revenue | % of Total |
|---------|---------|-----------|
| DyeSub Table Covers | $17,539,926 | 28.1% |
| DyeSub | $9,454,723 | 15.1% |
| Dyelux Flag | $7,320,460 | 11.7% |
| SP Table Covers | $4,972,649 | 8.0% |
| Screenprint (garment) | $4,743,649 | 7.6% |
| Shipping | $2,725,499 | 4.4% |
| Banner Stand | $2,227,723 | 3.6% |
| SP Banners | $2,175,162 | 3.5% |
| Embroidery | $2,056,591 | 3.3% |
| Services | $1,650,766 | 2.6% |
| Feather Flags | $1,592,912 | 2.6% |
| Solvent | $1,448,702 | 2.3% |
| Roll Up Banners | $816,944 | 1.3% |
| SubCon Table Cover | $763,355 | 1.2% |
| Dyesub Displays | $759,059 | 1.2% |
| All others | $2,153,687 | 3.4% |

**Key insights:**
- Top 3 products = 54.9% of all-time revenue (Table Covers + DyeSub + Dyelux Flag)
- Dye sublimation printing (DyeSub + Table Covers + Dyelux + Displays + SEG + Printed Fabric) = ~57% of revenue
- Screen printing (SP Table Covers + SP Banners + Screenprint garment + Screenprinting) = ~19% — significant legacy segment
- Garment decoration (Embroidery + Screenprint garment) = ~11%
- Gross margin ~82% in sample period — very healthy for manufacturing

--- (Documented 2026-02-07 from Chart_Of_Accounts_Listing.xls)

**Source:** Control report "Chart Of Accounts Listing" — maps to GLAccount table.

**Summary:** ~300+ accounts across standard accounting structure.

**ASSETS (71 accounts)**

| Category | Key Accounts |
|----------|-------------|
| Current Assets | Cash-Associated Bank (Checking, MM, Savings, Sweep), Cash on Hand |
| Undeposited Funds | MCVisa, Amex, Discover, Checks, Cash, Authorize.net, PayPal, Shopify |
| Receivables | Accounts Receivable, Allowance for Doubtful Accounts |
| Inventory | Raw Materials Inventory, Raw Materials - Dye Sub, Supplies Inventory, Digital Supplies, Finished Goods Inventory, Finished Goods - Feather Flag |
| Fixed Assets | Computer Equipment/Hardware, Fixtures & Equipment, Vehicles, Buildings, Building Improvements, Leasehold Improvement, Software, Land |
| Presale Assets | **WIP**, **Built** (these are the double-entry counterparts to Orders Due) |
| Transfer Accounts | Customer Credit Transfer, InterDivision COGS Transfer |
| Notes Receivable | NR Officers, NR Cain/Taylor/Gretel Goettelman |
| ZoomTex | Separate undeposited funds accounts (Amex, Cash/Check, Discover, MC/VS) |

**LIABILITIES (111 accounts)**

| Category | Key Accounts |
|----------|-------------|
| Current Liabilities | AP, Order Prepayments (+Contra), Credit Cards (US Bank, Chase Ink, Capital One, Amex), Payroll Liabilities (~20 sub-accounts), Sales Tax Payable, Baylake Bank LOC |
| Presale Liabilities | **Orders Due**, Customer Credit Balance (+Contra) |
| Long-term | EIDL Note, Land/Equipment/Vehicle Payable, Bank Loans, Deferred Revenue, Notes Payable (Cain, Taylor, Gretel), ENGS |
| Taxes | WI, Door County |

**EQUITY**
Common Stock, Opening Balance Equity, Owners Capital, Retained Earnings (+ Adjustments), Shareholder Distributions

**REVENUE (~55 Product Sales accounts + other income)**

Core product revenue accounts (under "Product Sales"):
- **Fabric printing:** DyeSub, DyeSub Table Covers, Dyelux Flag, Dyelux Golf Flag, Printed Fabric, SEG, Feather Flags, Swing Flags
- **Digital/solvent:** Product Sales - Digital, Solvent, Digital Prints, Printed Paper
- **Displays:** Pop-Up Tents, Dyesub Displays, Accessories and Displays, Carry Bags, Tri Fold Displays, Roll Up Banners, Banner Stand
- **Traditional signs:** Composition Signs, Cut Vinyl Signs, Dimensional Signs, Electric Signs, Vehicle Wraps
- **Screen printing:** Screenprinting, Screen Printing, SP Banners, SP Flags, SP Table Covers
- **Garments:** Garments, Embroidery, Screenprint (under garments)
- **Subcontract:** Sub Contract, Sub Con Embroidery/Screenprint, SubCon Flag/Sublimation/Vinyl/Table Cover
- **Hardware/accessories:** Post-Frames-Hardware, Golf Pin, Putting Cup, Flag Swivel, Stakes and Posts
- **Other:** Service, Services, Outsource, Textile, Promotion, Billboard Lease, Internet Sales
- **Shipping:** Shipping Sales, Shipping (two separate accounts)

Other revenue: Discounts Given, Credit Memo, Finance Charges, Order Rounding, Vendor Discounts, Interest Income, Miscellaneous Income, Other Income, Finance Charge Income, Freight Reimbursement, Returns and Allowances, Write Off, WI Grant Income

**COGS (under "Cost of Goods Sold1")**
- **Equipment:** Equipment Expenses, Installation/Production/Service Equipment Expense
- **Labor:** Labor Expenses, Design/Installation/Production/Service Labor Expense, Outsource Labor
- **Purchases:** Purchases, Contract Purchases, CC Charge Purchases, Dye Sub Purchases, Fabric Purchases, Garment Purchases, Table Cover Fabric Purchases, Subcontract Purchases

**EXPENSES (~100+ accounts)**

Major categories: Advertising, Bad Debt, Bank Charges, Car/Truck, Commissions & Fees, Credit Card Fees, Depreciation, Freight, Dues & Subscriptions, Insurance (Health, Property/Liability, Workers Comp, Benefits), Interest, Office Expense, Payroll (~30 sub-accounts for wages, taxes, benefits by department), Professional Fees (Accounting, Legal), Rent, Repairs & Maintenance, Supplies, Taxes (Property, Real Estate, Sales Tax), Telephone & Internet, Travel, Utilities, Vehicle, Website, Raw Material Purchases

**Key observations:**
1. Revenue accounts are very granular — maps directly to GL entries seen in order flow (e.g., "Feather Flags" revenue account matches GL credit on order 133819)
2. Presale Assets (WIP, Built) and Presale Liabilities (Orders Due) confirm the double-entry pattern for pre-sale orders
3. ZoomTex is a **legacy/defunct division** — accounts cannot be deactivated because they have historical entries. Exclude from all active queries and reporting.
4. Notes Receivable for each Goettelman family member — owner loan tracking
5. Notes Payable for Cain, Taylor, Gretel — owner debt to company
6. Duplicate-ish accounts exist (Screenprinting vs Screen Printing, two Shipping accounts) — likely legacy
7. "Install Sales" account exists at top (row 13) with SysAcct=1 — system account for installation revenue

---

```sql
-- AR Snapshot: Open invoices with balance due
SELECT a.CompanyName, th.OrderNumber, th.SaleDate, th.BalanceDue, th.TotalPrice,
    pt.TermsName, pt.GracePeriod
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
LEFT JOIN PaymentTerms pt ON a.PaymentTermsID = pt.ID
WHERE th.TransactionType = 1 AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.SaleDate IS NOT NULL
    AND th.StatusID != 9
ORDER BY th.BalanceDue DESC
```

```sql
-- AR Aging by bucket (from SaleDate — matches Control report)
SELECT 
    CASE 
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 30 THEN '0-30 days'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 60 THEN '31-60 days'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 90 THEN '61-90 days'
        ELSE '90+ days'
    END AS AgingBucket,
    COUNT(*) AS Invoices,
    SUM(th.BalanceDue) AS ARBalance
FROM TransHeader th
WHERE th.TransactionType = 1 AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.SaleDate IS NOT NULL
    AND th.StatusID != 9
GROUP BY 
    CASE 
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 30 THEN '0-30 days'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 60 THEN '31-60 days'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 90 THEN '61-90 days'
        ELSE '90+ days'
    END
```

```sql
-- AR by Payment Terms
SELECT pt.TermsName, COUNT(*) AS OpenInvoices, SUM(th.BalanceDue) AS ARBalance
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
LEFT JOIN PaymentTerms pt ON a.PaymentTermsID = pt.ID
WHERE th.TransactionType = 1 AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.SaleDate IS NOT NULL
    AND th.StatusID != 9
GROUP BY pt.TermsName
ORDER BY ARBalance DESC
```

**Note:** Simple DATEDIFF from SaleDate produces results matching the AR report totals ($80,899). A more precise aging calculation would incorporate PaymentTerms.GracePeriod to determine actual past-due status, but the SaleDate-based approach is sufficient for dashboard-level reporting.

---

## 10. CRYSTAL REPORTS

*Deferred to Phase 3 (Wiki Knowledge Formalization). Crystal Report SQL patterns will be cross-referenced with output/reports/ analysis.*

| Report Name | Purpose | Parameters | Output Format | Date Cataloged |
|------------|---------|------------|---------------|---------------|
| A/R Detail | AR aging by customer with WIP/Built visibility | Division (default: All), Date | XLS export available | 2026-02-07 |

---

## 11. CHAPI (Control HTTP API Interface)

*Deferred to Phase 3 (Wiki Knowledge Formalization). CHAPI architecture documented in output/wiki/extracts/database_integration_knowledge.md.*

| Endpoint | Method | Purpose | Required Fields | Date Documented |
|----------|--------|---------|----------------|----------------|
| | | | | |

---

## 12. OPEN QUESTIONS

| # | Question | Status | Date Opened | Date Resolved |
|---|----------|--------|-------------|---------------|
| 1 | What is TransactionType 2? | ✅ **RESOLVED: Estimates/Quotes** | 2025-02-07 | 2025-02-07 |
| 2 | StatusID 9 vs 1700 for voided — are these different states? | ✅ **RESOLVED: 1600/1700 do NOT appear in official wiki. StatusID 9 = Voided for orders. 1600/1700 were from incorrect "Cheat Sheet".** | 2025-02-07 | 2026-02-07 |
| 3 | What does StatusID 13 mean (all Type 2 records)? | ✅ **RESOLVED: "Converted" — estimate became an order** | 2025-02-07 | 2025-02-07 |
| 4 | Which UserField column stores Import_Order_Number? | ✅ **RESOLVED: TransHeaderUserField table** | 2025-02-07 | 2025-02-07 |
| 5 | How much revenue comes from web-imported orders? | ✅ **RESOLVED: $150,750 / 429 orders in 2025 (4.9% of revenue) — deduplicated** | 2025-02-07 | 2025-02-07 |
| 6 | Does the $589 revenue variance come from Type 2 partial fulfillment? | ❓ Unresolved — Type 2 is estimates, not web orders. Variance source still unknown. | 2025-02-07 | |
| 7 | Are there other UDFs on TransHeader that are operationally important? | ❓ Found 7 Import_ UDFs. Others not yet explored. | 2025-02-07 | |
| 8 | StatusID 1600 (Completed) and 1700 (Voided) from Cheat Sheet — do these exist? | ✅ **RESOLVED: NOT in official wiki. These are incorrect/deprecated. Use StatusID 4=Closed and 9=Voided.** | 2025-02-07 | 2026-02-07 |
| 9 | Type 1 StatusText "Closed" — is this different from "Completed" (StatusID 4)? | ✅ **RESOLVED: Closed = paid in full (BalanceDue=$0). Likely StatusID 4. Prepaid orders auto-close when marked Sales.** | 2025-02-07 | 2025-02-07 |
| 10 | What are the Type 2 top accounts' conversion patterns? Could inform sales strategy. | 💡 Opportunity — not blocking | 2025-02-07 | |
| 11 | Confirm exact StatusID values for Built and Closed on Type 1 | ✅ **RESOLVED via wiki: 2=Built, 4=Closed. Official reference: sql_structure_-_transheader_table** | 2025-02-07 | 2026-02-07 |
| 12 | What database tables store GL transactions? | ✅ **RESOLVED: Ledger table (all entries). GL is a VIEW filtering WHERE OffBalanceSheet=0. GLAccount stores chart of accounts.** | 2026-02-07 | 2026-02-07 |
| 13 | Where is DueDate stored/calculated for AR aging? | 🔶 **PARTIALLY RESOLVED: TransHeader.DueDate exists but is the PRODUCTION/SHIPPING due date, not payment due date. Payment due date is likely calculated at report runtime from SaleDate + PaymentTerms. Need to validate via PaymentTerms table or AR report comparison.** | 2026-02-07 | |
| 14 | Where are payment terms/grace periods stored? | ✅ **RESOLVED: Account.PaymentTermsID → PaymentTerms.ID (AR). Account.VendorPaymentTermsID → PaymentTerms.ID (AP). Also TransHeader.POPaymentTermsID for PO-specific terms.** | 2026-02-07 | 2026-02-07 |
| 15 | What GL accounts map to which revenue products? | 🔶 **PARTIALLY RESOLVED: GLAccount table exists, GLClassificationType 4000=Income. Need to query GLAccount to map product-specific revenue accounts.** | 2026-02-07 | |
| 16 | How are payments recorded in the database? | ✅ **RESOLVED: Payment table (split table with Journal, shared ID). ClassTypeID 20001=Order Payment, 20009=Bill Payment. Links to GLAccount via BankAccountID.** | 2026-02-07 | 2026-02-07 |

---

## 13. CORRECTIONS LOG

| Date | What Changed | Old Value | New Value | Why |
|------|-------------|-----------|-----------|-----|
| 2025-02-07 | TransactionType mappings | Three conflicting definitions across skill files | Validated mapping (see Section 1) | All three sources were wrong. Confirmed via GROUP BY query on live data. |
| 2025-02-07 | FP_ variable name | `FP_ProductionDescription` | `FP_ProductDescription` | Typo in original documentation. Confirmed via Variable table query. |
| 2025-02-07 | Revenue price field | `TotalPrice` (assumed) | `SubTotalPrice` (validated) | TotalPrice includes tax. SubTotalPrice matches FLS definition of "income" at 99.98%. |
| 2025-02-07 | Revenue date field | `OrderCreatedDate` (assumed) | `SaleDate` (validated) | OrderCreatedDate = entry date. SaleDate = when sale was finalized. Revenue must use SaleDate. |
| 2025-02-07 | TC_ variables | Assumed to persist to TransDetailParam | Do NOT persist — 0 records | TC_FabricCategory and TC_ProductName exist only in UI configurator. Use Description matching instead. |
| 2025-02-07 | Type 2 identity | Initial validation said "Web/Online Order" | **Estimate/Quote** with full lifecycle (Pending→Converted/Lost/Voided) | User confirmed FLS doesn't use Control web sales. Investigation confirmed StatusText="Converted"/"Lost", EstimateNumber linkage to Type 1, 56% conversion rate. |
| 2025-02-07 | Type 2 StatusID 13 | "Unknown — all Type 2 records" | "Converted" (one of 5 statuses: 1=WIP, 11=Pending, 12=Lost, 13=Converted, 14=Voided) | Initial validation only queried 2025 where most estimates were already converted. Full status distribution confirmed across all records. |
| 2025-02-07 | Type 1 Status lifecycle | StatusID 3="Sold", 4="Completed" (generic) | 3="Sales" (invoiced, SaleDate set), 4="Closed" (paid in full, BalanceDue=$0) | User confirmed business workflow: WIP→Built(rare)→Sales→Closed. Closed means paid, not just complete. Prepaid orders auto-close. |
| 2025-02-07 | **TransDetailParam query filters** | `IsActive=1 AND ParentClassTypeID=10100` | **No filters on these columns** | These filters excluded ~75% of valid records. DyeSub Print revenue: $464K → **$1,793,445** (+286%). Validated to within $3 of Control "Sales by Product" report. This was the single most impactful correction in the entire project. |
| 2025-02-07 | DyeSub Print % of revenue | 15% ($464K) | **58.7% ($1,793,445)** | Direct consequence of TransDetailParam filter fix. Flags hardware dropped from $871K to $4.9K — those were DyeSub products all along. |
| 2025-02-07 | Type 2 date field | `OrderCreatedDate` (NULL for all Type 2) | **`EstimateCreatedDate`** | OrderCreatedDate is not populated on Type 2 estimates. EstimateCreatedDate is the correct field for filtering estimates by creation date. |
| 2025-02-07 | Web order counts | 467 orders / $172,164 (raw) | **429 orders / $150,750 (deduplicated)** | Duplicate Import_Order_Number values are cloned orders in Control, not separate web orders. Only first order (lowest OrderNumber) per Import_Order_Number is a true web import. Control fixed 2026-02-07 to prevent future clones. |
| 2026-02-07 | TransHeader.DueDate meaning | Initially assumed to be payment due date (for AR aging) | **Production/shipping due date** (when order is due to customer) | User corrected: DueDate = delivery deadline, NOT payment deadline. Payment due date is likely calculated at report runtime from SaleDate + PaymentTerms. This matters for any AR aging queries. |

---

*Last updated: February 7, 2026*
*Last update: DueDate corrected (production, not payment). Wiki crawl captured GL/Ledger architecture, Payment/Journal split table, complete StatusID reference, Account credit fields. 7 open questions resolved, 4 remain.*
