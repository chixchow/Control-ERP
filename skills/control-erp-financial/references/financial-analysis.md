# Financial Analysis Reference — AR Detail, AP Detail, P&L Analysis, Cash Flow

**Parent skill:** `control-erp-financial`
**Extracted from:** `control-erp-financial-SKILL.md` (line count management)

This file contains the detailed query templates for AR/AP customer/vendor breakdowns, P&L comparison analysis, and cash flow/bank balance reporting. The main skill file contains summaries and NL routing; this file contains the full SQL.

---

## AR Detail with Customer Breakdown

Matches Control's A_R Detail report format: customer breakdown with aging columns, WIP/Built orders shown separately (not mixed into aging buckets), charge account customers marked with asterisk (*).

**AR ages from SaleDate (invoice date), NEVER DueDate.** DueDate on Type 1 orders is the production/shipping deadline, not the payment deadline. WIP/Built orders (SaleDate IS NULL) appear in a separate bucket because they have no invoice date to age from.

### Full AR Detail by Customer
```sql
-- AR Detail with customer breakdown and aging buckets
-- Ages from SaleDate (NEVER DueDate). WIP/Built orders shown separately.
SELECT
    a.CompanyName,
    CASE WHEN a.HasCreditAccount = 1 THEN '*' ELSE '' END AS CreditAcct,
    pt.TermsName,
    th.OrderNumber,
    th.SaleDate,
    th.Description,
    th.TotalPrice AS InvoiceTotal,
    th.BalanceDue,
    CASE WHEN th.SaleDate IS NULL THEN 'WIP/Built' ELSE '' END AS WIPFlag,
    -- Aging buckets (only for orders with SaleDate)
    CASE WHEN th.SaleDate IS NOT NULL AND DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 30
         THEN th.BalanceDue ELSE 0 END AS Current_0_30,
    CASE WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) BETWEEN 31 AND 60
         THEN th.BalanceDue ELSE 0 END AS Days_31_60,
    CASE WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) BETWEEN 61 AND 90
         THEN th.BalanceDue ELSE 0 END AS Days_61_90,
    CASE WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) > 90
         THEN th.BalanceDue ELSE 0 END AS Days_Over_90
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
LEFT JOIN PaymentTerms pt ON a.PaymentTermsID = pt.ID
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.StatusID != 9  -- exclude voided
ORDER BY a.CompanyName, th.SaleDate
```

### Single Customer AR Drill-Down
```sql
-- When user asks "show me AR for [customer name]"
-- Invoice-level detail for a specific customer
SELECT
    th.OrderNumber,
    th.InvoiceNumber,
    th.SaleDate,
    th.Description,
    th.TotalPrice AS InvoiceTotal,
    th.PaymentTotal,
    th.BalanceDue,
    DATEDIFF(DAY, th.SaleDate, GETDATE()) AS DaysOutstanding,
    th.StatusText,
    th.FinanceChargeAmount
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.StatusID != 9
    AND a.CompanyName LIKE '%' + @CustomerName + '%'
ORDER BY th.SaleDate
```

### AR Summary by Customer
```sql
-- Summary row per customer with aging bucket totals
-- Use when user asks "AR summary by customer" or "who owes us the most"
SELECT
    a.CompanyName,
    CASE WHEN a.HasCreditAccount = 1 THEN '*' ELSE '' END AS CreditAcct,
    COUNT(*) AS OpenInvoices,
    SUM(th.BalanceDue) AS TotalAR,
    SUM(CASE WHEN th.SaleDate IS NULL THEN th.BalanceDue ELSE 0 END) AS WIPBuilt,
    SUM(CASE WHEN th.SaleDate IS NOT NULL AND DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 30 THEN th.BalanceDue ELSE 0 END) AS Current_0_30,
    SUM(CASE WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) BETWEEN 31 AND 60 THEN th.BalanceDue ELSE 0 END) AS Days_31_60,
    SUM(CASE WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) BETWEEN 61 AND 90 THEN th.BalanceDue ELSE 0 END) AS Days_61_90,
    SUM(CASE WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) > 90 THEN th.BalanceDue ELSE 0 END) AS Days_Over_90
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 1 AND th.IsActive = 1 AND th.BalanceDue > 0 AND th.StatusID != 9
GROUP BY a.CompanyName, a.HasCreditAccount
ORDER BY TotalAR DESC
```

---

## AP Detail with Vendor Breakdown

Mirrors the AR Detail format but for vendor bills. AP ages from DueDate (vendor payment deadline) -- this is the OPPOSITE of AR which uses SaleDate.

**Key difference: AP uses DueDate, AR uses SaleDate.** DueDate on Type 8 bills is the vendor's payment deadline. DueDate on Type 1 orders is the production/shipping deadline (irrelevant for aging). This is the most important distinction in AR vs AP aging.

