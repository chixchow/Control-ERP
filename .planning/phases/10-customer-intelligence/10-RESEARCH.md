# Phase 10: Customer Intelligence - Research

**Researched:** 2026-02-09
**Domain:** Control ERP Customer Lookup, Segmentation, CLV, and Churn Detection
**Confidence:** HIGH

## Summary

This phase creates a new `control-erp-customers` skill enabling customer lookup, segmentation, CLV analysis, and at-risk detection. The research confirms that **all three requirements (CUST-01 through CUST-03) can be implemented using tables and join patterns already fully documented** in the existing schema files, common_joins reference, and advanced_analytics reference. No new table discovery is needed.

The Account table has 54,719 rows with explicit `IsClient` and `IsProspect` boolean flags. Account has direct FK-style references to AccountContact (via PrimaryContactID, AccountingContactID), Address (via BillingAddressID, ShippingAddressID), PhoneNumber (via MainPhoneNumberID), Employee (via SalesPersonID1/2/3), and PaymentTerms (via PaymentTermsID). The polymorphic ParentID pattern links AccountContact, PhoneNumber, AddressLink, and AccountUserField back to Account using ParentClassTypeID=2000.

The critical revenue query (`SUM(SubTotalPrice) WHERE TransactionType=1 AND IsActive=1 AND SaleDate IS NOT NULL`) is already validated to 99.98% accuracy against the core baseline ($3,053,541.85 known 2025 income). Customer-level breakdowns use `TransHeader.AccountID = Account.ID`. The advanced_analytics reference already contains working CLV, dormancy, retention, and Pareto analysis queries that serve as starting templates.

**Primary recommendation:** Create `skills/control-erp-customers/control-erp-customers-SKILL.md` with the same dependency pattern as other skills (depends on control-erp-core). Structure as: customer profile queries, search patterns, segmentation/ranking, RFM scoring, and at-risk detection. Keep the main SKILL.md under ~1,000 lines; extract detailed RFM methodology and dormancy algorithms to `references/customer-analysis.md` if needed.

## Standard Stack

This phase does not introduce new libraries or tools. It creates a new skill file using existing database tables.

### Core Tables
| Table | Row Count | Purpose | Key Columns |
|-------|-----------|---------|-------------|
| Account | 54,719 | Customer records | CompanyName, AccountNumber, IsClient, IsProspect, IsActive, IsVendor, SalesPersonID1/2/3, PaymentTermsID, CreditBalance, HasCreditAccount, DateCreated |
| AccountContact | 64,851 | Contact people | FirstName, LastName, EmailAddress, AccountID (polymorphic), IsPrimaryContact, IsAccountingContact, PrimaryNumber |
| TransHeader | 232,243 | Orders/sales | AccountID, TransactionType, SaleDate, SubTotalPrice, BalanceDue, StatusID, OrderNumber |
| Address | 125,614 | Physical addresses | StreetAddress1/2, City, State, PostalCode, Country |
| AddressLink | 304,987 | Polymorphic address linkage | ParentID, ParentClassTypeID, AddressID, AddressName, IsBillingAddress |
| PhoneNumber | 314,805 | Phone numbers (polymorphic) | ParentID, ParentClassTypeID, FormattedText, PhoneNumberTypeText |

### Supporting Tables
| Table | Row Count | Purpose | When to Use |
|-------|-----------|---------|-------------|
| Employee | 103 | Sales rep names | SalesPersonID1 lookup |
| PaymentTerms | 28 | Payment terms reference | Customer profile display |
| AccountUserField | 54,752 | Custom fields | ASI, PPAI, Sales_Status, marketing flags |
| Journal | 5,184,393 | Activity logging (split table with ContactActivity) | Last contact date lookup |
| ContactActivity | 163,716 | CRM activity details | Contact activity types and results |
| MarketingListItem | 96 | Industry/Origin/Region lists | Customer classification display |
| Element | 223 | Company stages (CRM pipeline) | Customer lifecycle stage |
| Payment | 286,084 | Payment records | Late payment detection for risk model |

