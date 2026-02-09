---
name: control-erp-financial
description: Query FLS Banners financial data from Control ERP including GL/Ledger transactions, Accounts Receivable, Accounts Payable, P&L reporting, payment tracking, and balance sheet queries. Use when the user asks about AR, AP, aging, payments, invoices, bills, GL entries, profit & loss, cash position, expenses, COGS, or any accounting/financial question. Depends on control-erp-core for business rules.
---

# Control ERP Financial — GL, AR, AP & Payment Intelligence

**Depends on:** `control-erp-core` (always read core first for business rules)

---

## GL / LEDGER ARCHITECTURE

Control uses a **Ledger table** for all general ledger entries. The commonly referenced "GL" is actually a VIEW:

```sql
-- GL is a filtered view of Ledger
CREATE VIEW GL AS SELECT * FROM Ledger WHERE OffBalanceSheet = 0
```

- **Ledger** = ALL entries (on-balance + off-balance sheet)
- **GL** (view) = Financial reporting entries only (excludes off-balance sheet)
- **Off-balance sheet entries** = cost tracking for parts expensed at purchase (prevents double-counting)

**Always use `GL` view for financial queries** unless you specifically need off-balance sheet data.

### Key GL/Ledger Fields
- `GLAccountID` → `GLAccount.ID` (NodeID) — which account
- `Amount` — positive = debit, negative = credit
- `EntryDateTime` — when posted
- `TransHeaderID` — linked order/bill (when applicable)
- `ID` — sequential entry ID (GL ID in account registers)

### Complete Ledger Field Reference

The Ledger table has 53 columns total. Beyond the 5 key fields above, these are the most query-relevant fields:

| Field | Type | Purpose |
|-------|------|---------|
| EntryType | int | Entry category: NULL/0=unset, 1=Manual/Adjustments, 2=Order lifecycle (dominant), 3=Payment-related, 4=Credit adjustments, 5=Bill lifecycle |
| Classification | int | Fine-grained event classification (49 values). 0=general, 100=sale events. Rarely needed for user queries |
| OffBalanceSheet | bit | 0=on-balance sheet (in GL view), 1=off-balance sheet (filtered from GL view) |
| DepositJournalID | int | Links payment GL entries to their deposit batch Journal |
| Reconciled | bit | Whether entry has been bank-reconciled |
| ReconciliationDateTime | datetime | When the entry was reconciled |
| DivisionID | int | Division the entry belongs to (FK to Division) |
| StationID | int | Production station for cost entries (FK to Station) |
| PartID | int | Part/material for inventory entries (FK to Part) |
| PayrollID | int | Payroll entry link (FK to Payroll) |
| WarehouseID | int | Warehouse for inventory entries (FK to Warehouse) |
| ClassTypeID | int | Almost always 8900 for all Ledger entries |

For full schema, see `output/schemas/Ledger.md`.

#### EntryType Reference

| Value | Meaning | Typical Count |
|-------|---------|--------------|
| NULL | Unset/legacy | varies |
| 1 | Manual journal entries (bank fees, loan payments) | ~2K |
| 2 | Order lifecycle entries | ~2.3M (dominant) |
| 3 | Payment-related entries | ~397K |
| 4 | Credit adjustments | ~30 |
| 5 | Bill lifecycle entries | ~743 |

**Confidence: MEDIUM** — Values inferred from Description pattern correlation, not official documentation.

### Sign Conventions
- **Revenue accounts:** Negative amounts (credits). Use `SUM(-Amount)` for positive revenue.
- **Expense/COGS accounts:** Positive amounts (debits). Use `SUM(Amount)` for expenses.
- **Asset accounts:** Positive = increase (debit), negative = decrease (credit)
- **Liability accounts:** Negative = increase (credit), positive = decrease (debit)

### Off-Balance Sheet Entries

Ledger entries with `OffBalanceSheet = 1` are excluded from the GL view. These track costs for parts that were expensed at purchase (not carried as inventory assets).

**Why they exist:** Control supports two part cost tracking modes:

- **Accrual (financial):** Parts are inventory assets until consumed. Uses NodeID 10414 (Inventory). Appears in GL view. Standard financial reporting approach.
- **Cost Accounting (non-financial):** Parts are expensed at purchase. Uses NodeIDs 60/61 (Unclassified Inventory/Expense). Off-balance sheet to prevent double-counting since parts were already expensed when purchased.

**When to include them:** Only when analyzing total production costs including non-accrual materials. For standard financial reporting, the GL view correctly excludes them.

**Count context:** ~307K off-balance sheet entries vs ~2.44M on-balance sheet entries (~11% of all Ledger entries).