### Full AP Detail by Vendor
```sql
-- AP Detail with vendor breakdown and aging buckets
-- NOTE: AP ages from DueDate (vendor payment deadline), NOT SaleDate
SELECT
    a.CompanyName AS Vendor,
    vpt.TermsName AS VendorTerms,
    th.BillNumber,
    th.DueDate,
    th.Description,
    th.TotalPrice AS BillAmount,
    th.BalanceDue,
    -- AP aging from DueDate (Current = not yet due)
    CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 0
         THEN th.BalanceDue ELSE 0 END AS Current,
    CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) BETWEEN 1 AND 30
         THEN th.BalanceDue ELSE 0 END AS Days_1_30,
    CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) BETWEEN 31 AND 60
         THEN th.BalanceDue ELSE 0 END AS Days_31_60,
    CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) BETWEEN 61 AND 90
         THEN th.BalanceDue ELSE 0 END AS Days_61_90,
    CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) > 90
         THEN th.BalanceDue ELSE 0 END AS Days_Over_90
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
LEFT JOIN PaymentTerms vpt ON a.VendorPaymentTermsID = vpt.ID
WHERE th.TransactionType = 8
    AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.StatusID != 9  -- exclude voided
ORDER BY a.CompanyName, th.DueDate
```

### AP Summary by Vendor
```sql
-- Summary row per vendor with aging bucket totals
-- Use when user asks "AP by vendor" or "who do we owe the most"
SELECT
    a.CompanyName AS Vendor,
    COUNT(*) AS OpenBills,
    SUM(th.BalanceDue) AS TotalAP,
    SUM(CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 0 THEN th.BalanceDue ELSE 0 END) AS Current,
    SUM(CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) BETWEEN 1 AND 30 THEN th.BalanceDue ELSE 0 END) AS Days_1_30,
    SUM(CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) BETWEEN 31 AND 60 THEN th.BalanceDue ELSE 0 END) AS Days_31_60,
    SUM(CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) BETWEEN 61 AND 90 THEN th.BalanceDue ELSE 0 END) AS Days_61_90,
    SUM(CASE WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) > 90 THEN th.BalanceDue ELSE 0 END) AS Days_Over_90
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 8 AND th.IsActive = 1 AND th.BalanceDue > 0 AND th.StatusID != 9
GROUP BY a.CompanyName
ORDER BY TotalAP DESC
```

### Vendor Payment History
```sql
-- When user asks "payment history for [vendor]" or "payments to [vendor]"
-- Shows all bill payments made to a specific vendor
SELECT
    p.PaymentDate,
    th.BillNumber,
    th.Description,
    j.DetailAmount AS PaymentAmount,
    CASE p.TenderType
        WHEN 0 THEN 'Cash'
        WHEN 1 THEN 'Check'
        WHEN 2 THEN 'Credit Card'
        WHEN 5 THEN 'ACH'
        WHEN 7 THEN 'Wire'
        ELSE 'Other'
    END AS PaymentMethod,
    p.DisplayNumber AS Reference
FROM Payment p
INNER JOIN Journal j ON p.ID = j.ID
INNER JOIN TransHeader th ON j.TransactionID = th.ID
INNER JOIN Account a ON th.AccountID = a.ID
WHERE p.ClassTypeID = 20009  -- Bill Payment
    AND p.IsActive = 1
    AND j.IsVoided = 0
    AND th.TransactionType = 8
    AND a.CompanyName LIKE '%' + @VendorName + '%'
ORDER BY p.PaymentDate DESC
```

**Vendor credits:** `Account.VendorCreditBalance` tracks outstanding vendor credits. When a vendor issues a credit, it reduces the effective AP balance. Check this field when reporting total AP exposure for a vendor.

---

## P&L Analysis

### P&L with Year-over-Year Comparison

Compare current YTD vs prior year same-period (apples-to-apples using same day-of-year cutoff). Default period: year-to-date.

```sql
-- Year-over-year P&L comparison (current YTD vs prior YTD, same day-of-year cutoff)
SELECT
    CASE ga.GLClassificationType
        WHEN 4000 THEN '1-Product Revenue'
        WHEN 4001 THEN '2-Other Income'
        WHEN 5001 THEN '3-Cost of Goods Sold'
        WHEN 5002 THEN '4-Operating Expenses'
    END AS Category,
    ga.AccountName,
    SUM(CASE WHEN YEAR(l.EntryDateTime) = YEAR(GETDATE())
        THEN CASE WHEN ga.GLClassificationType IN (4000, 4001) THEN -l.Amount ELSE l.Amount END
        ELSE 0 END) AS CurrentYear,
    SUM(CASE WHEN YEAR(l.EntryDateTime) = YEAR(GETDATE()) - 1
        THEN CASE WHEN ga.GLClassificationType IN (4000, 4001) THEN -l.Amount ELSE l.Amount END
        ELSE 0 END) AS PriorYear
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType IN (4000, 4001, 5001, 5002)
    AND l.EntryDateTime BETWEEN DATEADD(yy, DATEDIFF(yy, 0, GETDATE()) - 1, 0) AND GETDATE()
    AND DATEPART(DAYOFYEAR, l.EntryDateTime) <= DATEPART(DAYOFYEAR, GETDATE())
GROUP BY ga.GLClassificationType, ga.AccountName
HAVING SUM(l.Amount) != 0
ORDER BY Category
```