### No New Dependencies
All requirements use tables and join patterns already documented and validated. The primary new work is composing these into customer-focused query templates and natural language routing.

## Architecture Patterns

### Recommended Skill Structure
```
skills/control-erp-customers/
  control-erp-customers-SKILL.md     # Main skill file (~800-1,000 lines)
  references/                         # Create ONLY IF main file exceeds ~1,000 lines
    customer-analysis.md              # RFM methodology, dormancy algorithm, detailed queries
```

### Pattern 1: Customer Profile Lookup (CUST-01)

**What:** Search for a customer and return an executive summary profile with drill-down capability.
**Join path (all verified from common_joins.md):**

```sql
-- Executive Summary: Account + Primary Contact + Phone + Sales Rep + Payment Terms
SELECT
    a.CompanyName,
    a.AccountNumber,
    a.IsClient,
    a.IsProspect,
    a.IsActive,
    a.DateCreated AS CustomerSince,
    a.CreditBalance,
    a.CreditLimit,
    a.HasCreditAccount,
    -- Primary Contact
    pc.FirstName + ' ' + pc.LastName AS PrimaryContact,
    pc.EmailAddress AS PrimaryEmail,
    -- Phone (from Account directly via PrimaryNumber field)
    a.PrimaryNumber AS MainPhone,
    a.PriNumberTypeText AS PhoneType,
    -- Sales Rep
    e.FirstName + ' ' + e.LastName AS SalesRep,
    -- Payment Terms
    pt.TermsName AS PaymentTerms,
    pt.GracePeriod
FROM dbo.Account a
LEFT JOIN dbo.AccountContact pc ON a.PrimaryContactID = pc.ID
LEFT JOIN dbo.Employee e ON a.SalesPersonID1 = e.ID
LEFT JOIN dbo.PaymentTerms pt ON a.PaymentTermsID = pt.ID
WHERE a.ID = @AccountID
```

**CRITICAL:** Account has BOTH the old polymorphic pattern (AccountContact.ParentID = Account.ID, AccountContact.ParentClassTypeID = 2000) AND direct ID references (Account.PrimaryContactID = AccountContact.ID, Account.AccountingContactID = AccountContact.ID). The direct ID references are simpler for the primary contact. Use the polymorphic pattern to get ALL contacts.

**Phone number shortcut:** Account has denormalized `PrimaryNumber`, `SecondaryNumber`, `ThirdNumber` columns with corresponding `PriNumberTypeText`, `SecNumberTypeText`, `ThirdNumberTypeText`. These are simpler than joining PhoneNumber table for the profile summary. Use PhoneNumber table join only for the full detail view.

### Pattern 2: Customer Search (CUST-01)

**What:** Smart match -- try exact first, fall back to fuzzy. Search CompanyName, AccountContact name, and AccountNumber.

```sql
-- Step 1: Try exact match on AccountNumber
SELECT ID, CompanyName, AccountNumber FROM dbo.Account
WHERE AccountNumber = TRY_CAST(@SearchTerm AS int)
  AND IsActive = 1

-- Step 2: If no result, try LIKE on CompanyName
SELECT ID, CompanyName, AccountNumber FROM dbo.Account
WHERE CompanyName LIKE '%' + @SearchTerm + '%'
  AND IsActive = 1
ORDER BY CompanyName

-- Step 3: If no result, search contacts
SELECT a.ID, a.CompanyName, a.AccountNumber,
       ac.FirstName + ' ' + ac.LastName AS ContactName
FROM dbo.Account a
JOIN dbo.AccountContact ac ON ac.AccountID = a.ID
WHERE (ac.FirstName LIKE '%' + @SearchTerm + '%'
    OR ac.LastName LIKE '%' + @SearchTerm + '%'
    OR ac.FirstName + ' ' + ac.LastName LIKE '%' + @SearchTerm + '%')
  AND a.IsActive = 1
ORDER BY a.CompanyName
```