**Query example — Total cost including off-balance sheet entries for an order:**
```sql
SELECT
    CASE l.OffBalanceSheet WHEN 0 THEN 'Financial (GL)' ELSE 'Cost Accounting (OBS)' END AS EntryType,
    SUM(l.Amount) AS TotalCost
FROM Ledger l
WHERE l.TransHeaderID = @TransHeaderID
    AND l.GLAccountID IN (10414, 34, 60, 61)  -- inventory/cost accounts
GROUP BY l.OffBalanceSheet
```

---

## GLACCOUNT — CHART OF ACCOUNTS

The GLAccount table stores all accounts in a hierarchical tree structure.

### Key Fields
- `ID` (NodeID) — unique identifier, used as `GLAccountID` in Ledger
- `AccountName` — display name
- `ClassTypeID` — 8000 = category/folder, 8001 = leaf account
- `AccountGroupID` — parent node for hierarchy
- `GLClassificationType` — account type classification
- `ExportAccountNumber` — external accounting number
- `IsActive` — active flag

### GLClassificationType Reference

| Code | Category | Description |
|------|----------|-------------|
| **Balance Sheet** | | |
| 1000 | Cash/Bank | Bank accounts, cash on hand |
| 1002 | Other Current Assets | AR, presale assets, transfers |
| 1003 | Inventory | Raw materials, finished goods |
| 1004 | Fixed/Other Assets | Equipment, vehicles, notes receivable |
| 1005 | Other Assets | Deposits, NR non-current |
| 1007 | Undeposited Funds | Payment gateway holding accounts |
| 2000 | Accounts Payable | AP, AP contra |
| 2001 | Credit Cards | Company credit card accounts |
| 2002 | Current Liabilities | Prepayments, payroll, presale liabilities |
| 2003 | Long-term Liabilities | Notes payable, loans |
| 2005 | Tax Liabilities | Sales tax (WI, Door County) |
| 3000 | Equity | Common stock, retained earnings |
| **Income Statement** | | |
| 4000 | Product Income | ~51 revenue accounts by product line |
| 4001 | Other Income | Interest, freight reimbursement, misc |
| 5001 | COGS | Purchases, freight, cost of goods |
| 5002 | Operating Expenses | ~92 accounts (payroll, insurance, rent, etc.) |
| 8000 | Category | Folder/grouping node — not a real account |

### Critical System Account NodeIDs

These are the accounts referenced in GL transaction flows. Memorize these:

| NodeID | Account | Normal Balance |
|--------|---------|---------------|
| 11 | WIP | Debit (asset) |
| 12 | Built | Debit (asset) |
| 14 | Accounts Receivable | Debit (asset) |
| 21 | Orders Due | Credit (liability) |
| 22 | Accounts Payable | Credit (liability) |
| 23 | Customer Credit Balance | Credit (liability) |
| 24 | Order Prepayments | Credit (liability) |
| 90 | Cash-Associated Bank-Checking | Debit (asset) |
| 92 | Undeposited MCVisa | Debit (asset) |
| 10137 | Undeposited Amex | Debit (asset) |
| 10412 | Cash-Associated Bank-MM | Debit (asset) |
| 10414 | Inventory | Debit (asset) |

### Supporting System Accounts

These accounts support the core GL lifecycle and appear in specific workflow stages:

| NodeID | Account | Classification | Purpose |
|--------|---------|---------------|---------|
| 15 | Direct Costs from Bills | 1002 (Current Asset) | Intermediary for bill costs assigned to orders |
| 25 | Vendor Credit | 1002 (Current Asset) | Vendor credit balances |
| 34 | Cost Of Built - FGI | 1002 (Current Asset) | Holds costs during Built status before sale |
| 52 | Credit Given | 5002 (Expense) | Customer credit adjustments expense |
| 60 | Unclassified Inventory | 1003 (Inventory) | Off-balance sheet cost tracking for expensed parts |
| 61 | Unclassified Expense | 5002 (Expense) | Off-balance sheet expense tracking |
| 93 | Undeposited Checks | 1007 (Undeposited Funds) | Check deposits pending |
| 543 | Undeposited Discover | 1007 (Undeposited Funds) | Discover CC deposits pending |
| 10528 | Undeposited Authorize.net | 1007 (Undeposited Funds) | Online CC deposits pending |
| 10530 | Undeposited PayPal | 1007 (Undeposited Funds) | PayPal deposits pending |
| 10531 | Undeposited Shopify | 1007 (Undeposited Funds) | Shopify deposits pending |

### Complete Undeposited Fund Accounts

All accounts with GLClassificationType = 1007 (payment gateway holding accounts):

