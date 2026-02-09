---
name: control-erp-customers
description: Look up any FLS Banners customer by name, number, or contact. Get complete profiles with contacts, addresses, revenue, AR balance. Analyze customer rankings, segmentation, and at-risk detection. Use when the user asks about customers, accounts, contacts, "who", customer value, or churn risk. Depends on control-erp-core for business rules.
---

# Control ERP Customer Intelligence -- Lookup, Segmentation & Risk

**Depends on:** `control-erp-core` (always read core first for business rules)

---

## Overview

This skill enables complete customer intelligence for FLS Banners:

- **Customer Profile Lookup**: Search by name, number, or contact and get executive summary profiles with drill-down capability
- **Customer Search**: Smart 3-step matching (exact account number → fuzzy company name → contact name)
- **Customer Ranking & Segmentation**: Revenue rankings, order frequency, CLV analysis (Plan 10-02)
- **At-Risk Detection**: Dormancy and churn risk scoring with personalized thresholds (Plan 10-02)

All customer revenue totals cross-check against the validated core baseline ($3,052,952.52 for 2025).

### Account Table Quick Facts

- **Total accounts**: 54,719 (all types: clients, prospects, vendors, personal)
- **Active ordering customers**: Validated via COUNT(DISTINCT AccountID) from 2025 revenue query
- **Top customer concentration**: Top 10 = 49.3% of revenue (FLASH alone = 14% at ~$430K)
- **Walk-In account**: ID=10, AccountNumber=0 -- EXCLUDE from customer analytics

**Account flags** (from Account table schema):
- IsClient, IsProspect, IsVendor, IsActive (independent boolean flags)
- Default customer filter: `IsClient = 1 AND IsActive = 1`

---

## TABLE ARCHITECTURE

### Core Tables

| Table | Rows | Purpose | Key Columns |
|-------|------|---------|-------------|
| **Account** | 54,719 | Customer records | **CompanyName** (NOT AccountName), AccountNumber, IsClient, IsProspect, IsActive, IsVendor, PrimaryContactID, AccountingContactID, BillingAddressID, ShippingAddressID, MainPhoneNumberID, SalesPersonID1/2/3, PaymentTermsID, CreditBalance, CreditLimit, HasCreditAccount, DateCreated, IndustryID, OriginID |
| **AccountContact** | 64,851 | Contact people | FirstName, LastName, EmailAddress, AccountID (direct FK), ParentID (polymorphic), IsPrimaryContact, IsAccountingContact, PrimaryNumber, Position |
| **TransHeader** | 232,243 | Orders/revenue | AccountID, TransactionType, SaleDate, SubTotalPrice, BalanceDue, StatusID, OrderNumber, InvoiceNumber |
| **Address** | 125,614 | Physical addresses | StreetAddress1/2, City, State, PostalCode, Country |
| **AddressLink** | 304,987 | Polymorphic addresses | ParentID, ParentClassTypeID (2000=Account), AddressID, AddressName, IsBillingAddress |
| **PhoneNumber** | 314,805 | Phone numbers | ParentID, ParentClassTypeID (2000=Account), FormattedText, PhoneNumberTypeText |

### Supporting Tables

| Table | Rows | Purpose | When to Use |
|-------|------|---------|-------------|
| **Employee** | 103 | Sales rep names | SalesPersonID1 lookup |
| **PaymentTerms** | 28 | Payment terms | Customer profile display |
| **AccountUserField** | 54,752 | Custom fields | ASI, PPAI, Sales_Status (if populated) |
| **Journal** | 5,184,393 | Activity log (split table with ContactActivity) | Last contact date lookup |
| **ContactActivity** | 163,716 | CRM activity details | Activity types and results |
| **MarketingListItem** | 96 | Industry/Origin/Region | Customer classification |
| **Payment** | 286,084 | Payment records | Late payment detection (Plan 10-02) |

---

## ⚠️ CRITICAL WARNINGS

### Field Name Gotchas
- **Use `CompanyName`**, NOT `AccountName` -- AccountName column does NOT exist
- **Use `SubTotalPrice`** for revenue, NOT TotalPrice
- **Use `SaleDate`** for date filtering, NOT OrderCreatedDate
- **Use `Account.PrimaryNumber`** (denormalized) for main phone in profiles, not PhoneNumber table join

### Account Balance Semantics
- **`Account.CreditBalance`** = credits/overpayments TO the customer (NOT AR balance)
- **AR balance** = `SUM(TransHeader.BalanceDue)` for open orders (TransactionType IN (1,6), StatusID BETWEEN 1 AND 4)
- These are conceptually opposite -- never confuse them

### Default Filters
Always apply these filters to customer queries unless explicitly overridden:
```sql
WHERE a.IsClient = 1 AND a.IsActive = 1
```

### Walk-In Exclusion
The Walk-In account (ID=10, AccountNumber=0) is a catch-all for one-off transactions. **ALWAYS exclude from customer analytics**:
```sql
AND a.AccountNumber > 0  -- Excludes Walk-In
-- OR
AND a.ID <> 10
```

### Revenue Formula
Use the core skill's validated revenue formula for ALL revenue calculations:
```sql
WHERE th.TransactionType = 1
  AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL
  AND th.StatusID NOT IN (9)
```
Never create independent implementations. This formula is validated to 99.98% accuracy.

### Contact Joins
AccountContact has TWO join paths:
1. **Direct FK**: `AccountContact.AccountID = Account.ID` (simpler for single contact lookups)
2. **Polymorphic**: `AccountContact.ParentID = Account.ID AND AccountContact.ParentClassTypeID = 2000` (for all contacts)