**IMPORTANT NOTE on AccountContact.AccountID:** The AccountContact schema shows an `AccountID` column (int, nullable) which is a direct FK to Account.ID. This is simpler than the polymorphic ParentID pattern for this specific join. The common_joins.md also shows the polymorphic pattern (`JOIN dbo.AccountContact ac ON ac.ParentID = a.ID AND ac.ParentClassTypeID = 2000`). Both work; use whichever is more appropriate for the query context.

### Pattern 3: Customer Revenue Aggregation (CUST-02)

**What:** Revenue ranking, order frequency, average order value, lifetime value.

```sql
-- Source: output/skill/references/advanced_analytics.md (verified pattern)
SELECT TOP @N
    a.CompanyName,
    a.AccountNumber,
    MIN(th.SaleDate) AS FirstOrder,
    MAX(th.SaleDate) AS LastOrder,
    COUNT(th.ID) AS TotalOrders,
    SUM(th.SubTotalPrice) AS LifetimeRevenue,
    AVG(th.SubTotalPrice) AS AvgOrderValue,
    DATEDIFF(DAY, MIN(th.SaleDate), MAX(th.SaleDate)) AS CustomerDays
FROM dbo.Account a
JOIN dbo.TransHeader th ON th.AccountID = a.ID
WHERE a.IsActive = 1 AND a.IsClient = 1
  AND th.TransactionType = 1
  AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL
  AND th.StatusID NOT IN (9)
GROUP BY a.CompanyName, a.AccountNumber
ORDER BY LifetimeRevenue DESC
```

**YoY comparison pattern:**
```sql
-- Add YoY comparison columns
SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE()) THEN th.SubTotalPrice ELSE 0 END) AS Trailing12Mo,
SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
          AND th.SaleDate < DATEADD(YEAR, -1, GETDATE()) THEN th.SubTotalPrice ELSE 0 END) AS Prior12Mo
```

**Revenue baseline cross-check:** The core skill's validated revenue formula (`SUM(SubTotalPrice) WHERE TransactionType=1 AND IsActive=1 AND SaleDate IS NOT NULL`) produces $3,052,952.52 for 2025. Customer-level GROUP BY of this same query must sum to the same total. This is the internal consistency check required by Success Criteria #4.

### Pattern 4: RFM Segmentation (CUST-02)

**What:** Recency/Frequency/Monetary scoring with named segments.

**Approach:** Use NTILE(5) window functions to create quintile scores, then map combinations to named segments.

```sql
-- RFM scoring with NTILE
;WITH CustomerMetrics AS (
    SELECT
        a.ID AS AccountID,
        a.CompanyName,
        DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) AS Recency,
        COUNT(th.ID) AS Frequency,
        SUM(th.SubTotalPrice) AS Monetary
    FROM dbo.Account a
    JOIN dbo.TransHeader th ON th.AccountID = a.ID
    WHERE a.IsActive = 1 AND a.IsClient = 1
      AND th.TransactionType = 1 AND th.IsActive = 1
      AND th.SaleDate IS NOT NULL AND th.StatusID NOT IN (9)
      AND th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
    GROUP BY a.ID, a.CompanyName
),
RFMScores AS (
    SELECT *,
        NTILE(5) OVER (ORDER BY Recency DESC) AS R_Score,   -- Lower recency = higher score
        NTILE(5) OVER (ORDER BY Frequency ASC) AS F_Score,
        NTILE(5) OVER (ORDER BY Monetary ASC) AS M_Score
    FROM CustomerMetrics
)
SELECT *,
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

**Claude's discretion:** RFM segment boundaries and the exact score-to-segment mapping. The above is a starting template; the planner should document the named segments to use and let the executor refine boundaries.

### Pattern 5: Dormancy Detection with Personalized Thresholds (CUST-03)

**What:** Multi-signal risk model with personalized dormancy thresholds based on each customer's historical ordering pattern.

**Personalized dormancy algorithm:**
```sql
-- Calculate each customer's typical order gap
;WITH OrderGaps AS (
    SELECT
        th.AccountID,
        th.SaleDate,
        LAG(th.SaleDate) OVER (PARTITION BY th.AccountID ORDER BY th.SaleDate) AS PrevSaleDate,
        DATEDIFF(DAY,
            LAG(th.SaleDate) OVER (PARTITION BY th.AccountID ORDER BY th.SaleDate),
            th.SaleDate
        ) AS DaysBetweenOrders
    FROM dbo.TransHeader th
    WHERE th.TransactionType = 1 AND th.IsActive = 1
      AND th.SaleDate IS NOT NULL AND th.StatusID NOT IN (9)
),
CustomerCadence AS (
    SELECT
        AccountID,
        COUNT(*) AS OrderCount,
        AVG(DaysBetweenOrders * 1.0) AS AvgGap,
        STDEV(DaysBetweenOrders) AS StdDevGap,
        MAX(DaysBetweenOrders) AS MaxGap,
        -- Personal threshold = avg gap + 1.5 std deviations (or 90 days minimum)
        CASE
            WHEN COUNT(*) >= 4 -- Need enough history to establish a pattern
            THEN CAST(AVG(DaysBetweenOrders * 1.0) + 1.5 * ISNULL(STDEV(DaysBetweenOrders), 0) AS INT)
            ELSE 90  -- Fallback for customers without enough history
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
    -- Trailing 12mo revenue for "revenue at risk"
    SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE()) THEN th.SubTotalPrice ELSE 0 END) AS Trailing12MoRevenue,
    SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
              AND th.SaleDate < DATEADD(YEAR, -1, GETDATE()) THEN th.SubTotalPrice ELSE 0 END) AS Prior12MoRevenue