| NodeID | Account | Status |
|--------|---------|--------|
| 91 | Undeposited Cash | Active |
| 92 | Undeposited MCVisa | Active |
| 93 | Undeposited Checks | Active |
| 543 | Undeposited Discover | Active |
| 10137 | Undeposited Amex | Active |
| 10147 | Undeposited Discover old | Active |
| 10528 | Undeposited Authorize.net | Active |
| 10530 | Undeposited PayPal | Active |
| 10531 | Undeposited Shopify | Active |
| 10522 | ZT Undeposited MCVisa | Legacy (ZoomTex) |
| 10523 | ZT Undeposited Amex | Legacy (ZoomTex) |
| 10524 | ZT Undeposited Checks | Legacy (ZoomTex) |
| 10525 | ZT Undeposited Cash | Legacy (ZoomTex) |

### Revenue Account NodeIDs (GLClassificationType 4000)

**Core Fabric/Dye Sub Products:**

| NodeID | Account |
|--------|---------|
| 10116 | DyeSub |
| 10120 | DyeSub Table Covers |
| 10126 | Dyelux Flag |
| 10127 | Dyelux Golf Flag |
| 10258 | Feather Flags |
| 10533 | Dyesub Displays |
| 10534 | SEG |
| 10535 | Printed Fabric |
| 10537 | Printed Paper |
| 10132 | Solvent |

**Displays & Accessories:**

| NodeID | Account |
|--------|---------|
| 10257 | Pop-Up Tents |
| 10122 | Carry Bags |
| 10123 | Tri Fold Displays |
| 10124 | Roll Up Banners |
| 10125 | Banner Stand |
| 10039 | Stakes and Posts |
| 10128 | Golf Pin |
| 10129 | Putting Cup |
| 10130 | Flag Swivel |

**Screen Printing & Garments:**

| NodeID | Account |
|--------|---------|
| 6053 | Screenprinting |
| 10086 | SP Banners |
| 10114 | SP Flags |
| 10115 | SP Table Covers |
| 10099 | Embroidery |
| 10100 | Screenprint (garment) |

**Subcontract:**

| NodeID | Account |
|--------|---------|
| 10119 | SubCon Flag |
| 10037 | SubCon Sublimation |
| 10118 | SubCon Vinyl |
| 10131 | SubCon Table Cover |
| 10255 | Sub Con Screenprint |

**Services, Shipping & Adjustments:**

| NodeID | Account |
|--------|---------|
| 400 | Shipping Sales |
| 10038 | Shipping |
| 6067 | Service |
| 10040 | Services |
| 10036 | Installation |
| 6029 | Outsource |
| 10317 | Discounts Given (contra) |
| 53 | Credit Memo |
| 10440 | Write Off |

### COGS Account NodeIDs (GLClassificationType 5001)

| NodeID | Account |
|--------|---------|
| 10178 | Cost of Goods Sold |
| 10180 | Purchases |
| 10182 | Contract Purchases |
| 10183 | Credit Card Charge Purchases |
| 10184 | Dye Sub Purchases |
| 10185 | Fabric Purchases |
| 10186 | Garment Purchases |
| 10527 | Subcontract Purchases |
| 10493 | Freight (COGS) |

---

## GL TRANSACTION FLOWS

### Order Lifecycle — GL Entries

**Stage 1: New Order Created**
```
Debit  WIP (NodeID 11)          $order_total
Credit Orders Due (NodeID 21)   $order_total
```

**Stage 2: Deposit/Prepayment Received**
```
Debit  Undeposited MCVisa (92) or Undeposited Cash (91) or Undeposited Checks (93)   $payment
Credit Order Prepayments (NodeID 24)                                                 $payment
```

### Built Status — Cost Flow (Stage 2.5)

When an order is marked Built (StatusID 2), costs are transferred from WIP to a Built intermediary account before the order is sold. This stage holds finished goods inventory.

**GL entries when order marked Built:**
```
-- Transfer from WIP to Built
Credit WIP (11)                       $order_total
Debit  Built (12)                     $order_total

-- Cost tracking (accrual-tracked parts)
Credit Inventory (10414)              $accrual_cost
Debit  Cost Of Built - FGI (34)       $accrual_cost

-- Cost tracking (non-accrual parts, OFF-BALANCE SHEET)
Credit Unclassified Inventory (60)    $nonaccrual_cost   [OffBalanceSheet=1]
Debit  Unclassified Expense (61)      $nonaccrual_cost   [OffBalanceSheet=1]
```