**Note:** The prior year comparison uses the same day-of-year cutoff (`DATEPART(DAYOFYEAR)`) to ensure an apples-to-apples comparison. For example, if today is February 9, both years include only Jan 1 through Feb 9.

### P&L with Month-over-Month Comparison

Compare current month vs prior month. Useful for trend analysis.

```sql
-- Month-over-month P&L comparison
DECLARE @CurrentMonthStart DATETIME = DATEADD(mm, DATEDIFF(mm, 0, GETDATE()), 0);
DECLARE @PriorMonthStart DATETIME = DATEADD(mm, -1, @CurrentMonthStart);
DECLARE @PriorMonthEnd DATETIME = DATEADD(dd, -1, @CurrentMonthStart);

SELECT
    CASE ga.GLClassificationType
        WHEN 4000 THEN '1-Product Revenue'
        WHEN 4001 THEN '2-Other Income'
        WHEN 5001 THEN '3-Cost of Goods Sold'
        WHEN 5002 THEN '4-Operating Expenses'
    END AS Category,
    ga.AccountName,
    SUM(CASE WHEN l.EntryDateTime >= @CurrentMonthStart
        THEN CASE WHEN ga.GLClassificationType IN (4000, 4001) THEN -l.Amount ELSE l.Amount END
        ELSE 0 END) AS CurrentMonth,
    SUM(CASE WHEN l.EntryDateTime BETWEEN @PriorMonthStart AND @PriorMonthEnd
        THEN CASE WHEN ga.GLClassificationType IN (4000, 4001) THEN -l.Amount ELSE l.Amount END
        ELSE 0 END) AS PriorMonth
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType IN (4000, 4001, 5001, 5002)
    AND l.EntryDateTime >= @PriorMonthStart
GROUP BY ga.GLClassificationType, ga.AccountName
HAVING SUM(l.Amount) != 0
ORDER BY Category
```

### Product-Line Margin Analysis

Revenue can be broken down by product line at the GL level, but COGS is aggregate. Run **both queries** to get the full margin picture.

**Query A -- Revenue Breakdown by Product Line:**
```sql
-- Revenue breakdown by product line (GLClassificationType 4000)
-- NOTE: COGS is aggregate (not per-product) at GL level. Per-product COGS requires TransDetail analysis.
DECLARE @StartDate DATETIME = DATEADD(yy, DATEDIFF(yy, 0, GETDATE()), 0);  -- YTD default
DECLARE @EndDate DATETIME = GETDATE();

SELECT
    ga.AccountName AS ProductLine,
    SUM(-l.Amount) AS Revenue
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 4000
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY ga.AccountName
ORDER BY Revenue DESC;
```

**Query B -- Aggregate COGS (separate, cannot be broken by product):**
```sql
-- Separate query for total COGS (cannot be broken by product at GL level)
DECLARE @StartDate DATETIME = DATEADD(yy, DATEDIFF(yy, 0, GETDATE()), 0);  -- YTD default
DECLARE @EndDate DATETIME = GETDATE();

SELECT
    SUM(l.Amount) AS TotalCOGS
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 5001
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate;
```

**How to use:** Run Query A for per-product revenue, Query B for total COGS, then compute gross margin = (Total Revenue from A) - (Total COGS from B). This gives aggregate gross margin percentage but not per-product margins.

> **COGS-to-product mapping:** Revenue GL accounts have per-product NodeIDs (e.g., 10116=DyeSub, 10120=Table Covers), but COGS accounts (GLClassificationType 5001) are aggregate. Product-line P&L shows revenue breakdown by product with total COGS. Per-product margin requires TransDetail-level cost analysis (different approach, not GL-based).

### P&L Formatting Guidance

- **Default date range:** Year-to-date (`DATEADD(yy, DATEDIFF(yy, 0, GETDATE()), 0)` to `GETDATE()`)
- **Currency format:** $1,234.56 with comma separators
- **Negative numbers:** Accounting parentheses -- ($1,234.56)
- **Computed rows:** Always include Gross Margin (Revenue - COGS) and Net Income (Revenue - COGS - OpEx)
- **Result sets:** Full list, no truncation