FROM dbo.Account a
JOIN dbo.TransHeader th ON th.AccountID = a.ID
LEFT JOIN CustomerCadence cc ON cc.AccountID = a.ID
WHERE a.IsActive = 1 AND a.IsClient = 1
  AND th.TransactionType = 1 AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL AND th.StatusID NOT IN (9)
GROUP BY a.CompanyName, a.AccountNumber, cc.AvgGap, cc.PersonalThreshold
HAVING DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) > ISNULL(cc.PersonalThreshold, 90)
ORDER BY
    -- Combined score: revenue impact * overdue severity
    SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE()) THEN th.SubTotalPrice ELSE 0 END)
    * (DATEDIFF(DAY, MAX(th.SaleDate), GETDATE()) * 1.0 / ISNULL(NULLIF(cc.PersonalThreshold, 0), 90))
    DESC
```

**Minimum history for pattern detection (Claude's discretion):** Require >= 4 orders with measurable gaps to establish a pattern. Customers with fewer orders default to the 90-day fallback threshold. This is conservative but prevents false alarms from one-time buyers.

### Pattern 6: Last Contact Activity (for Customer Profile)

**What:** Get last CRM contact activity date per customer.
**Key insight (from wiki):** ContactActivity is a "split table" with Journal. They share the same ID space. Journal.AccountID links activities to accounts. Journal.ContactID links to contacts.

```sql
-- Last contact activity for a customer
SELECT
    MAX(j.CompletedDateTime) AS LastContactDate,
    MAX(j.Description) AS LastContactDescription  -- only meaningful with a subquery
FROM dbo.Journal j
WHERE j.AccountID = @AccountID
  AND j.ClassTypeID = 11  -- ContactActivity ClassTypeID
  AND j.IsActive = 1
  AND j.CompletedDateTime IS NOT NULL
```

**Alternative (more detailed):**
```sql
-- Last contact with details
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

**IMPORTANT:** Journal.ClassTypeID for ContactActivity entries needs verification. The wiki states ContactActivity ClassTypeID is 11, and Journal.JournalActivityType determines the activity subtype. The exact ClassTypeID used in Journal for contact activities may differ. **This is a LOW confidence finding that must be validated during execution.**

### Pattern 7: AR Balance per Customer

**What:** Show outstanding AR balance in customer profile.

**Two approaches (wiki-verified):**
1. **Account.CreditBalance** -- This tracks customer *credit* balance (overpayments/credits), NOT AR. This is NOT the AR balance.
2. **TransHeader.BalanceDue** summed per customer -- This IS the AR balance.