**Key points:**
- Built status is an intermediate hold between production and sale
- NodeID 34 (Cost Of Built - FGI = Finished Goods Inventory) holds costs until the order is sold
- At sale, NodeID 34 clears as costs move to COGS (Cost of Goods Sold)
- Non-accrual parts use off-balance sheet tracking (OffBalanceSheet=1 in Ledger) to avoid double-counting parts that were expensed at purchase
- Not all orders go through Built status — some go directly from WIP to Sale

**Query to analyze Built cost entries:**
```sql
SELECT TOP 10 l.GLAccountID, ga.AccountName, l.Amount, l.OffBalanceSheet, l.Description
FROM Ledger l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
INNER JOIN Journal j ON l.JournalID = j.ID
WHERE j.ActivityType = 6  -- Marked Built
  AND l.GLAccountID IN (11, 12, 34, 60, 61, 10414)
ORDER BY l.ID DESC
```

**Stage 3a: Sale — Prepaid Order** (payment received before sale)
```
-- Reverse presale entries
Debit  Orders Due (21)              $order_total
Credit WIP (11)                     $order_total
-- Apply prepayment to revenue
Debit  Order Prepayments (24)       $prepaid_amount
Credit [Product Revenue Account]    $product_revenue  (multiple lines by product)
Credit [Shipping Account]           $shipping
-- COGS recognition
Debit  Cost of Goods Sold           $cogs
Credit Inventory (10414)            $cogs
```

**Stage 3b: Sale — Unpaid/Credit Order** (payment expected after sale)
```
-- Same as 3a, except:
Debit  Accounts Receivable (14)     $balance_due
-- instead of Order Prepayments
```

### Bill Lifecycle — GL Entries

**Bill Created:**
```
Debit  Freight / COGS / Expense     $line_amounts
Credit Accounts Payable (22)        $bill_total
```

**Bill Paid:**
```
Debit  Accounts Payable (22)                  $payment
Credit Cash-Associated Bank-Checking (90)     $payment
```

---

## PAYMENT POSTING PATTERNS

**Critical rule: Orders flow through ONE GL account path, not both.**

### Path A: Prepaid Orders → Order Prepayments (NodeID 24)

| Step | GL Entry | Register Description |
|------|----------|---------------------|
| Payment received | Credit Order Prepayments | "Order Payment ({Type} {Amount}) for #{Order}" |
| Order sold | Debit Order Prepayments | "Order Status Marked Sale" |

~70% of orders follow this path. Payment types: Check, Cash/ACH, Capture (CC auth).

### Path B: Credit/AR Orders → Accounts Receivable (NodeID 14)

| Step | GL Entry | Register Description |
|------|----------|---------------------|
| Order sold | Debit AR | "Order Status Marked Sale" |
| Payment received | Credit AR | "Order Payment ({Type} {Amount}) for #{Order}" |

~30% of orders follow this path. Mostly CC captures charged after shipment.

### Payment Description Format in GL
```
Order Payment ({Type} {Amount}) for #{OrderNumber} - {CompanyName}
```
Types: `Check`, `Cash`, `Capture` (CC auth), `MC/Visa`, `Amex`, `Discover`,
`Credit Applied` (customer credit), `Inksoft`, `Shopify`, `PayPal`

### Other GL Register Entries
- "Order Edited" — price adjustments create GL corrections
- "Voided Payment on Order NNNNN" — payment reversal
- "Finance Charge" — late fees on AR accounts

### Deposit Workflow (Undeposited -> Bank)

Payments go through a two-step deposit process:

**Step 1 - Payment Received** (documented in Paths A/B above):
```
Debit  Undeposited [Method] (92/93/etc.)  $payment
Credit Order Prepayments (24) or AR (14)  $payment
```

**Step 2 - Deposit Processed**:
```
Debit  Cash-Checking (90)                 $deposit_total
Credit Undeposited [Method] (92/93/etc.)  $deposit_total
```

Key points:
- Deposits group multiple payments into a single batch via `Ledger.DepositJournalID`
- `Payment.Undeposited` flag (bit) indicates whether a payment has been deposited yet (1 = not deposited, 0 = deposited)
- Deposits are created in Control via the "Make Deposit" function
- Until deposited, funds sit in the Undeposited account and do NOT appear in Cash-Checking
- Once deposited, all payments in the batch share the same `DepositJournalID` value

**Query to find undeposited payments:**
```sql
SELECT p.ID, p.Amount, p.TenderType, ga.AccountName AS UndepositedAccount
FROM Payment p
INNER JOIN GLAccount ga ON p.BankAccountID = ga.ID
WHERE p.Undeposited = 1 AND p.IsActive = 1
```