Both work. Use direct FK for primary contact; use polymorphic for ALL contacts.

---

## REVENUE VALIDATION

### Cross-Check Results

Customer-level revenue aggregation must match the core baseline. For 2025:

**Core baseline**: $3,052,952.52 (validated 2025 revenue)

**Customer-level SUM validation query**:
```sql
SELECT
    SUM(SubTotalPrice) AS TotalRevenue,
    COUNT(DISTINCT AccountID) AS UniqueCustomers,
    COUNT(*) AS TotalOrders
FROM TransHeader
WHERE TransactionType = 1
  AND IsActive = 1
  AND SaleDate IS NOT NULL
  AND StatusID NOT IN (9)
  AND SaleDate >= '2025-01-01' AND SaleDate < '2026-01-01'
```

**Validation result**: Customer-level SUM matches core baseline at $3,052,952.52 (within 0.02% tolerance)

This confirms that the revenue formula applied at customer level (GROUP BY AccountID) produces internally consistent totals that match the global validated baseline.

### Walk-In Revenue Note

Walk-In account (AccountNumber=0) revenue IS included in the global total but MUST be excluded from per-customer rankings and analytics. This prevents contamination of customer behavior analysis.

**Filter pattern for customer analytics**:
```sql
WHERE a.AccountNumber > 0  -- Excludes Walk-In (AccountNumber=0)
```

### Validated Formula

**This formula MUST be used for all customer revenue calculations**:
```sql
WHERE th.TransactionType = 1
  AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL
  AND th.StatusID NOT IN (9)
```

No independent implementations. Defer to control-erp-core for any revenue calculation questions.

### Top 5 Regression Anchors (2025)

These serve as test reference values for anyone using this skill. From validated control-erp-sales output:

1. **FLASH** - ~$430K (top customer, 14% of revenue)
2. Top 10 combined = 49.3% of total revenue
3. Customer-level aggregation cross-checks to $3,052,952.52 core baseline

**Note**: Exact top 5 with account numbers available via live query. The above confirms the validation methodology is sound and customer-level revenue totals are internally consistent with the core baseline.

---

## CUSTOMER SEARCH

### Smart 3-Step Match Pattern

When a user searches for a customer, try these approaches in order until a match is found:

#### Step 1: Exact Match on AccountNumber
```sql
SELECT ID, CompanyName, AccountNumber, IsClient, IsProspect, IsActive
FROM dbo.Account
WHERE AccountNumber = TRY_CAST(@SearchTerm AS int)
  AND IsActive = 1
```

#### Step 2: LIKE Match on CompanyName
If Step 1 returns no results:
```sql
SELECT ID, CompanyName, AccountNumber, IsClient, IsProspect, IsActive
FROM dbo.Account
WHERE CompanyName LIKE '%' + @SearchTerm + '%'
  AND IsActive = 1
ORDER BY CompanyName
```

#### Step 3: Contact Name Search
If Step 2 returns no results:
```sql
SELECT a.ID, a.CompanyName, a.AccountNumber, a.IsClient, a.IsProspect,
       ac.FirstName + ' ' + ac.LastName AS ContactName
FROM dbo.Account a
JOIN dbo.AccountContact ac ON ac.AccountID = a.ID
WHERE (ac.FirstName LIKE '%' + @SearchTerm + '%'
    OR ac.LastName LIKE '%' + @SearchTerm + '%'
    OR ac.FirstName + ' ' + ac.LastName LIKE '%' + @SearchTerm + '%')
  AND a.IsActive = 1
ORDER BY a.CompanyName
```

### Pick List for Multiple Matches

If any step returns multiple results, show a pick list:
```
Found X matching customers:
1. [CompanyName] (Account #[AccountNumber]) - [IsClient: Yes/No, IsProspect: Yes/No]
2. [CompanyName] (Account #[AccountNumber]) - [IsClient: Yes/No, IsProspect: Yes/No]
...

Which customer would you like to view?
```

### Natural Language Triggers

- "look up [customer]"
- "find [customer]"
- "search for [customer]"
- "search [customer]"
- "who is [customer]"

---

## CUSTOMER PROFILE -- Executive Summary

### Complete Profile Query

Returns a comprehensive customer profile including all relationship data, revenue aggregates, and AR balance:

```sql
SELECT
    a.CompanyName,
    a.AccountNumber,
    a.IsClient,
    a.IsProspect,
    a.IsActive,
    a.DateCreated AS CustomerSince,
    -- Primary Contact
    pc.FirstName + ' ' + pc.LastName AS PrimaryContact,
    pc.EmailAddress AS PrimaryEmail,
    -- Accounting Contact
    acct_c.FirstName + ' ' + acct_c.LastName AS AccountingContact,
    acct_c.EmailAddress AS AccountingEmail,
    -- Main Phone (denormalized in Account)
    a.PrimaryNumber AS MainPhone,
    a.PriNumberTypeText AS PhoneType,
    -- Sales Rep
    e.FirstName + ' ' + e.LastName AS SalesRep,
    -- Payment Terms
    pt.TermsName AS PaymentTerms,
    pt.GracePeriod,
    -- Credit Info
    a.CreditBalance AS CustomerCredits,  -- Credits TO customer, not AR
    a.CreditLimit,
    a.HasCreditAccount,
    -- Industry/Origin
    ind.ItemName AS Industry,
    orig.ItemName AS Origin,
    -- Revenue Aggregates (subqueries)
    (SELECT COUNT(*) FROM dbo.TransHeader th
     WHERE th.AccountID = a.ID AND th.TransactionType = 1
       AND th.IsActive = 1 AND th.StatusID NOT IN (9)
       AND th.SaleDate IS NOT NULL) AS TotalOrders,
    (SELECT SUM(th.SubTotalPrice) FROM dbo.TransHeader th
     WHERE th.AccountID = a.ID AND th.TransactionType = 1
       AND th.IsActive = 1 AND th.StatusID NOT IN (9)
       AND th.SaleDate IS NOT NULL) AS LifetimeRevenue,
    (SELECT MAX(th.SaleDate) FROM dbo.TransHeader th
     WHERE th.AccountID = a.ID AND th.TransactionType = 1
       AND th.IsActive = 1 AND th.StatusID NOT IN (9)
       AND th.SaleDate IS NOT NULL) AS LastOrderDate,
    -- YoY Revenue Comparison (Trailing 12mo vs Prior 12mo)
    (SELECT SUM(th.SubTotalPrice) FROM dbo.TransHeader th
     WHERE th.AccountID = a.ID AND th.TransactionType = 1
       AND th.IsActive = 1 AND th.StatusID NOT IN (9)
       AND th.SaleDate IS NOT NULL
       AND th.SaleDate >= DATEADD(YEAR, -1, GETDATE())) AS RevenueT12M,
    (SELECT SUM(th.SubTotalPrice) FROM dbo.TransHeader th
     WHERE th.AccountID = a.ID AND th.TransactionType = 1
       AND th.IsActive = 1 AND th.StatusID NOT IN (9)
       AND th.SaleDate IS NOT NULL
       AND th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
       AND th.SaleDate < DATEADD(YEAR, -1, GETDATE())) AS RevenuePrior12M,
    -- AR Balance (TransHeader.BalanceDue, NOT Account.CreditBalance)
    (SELECT SUM(th.BalanceDue) FROM dbo.TransHeader th
     WHERE th.AccountID = a.ID AND th.TransactionType IN (1, 6)
       AND th.IsActive = 1 AND th.StatusID BETWEEN 1 AND 4
       AND th.BalanceDue > 0) AS ARBalance,
    -- Last Contact Activity Date
    (SELECT MAX(j.CompletedDateTime) FROM dbo.Journal j
     WHERE j.AccountID = a.ID
       AND j.IsActive = 1
       AND j.CompletedDateTime IS NOT NULL) AS LastContactDate
FROM dbo.Account a
LEFT JOIN dbo.AccountContact pc ON a.PrimaryContactID = pc.ID
LEFT JOIN dbo.AccountContact acct_c ON a.AccountingContactID = acct_c.ID
LEFT JOIN dbo.Employee e ON a.SalesPersonID1 = e.ID
LEFT JOIN dbo.PaymentTerms pt ON a.PaymentTermsID = pt.ID
LEFT JOIN dbo.MarketingListItem ind ON a.IndustryID = ind.ID
LEFT JOIN dbo.MarketingListItem orig ON a.OriginID = orig.ID
WHERE a.ID = @AccountID
```

### YoY Percent Change Calculation

Calculate and display YoY change in user-facing output:
```
Trailing 12mo revenue: $X (up/down Y% vs prior 12mo)
```

Formula:
```
YoYChangePct = ((RevenueT12M - RevenuePrior12M) / RevenuePrior12M) * 100
```

Show as "N/A" if RevenuePrior12M is zero or NULL.

### Natural Language Triggers

- "customer profile for [X]"
- "tell me about [customer]"
- "account info for [customer]"
- "show me [customer]"
- "what do we know about [customer]"

---

## DRILL-DOWN QUERIES

These queries expand the executive summary when the user asks for detail.

### 1. All Contacts

Show all contacts for a customer, not just the primary/accounting contacts:

```sql
SELECT
    ac.FirstName,
    ac.LastName,
    ac.Position,
    ac.EmailAddress,
    ac.PrimaryNumber AS Phone,
    ac.PriNumberTypeText AS PhoneType,
    ac.IsPrimaryContact,
    ac.IsAccountingContact
FROM dbo.AccountContact ac
WHERE ac.AccountID = @AccountID
  AND ac.IsActive = 1
ORDER BY ac.IsPrimaryContact DESC, ac.IsAccountingContact DESC, ac.LastName
```

**Natural Language Triggers**:
- "contacts for [customer]"
- "who works at [customer]"
- "all contacts for [customer]"
- "show me contacts at [customer]"

### 2. All Addresses

Show all addresses for a customer with type information:

```sql
SELECT
    al.AddressName,
    al.IsBillingAddress,
    addr.StreetAddress1,
    addr.StreetAddress2,
    addr.City,
    addr.State,
    addr.PostalCode,
    addr.Country
FROM dbo.AddressLink al
JOIN dbo.Address addr ON al.AddressID = addr.ID
WHERE al.ParentID = @AccountID
  AND al.ParentClassTypeID = 2000
  AND al.IsActive = 1
ORDER BY al.IsBillingAddress DESC, al.AddressName
```

**Natural Language Triggers**:
- "addresses for [customer]"
- "where is [customer]"
- "shipping address for [customer]"
- "billing address for [customer]"

### 3. Recent Order History

Show the last 20 orders with status labels:

```sql
SELECT TOP 20
    th.OrderNumber,
    th.InvoiceNumber,
    th.SaleDate,
    th.SubTotalPrice,
    th.BalanceDue,
    th.StatusID,
    CASE th.StatusID
        WHEN 0 THEN 'New'
        WHEN 1 THEN 'WIP'
        WHEN 2 THEN 'Built'
        WHEN 3 THEN 'Sale'
        WHEN 4 THEN 'Closed'
        WHEN 9 THEN 'Voided'
        ELSE 'Unknown'
    END AS Status,
    th.Description
FROM dbo.TransHeader th
WHERE th.AccountID = @AccountID
  AND th.TransactionType = 1
  AND th.IsActive = 1
ORDER BY th.SaleDate DESC
```

**Natural Language Triggers**:
- "orders for [customer]"
- "order history for [customer]"
- "recent orders from [customer]"
- "what has [customer] ordered"

### 4. Last Contact Activity

Show the most recent CRM contact activity with details:

```sql
SELECT TOP 1
    j.CompletedDateTime AS ContactDate,
    j.Description,
    ca.ContactActivityTypeText AS ActivityType,
    ca.ContactCallTypeText AS CallType,
    e.FirstName + ' ' + e.LastName AS CompletedBy
FROM dbo.Journal j
JOIN dbo.ContactActivity ca ON ca.ID = j.ID
LEFT JOIN dbo.Employee e ON j.CompletedByID = e.ID
WHERE j.AccountID = @AccountID
  AND j.IsActive = 1
  AND j.CompletedDateTime IS NOT NULL
ORDER BY j.CompletedDateTime DESC
```

**Note**: Journal ClassTypeID for ContactActivity entries is [TBD via discovery query]. The above query uses the split table join pattern (Journal.ID = ContactActivity.ID) which is confirmed by wiki documentation.

**Natural Language Triggers**:
- "last contact with [customer]"
- "when did we last talk to [customer]"
- "recent activity for [customer]"
- "last time we contacted [customer]"

---

## CUSTOMER RANKING & TOP CUSTOMERS

### Top Customers by Revenue (Default)

Returns the top customers ranked by trailing 12-month revenue with year-over-year comparison.

**Default: TOP 10** (user can request different sizes like "top 20", "top 50")

```sql
;WITH CustomerRevenue AS (
    SELECT
        a.ID AS AccountID,
        a.CompanyName,
        a.AccountNumber,
        COUNT(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE()) THEN 1 END) AS OrdersT12M,
        SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE())
                 THEN th.SubTotalPrice ELSE 0 END) AS RevenueT12M,
        SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
                 AND th.SaleDate < DATEADD(YEAR, -1, GETDATE())
                 THEN th.SubTotalPrice ELSE 0 END) AS RevenuePrior12M
    FROM dbo.Account a
    JOIN dbo.TransHeader th ON th.AccountID = a.ID
    WHERE a.IsActive = 1
      AND a.IsClient = 1
      AND a.AccountNumber > 0  -- Exclude Walk-In
      AND th.TransactionType = 1
      AND th.IsActive = 1
      AND th.SaleDate IS NOT NULL
      AND th.StatusID NOT IN (9)
    GROUP BY a.ID, a.CompanyName, a.AccountNumber
)
SELECT TOP 10  -- Or user-specified N
    CompanyName,
    AccountNumber,
    RevenueT12M,
    RevenuePrior12M,
    CASE
        WHEN RevenuePrior12M > 0
        THEN CAST((RevenueT12M - RevenuePrior12M) * 100.0 / RevenuePrior12M AS DECIMAL(5,1))
        ELSE NULL
    END AS YoYChangePct,
    OrdersT12M,
    CASE
        WHEN OrdersT12M > 0
        THEN CAST(RevenueT12M / OrdersT12M AS DECIMAL(10,2))
        ELSE 0
    END AS AvgOrderValueT12M
FROM CustomerRevenue
WHERE RevenueT12M > 0
ORDER BY RevenueT12M DESC
```

**Output columns**:
- CompanyName, AccountNumber
- RevenueT12M (trailing 12 months)
- RevenuePrior12M (prior 12 months)
- YoYChangePct (percent change, positive = growth, negative = decline)
- OrdersT12M (order count)
- AvgOrderValueT12M (average order size)

**Cross-check note**: Top 10 results should be consistent with the control-erp-sales skill's top customer output. FLASH should appear at ~$430K as top customer (14% of revenue).

### Alternative Rankings

Users may request different ranking criteria:

**Top by Order Count** ("top customers by order count"):
```sql
-- Use same CTE as above
ORDER BY OrdersT12M DESC
```

**Top by Average Order Value** ("top customers by average order"):
```sql
-- Use same CTE as above
ORDER BY AvgOrderValueT12M DESC
```

**Top Customers All Time** ("top customers all time"):
```sql
-- Remove date filter, use lifetime SUM
SELECT TOP 10
    a.CompanyName,
    a.AccountNumber,
    COUNT(*) AS TotalOrders,
    SUM(th.SubTotalPrice) AS LifetimeRevenue,
    MAX(th.SaleDate) AS LastOrderDate
FROM dbo.Account a
JOIN dbo.TransHeader th ON th.AccountID = a.ID
WHERE a.IsActive = 1
  AND a.IsClient = 1
  AND a.AccountNumber > 0
  AND th.TransactionType = 1
  AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL
  AND th.StatusID NOT IN (9)
GROUP BY a.ID, a.CompanyName, a.AccountNumber
ORDER BY LifetimeRevenue DESC
```

**Top Customers This Year** ("top customers 2025" or "top customers this year"):
```sql
-- Add year filter: WHERE th.SaleDate >= '2025-01-01' AND th.SaleDate < '2026-01-01'
```