```sql
-- Customer AR balance (correct approach)
SELECT
    SUM(CASE WHEN th.BalanceDue > 0 AND th.StatusID BETWEEN 1 AND 4
             THEN th.BalanceDue ELSE 0 END) AS ARBalance,
    SUM(CASE WHEN th.BalanceDue > 0 AND th.StatusID = 3
             AND DATEDIFF(DAY, th.SaleDate, GETDATE()) > 30
             THEN th.BalanceDue ELSE 0 END) AS OverdueAR
FROM dbo.TransHeader th
WHERE th.AccountID = @AccountID
  AND th.TransactionType IN (1, 6)
  AND th.IsActive = 1
```

**CRITICAL GOTCHA:** `Account.CreditBalance` is customer CREDITS (overpayments owed TO the customer), not AR. The AR balance is the sum of `TransHeader.BalanceDue` for open orders. These are conceptually opposite. The existing financial skill documents this correctly.

### Anti-Patterns to Avoid

- **Using AccountName instead of CompanyName:** Account table does NOT have an AccountName column. Always use `CompanyName`.
- **Using TotalPrice instead of SubTotalPrice for revenue:** SubTotalPrice is pre-tax and matches FLS's definition of "income."
- **Using OrderCreatedDate instead of SaleDate:** SaleDate is the revenue recognition date.
- **Filtering ContactActivity directly without Journal join:** ContactActivity does not have AccountID; you must join through Journal.
- **Confusing Account.CreditBalance with AR balance:** CreditBalance = credits owed TO customer; AR = TransHeader.BalanceDue sum.
- **Forgetting IsClient=1 filter:** Account table contains prospects, vendors, and personal accounts. Default customer queries must filter `IsClient=1 AND IsActive=1`.
- **Using SaleDate on Type 2 estimates:** SaleDate is always NULL for Type 2. Use EstimateCreatedDate.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Revenue calculation | Custom SUM logic | Core skill's validated formula | Already validated to 99.98% accuracy |
| AR aging buckets | Custom date math | Financial skill's existing pattern | Already handles edge cases (WIP vs Sale, PercentComplete) |
| Join patterns | Custom FK discovery | common_joins.md templates | All polymorphic patterns documented |
| ClassTypeID lookups | Hardcoded values | classtypeid_reference.md | 350+ values already catalogued |
| Payment terms display | Custom lookup | PaymentTerms.TermsName JOIN | Only 28 rows, simple LEFT JOIN |

## Common Pitfalls

### Pitfall 1: Walk-In Account Contamination
**What goes wrong:** The default "Walk-In" account (ID=10, AccountNumber=0) accumulates one-off transactions and inflates customer counts or skews CLV metrics.
**Why it happens:** Walk-In is a catch-all used when no customer account is specified.
**How to avoid:** Exclude AccountNumber=0 or Account.ID=10 from customer analytics. Include a filter: `AND a.AccountNumber > 0` or `AND a.ID <> 10`.
**Warning signs:** If "Walk-In" appears in top customers or customer counts seem inflated.

### Pitfall 2: Voided Orders in Revenue Calculations
**What goes wrong:** Including StatusID=9 (voided) orders inflates revenue figures.
**Why it happens:** Voided orders still have SubTotalPrice values.
**How to avoid:** Always include `StatusID NOT IN (9)` in revenue queries. The core skill mandates this.
**Warning signs:** Revenue totals significantly exceed the $3,052,952.52 2025 baseline.

### Pitfall 3: IsClient vs IsProspect Semantics
**What goes wrong:** Assuming all non-prospects are clients, or vice versa.
**Why it happens:** IsClient and IsProspect are independent boolean flags. An account could theoretically have both set, or neither set.
**How to avoid:** Always explicitly filter `IsClient=1` for customer queries. Don't use `IsProspect=0` as a proxy for IsClient.
**Warning signs:** Customer counts that don't match expectations; vendor or personal accounts appearing in customer lists.