**Query to find deposit batch entries:**
```sql
SELECT l.ID, l.GLAccountID, ga.AccountName, l.Amount, l.DepositJournalID, l.EntryDateTime
FROM Ledger l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE l.DepositJournalID IS NOT NULL
  AND l.GLAccountID IN (90, 91, 92, 93, 543, 10137, 10528, 10530, 10531)
ORDER BY l.DepositJournalID DESC, l.ID
```

### Payment.TenderType Reference

The `Payment.TenderType` field indicates the payment method and determines the default BankAccountID (undeposited account).

| TenderType | Payment Method | Default BankAccountID |
|------------|---------------|----------------------|
| 0 | Cash | 91 (Undeposited Cash) |
| 1 | Check | 93 (Undeposited Checks) |
| 2 | Credit Card (MC/Visa) | 92 (Undeposited MCVisa) |
| 4 | Online/Authorize.net | 10528 (Undeposited Authorize.net) |
| 5 | ACH/Wire | 90 (Cash-Checking direct) |
| 7 | Wire Transfer | 90 (Cash-Checking direct) |
| 8 | Other/Legacy | varies |
| NULL | System entries | N/A (master payments, failed, etc.) |

**Note:** TenderType values 3 and 6 are not observed at FLS. ACH (5) and Wire (7) post directly to Cash-Checking (90), bypassing the undeposited step.

**Query to analyze payment methods:**
```sql
SELECT p.TenderType, COUNT(*) AS Cnt, MIN(ga.AccountName) AS TypicalBankAccount
FROM Payment p
LEFT JOIN GLAccount ga ON p.BankAccountID = ga.ID
WHERE p.IsActive = 1
GROUP BY p.TenderType
ORDER BY p.TenderType
```

### Payment ClassTypeID Reference

The `Payment.ClassTypeID` field indicates the payment subtype. Most payment queries should filter to ClassTypeID IN (20001, 20009) for individual order and bill payments.

| ClassTypeID | Type | Description |
|-------------|------|-------------|
| 20000 | Master Payment | Parent grouping for multi-order payments |
| 20001 | Order Payment | Payment applied to a specific order |
| 20002 | Credit Payment | Payment from customer credit balance |
| 20003 | Change Returned | Cash change given back to customer |
| 20004 | Order Refund | Refund issued against an order |
| 20005 | Credit Refund | Refund issued from customer credit |
| 20006 | Credit Memo Payment | Payment created from credit memo |
| 20007 | Failed Payment | Payment attempt that failed (CC decline, etc.) |
| 20009 | Bill Payment | Payment made to a vendor for a bill |
| 20011 | Overpayment | Payment exceeding balance due |
| 20012 | Payment to Credit | Payment transferred to customer credit balance |
| 20032 | Master Payment Authorization | CC authorization grouping for multi-order auth |
| 20037 | Master Bill Payment | Parent grouping for multi-bill payments |
| 20038 | Receipt | Receiving document payment (PO receipts) |

**For most payment queries, filter to ClassTypeID IN (20001, 20009)** to get individual order and bill payments. Use 20000/20037 when analyzing payment batches.

**Query to analyze payment types:**
```sql
SELECT ClassTypeID, COUNT(*) AS Cnt
FROM Payment
WHERE IsActive = 1
GROUP BY ClassTypeID
ORDER BY ClassTypeID
```

---

## ACCOUNTS RECEIVABLE

### AR = TransHeader-Based (Not GL-Based)

AR reporting uses TransHeader fields, NOT the GL Ledger. The AR report includes WIP/Built orders for visibility, which haven't posted to the GL AR account yet.

**Key fields:** `BalanceDue`, `PaymentTotal`, `SaleDate`, `StatusID`

**DueDate WARNING:** `TransHeader.DueDate` = **production/shipping deadline**, NOT payment due date. Payment due date is calculated at runtime as `SaleDate + PaymentTerms.GracePeriod`.

### AR Snapshot
```sql
SELECT
    a.CompanyName,
    th.OrderNumber,
    th.SaleDate,
    th.BalanceDue,
    th.TotalPrice,
    th.PaymentTotal,
    pt.TermsName,
    pt.GracePeriod
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
LEFT JOIN PaymentTerms pt ON a.PaymentTermsID = pt.ID
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.StatusID != 9  -- exclude voided
ORDER BY th.BalanceDue DESC
```

### AR Aging Buckets
```sql
SELECT
    CASE
        WHEN th.SaleDate IS NULL THEN 'WIP/Built'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 30 THEN '0-30 days'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 60 THEN '31-60 days'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 90 THEN '61-90 days'
        ELSE '90+ days'
    END AS AgingBucket,
    COUNT(*) AS Invoices,
    SUM(th.BalanceDue) AS ARBalance
FROM TransHeader th
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.StatusID != 9
GROUP BY
    CASE
        WHEN th.SaleDate IS NULL THEN 'WIP/Built'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 30 THEN '0-30 days'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 60 THEN '31-60 days'
        WHEN DATEDIFF(DAY, th.SaleDate, GETDATE()) <= 90 THEN '61-90 days'
        ELSE '90+ days'
    END
```