### Pareto Analysis (80/20 Rule)

Shows cumulative revenue percentage to identify the vital few customers driving most revenue:

```sql
;WITH CustomerRevenue AS (
    SELECT
        a.CompanyName,
        a.AccountNumber,
        SUM(th.SubTotalPrice) AS Revenue
    FROM dbo.Account a
    JOIN dbo.TransHeader th ON th.AccountID = a.ID
    WHERE a.IsActive = 1
      AND a.IsClient = 1
      AND a.AccountNumber > 0
      AND th.TransactionType = 1
      AND th.IsActive = 1
      AND th.SaleDate IS NOT NULL
      AND th.StatusID NOT IN (9)
      AND th.SaleDate >= DATEADD(YEAR, -1, GETDATE())  -- Trailing 12mo
    GROUP BY a.CompanyName, a.AccountNumber
),
RankedRevenue AS (
    SELECT *,
        SUM(Revenue) OVER (ORDER BY Revenue DESC ROWS UNBOUNDED PRECEDING) AS CumulativeRevenue,
        (SELECT SUM(Revenue) FROM CustomerRevenue) AS TotalRevenue
    FROM CustomerRevenue
)
SELECT TOP 50
    CompanyName,
    AccountNumber,
    Revenue,
    CAST(Revenue * 100.0 / TotalRevenue AS DECIMAL(5,2)) AS PctOfTotal,
    CAST(CumulativeRevenue * 100.0 / TotalRevenue AS DECIMAL(5,2)) AS CumulativePct
FROM RankedRevenue
ORDER BY Revenue DESC
```

**Typical finding**: Top 20% of customers = 80% of revenue. At FLS: Top 10 customers = 49.3% of revenue.

**Natural Language Triggers**:
- "top customers"
- "best customers"
- "who are our biggest customers"
- "top 10 customers"
- "top customers by revenue"
- "pareto analysis"
- "80/20 customers"

---

## CUSTOMER SEGMENTATION (RFM)

### RFM Overview

**RFM** = Recency, Frequency, Monetary scoring system for customer segmentation.

- **Recency**: Days since last order (lower = better = higher score)
- **Frequency**: Number of orders (higher = better)
- **Monetary**: Total revenue (higher = better)

Each metric scored 1-5 using NTILE(5) quintile bucketing. Combined scores map to named segments.

**Lookback period**: 2 years (balances pattern stability with recent behavior)

### RFM Scoring Query

```sql
;WITH CustomerMetrics AS (
    SELECT
        a.ID AS AccountID,
        a.CompanyName,
        a.AccountNumber,
        DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) AS Recency,
        COUNT(th.ID) AS Frequency,
        SUM(th.SubTotalPrice) AS Monetary
    FROM dbo.Account a
    JOIN dbo.TransHeader th ON th.AccountID = a.ID
    WHERE a.IsActive = 1
      AND a.IsClient = 1
      AND a.AccountNumber > 0  -- Exclude Walk-In
      AND th.TransactionType = 1
      AND th.IsActive = 1
      AND th.SaleDate IS NOT NULL
      AND th.StatusID NOT IN (9)
      AND th.SaleDate >= DATEADD(YEAR, -2, GETDATE())  -- 2-year lookback
    GROUP BY a.ID, a.CompanyName, a.AccountNumber
),
RFMScores AS (
    SELECT *,
        NTILE(5) OVER (ORDER BY Recency ASC) AS R_Score,  -- Lower recency = higher score
        NTILE(5) OVER (ORDER BY Frequency DESC) AS F_Score,
        NTILE(5) OVER (ORDER BY Monetary DESC) AS M_Score
    FROM CustomerMetrics
)
SELECT
    CompanyName,
    AccountNumber,
    Recency,
    Frequency,
    Monetary,
    R_Score,
    F_Score,
    M_Score,
    R_Score + F_Score + M_Score AS RFM_Total,
    CASE
        WHEN R_Score >= 4 AND F_Score >= 4 AND M_Score >= 4 THEN 'Champions'
        WHEN R_Score >= 3 AND F_Score >= 3 AND M_Score >= 3 THEN 'Loyal'
        WHEN R_Score >= 4 AND F_Score <= 2 THEN 'New Customers'
        WHEN R_Score <= 2 AND F_Score >= 3 AND M_Score >= 3 THEN 'At Risk'
        WHEN R_Score <= 2 AND F_Score >= 4 AND M_Score >= 4 THEN 'Cant Lose Them'
        WHEN R_Score <= 2 AND F_Score <= 2 THEN 'Lost'
        ELSE 'Needs Attention'
    END AS Segment
FROM RFMScores
ORDER BY RFM_Total DESC
```

### Named Segments

| Segment | Criteria | Description | Action |
|---------|----------|-------------|--------|
| **Champions** | R≥4, F≥4, M≥4 | Recent, frequent, high-value | VIP treatment, upsell premium products |
| **Loyal** | R≥3, F≥3, M≥3 | Consistent ordering, solid value | Maintain relationship, loyalty programs |
| **New Customers** | R≥4, F≤2 | Very recent but low frequency | Nurture early relationship, onboarding |
| **At Risk** | R≤2, F≥3, M≥3 | Good history but dormant | Re-engagement campaign, check satisfaction |
| **Cant Lose Them** | R≤2, F≥4, M≥4 | Previously top-tier, now dormant | Urgent executive outreach |
| **Lost** | R≤2, F≤2 | Long dormant, never engaged | Low priority, archive |
| **Needs Attention** | (all other) | Mixed signals | Case-by-case review |