### Pitfall 4: Dormancy False Positives for Seasonal/Infrequent Buyers
**What goes wrong:** Flagging customers as "at risk" when their ordering pattern is naturally infrequent.
**Why it happens:** Flat dormancy thresholds (e.g., 90 days) don't account for customers who only order annually or seasonally.
**How to avoid:** Use the personalized threshold algorithm (avg gap + 1.5 stdev). Require minimum order history (>=4 orders) before establishing a pattern.
**Warning signs:** Customers appearing as "at risk" who have a consistent yearly ordering pattern.

### Pitfall 5: Confusing CreditBalance with AR Balance
**What goes wrong:** Reporting Account.CreditBalance as "amount owed by customer" when it's actually "amount owed TO customer."
**Why it happens:** Field name is misleading. CreditBalance tracks overpayments and credits, not receivables.
**How to avoid:** Use `SUM(TransHeader.BalanceDue)` for AR. Use `Account.CreditBalance` only for customer credit display.
**Warning signs:** Negative values appearing where positive AR balances are expected, or vice versa.

### Pitfall 6: ContactActivity Join Path
**What goes wrong:** Trying to join ContactActivity directly to Account using a ParentID pattern.
**Why it happens:** ContactActivity is a "split table" with Journal but lacks its own AccountID column.
**How to avoid:** Join ContactActivity to Journal on shared ID (ContactActivity.ID = Journal.ID), then use Journal.AccountID.
**Warning signs:** NULL values in account joins, or missing activity data.

## Code Examples

All patterns verified from existing skill files and wiki extracts.

### Complete Customer Profile Query (CUST-01)
```sql
-- Source: Composed from common_joins.md Account patterns + advanced_analytics.md
SELECT
    a.CompanyName,
    a.AccountNumber,
    a.IsClient,
    a.IsProspect,
    a.IsActive,
    a.DateCreated AS CustomerSince,
    a.PrimaryNumber AS MainPhone,
    a.PriNumberTypeText AS PhoneType,
    -- Primary Contact
    pc.FirstName + ' ' + pc.LastName AS PrimaryContact,
    pc.EmailAddress AS PrimaryEmail,
    -- Accounting Contact
    acct_c.FirstName + ' ' + acct_c.LastName AS AccountingContact,
    acct_c.EmailAddress AS AccountingEmail,
    -- Sales Rep
    e.FirstName + ' ' + e.LastName AS SalesRep,
    -- Payment Terms
    pt.TermsName AS PaymentTerms,
    pt.GracePeriod,
    -- Credit Info
    a.CreditBalance AS CustomerCredits,
    a.CreditLimit,
    a.HasCreditAccount,
    -- Industry/Origin
    ind.ItemName AS Industry,
    orig.ItemName AS Origin,
    -- Revenue Aggregates (subqueries for summary)
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
    -- AR Balance
    (SELECT SUM(th.BalanceDue) FROM dbo.TransHeader th
     WHERE th.AccountID = a.ID AND th.TransactionType IN (1, 6)
       AND th.IsActive = 1 AND th.StatusID BETWEEN 1 AND 4
       AND th.BalanceDue > 0) AS ARBalance
FROM dbo.Account a
LEFT JOIN dbo.AccountContact pc ON a.PrimaryContactID = pc.ID
LEFT JOIN dbo.AccountContact acct_c ON a.AccountingContactID = acct_c.ID
LEFT JOIN dbo.Employee e ON a.SalesPersonID1 = e.ID
LEFT JOIN dbo.PaymentTerms pt ON a.PaymentTermsID = pt.ID
LEFT JOIN dbo.MarketingListItem ind ON a.IndustryID = ind.ID
LEFT JOIN dbo.MarketingListItem orig ON a.OriginID = orig.ID
WHERE a.ID = @AccountID
```

### All Contacts for Customer (Drill-Down)
```sql
-- Source: common_joins.md "Account + Contacts + Phone Numbers"
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
ORDER BY ac.IsPrimaryContact DESC, ac.LastName
```

