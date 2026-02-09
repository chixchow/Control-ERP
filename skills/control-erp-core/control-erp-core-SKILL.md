---
name: control-erp-core
description: Foundation reference for the Control ERP StoreData database at FLS Banners. Contains validated business rules, corrected TransactionType mappings, standard query filters, pricing logic, and schema relationships. Required dependency for all other control-erp skills. Use whenever querying, analyzing, or writing SQL for the Control ERP system.
---

# Control ERP Core — Validated Business Rules & Schema Reference

**Last Validated:** February 7, 2026
**Validated Against:** $3,053,541.85 known 2025 income (matched to 99.98%)
**Database:** StoreData (SQL Server)

---

## CRITICAL: Revenue Query Formula

```sql
-- THE canonical revenue query. Validated to 99.98% accuracy.
SELECT SUM(SubTotalPrice) AS Revenue
FROM TransHeader
WHERE TransactionType = 1
    AND IsActive = 1
    AND SaleDate IS NOT NULL
    AND YEAR(SaleDate) = @Year
```

**Three things that are NOT obvious:**
1. Use `SubTotalPrice` (pre-tax), NOT `TotalPrice` — SubTotalPrice matches FLS's definition of "income"
2. Use `SaleDate` for date filtering, NOT `OrderCreatedDate` — SaleDate reflects when the sale was finalized
3. Always use header-level `TransHeader.SubTotalPrice` for totals, NOT the sum of `TransDetail.SubTotalPrice` (there's a ~$7K gap due to header-level adjustments)

---

## TransactionType Reference — VALIDATED

These mappings were validated against live 2025 data. All previous skill file definitions were incorrect.

| Type | Identity | Number Field | Date Field | Notes |
|------|----------|-------------|------------|-------|
| **1** | **Order/Sale** | OrderNumber, InvoiceNumber (same value) | SaleDate | **Primary revenue record.** StatusID: 0=New, 1=WIP, 2=Built, 3=Sale, 4=Closed, 9=Voided |
| **2** | **Estimate/Quote** | EstimateNumber | SaleDate is always NULL | Lifecycle: StatusID 11=Pending → 13=Converted (56%) / 12=Lost (41%) / 14=Voided (2%). Converted estimates link to Type 1 orders via shared EstimateNumber. |
| **3** | **Recurring Order** | — | — | StatusID 21. Zero records at FLS. Documented in wiki. |
| **4** | **Credit Memo** | — | — | StatusID 20=Credit Memo, 9=Voided. Zero records at FLS. |
| **5** | **UNUSED** | — | — | No records exist at FLS |
| **6** | **Service Ticket** | OrderNumber | Varies | Rarely used (1 record in all of 2025) |
| **7** | **Vendor Purchase Order** | PurchaseOrderNumber | — | StatusID 6=Open, 25=Requested, 26=Approved, 27=Ordered, 28=Closed, 29=Cancelled, 30=Rejected, 31=Received. Part of vendor chain: 7→8→9 |
| **8** | **Vendor Bill (Payment)** | PurchaseOrderNumber | — | StatusID 4=Closed, 6=Open, 9=Voided |
| **9** | **Vendor Bill (Received)** | PurchaseOrderNumber | — | StatusID 4=Closed, 6=Open, 9=Voided |
| **10** | **Vendor Credit Memo** | — | — | StatusID 24. Not validated at FLS. |

### FLS Workflow
- **Type 1** is the main sales record — this is where revenue lives. Includes web-imported orders (identified by Import_Order_Number UDF).
- **Type 2** is estimates/quotes — 56% convert to Type 1 orders (linked via EstimateNumber), 41% are lost. Never include in revenue.
- **Types 3, 4, 5, 10** have zero or near-zero records at FLS (documented from wiki for completeness)
- **Type 7→8→9** is the vendor purchasing chain (PO → Bill Payment → Bill Received)

### Estimate → Order Linkage
Type 2 and Type 1 share `EstimateNumber`. When an estimate converts, a Type 1 order is created with the same EstimateNumber but its own OrderNumber/InvoiceNumber. Prices may change between estimate and order.

### Web Import Identification
Web orders imported via CHAPI are Type 1 orders (NOT Type 2). Identified by `Import_Order_Number` UDF, but **deduplication is mandatory** — duplicate Import_Order_Number values are cloned orders, not separate web imports.

```sql
-- CORRECT: Deduplicated web import query
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
Web imports (deduplicated) = ~4.9% of revenue, ~10.3% of order count (2025).
Note: Control was fixed Feb 7, 2026 to prevent future clones, but historical data still needs dedup.

---

## Natural Language → TransactionType Mapping

| User Says | TransactionType | Price Field |
|-----------|----------------|-------------|
| "sales", "revenue", "income" | 1 | SubTotalPrice |
| "orders" (in revenue context) | 1 | SubTotalPrice |
| "web orders", "online orders", "website orders" | 1 (filter by Import_Order_Number UDF) | SubTotalPrice |
| "all sales" (inclusive) | 1 | SubTotalPrice |
| "estimates", "quotes" | 2 | SubTotalPrice (quoted value, NOT revenue) |
| "conversion rate", "quote to close" | 2 (StatusID 13 vs 12) | Compare Type 2 vs matched Type 1 SubTotalPrice |
| "pending estimates", "open quotes" | 2 WHERE StatusID = 11 | SubTotalPrice |
| "lost estimates", "lost quotes" | 2 WHERE StatusID = 12 | SubTotalPrice |
| "purchase orders", "POs" (vendor) | 7 | TotalPrice |
| "vendor bills", "AP" | 8, 9 | TotalPrice |
| "cost of goods", "COGS", "spending" | 8 | TotalPrice |

**Default assumption:** When a user asks about "sales" or "revenue" without qualification, use TransactionType = 1 only. Only include Type 2 (web orders) if explicitly asked about web/online orders or "all channels."

---

## Standard Query Filters — Always Apply

Every query against TransHeader MUST include these unless explicitly overridden:

```sql
WHERE th.IsActive = 1          -- Active records only
    AND th.StatusID != 9        -- Exclude voided (StatusID 9 for Type 1)
```

Every query against TransDetail MUST include:
```sql
WHERE td.IsActive = 1          -- Active line items only
```

**Revenue-Specific Filters (in addition to above):**
```sql
AND th.SaleDate IS NOT NULL    -- Exclude orders not yet invoiced
```
Without this filter, WIP and Built orders (no SaleDate) would be included in revenue totals.

Every query against TransDetailParam:
```sql
-- CORRECT: No IsActive or ParentClassTypeID filter
WHERE tdp.VariableID = @VariableID  -- Filter by specific variable ID for performance
```

**⚠️ CRITICAL: Do NOT filter TransDetailParam by `IsActive = 1` or `ParentClassTypeID = 10100`. These filters exclude ~75% of valid records and produce drastically wrong results (e.g., DyeSub Print revenue appears as $464K instead of the correct $1.79M). This was validated 2025-02-07 against the Control "Sales by Product" report.**

---

## Date Filtering Patterns

**Always use `SaleDate` for revenue queries on Type 1 transactions.**
`OrderCreatedDate` reflects when the order was entered, not when it was sold.

**Always use `EstimateCreatedDate` for Type 2 estimate date filtering.**
`OrderCreatedDate` is NULL for all Type 2 records. `SaleDate` is also always NULL on Type 2.

```sql
-- This year
AND YEAR(th.SaleDate) = YEAR(GETDATE())

-- Specific year
AND YEAR(th.SaleDate) = @Year

-- Date range
AND th.SaleDate >= @StartDate AND th.SaleDate < @EndDate

-- YTD
AND th.SaleDate >= DATEFROMPARTS(YEAR(GETDATE()), 1, 1)

-- Quarter (e.g., Q4)
AND th.SaleDate >= DATEFROMPARTS(@Year, 10, 1)
AND th.SaleDate < DATEFROMPARTS(@Year + 1, 1, 1)

-- Month
AND YEAR(th.SaleDate) = @Year AND MONTH(th.SaleDate) = @Month

-- Null safety (always include for SaleDate queries)
AND th.SaleDate IS NOT NULL

-- ESTIMATES (Type 2): Use EstimateCreatedDate, NOT OrderCreatedDate
AND YEAR(th.EstimateCreatedDate) = @Year
```

---

## Schema Quick Reference

### Core Transaction Hierarchy
```
TransHeader (order)                    → ClassTypeID 10000
  └── TransDetail (line item)          → ClassTypeID 10100
      ├── TransDetailParam (variables) → ClassTypeID 10150
      ├── TransMod (modifiers)         → ClassTypeID 10200
      └── TransPart (materials)        → ClassTypeID 10300
```

### Key JOIN Patterns

```sql
-- Header → Detail
INNER JOIN TransDetail td ON th.ID = td.TransHeaderID AND td.IsActive = 1

-- Detail → Parameter (Variable Value)
INNER JOIN TransDetailParam tdp ON tdp.ParentID = td.ID
    AND tdp.VariableID = @VariableID  -- Always filter by specific VariableID

-- Parameter → Variable Definition
INNER JOIN Variable v ON tdp.VariableID = v.ID

-- Header → Customer
INNER JOIN Account a ON th.AccountID = a.ID

-- Header → Salesperson
LEFT JOIN Employee e ON th.SalesPerson1ID = e.ID
```

### TransDetailParam Value Storage
Values are stored in typed columns based on `ValueType`:

| ValueType | Column | Use |
|-----------|--------|-----|
| 256 | `ValueAsStr25` (nvarchar 25) | **Most common for FP_/TC_ variables** |
| 256 | `ValueAsString` (text) | Longer string values |
| 11 | `ValueAsFloat` | Numeric values |
| — | `ValueAsInteger` | Integer values |
| — | `ValueAsDateTime` | Date values |

**For FP_ProductDescription and FP_ProductID, values are in `ValueAsStr25`.**

---

## Pricing Fields Reference

### TransHeader Level
| Field | Meaning | Use For |
|-------|---------|---------|
| `SubTotalPrice` | Pre-tax total | **Revenue reporting** |
| `TotalPrice` | Total including tax | Gross billing amount |
| `TaxesPrice` | Tax amount | Tax reporting |
| `PaymentTotal` | Amount paid | Collections reporting |
| `BalanceDue` | Outstanding amount | AR reporting |
| `ShippingPrice` | Shipping charges | Shipping revenue |

### TransDetail Level
| Field | Meaning | Use For |
|-------|---------|---------|
| `SubTotalPrice` | Line-item pre-tax total | Product-level revenue (note: sums ~$7K less than header) |
| `TotalPrice` | Line-item total with tax | Line-level billing |
| `Quantity` | Units ordered | Volume reporting |
| `BasePrice` | Unit price | Pricing analysis |

**Rule:** Use `TransHeader.SubTotalPrice` for total revenue. Use `TransDetail.SubTotalPrice` only when you need a product-level or line-item breakdown, and acknowledge the ~$7K header-level adjustment gap.

---

## StatusID Reference (Confirmed via Control Wiki)

### Type 1 (Orders) — Full Lifecycle: WIP → Built → Sales → Closed
| StatusID | StatusText | Meaning | Include in Revenue? |
|----------|-----------|---------|-------------------|
| 0 | New | Just created | No |
| 1 | WIP | Work in Process — order entered, being produced | No (SaleDate not yet set) |
| 2 | Built | Production complete, not yet invoiced (rarely used at FLS) | No |
| 3 | Sales | Invoiced — **SaleDate set here = revenue recognition** | **YES** |
| 4 | Closed | Paid in full — BalanceDue = $0 | **YES** |
| 9 | Voided | Cancelled | **NO — always exclude** |

**Key rules:**
- Revenue = SaleDate IS NOT NULL (captures both Sales and Closed orders)
- Sales with BalanceDue > 0 = AR (awaiting payment)
- Closed = BalanceDue = $0 (fully paid)
- Prepaid orders auto-close on sale (deposit = order value, so BalanceDue = $0 immediately)

### Type 2 (Estimates)
| StatusID | Meaning | % of Estimates |
|----------|---------|---------------|
| 11 | Pending | <1% |
| 12 | Lost | 41% |
| 13 | Converted (became Type 1 order) | 56% |
| 14 | Voided | 2% |

### Type 7 (Vendor PO)
| StatusID | Meaning |
|----------|---------|
| 6 | Open |
| 25 | Requested |
| 26 | Approved |
| 27 | Ordered |
| 28 | Closed |
| 29 | Cancelled |
| 30 | Rejected |
| 31 | Received |

### Type 8 (Bill) / Type 9 (Receiving Document)
| StatusID | Meaning |
|----------|---------|
| 4 | Closed |
| 6 | Open |
| 9 | Voided |

### Type 4 (Credit Memo)
| StatusID | Meaning |
|----------|---------|
| 20 | Credit Memo |
| 9 | Voided |

### Type 10 (Vendor Credit Memo)
| StatusID | Meaning |
|----------|---------|
| 24 | Vendor Credit Memo |

---

## High-Volume Tables — Performance Notes

| Table | Records | Index Strategy |
|-------|---------|---------------|
| TransPart | 2,518,931+ | Always filter by TransHeaderID or TransDetailID |
| TransDetailParam | 759,245 | Filter by VariableID for product queries (use ID, not name) |
| TransDetail | 535,126+ | Filter by TransHeaderID |
| TransVariation | 231,648 | Filter by TransHeaderID |
| TransHeader | 230,849 | Filter by TransactionType + SaleDate |

**Performance tip:** When querying TransDetailParam, filter by `VariableID = [specific ID]` rather than joining to Variable and filtering by `VariableName`. The ID-based filter is significantly faster on 759K records.

---

## GL / Ledger Architecture

**⚠️ "GL" is a VIEW, not a table.** Actual data is in the **Ledger** table.

```sql
-- GL View = on-balance sheet entries only (for financial reports)
CREATE VIEW GL AS SELECT * FROM Ledger WHERE OffBalanceSheet = 0

-- Ledger table = ALL entries (for cost analysis)
SELECT * FROM Ledger
```

Key GL/Ledger columns: `Amount`, `GLAccountID`→GLAccount, `GLClassificationType`, `JournalID`→Journal, `TransactionID`→TransHeader, `TransDetailID`→TransDetail

**Sign conventions:** Revenue accounts are negative (credits), expenses are positive (debits). Use `SUM(-Amount)` for revenue, `SUM(Amount)` for expenses.

**GLClassificationType codes:** 1000=Cash, 1002=Current Assets, 1003=Inventory, 1007=UndepositedFunds, 2000=AP, 2002=Current Liabilities, 3000=Equity, 4000=Product Revenue, 4001=Other Income, 5001=COGS, 5002=Expenses

**Payment table** is a split table with Journal (`Payment.ID = Journal.ID`). ClassTypeID 20001=Order Payment, 20009=Bill Payment.

**For detailed GL/AR/AP queries, P&L reporting, and complete NodeID mappings, see `control-erp-financial` skill.**

---

## Multi-Division Context

| Division | Warehouses | Products |
|----------|-----------|----------|
| Company (Banners/Signs) | 10, 11 | DyeSub Print, DyeLux Table Covers, hardware |
| Apparel | 10000, 10001 | Garments, embroidery |

---

## History Tables

All core transaction tables have corresponding `*History` tables (TransHeaderHistory, TransDetailHistory, etc.) that provide complete audit trails via `(ID, SeqID)` composite keys. History tables are append-only.

---

*This skill contains business rules validated against FLS Banners' actual 2025 financial data. All TransactionType mappings, price field selections, and query patterns have been confirmed through direct database analysis.*
