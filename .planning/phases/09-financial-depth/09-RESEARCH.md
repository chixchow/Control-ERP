# Phase 9: Financial Depth - Research

**Researched:** 2026-02-09
**Domain:** Control ERP Financial Reporting (AR/AP/P&L/Cash Flow)
**Confidence:** HIGH

## Summary

This phase extends the existing `control-erp-financial` skill (856 lines) with deeper analytical queries for AR aging with customer breakdown, AP tracking with vendor breakdown, P&L with product-line analysis, and cash flow/bank balance reporting. The existing skill already contains the foundational GL architecture, sign conventions, payment patterns, and basic AR/AP/P&L queries -- this phase adds the analytical depth users need for day-to-day financial management.

The research confirms that **all four requirements (FIN-04 through FIN-07) can be implemented entirely with existing validated patterns**. No new tables, no new joins, no new libraries. The work is writing SQL query templates that compose existing patterns (GL view, TransHeader fields, GLAccount hierarchy, Payment table) into the specific report formats the user expects. The primary risk is not technical complexity but **getting the details right**: correct sign conventions per account type, correct date fields per transaction type, correct aging logic that matches Control's built-in reports.

**Primary recommendation:** Add four new sections to the existing financial skill file (AR Detail, AP Detail, P&L Analysis, Cash Flow), each with multiple query templates and natural language routing. Estimate ~350-450 new lines, keeping the total under the 1,200-line extraction threshold. If it exceeds 1,200 lines, extract the new content to `references/financial-analysis.md`.

## Standard Stack

This phase does not introduce new libraries or tools. It extends existing skill content.

### Core (already in place)
| Component | Version | Purpose | Why Standard |
|-----------|---------|---------|--------------|
| GL view | N/A (SQL view) | All financial queries | Filters off-balance-sheet, validated in v1.0 |
| TransHeader | N/A (table) | AR/AP source of truth | BalanceDue, SaleDate, DueDate by transaction type |
| GLAccount | N/A (table) | Chart of accounts tree | GLClassificationType for P&L categories |
| Payment | N/A (table) | Payment history | TenderType, BankAccountID, ClassTypeID for filtering |
| PaymentTerms | N/A (table) | Grace period for AR aging context | GracePeriod field |

### Supporting (already documented)
| Component | Purpose | When to Use |
|-----------|---------|-------------|
| Account table | Customer/vendor names, HasCreditAccount, CreditBalance, VendorCreditBalance | AR/AP customer/vendor breakdown |
| Journal table | Payment activity dates and descriptions | Cash flow categorization by activity type |
| Closeout table | Period locking boundaries | Warn user about locked periods when querying historical data |

### No New Dependencies
All four requirements use tables and patterns already documented and validated in the v1.0 financial skill. No new table mappings, no new join patterns, no external tools.

## Architecture Patterns

### Existing Skill Structure (extend, not replace)
```
skills/control-erp-financial/
  control-erp-financial-SKILL.md    # 856 lines currently, add ~350-450
  references/                        # Create ONLY IF >1,200 lines total
    financial-analysis.md            # Overflow: AR detail, AP detail, P&L analysis, cash flow
```

### Pattern 1: AR Aging with Customer Breakdown (FIN-04)

**What:** AR aging report that mirrors Control's A_R Detail report -- grouped by customer, with aging bucket columns.
**Key insight from wiki:** Control's Historical AR Report sources from GL (GLAccountID=14) for historical views, but the live AR report uses TransHeader fields. Our existing skill correctly uses TransHeader as the source.

**Critical rules (verified from existing skill and wiki):**
- AR ages from `SaleDate` (invoice date), NEVER `DueDate` (production deadline)
- Include WIP/Built orders (StatusID 1,2) separately from Sale orders (StatusID 3+) -- Control's report shows them for visibility
- `BalanceDue > 0` filters outstanding invoices
- `Account.HasCreditAccount = 1` marks charge account customers (shown as `*` prefix in Control)
- `Account.AccountingContactID` links to AR contact, NOT `Account.PrimaryContactID`