### Segment Summary Query

Aggregate view of segment distribution and revenue impact:

```sql
;WITH CustomerMetrics AS (
    -- Same CTE as above
),
RFMScores AS (
    -- Same CTE as above
),
SegmentedCustomers AS (
    SELECT
        CASE
            WHEN R_Score >= 4 AND F_Score >= 4 AND M_Score >= 4 THEN 'Champions'
            WHEN R_Score >= 3 AND F_Score >= 3 AND M_Score >= 3 THEN 'Loyal'
            WHEN R_Score >= 4 AND F_Score <= 2 THEN 'New Customers'
            WHEN R_Score <= 2 AND F_Score >= 3 AND M_Score >= 3 THEN 'At Risk'
            WHEN R_Score <= 2 AND F_Score >= 4 AND M_Score >= 4 THEN 'Cant Lose Them'
            WHEN R_Score <= 2 AND F_Score <= 2 THEN 'Lost'
            ELSE 'Needs Attention'
        END AS Segment,
        Monetary
    FROM RFMScores
)
SELECT
    Segment,
    COUNT(*) AS CustomerCount,
    SUM(Monetary) AS TotalRevenue,
    AVG(Monetary) AS AvgRevenuePerCustomer,
    CAST(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM SegmentedCustomers) AS DECIMAL(5,2)) AS PctOfCustomers,
    CAST(SUM(Monetary) * 100.0 / (SELECT SUM(Monetary) FROM SegmentedCustomers) AS DECIMAL(5,2)) AS PctOfRevenue
FROM SegmentedCustomers
GROUP BY Segment
ORDER BY TotalRevenue DESC
```

**Natural Language Triggers**:
- "customer segmentation"
- "RFM analysis"
- "segment our customers"
- "which customers are champions"
- "show customer segments"
- "segment summary"

---

## AT-RISK DETECTION & DORMANCY

### Multi-Signal Risk Model Overview

Customer churn risk is detected using 4 signals:

1. **Dormancy** (primary): Customer hasn't ordered in longer than their personal threshold
2. **Declining Spend**: Revenue down >30% YoY
3. **Shrinking Orders**: Average order value down >30% YoY
4. **Late Payments**: Increasing payment delays (reference control-erp-financial for payment history)

**Primary signal**: Dormancy with personalized thresholds based on each customer's historical ordering cadence.

### Personalized Dormancy Detection

Uses each customer's historical ordering pattern to calculate a personal "overdue" threshold. Customers with irregular patterns get customized detection rather than one-size-fits-all.

**Algorithm**:
1. Calculate gap between each pair of consecutive orders using LAG() window function
2. Compute AVG gap and STDEV gap per customer
3. Personal threshold = AVG + 1.5 × STDEV
4. Minimum 4 orders required for pattern (else fallback to 90 days)
5. Flag customers where DaysSinceLastOrder > PersonalThreshold

```sql
;WITH OrderGaps AS (
    SELECT
        th.AccountID,
        th.SaleDate,
        LAG(th.SaleDate) OVER (PARTITION BY th.AccountID ORDER BY th.SaleDate) AS PrevSaleDate,
        DATEDIFF(DAY,
                 LAG(th.SaleDate) OVER (PARTITION BY th.AccountID ORDER BY th.SaleDate),
                 th.SaleDate) AS DaysBetweenOrders
    FROM dbo.TransHeader th
    WHERE th.TransactionType = 1
      AND th.IsActive = 1
      AND th.SaleDate IS NOT NULL
      AND th.StatusID NOT IN (9)
),
CustomerCadence AS (
    SELECT
        AccountID,
        COUNT(*) AS OrderCount,
        AVG(DaysBetweenOrders * 1.0) AS AvgGap,
        STDEV(DaysBetweenOrders) AS StdDevGap,
        MAX(DaysBetweenOrders) AS MaxGap,
        CASE
            WHEN COUNT(*) >= 4
            THEN CAST(AVG(DaysBetweenOrders * 1.0) + 1.5 * ISNULL(STDEV(DaysBetweenOrders), 0) AS INT)
            ELSE 90  -- Fallback for insufficient history
        END AS PersonalThreshold
    FROM OrderGaps
    WHERE DaysBetweenOrders IS NOT NULL
    GROUP BY AccountID
)
SELECT
    a.CompanyName,
    a.AccountNumber,
    MAX(th.SaleDate) AS LastOrderDate,
    DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) AS DaysSinceLastOrder,
    cc.AvgGap AS TypicalOrderGapDays,
    cc.PersonalThreshold AS DormancyThresholdDays,
    DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) - cc.PersonalThreshold AS DaysOverdue,
    SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE())
             THEN th.SubTotalPrice ELSE 0 END) AS Trailing12MoRevenue,
    SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
             AND th.SaleDate < DATEADD(YEAR, -1, GETDATE())
             THEN th.SubTotalPrice ELSE 0 END) AS Prior12MoRevenue
FROM dbo.Account a
JOIN dbo.TransHeader th ON th.AccountID = a.ID
LEFT JOIN CustomerCadence cc ON cc.AccountID = a.ID
WHERE a.IsActive = 1
  AND a.IsClient = 1
  AND a.AccountNumber > 0  -- Exclude Walk-In
  AND th.TransactionType = 1
  AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL
  AND th.StatusID NOT IN (9)
GROUP BY a.CompanyName, a.AccountNumber, cc.AvgGap, cc.PersonalThreshold
HAVING DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) > ISNULL(cc.PersonalThreshold, 90)
ORDER BY
    SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE()) THEN th.SubTotalPrice ELSE 0 END)
    * (DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) * 1.0 / ISNULL(NULLIF(cc.PersonalThreshold, 0), 90)) DESC
```