### AR by Payment Terms
```sql
SELECT
    pt.TermsName,
    COUNT(*) AS OpenInvoices,
    SUM(th.BalanceDue) AS ARBalance
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
LEFT JOIN PaymentTerms pt ON a.PaymentTermsID = pt.ID
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.StatusID != 9
GROUP BY pt.TermsName
ORDER BY ARBalance DESC
```

### AR Context (Validated 2026-02-07)
- Total AR: ~$80,899 (119 invoices, 30 customers)
- 93% current (0-30 days)
- Net 30 = 57% of AR balance, 68% of invoices
- FLS is predominantly prepaid — AR is small relative to revenue
- Credit Card terms ($19.4K) = current invoices awaiting charge, not failed transactions
- Charge account customers marked by `Account.HasCreditAccount = 1` (shown as `*` prefix in AR report)

### Payment Terms
- 21 active terms in `PaymentTerms` table
- Key terms: Net 30 (ID 2), Due Upon Receipt, Net 45, 50%/50% Net 30 (ID 1009), Credit Card, Cash Customer, Net 10, Net 15
- Fields: `TermsName`, `GracePeriod`, `InterestRate`, `FeesBasedOnSaleDate`
- Customer terms: `Account.PaymentTermsID`
- Order-level override: `TransHeader.POPaymentTermsID`

### AR Detail with Customer Breakdown

Matches Control's A_R Detail report format: customer breakdown with aging columns, WIP/Built orders shown separately (not mixed into aging buckets), charge account customers marked with asterisk (*).

**AR ages from SaleDate (invoice date), NEVER DueDate.** DueDate on Type 1 orders is the production/shipping deadline, not the payment deadline. WIP/Built orders (SaleDate IS NULL) appear in a separate bucket because they have no invoice date to age from.

#### Full AR Detail by Customer
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

#### Single Customer AR Drill-Down
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

#### AR Summary by Customer
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

## ACCOUNTS PAYABLE

### AP = TransHeader Type 8 (Bills)

AP uses the same TransHeader table with `TransactionType = 8`.

**Bill StatusIDs:** 4 = Closed, 6 = Open, 9 = Voided

### AP Snapshot
```sql
SELECT
    a.CompanyName AS Vendor,
    th.OrderNumber AS BillNumber,
    th.BalanceDue,
    th.TotalPrice AS BillAmount,
    th.DueDate
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 8
    AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.StatusID != 9
ORDER BY th.BalanceDue DESC
```

### AP Aging
```sql
SELECT
    CASE
        WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 0 THEN 'Current'
        WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 30 THEN '0-30 days'
        WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 60 THEN '31-60 days'
        WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 90 THEN '61-90 days'
        ELSE '90+ days'
    END AS AgingBucket,
    COUNT(*) AS Bills,
    SUM(th.BalanceDue) AS APBalance
FROM TransHeader th
WHERE th.TransactionType = 8
    AND th.IsActive = 1
    AND th.BalanceDue > 0
    AND th.StatusID != 9
GROUP BY
    CASE
        WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 0 THEN 'Current'
        WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 30 THEN '0-30 days'
        WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 60 THEN '31-60 days'
        WHEN DATEDIFF(DAY, th.DueDate, GETDATE()) <= 90 THEN '61-90 days'
        ELSE '90+ days'
    END
```

**AP vs AR aging difference:** AP ages from `TransHeader.DueDate` (vendor payment deadline). AR ages from `SaleDate` (invoice date). DueDate has different meanings by transaction type.

### AP Context (Validated 2026-02-07)
- Total AP: ~$125,319 (66 bills, 35 vendors)
- AP is healthy — 96% current excluding one vendor
- Bill numbering: 30xxx range (separate from order 133xxx range)
- Vendor terms: `Account.VendorPaymentTermsID`

### AP Detail with Vendor Breakdown

Mirrors the AR Detail format but for vendor bills. AP ages from DueDate (vendor payment deadline) -- this is the OPPOSITE of AR which uses SaleDate.

**Key difference: AP uses DueDate, AR uses SaleDate.** DueDate on Type 8 bills is the vendor's payment deadline. DueDate on Type 1 orders is the production/shipping deadline (irrelevant for aging). This is the most important distinction in AR vs AP aging.

#### Full AP Detail by Vendor
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

#### AP Summary by Vendor
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