**Source:** Existing skill lines 502-590, wiki `historical_ar_report_standard`, wiki `control_sql_-_ar_integrity_check`

### Pattern 2: AP Aging with Vendor Breakdown (FIN-05)

**What:** AP aging report grouped by vendor, with aging bucket columns.
**Key insight:** AP ages from `DueDate` (vendor payment deadline) -- this is the OPPOSITE of AR which uses SaleDate. DueDate has different semantics by transaction type.

**Critical rules (verified from existing skill):**
- AP uses TransactionType = 8 (bills), StatusID 6 = Open, 9 = Voided
- AP ages from `DueDate` (the vendor's payment deadline)
- `Account.VendorPaymentTermsID` links to vendor's default payment terms
- Vendor identification: `Account.IsVendor = 1`
- `Account.VendorCreditBalance` tracks outstanding vendor credits
- Bill numbers are in `TransHeader.BillNumber` field (30xxx range, separate from order numbering)

**Source:** Existing skill lines 593-653, wiki `accounting_ch_09-bills_from_vendors`, wiki `control_sql_-_ap_integrity_check`

### Pattern 3: P&L with Product-Line Breakdown (FIN-06)

**What:** Two P&L modes -- traditional format and product-line breakdown.
**Key insight:** The existing skill already has Revenue by Product Line, Full P&L, and P&L Summary queries. The extension adds comparison periods, product-line margin analysis, and formatted output templates.

**Critical rules (verified from existing skill):**
- Revenue (GLClassificationType 4000, 4001): `SUM(-Amount)` -- stored as negative credits
- COGS (GLClassificationType 5001): `SUM(Amount)` -- stored as positive debits
- Operating Expenses (GLClassificationType 5002): `SUM(Amount)` -- stored as positive debits
- Gross Margin = Revenue - COGS
- Net Income = Revenue - COGS - Operating Expenses
- Date filter on `GL.EntryDateTime BETWEEN @StartDate AND @EndDate`
- Default period: year-to-date (per context decision)
- Product lines map to GLAccount.AccountName where GLClassificationType = 4000

**COGS-to-product mapping limitation:** COGS accounts (GLClassificationType 5001) are not granularly mapped to individual product lines at the GL level. Revenue accounts have product-specific NodeIDs (e.g., 10116=DyeSub, 10120=DyeSub Table Covers), but COGS is recorded in general accounts (10178=Cost of Goods Sold, 10180=Purchases, etc.). Product-line P&L therefore shows revenue breakdown by product but COGS as aggregate. True per-product margin requires TransDetail-level analysis (different approach).

**Source:** Existing skill lines 656-710, GLAccount NodeID reference lines 194-273

### Pattern 4: Cash Flow and Bank Balances (FIN-07)

**What:** Current bank balances plus categorized cash in/out summary.
**Key insight:** Bank accounts are identified by GLClassificationType = 1000 (Cash/Bank). Undeposited funds (GLClassificationType = 1007) should be shown separately as "in transit."

**Critical rules (verified from existing skill):**
- Bank accounts: GLClassificationType = 1000 (NodeIDs 90=Checking, 10412=MM)
- Undeposited accounts: GLClassificationType = 1007 (NodeIDs 91, 92, 93, 543, 10137, 10528, 10530, 10531)
- Bank balance = `SUM(GL.Amount)` where GLAccountID is a bank account (cumulative, no date filter for "current" balance)
- Cash in = customer payments (debit to bank/undeposited accounts)
- Cash out = vendor payments (credit from bank accounts), payroll
- `Payment.Undeposited = 1` identifies payments not yet deposited to bank
- Time-series view: GROUP BY YEAR(EntryDateTime), MONTH(EntryDateTime)
- Exclude ZoomTex legacy accounts (NodeIDs 10522-10525)

**Cash flow categorization approach:** Use GL.GLClassificationType to categorize inflows vs outflows:
- Customer receipts: GL entries hitting GLClassificationType 1000/1007 with matching AR/deposit offset
- Vendor payments: GL entries where AP (NodeID 22) is debited and bank (NodeID 90) is credited
- Payroll: GL entries with PayrollID IS NOT NULL
- Other: Manual journal entries, bank fees, etc.

**Source:** Existing skill lines 714-732 (Balance Sheet Key Accounts), lines 400-440 (Deposit Workflow)

### Anti-Patterns to Avoid

- **Using DueDate for AR aging:** DueDate on Type 1 = production/ship deadline. AR aging MUST use SaleDate.
- **Using Ledger instead of GL view:** Includes ~307K off-balance-sheet entries that distort financial reports.
- **Forgetting SUM(-Amount) for revenue:** Revenue stored as negative credits. Missing the negation is the most common sign convention error.
- **Filtering IsActive on Ledger/GL:** Ledger entries are always IsActive=1 in practice. Adding the filter is harmless but unnecessary and may cause confusion.
- **Mixing WIP/Built AR with Sale AR:** WIP/Built orders have BalanceDue but are NOT in GL AR (NodeID 14). Report should show them separately with a "WIP/Built" bucket, not mixed into aging.
- **Truncating output:** Context decision explicitly says "full list always -- no truncation." All queries should return complete result sets.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| AR aging calculation | Custom aging logic | DATEDIFF(DAY, SaleDate, GETDATE()) with CASE | The existing pattern in lines 535-558 is validated against Control's report |
| GL sign convention | Per-query sign handling | Standard pattern: SUM(-Amount) for GLClassType 4000/4001, SUM(Amount) for 5001/5002 | One pattern, used everywhere |
| Payment terms grace period | Manual due date calculation | SaleDate + PaymentTerms.GracePeriod | PaymentTerms table has the authoritative grace periods |
| Period date defaults | Per-query date handling | Standard YTD default: `DATEADD(yy, DATEDIFF(yy, 0, GETDATE()), 0)` to `GETDATE()` | Consistent default per context decision |
| Account type identification | Custom flags | `Account.IsClient`, `Account.IsVendor`, `Account.HasCreditAccount` | Built-in flags, already validated |
| GL hierarchy for P&L | Manual category grouping | `GLAccount.GLClassificationType` IN (4000, 4001, 5001, 5002) | GLAccount tree provides the hierarchy |

**Key insight:** Every query in this phase is a composition of existing validated patterns. The financial skill already has all the building blocks; this phase assembles them into analytical report formats.

## Common Pitfalls

### Pitfall 1: AR/AP Date Field Confusion
**What goes wrong:** Using DueDate for AR aging produces nonsensical results because DueDate on Type 1 orders is the production/shipping deadline, not the payment deadline.
**Why it happens:** "DueDate" sounds like it should be the payment due date. It is -- but only on Type 8 bills (vendor payment deadline).
**How to avoid:** AR ages from SaleDate. AP ages from DueDate. Document this difference explicitly in every AR/AP query comment.
**Warning signs:** AR aging shows most invoices as "90+ days" when they're actually current -- because DueDate was set weeks before SaleDate.

### Pitfall 2: GL Sign Convention Inversion
**What goes wrong:** Revenue shows as negative, or P&L shows losses when the company is profitable.
**Why it happens:** GL uses accounting double-entry: revenue = credit = negative amount in Ledger. Forgetting SUM(-Amount) for revenue accounts inverts the sign.
**How to avoid:** Standard rule: SUM(-Amount) for GLClassificationType IN (4000, 4001). SUM(Amount) for everything else. Encode this in every P&L query.
**Warning signs:** Revenue totals are negative. Net income is the opposite of expected.

### Pitfall 3: WIP/Built Orders in AR Report
**What goes wrong:** Including WIP/Built orders in AR aging buckets inflates the aging balances and misrepresents overdue amounts.
**Why it happens:** WIP/Built orders have BalanceDue > 0 but haven't been sold yet (SaleDate IS NULL). They can't be "aged" because there's no invoice date.
**How to avoid:** Show WIP/Built orders in a separate bucket labeled "WIP/Built (not yet invoiced)" rather than in the aging columns. Control's own AR report does this.
**Warning signs:** A large "WIP/Built" bucket in the AR report, or orders with NULL SaleDate appearing in the 90+ day bucket.

### Pitfall 4: P&L Product-Line COGS Gap
**What goes wrong:** User asks for margin by product line but COGS is shown as a single number, not broken out by product.
**Why it happens:** COGS GL accounts (GLClassificationType 5001) are not product-specific. Revenue accounts have per-product NodeIDs, but COGS accounts are aggregate.
**How to avoid:** Document this limitation explicitly. Revenue breakdown by product is available; COGS breakdown requires TransDetail-level analysis (not GL-based). Offer aggregate margin as the default, note that per-product COGS would require a different approach.
**Warning signs:** User expects per-product margins but gets total COGS across all products.

### Pitfall 5: Cash Flow Double-Counting via Undeposited Accounts
**What goes wrong:** Cash inflows are counted twice -- once when payment hits undeposited account, again when deposit moves to bank.
**Why it happens:** Control uses a two-step deposit process: Payment -> Undeposited -> Bank. Both steps create GL entries.
**How to avoid:** For cash flow, track only the final movement to/from bank accounts (GLClassificationType = 1000), OR track only the initial payment receipt to undeposited accounts, but never both.
**Warning signs:** Cash inflows are roughly double the expected amount. Bank balance doesn't match GL balance.

### Pitfall 6: Historical vs Current Balance Confusion
**What goes wrong:** Bank balance query includes a date range filter, producing a "period activity" number instead of the cumulative balance.
**Why it happens:** P&L queries use date ranges; bank balance queries should NOT (they need cumulative all-time totals).
**How to avoid:** Bank balance = SUM(GL.Amount) with NO date filter (or WHERE EntryDateTime <= @AsOfDate for point-in-time). Cash flow activity = SUM with date range. Different queries for different purposes.
**Warning signs:** "Bank balance" shows a small number that looks like monthly activity rather than an account balance.

## Code Examples

### AR Aging with Customer Breakdown (FIN-04)

```sql
-- Source: Existing skill AR aging pattern + wiki Historical AR Report structure
-- AR Detail with customer breakdown and aging buckets
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

### Single Customer AR Drill-Down (FIN-04 extension)

```sql
-- When user asks "show me AR for [customer name]"
-- Same query as above but with customer filter and invoice-level detail
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

### AP Aging with Vendor Breakdown (FIN-05)

```sql
-- Source: Existing skill AP aging pattern
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
    -- AP aging from DueDate
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

### Vendor Payment History (FIN-05 extension)

```sql
-- When user asks "payment history for [vendor]"
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

### P&L Traditional Format (FIN-06)

```sql
-- Source: Existing skill Full P&L + P&L Summary patterns
-- Default: year-to-date
DECLARE @StartDate DATETIME = DATEADD(yy, DATEDIFF(yy, 0, GETDATE()), 0); -- Jan 1 of current year
DECLARE @EndDate DATETIME = GETDATE();

SELECT
    CASE ga.GLClassificationType
        WHEN 4000 THEN '1-Product Revenue'
        WHEN 4001 THEN '2-Other Income'
        WHEN 5001 THEN '3-Cost of Goods Sold'
        WHEN 5002 THEN '4-Operating Expenses'
    END AS Category,
    ga.AccountName,
    CASE
        WHEN ga.GLClassificationType IN (4000, 4001) THEN SUM(-l.Amount)
        ELSE SUM(l.Amount)
    END AS Amount
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType IN (4000, 4001, 5001, 5002)
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY ga.GLClassificationType, ga.AccountName
HAVING SUM(l.Amount) != 0
ORDER BY Category, Amount DESC
```

### P&L with Period Comparison (FIN-06 extension)

```sql
-- Year-over-year comparison (current YTD vs prior YTD)
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

### Bank Balance and Cash Position (FIN-07)

```sql
-- Source: Existing skill Balance Sheet Key Accounts pattern
-- Current bank balances (cumulative, no date filter)
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

### Cash Flow Summary (FIN-07 extension)

```sql
-- Cash in/out by category for a date range (default YTD)
DECLARE @StartDate DATETIME = DATEADD(yy, DATEDIFF(yy, 0, GETDATE()), 0);
DECLARE @EndDate DATETIME = GETDATE();

SELECT
    CASE
        WHEN l.GLAccountID IN (14, 24) AND l.Amount < 0 THEN 'Customer Receipts'
        WHEN l.GLAccountID = 22 AND l.Amount > 0 THEN 'Vendor Payments'
        WHEN l.PayrollID IS NOT NULL THEN 'Payroll'
        WHEN l.EntryType = 1 THEN 'Manual/Adjustments'
        ELSE 'Other'
    END AS CashCategory,
    SUM(l.Amount) AS NetAmount
FROM GL l
WHERE l.GLAccountID IN (90, 10412)  -- Bank accounts only (final destination)
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY
    CASE
        WHEN l.GLAccountID IN (14, 24) AND l.Amount < 0 THEN 'Customer Receipts'
        WHEN l.GLAccountID = 22 AND l.Amount > 0 THEN 'Vendor Payments'
        WHEN l.PayrollID IS NOT NULL THEN 'Payroll'
        WHEN l.EntryType = 1 THEN 'Manual/Adjustments'
        ELSE 'Other'
    END
ORDER BY NetAmount DESC
```

**Note on cash flow categorization:** The categorization above is approximate. GL entries hitting bank accounts don't always carry clear source indicators. The most reliable categories are:
- **Customer receipts:** Deposit journal entries (DepositJournalID IS NOT NULL) debiting bank accounts
- **Vendor payments:** Bill payment entries (linked to TransactionType=8 via TransactionID)
- **Payroll:** Entries with PayrollID IS NOT NULL

A more precise approach queries the bank account GL entries and joins back to Journal.ClassTypeID to identify the activity type:
- ClassTypeID 20001 = Order Payment
- ClassTypeID 20009 = Bill Payment
- ClassTypeID 20500 = Manual Journal Entry

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Query Ledger directly | Use GL view | v1.0 Phase 5 | Excludes ~307K off-balance-sheet entries |
| Guess at NodeIDs | Complete NodeID reference in skill | v1.0 Phase 5 | All 452 GLAccount entries documented |
| Manual sign handling | Standard sign convention rules | v1.0 Phase 5 | SUM(-Amount) for 4000/4001, SUM(Amount) otherwise |
| No date defaults | YTD default for all period queries | Phase 9 (new) | Consistent user experience across P&L and cash flow |
| Summary-only AR/AP | Customer/vendor breakdown with aging | Phase 9 (new) | Matches Control's built-in A_R Detail report format |

**Nothing deprecated or outdated.** All patterns from v1.0 remain valid. This phase adds analytical depth, not replacements.

## Sizing and Structure Decision

**Current skill:** 856 lines
**Estimated new content:** ~350-450 lines (4 sections x ~100 lines each)
**Projected total:** ~1,200-1,300 lines

**Recommendation:** Start by adding content to the main SKILL.md file. If the total exceeds 1,200 lines after all four sections are added, extract the new analytical sections to `references/financial-analysis.md` and keep summaries + routing in the main file. The 1,200-line threshold was set in the context decisions as the extraction trigger.

**Section placement in existing skill:**
1. After "## ACCOUNTS RECEIVABLE" section (line ~500): Add AR Detail with customer breakdown
2. After "## ACCOUNTS PAYABLE" section (line ~593): Add AP Detail with vendor breakdown
3. After "## P&L FROM GL" section (line ~656): Add P&L comparison and product-line analysis
4. After "## ALL-TIME REVENUE CONTEXT" section (line ~756): Add Cash Flow and Bank Balance section
5. Extend "## NATURAL LANGUAGE INTERPRETATION" table (line ~760) with new routing entries

## Validation Strategy

Each requirement maps to a specific Control built-in report for validation:

| Requirement | Control Report to Match | Key Validation Point |
|-------------|------------------------|---------------------|
| FIN-04 (AR aging) | A_R Detail report | Total AR should match ~$80,899 (or current value). Customer breakdown should match. |
| FIN-05 (AP tracking) | A/P Aging Detail | Total AP should match ~$125,319 (or current value). Vendor breakdown should match. |
| FIN-06 (P&L) | Income Statement / Trial Balance | YTD revenue should approximate core baseline ($3.05M annualized). GL totals should match Trial Balance. |
| FIN-07 (Cash flow) | Bank Account registers | Bank balances should match GL account registers. Cash in/out should reconcile with Payment table totals. |

**Validation approach:** Run each new query template against the live database and compare output with the corresponding Control report. Use the GL Balance Integrity Check query (from wiki) as a cross-validation for GL-sourced queries.

## Open Questions

1. **Cash flow categorization precision**
   - What we know: GL entries hitting bank accounts can be categorized by joining to Journal.ClassTypeID and Payment.ClassTypeID
   - What's unclear: Whether the categorization is granular enough for user expectations (e.g., separating "customer payments" from "deposits of undeposited funds" cleanly)
   - Recommendation: Start with the simpler approach (categorize by GL entry source), refine if validation shows gaps. The main categories (customer receipts, vendor payments, payroll) should cover 90%+ of entries.

2. **Product-line COGS granularity**
   - What we know: Revenue has per-product GL accounts; COGS does not
   - What's unclear: Whether users will expect per-product margin or accept aggregate COGS
   - Recommendation: Document the limitation in the skill. Revenue by product line is available; COGS is aggregate. If per-product margin is needed, it requires a TransDetail-level query (different from GL-based P&L).

3. **Payment forecast (mentioned in FIN-07 requirement)**
   - What we know: The requirement mentions "payment forecast" but the context decisions focus on bank balance + cash in/out
   - What's unclear: Whether a forward-looking forecast is expected or just historical cash flow
   - Recommendation: Implement historical cash flow first. A simple forecast could extrapolate average daily cash flow, but this is speculative. Defer forecasting to an analytics milestone.

## Sources

### Primary (HIGH confidence)
- `skills/control-erp-financial/control-erp-financial-SKILL.md` -- existing validated skill (856 lines), contains all GL architecture, sign conventions, AR/AP/P&L patterns, payment flows
- `output/schemas/TransHeader.md` -- full column reference for TransHeader (AR/AP source table)
- `output/schemas/GLAccount.md` -- full column reference for GLAccount (chart of accounts)
- `output/schemas/Ledger.md` -- full column reference for Ledger/GL (53 columns)
- `output/schemas/Payment.md` -- full column reference for Payment table
- `output/schemas/PaymentTerms.md` -- full column reference for PaymentTerms (grace periods, early payment discounts)
- `output/schemas/Account.md` -- full column reference for Account (HasCreditAccount, CreditBalance, VendorCreditBalance, VendorPaymentTermsID)

### Secondary (HIGH confidence -- from wiki, verified against live data in v1.0)
- `output/wiki/extracts/orders_accounting_knowledge.md` -- GL lifecycle, payment processing, AP/PO lifecycle
- `output/wiki/extracts/sql_queries_reference.md` -- GL integrity checks, AR/AP integrity checks, bank register SQL
- `output/wiki/reports/historical_ar_report_standard.md` -- Control's Historical AR Report structure (tables, joins, fields)
- `output/wiki/support/historical_ar_balance_by_order_as_of_a_specified_date.md` -- GL-based historical AR query
- `output/report_join_patterns.md` -- Crystal Reports inferred join patterns for AR, AP, Financial reports

### Tertiary (MEDIUM confidence)
- `output/wiki/support/control_sql_-_plug_all_ar_balances_for_orders_not_in_sale.md` -- AR repair patterns (confirms GLAccountID=14 semantics, confirms StatusID=3 requirement for AR GL entries)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- All tables/views documented and validated in v1.0
- Architecture: HIGH -- Extending existing validated patterns, not introducing new ones
- Pitfalls: HIGH -- All pitfalls documented from real v1.0 validation experience (DueDate bug, sign convention, GL view vs Ledger)
- Code examples: HIGH -- Based on existing validated queries with composition into new formats

**Research date:** 2026-02-09
**Valid until:** Indefinite (stable database schema, no expected changes to Control ERP GL architecture)