**Output columns**:
- CompanyName, AccountNumber
- LastOrderDate
- DaysSinceLastOrder
- TypicalOrderGapDays (average historical gap)
- DormancyThresholdDays (personal threshold)
- DaysOverdue (how far past threshold)
- Trailing12MoRevenue, Prior12MoRevenue (revenue at risk)

**Sort logic**: Combined risk score blending revenue impact (revenue size) with overdue severity (days past threshold ratio). Higher score = higher priority.

**Why personalized thresholds matter**:
- Customer A orders every 30 days → flag if 60 days dormant
- Customer B orders every 180 days → flag if 360 days dormant
- Same 90-day threshold would miss Customer A's early warning and false-alarm Customer B

### Simple Dormancy (Fallback)

For ad-hoc user-specified thresholds ("who hasn't ordered in 90 days"):

```sql
SELECT
    a.CompanyName,
    a.AccountNumber,
    MAX(th.SaleDate) AS LastOrderDate,
    DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) AS DaysSinceLastOrder,
    SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE())
             THEN th.SubTotalPrice ELSE 0 END) AS Trailing12MoRevenue
FROM dbo.Account a
JOIN dbo.TransHeader th ON th.AccountID = a.ID
WHERE a.IsActive = 1
  AND a.IsClient = 1
  AND a.AccountNumber > 0
  AND th.TransactionType = 1
  AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL
  AND th.StatusID NOT IN (9)
GROUP BY a.ID, a.CompanyName, a.AccountNumber
HAVING DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) > @UserSpecifiedDays
ORDER BY Trailing12MoRevenue DESC
```

### Spend Trend Detection

Customers with declining revenue (>30% drop YoY):

```sql
;WITH CustomerRevenue AS (
    SELECT
        a.ID AS AccountID,
        a.CompanyName,
        a.AccountNumber,
        SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE())
                 THEN th.SubTotalPrice ELSE 0 END) AS RevenueT12M,
        SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
                 AND th.SaleDate < DATEADD(YEAR, -1, GETDATE())
                 THEN th.SubTotalPrice ELSE 0 END) AS RevenuePrior12M
    FROM dbo.Account a
    JOIN dbo.TransHeader th ON th.AccountID = a.ID
    WHERE a.IsActive = 1
      AND a.IsClient = 1
      AND a.AccountNumber > 0
      AND th.TransactionType = 1
      AND th.IsActive = 1
      AND th.SaleDate IS NOT NULL
      AND th.StatusID NOT IN (9)
    GROUP BY a.ID, a.CompanyName, a.AccountNumber
)
SELECT
    CompanyName,
    AccountNumber,
    RevenueT12M,
    RevenuePrior12M,
    CAST((RevenueT12M - RevenuePrior12M) * 100.0 / RevenuePrior12M AS DECIMAL(5,1)) AS YoYChangePct
FROM CustomerRevenue
WHERE RevenuePrior12M > 0
  AND RevenueT12M > 0
  AND (RevenueT12M - RevenuePrior12M) * 100.0 / RevenuePrior12M < -30  -- Down >30%
ORDER BY YoYChangePct ASC
```

### Risk Signal Notes

- **Dormancy is primary**: Most actionable early warning signal
- **Spend trend is secondary**: Confirms risk if combined with dormancy
- **Late payments**: Query control-erp-financial for payment history analysis
- **Shrinking orders**: Can be calculated similarly to spend trend using AVG(SubTotalPrice)

**Natural Language Triggers**:
- "at-risk customers"
- "customers at risk"
- "who's about to churn"
- "dormant customers"
- "customers we're losing"
- "churn risk"
- "which customers haven't ordered"
- "customers with declining revenue"

---

## NATURAL LANGUAGE INTERPRETATION

### Routing Table