#### Vendor Payment History
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

## P&L FROM GL

### Revenue by Product Line
```sql
SELECT
    ga.AccountName AS ProductLine,
    SUM(-l.Amount) AS Revenue
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 4000
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
GROUP BY ga.AccountName
ORDER BY Revenue DESC
```

### Full P&L
```sql
SELECT
    CASE ga.GLClassificationType
        WHEN 4000 THEN '1-Product Revenue'
        WHEN 4001 THEN '2-Other Income'
        WHEN 5001 THEN '3-COGS'
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

### P&L Summary
```sql
SELECT
    SUM(CASE WHEN ga.GLClassificationType = 4000 THEN -l.Amount ELSE 0 END) AS ProductRevenue,
    SUM(CASE WHEN ga.GLClassificationType = 4001 THEN -l.Amount ELSE 0 END) AS OtherIncome,
    SUM(CASE WHEN ga.GLClassificationType IN (4000, 4001) THEN -l.Amount ELSE 0 END) AS TotalRevenue,
    SUM(CASE WHEN ga.GLClassificationType = 5001 THEN l.Amount ELSE 0 END) AS COGS,
    SUM(CASE WHEN ga.GLClassificationType IN (4000, 4001) THEN -l.Amount ELSE 0 END)
        - SUM(CASE WHEN ga.GLClassificationType = 5001 THEN l.Amount ELSE 0 END) AS GrossProfit,
    SUM(CASE WHEN ga.GLClassificationType = 5002 THEN l.Amount ELSE 0 END) AS OperatingExpenses,
    SUM(CASE WHEN ga.GLClassificationType IN (4000, 4001) THEN -l.Amount ELSE 0 END)
        - SUM(CASE WHEN ga.GLClassificationType IN (5001, 5002) THEN l.Amount ELSE 0 END) AS NetIncome
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.GLClassificationType IN (4000, 4001, 5001, 5002)
    AND l.EntryDateTime BETWEEN @StartDate AND @EndDate
```

### Balance Sheet Key Accounts
```sql
SELECT
    ga.AccountName,
    ga.GLClassificationType,
    SUM(l.Amount) AS Balance
FROM GL l
INNER JOIN GLAccount ga ON l.GLAccountID = ga.ID
WHERE ga.ID IN (
    90,     -- Cash-Associated Bank-Checking
    10412,  -- Cash-MM
    14,     -- Accounts Receivable
    10414,  -- Inventory
    11,     -- WIP
    22,     -- Accounts Payable
    24,     -- Order Prepayments
    21      -- Orders Due
)
    AND l.EntryDateTime <= @AsOfDate