---

## CASH FLOW AND BANK BALANCES

### Bank Balance (Current Cash Position)

Cumulative balance on bank accounts. **NO date filter** for "current" balance -- this is all-time cumulative. Exclude ZoomTex legacy accounts (10522-10525).

```sql
-- Current bank balances (cumulative, NO date filter)
SELECT
    ga.AccountName,
    SUM(l.Amount) AS Balance
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 1000  -- Cash/Bank accounts
    AND ga.IsActive = 1
    AND ga.ID NOT IN (10522, 10523, 10524, 10525)  -- Exclude ZoomTex legacy
GROUP BY ga.AccountName, ga.ID
ORDER BY ga.ID
```

Add a total row when presenting results: sum all bank account balances for total cash position.

### Undeposited Funds (In Transit)

Funds received but not yet deposited to bank. Shown separately from bank balance.

```sql
-- Undeposited funds by account (in transit, not yet in bank)
SELECT
    ga.AccountName,
    SUM(l.Amount) AS Balance
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 1007  -- Undeposited Funds
    AND ga.IsActive = 1
    AND ga.ID NOT IN (10522, 10523, 10524, 10525)  -- Exclude ZoomTex
GROUP BY ga.AccountName, ga.ID
HAVING SUM(l.Amount) != 0
ORDER BY ga.ID
```

### Cash Flow Summary (YTD)

Categorized cash in/out for a date range. Tracks only bank account movements (GLClassificationType = 1000) to avoid double-counting the undeposited-to-bank two-step process. Default period: year-to-date.

```sql
DECLARE @StartDate DATETIME = DATEADD(yy, DATEDIFF(yy, 0, GETDATE()), 0);
DECLARE @EndDate DATETIME = GETDATE();

-- Cash flow: entries hitting bank accounts, categorized by activity type
SELECT
    CASE
        WHEN l.DepositJournalID IS NOT NULL AND l.Amount > 0 THEN 'Customer Deposits'
        WHEN j.ClassTypeID = 20009 THEN 'Vendor Payments'
        WHEN l.PayrollID IS NOT NULL THEN 'Payroll'
        WHEN l.EntryType = 1 THEN 'Manual/Adjustments'
        WHEN l.Amount > 0 THEN 'Other Inflows'
        ELSE 'Other Outflows'
    END AS CashCategory,
    SUM(l.Amount) AS NetAmount,
    COUNT(*) AS Entries
FROM GL l
LEFT JOIN Journal j ON l.JournalID = j.ID
WHERE l.GLAccountID IN (90, 10412)  -- Bank accounts only
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY
    CASE
        WHEN l.DepositJournalID IS NOT NULL AND l.Amount > 0 THEN 'Customer Deposits'
        WHEN j.ClassTypeID = 20009 THEN 'Vendor Payments'
        WHEN l.PayrollID IS NOT NULL THEN 'Payroll'
        WHEN l.EntryType = 1 THEN 'Manual/Adjustments'
        WHEN l.Amount > 0 THEN 'Other Inflows'
        ELSE 'Other Outflows'
    END
ORDER BY NetAmount DESC
```

### Monthly Cash Flow Time Series

When user asks for cash flow over multiple months or cash flow trend:

```sql
DECLARE @StartDate DATETIME = DATEADD(yy, DATEDIFF(yy, 0, GETDATE()), 0);
DECLARE @EndDate DATETIME = GETDATE();

SELECT
    YEAR(l.EntryDateTime) AS Yr,
    MONTH(l.EntryDateTime) AS Mo,
    SUM(CASE WHEN l.Amount > 0 THEN l.Amount ELSE 0 END) AS CashIn,
    SUM(CASE WHEN l.Amount < 0 THEN l.Amount ELSE 0 END) AS CashOut,
    SUM(l.Amount) AS NetCashFlow
FROM GL l
WHERE l.GLAccountID IN (90, 10412)  -- Bank accounts
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY YEAR(l.EntryDateTime), MONTH(l.EntryDateTime)
ORDER BY Yr, Mo
```

### Cash Flow Notes

- **Bank balance = cumulative** (no date filter). **Cash flow = period activity** (with date filter). Different queries for different purposes.
- Cash flow tracks only bank account movements (GLClassificationType = 1000) to avoid double-counting the undeposited-to-bank two-step deposit process.
- Categorization covers ~90% of entries. "Other" bucket captures miscellaneous GL entries (bank fees, loan payments, etc.).
- Default period: year-to-date (consistent with P&L).
- Exclude ZoomTex legacy accounts (10522-10525) from all bank and cash flow queries.
- **Payment forecast is deferred** to the analytics milestone. This skill provides historical cash flow only. If user asks for projected or forecasted cash flow, respond that this capability is planned but not yet available, and offer historical cash flow instead.