### All Addresses for Customer (Drill-Down)
```sql
-- Source: common_joins.md "Account + Addresses"
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

### Customer Revenue with YoY (CUST-02)
```sql
-- Source: advanced_analytics.md Pareto Analysis + YoY pattern
;WITH CustomerRevenue AS (
    SELECT
        a.ID AS AccountID,
        a.CompanyName,
        a.AccountNumber,
        COUNT(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE()) THEN 1 END) AS OrdersT12M,
        SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE()) THEN th.SubTotalPrice ELSE 0 END) AS RevenueT12M,
        SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
                  AND th.SaleDate < DATEADD(YEAR, -1, GETDATE()) THEN th.SubTotalPrice ELSE 0 END) AS RevenuePrior12M
    FROM dbo.Account a
    JOIN dbo.TransHeader th ON th.AccountID = a.ID
    WHERE a.IsActive = 1 AND a.IsClient = 1 AND a.AccountNumber > 0
      AND th.TransactionType = 1 AND th.IsActive = 1
      AND th.SaleDate IS NOT NULL AND th.StatusID NOT IN (9)
    GROUP BY a.ID, a.CompanyName, a.AccountNumber
)
SELECT TOP @N
    CompanyName,
    AccountNumber,
    RevenueT12M,
    RevenuePrior12M,
    CASE WHEN RevenuePrior12M > 0
         THEN CAST((RevenueT12M - RevenuePrior12M) * 100.0 / RevenuePrior12M AS DECIMAL(5,1))
         ELSE NULL END AS YoYChangePct,
    OrdersT12M,
    CASE WHEN OrdersT12M > 0
         THEN CAST(RevenueT12M / OrdersT12M AS DECIMAL(10,2))
         ELSE 0 END AS AvgOrderValueT12M
FROM CustomerRevenue
WHERE RevenueT12M > 0
ORDER BY RevenueT12M DESC
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Flat 6-month dormancy threshold | Personalized threshold per customer's ordering cadence | Phase 10 (new) | Eliminates false positives for seasonal/infrequent buyers |
| Simple revenue ranking | RFM segmentation with named segments | Phase 10 (new) | Multi-dimensional customer intelligence |
| Manual "who hasn't ordered" reports | Automated multi-signal risk scoring | Phase 10 (new) | Proactive rather than reactive customer management |
| GL-based customer sales (GLClassificationType 8012) | TransHeader-based revenue (validated formula) | Phase 1/v1.0 | 99.98% accuracy vs GL pattern |

**Note on GL-based approach:** The wiki shows a pattern using GL with `GLClassificationType = 8012` for customer pre-tax sales (see CRM wiki extract, "Customer Sales SQL Pattern"). This works but the TransHeader-based approach is already validated and is the standard across all existing skills. Do not use the GL approach.

## Open Questions

### 1. ContactActivity Journal ClassTypeID
- **What we know:** ContactActivity is a split table with Journal. Journal.AccountID links to Account.
- **What's unclear:** The exact Journal.ClassTypeID value for ContactActivity entries. The ContactActivity table has ClassTypeID=11 in its own schema, but the Journal table may use a different ClassTypeID to identify these entries (possibly 11, but wiki shows ClassTypeID 59 for Journal sample data, and 21200 for macro activities, 20060 for emails).
- **Recommendation:** During execution, run a discovery query: `SELECT DISTINCT ClassTypeID, JournalActivityType, JournalActivityText, COUNT(*) FROM Journal WHERE AccountID > 0 GROUP BY ClassTypeID, JournalActivityType, JournalActivityText ORDER BY COUNT(*) DESC`. This will reveal which ClassTypeID values correspond to contact activities.

### 2. Account Flag Distribution (IsClient/IsProspect/IsVendor actual counts)
- **What we know:** Account has IsClient, IsProspect, IsVendor, IsPersonal boolean flags. There are 54,719 total rows.
- **What's unclear:** Exact distribution without MCP access. The schema confirms these are separate boolean columns.
- **Recommendation:** During execution, run: `SELECT IsClient, IsProspect, IsVendor, IsActive, COUNT(*) FROM Account GROUP BY IsClient, IsProspect, IsVendor, IsActive ORDER BY COUNT(*) DESC`. This informs default filter decisions and helps estimate the size of the active client base.