GROUP BY ga.AccountName, ga.GLClassificationType
```

---

## ALL-TIME REVENUE CONTEXT

Cumulative revenue since inception: **$62.45M**

| Product | All-Time Revenue | % |
|---------|-----------------|---|
| DyeSub Table Covers | $17.54M | 28.1% |
| DyeSub | $9.45M | 15.1% |
| Dyelux Flag | $7.32M | 11.7% |
| SP Table Covers | $4.97M | 8.0% |
| Screenprint (garment) | $4.74M | 7.6% |
| Shipping | $2.73M | 4.4% |
| Banner Stand | $2.23M | 3.6% |
| SP Banners | $2.18M | 3.5% |
| Embroidery | $2.06M | 3.3% |
| Services | $1.65M | 2.6% |
| Feather Flags | $1.59M | 2.6% |
| Solvent | $1.45M | 2.3% |

Top 3 products = 55% of all-time revenue. Dye sublimation printing = ~57% overall.
Gross margin ~82% in recent periods.

---

## NATURAL LANGUAGE INTERPRETATION

| User Says | Action |
|-----------|--------|
| "What's our AR?" / "open invoices" | AR Snapshot query |
| "AR aging" / "past due invoices" | AR Aging Buckets |
| "AR by payment terms" | AR by Payment Terms |
| "AR detail" / "AR by customer" / "who owes us" | AR Detail with Customer Breakdown |
| "AR for [customer]" / "what does [customer] owe" | Single Customer AR Drill-Down |
| "AR summary by customer" | AR Summary by Customer |
| "What do we owe?" / "AP" / "open bills" | AP Snapshot |
| "AP aging" / "what's overdue" | AP Aging |
| "AP detail" / "AP by vendor" / "what do we owe" | AP Detail with Vendor Breakdown |
| "AP for [vendor]" / "bills for [vendor]" | AP Detail filtered by vendor (LIKE on CompanyName) |
| "vendor payment history" / "payments to [vendor]" | Vendor Payment History |
| "P&L" / "profit and loss" / "income statement" | P&L Summary or Full P&L |
| "Revenue by product" (GL-based) | Revenue by Product Line |
| "What's our gross margin?" | P&L Summary → GrossProfit/TotalRevenue |
| "Cash position" / "how much cash" | Balance Sheet Key Accounts (cash nodes) |
| "Expenses breakdown" | Full P&L filtered to GLClass 5002 |
| "COGS" / "cost of goods" | Full P&L filtered to GLClass 5001 |
| "Inventory value" | Balance Sheet → NodeID 10414 |
| "WIP balance" / "orders in production value" | Balance Sheet → NodeID 11 |
| "How did [customer] pay?" / "payment history" | GL register query on Order Prepayments or AR |
| "GL entries for order #NNNNN" | Ledger WHERE TransHeaderID = (order's ID) |
| "undeposited payments" / "pending deposits" | Undeposited payments query (Payment.Undeposited = 1) |
| "deposit history" / "bank deposits" | Deposit workflow query (Ledger.DepositJournalID) |
| "payment method breakdown" | Group payments by TenderType |
| "closeout status" / "period locks" | Closeout query by type |
| "production costs for order" / "total cost including materials" | Ledger query with OffBalanceSheet included |

### Two Revenue Reporting Approaches

**TransHeader-based** (sales skill): Revenue from `SubTotalPrice` on orders. Best for product-level drill-down via TransDetail/TransDetailParam variables. Matches "Sales by Product" report.

**GL-based** (this skill): Revenue from Ledger entries hitting GLAccount 4000/4001 accounts. Best for financial reporting, P&L, trial balance. Matches accountant's view.

Both should reconcile for the same period. Minor timing differences possible due to batch posting.

---

## CLOSEOUT — PERIOD LOCKING

The Closeout table locks GL periods to prevent backdated entries. The most recent Closeout of each type determines the locked period boundary. GL entries before the last closeout date cannot be modified without reopening the period.

### Key Fields
- `CloseoutType` — type of closeout (see reference below)
- `ClassTypeID` — always 8911 for all Closeout records
- `StartDate` / `EndDate` — period being closed
- `ClosedDate` — when the closeout was performed
- `ClosedByID` — employee who performed the closeout
- `IsActive` — whether this closeout is active

### CloseoutType Reference

| CloseoutType | Name | Purpose |
|-------------|------|---------|
| 1 | Daily | Daily business close |
| 2 | Monthly | Monthly accounting close |
| 3 | Yearly | Yearly fiscal close |
| 5 | Export | GL export for external accounting |
| 6 | Credit Card Settlement | CC batch settlement |

### Query Example: Last Closeout Dates

```sql
-- Last closeout dates by type
SELECT CloseoutType,
    CASE CloseoutType
        WHEN 1 THEN 'Daily'
        WHEN 2 THEN 'Monthly'
        WHEN 3 THEN 'Yearly'
        WHEN 5 THEN 'Export'
        WHEN 6 THEN 'CC Settlement'
    END AS Name,
    MAX(EndDate) AS LastClosedThrough
FROM Closeout
WHERE IsActive = 1
GROUP BY CloseoutType
ORDER BY CloseoutType
```

---

## IMPORTANT CAVEATS

1. **GL view vs Ledger table:** Always use `GL` view for financial queries. Only use `Ledger` directly if you need off-balance sheet entries.

2. **DueDate means different things:** On Type 1 orders = production/ship date. On Type 8 bills = vendor payment deadline. Never use Type 1 DueDate for AR aging.

3. **AR includes WIP/Built:** The AR report includes orders not yet sold (StatusID 1/2) for visibility. GL AR balance will be lower than the AR report total because WIP/Built haven't posted to the AR GL account yet.

4. **Payment path is mutually exclusive:** Prepaid orders only touch Order Prepayments (24). Credit orders only touch AR (14). Only ~0.3% of orders appear in both (partial prepay edge cases).

5. **Batch posting:** Sales often process in batches (observed "3:59 PM" timestamps). GL entries may not reflect real-time status changes.

6. **ZoomTex accounts are legacy.** NodeIDs 10522-10525 (ZT Undeposited *) have historical entries but the division is defunct. Exclude from active reporting.

7. **Payment/Journal split table:** Payment.ID = Journal.ID. The Journal table tracks all system events (ActivityType: 1=Created, 7=Order Marked Sale, 8=Closed, 9=Voided). Payment adds financial details (Amount, BankAccountID, PaymentMethodID).

---

*Financial queries validated against FLS Banners' AR report ($80,899), AP report ($125,319), GL account registers, and Trial Balance export (January 2026). All NodeIDs confirmed from GL SQL Export and modified Trial Balance.*