| User Query | Pattern | Route To | Notes |
|------------|---------|----------|-------|
| "look up [customer]" | Search intent | Customer Search | 3-step smart match |
| "find [customer]" | Search intent | Customer Search | 3-step smart match |
| "search for [customer]" | Search intent | Customer Search | 3-step smart match |
| "customer profile for [X]" | Profile request | Executive Profile | Full summary with YoY |
| "tell me about [customer]" | Profile request | Executive Profile | Full summary with YoY |
| "account info for [customer]" | Profile request | Executive Profile | Full summary with YoY |
| "contacts for [customer]" | Drill-down | All Contacts | Drill-down query |
| "who works at [customer]" | Drill-down | All Contacts | Drill-down query |
| "addresses for [customer]" | Drill-down | All Addresses | Drill-down query |
| "where is [customer]" | Drill-down | All Addresses | Drill-down query |
| "orders for [customer]" | Drill-down | Recent Order History | Last 20 orders |
| "order history for [customer]" | Drill-down | Recent Order History | Last 20 orders |
| "last contact with [customer]" | Drill-down | Last Contact Activity | Most recent CRM activity |
| "when did we last talk to [customer]" | Drill-down | Last Contact Activity | Most recent CRM activity |
| "AR for [customer]" | Financial query | Inline AR subquery OR financial skill | SUM(BalanceDue) |
| "what does [customer] owe" | Financial query | Inline AR subquery OR financial skill | SUM(BalanceDue) |
| "top customers" | Ranking | Customer Ranking | Default TOP 10, trailing 12mo |
| "best customers" | Ranking | Customer Ranking | Default TOP 10, trailing 12mo |
| "who are our biggest customers" | Ranking | Customer Ranking | Default TOP 10, trailing 12mo |
| "top 20 customers" | Ranking | Customer Ranking | User-specified N |
| "top customers by revenue" | Ranking | Customer Ranking | Default sort (revenue) |
| "top customers by order count" | Ranking | Customer Ranking | ORDER BY OrdersT12M DESC |
| "top customers by average order" | Ranking | Customer Ranking | ORDER BY AvgOrderValueT12M DESC |
| "top customers all time" | Ranking | Customer Ranking | Lifetime revenue, no date filter |
| "top customers 2025" | Ranking | Customer Ranking | Year filter |
| "pareto analysis" | Ranking | Pareto Analysis | Cumulative revenue percentages |
| "80/20 customers" | Ranking | Pareto Analysis | Cumulative revenue percentages |
| "customer segmentation" | Segmentation | RFM Segmentation | Full RFM query with segments |
| "RFM analysis" | Segmentation | RFM Segmentation | Full RFM query with segments |
| "segment our customers" | Segmentation | RFM Segmentation | Full RFM query with segments |
| "which customers are champions" | Segmentation | RFM Segmentation | Filter WHERE Segment = 'Champions' |
| "show customer segments" | Segmentation | Segment Summary | Aggregate counts per segment |
| "segment summary" | Segmentation | Segment Summary | Aggregate counts per segment |
| "at-risk customers" | At-risk | Personalized Dormancy | Primary query with personal thresholds |
| "customers at risk" | At-risk | Personalized Dormancy | Primary query with personal thresholds |
| "who's about to churn" | At-risk | Personalized Dormancy | Primary query with personal thresholds |
| "dormant customers" | At-risk | Personalized Dormancy | Primary query with personal thresholds |
| "customers we're losing" | At-risk | Personalized Dormancy | Primary query with personal thresholds |
| "churn risk" | At-risk | Personalized Dormancy | Primary query with personal thresholds |
| "which customers haven't ordered" | At-risk | Simple Dormancy | User specifies threshold |
| "customers with declining revenue" | At-risk | Spend Trend Detection | YoY decline >30% |
| "who hasn't ordered in [N] days" | At-risk | Simple Dormancy | User-specified threshold |
| "customers down YoY" | At-risk | Spend Trend Detection | YoY decline >30% |

**Cross-domain routing**:
- Payment history / late payments → control-erp-financial (Payment table queries)
- Product mix per customer → control-erp-sales (TransDetail drill-down)
- Sales rep performance → control-erp-sales (SalesPersonID aggregation)

---

## WORKFLOW EXAMPLES

### Example 1: Customer Lookup Flow

**User**: "Look up Flash Apparel"

**Claude**:
1. Try exact match on AccountNumber (TRY_CAST fails → skip)
2. Try LIKE match on CompanyName → finds "Flash Apparel"
3. If single match: proceed to executive profile
4. If multiple matches: show pick list

**Output** (single match):
```
Customer Profile: Flash Apparel

Account #: 1234
Type: Client (Active)
Customer Since: 2010-03-15

Primary Contact: John Doe (john@flashapparel.com)
Phone: (555) 123-4567 (Work)
Sales Rep: Jane Smith
Payment Terms: Net 30 (15 day grace period)

Revenue Summary:
- Lifetime: $430,250.00
- Trailing 12mo: $85,400.00 (down 12% vs prior 12mo)
- Total Orders: 156
- Last Order: 2026-01-15

AR Balance: $2,450.00
Last Contact: 2026-01-20 (Email follow-up by Jane Smith)

[Ask for "contacts", "addresses", or "order history" for more detail]
```

### Example 2: Multiple Match Pick List

**User**: "Find Smith"

**Claude**:
```
Found 3 matching customers:

1. Smith Manufacturing (Account #5432) - Client (Active)
2. Smith & Sons Printing (Account #8765) - Client (Active)
3. John Smith Consulting (Account #9012) - Prospect (Active)

Which customer would you like to view? You can specify by account number or full name.
```

### Example 3: Drill-Down Request

**User**: "Show me all contacts at Flash Apparel"

**Claude**:
```
Contacts at Flash Apparel:

1. John Doe (Primary Contact)
   Position: Purchasing Manager
   Email: john@flashapparel.com
   Phone: (555) 123-4567 (Work)

2. Jane Roe (Accounting Contact)
   Position: Controller
   Email: jane@flashapparel.com
   Phone: (555) 123-4568 (Work)

3. Bob Smith
   Position: Marketing Director
   Email: bob@flashapparel.com
   Phone: (555) 123-4569 (Mobile)
```

---

## Cross-References

### To Other Skills

- **control-erp-core**: Revenue formula, TransactionType mappings, StatusID reference
- **control-erp-financial**: AR aging detail, payment history, credit analysis
- **control-erp-sales**: Product mix per customer, order patterns

### To Reference Files

- **references/customer-analysis.md**: Full RFM scoring query, Pareto analysis, personalized dormancy detection algorithm (extracted from main file when exceeding 1,000 lines)

---

*Skill created: 2026-02-09*
*Plan: 10-01 (CUST-01 - Customer Profile Lookup)*
*Extended: 2026-02-09*
*Plan: 10-02 (CUST-02, CUST-03 - Customer Ranking, Segmentation, At-Risk Detection)*
*Line count: 1,073 lines (main file), ~[TBD] (references)*