### 3. Unique Customers with Orders (actual count)
- **What we know:** Account has 54,719 rows total. TransHeader has 232,243 rows with AccountID FK.
- **What's unclear:** How many unique Account IDs appear in TransHeader with TransactionType=1 and valid SaleDate.
- **Recommendation:** During execution, run: `SELECT COUNT(DISTINCT AccountID) FROM TransHeader WHERE TransactionType=1 AND IsActive=1 AND SaleDate IS NOT NULL AND StatusID NOT IN (9)`. This is needed to calibrate NTILE buckets for RFM scoring.

### 4. AccountUserField Custom Fields Relevance
- **What we know:** Custom fields include EmailMarketing, DoNot Email/Mail Market, ASI, PPAI, W9 Filed, Invoicing Email, Artwork Line Not Needed, Sales_Status.
- **What's unclear:** Which of these are populated and useful for customer profiles.
- **Recommendation:** During execution, sample: `SELECT TOP 10 * FROM AccountUserField WHERE ASI IS NOT NULL OR Sales_Status IS NOT NULL`. If Sales_Status is meaningfully populated, include it in the customer profile.

### 5. Ordering Cadence Distribution
- **What we know:** FLS has both regular-frequency orderers and as-needed orderers.
- **What's unclear:** The actual distribution of order gaps across the customer base (mean, median, percentiles).
- **Recommendation:** During execution, compute the gap distribution to calibrate the dormancy algorithm: average gap, median gap, 75th/90th percentile. This helps validate the "avg + 1.5 stdev" threshold formula.

## Sources

### Primary (HIGH confidence)
- `output/schemas/Account.md` -- Full Account table schema with 54,719 rows
- `output/schemas/AccountContact.md` -- Contact table schema with AccountID column
- `output/schemas/TransHeader.md` -- Transaction header with AccountID FK
- `output/schemas/AddressLink.md` -- Polymorphic address linkage (ParentClassTypeID=2000 for Account)
- `output/schemas/PhoneNumber.md` -- Polymorphic phone numbers
- `output/schemas/Journal.md` -- Journal table with AccountID, ContactID (5.1M rows)
- `output/schemas/ContactActivity.md` -- Contact activity split table (163K rows)
- `output/skill/references/common_joins.md` -- All verified join patterns including Account joins
- `output/skill/references/advanced_analytics.md` -- CLV, dormancy, retention, Pareto queries
- `output/skill/references/relationships.md` -- Polymorphic pattern documentation
- `output/skill/references/field_values.md` -- TransactionType, StatusID, ClassTypeID reference
- `skills/control-erp-core/control-erp-core-SKILL.md` -- Validated revenue formula ($3,053,541.85)

### Secondary (MEDIUM confidence)
- `output/wiki/extracts/crm_payroll_system_knowledge.md` -- CRM wiki extract confirming IsClient/IsProspect flags, Company Stages, Credit Balance behavior, Customer Sales SQL pattern
- `output/wiki/extracts/sql_queries_reference.md` -- GL integrity checks, Customer Credit Balance integrity check (confirms Account.CreditBalance is credits TO customer)
- `output/wiki/extracts/macros_automation_knowledge.md` -- Journal/ContactActivity split table pattern explanation

### Tertiary (LOW confidence)
- ContactActivity Journal ClassTypeID mapping (needs live verification, see Open Question #1)
- Account flag distribution counts (needs live query, see Open Question #2)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- All tables fully documented in existing schema files
- Architecture patterns: HIGH -- All join patterns verified from common_joins.md and advanced_analytics.md
- Revenue formulas: HIGH -- Validated against $3,053,541.85 baseline in core skill
- Dormancy algorithm: MEDIUM -- Pattern is sound but thresholds need calibration with live data
- ContactActivity linkage: MEDIUM -- Split table pattern confirmed by wiki, but exact ClassTypeID needs verification
- Account flag distribution: LOW -- Flags confirmed to exist, but counts need live query

**Research date:** 2026-02-09
**Valid until:** 2026-03-09 (30 days -- schema is stable, only data distribution questions remain)
