# Crystal Reports SQL Patterns Reference

Authoritative SQL join patterns extracted from wiki-documented standard reports.

This reference consolidates the 13 wiki-documented standard Crystal Reports with their documented table relationships, join patterns, and field usage. These patterns represent the **official** Cyrious-documented approaches to common reporting scenarios.

---

## Report Catalog

| Report | Purpose | Primary Source | Key Joins |
|--------|---------|----------------|-----------|
| AR Detail Report | Accounts receivable detail by company | TransHeader | TransHeader → Account, PaymentTerms, Employee, EmployeeGroup |
| AR Summary Report | Accounts receivable summary by company | TransHeader | TransHeader → Account, AccountContact, PaymentTerms, Employee, EmployeeGroup |
| Deposits Made Report | Deposit breakdown by payment method | Journal | Journal → Payment → PaymentAccount, GLAccount, TransHeader, Account |
| Estimate Report | Primary estimate/quotation report | TransHeader | TransHeader → TransVariation → TransDetail, Account, Employee, Address |
| GL Listing Report | General ledger entries | Ledger | Ledger → GLAccount, TransHeader, Account, Journal, EmployeeGroup |
| Historical AR Report | Historical accounts receivable (GL-based) | GL | GL → TransHeader → Account, AccountContact, PaymentTerms, Employee |
| Invoice Report | Primary invoice report | TransHeader | TransHeader → TransDetail, Account, AccountContact, Address, Employee, Journal, Payment |
| Orders Placed Between Report | Orders by status change date range | Ledger | Ledger → TransHeader → Account, AccountContact, Employee, EmployeeGroup |
| Sales By X Report | Sales by various dimensions (product, customer, etc.) | GL (Ledger view) | GL → TransHeader → Account, TransDetail, Employee, GLAccount, Address, MarketingListItem |
| Sales Report | Sales activity from GL | GL (Ledger view) | GL → TransHeader → Account, Employee, EmployeeGroup |
| WIP and Built Report | Production management by order | TransHeader | TransHeader → Account, AccountContact, Station, Employee, EmployeeGroup |
| WIP By Line Item Report | Production management by line item | TransDetail | TransDetail → TransHeader → Account, Station, Employee, EmployeeGroup |
| Work Order Report | Production work order with parts/shipping | TransHeader | TransHeader → TransDetail → TransPart, Part, Shipment, Account, Employee |

---

## Report Details

### AR Detail Report

**Purpose:** Accounts receivable detail by company with aging buckets.

**Primary Data Source:** TransHeader

**Tables and Joins:**

```
TransHeader (base)
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
  LEFT JOIN PaymentTerms ON PaymentTerms.ID = Account.PaymentTermsID
  LEFT JOIN AccountContact ON AccountContact.ID = Account.AccountingContactID
  LEFT JOIN Employee ON Employee.ID = TransHeader.Salesperson1ID
  LEFT JOIN EmployeeGroup ON EmployeeGroup.ID = TransHeader.DivisionID
```

**WHERE Clause:**
- TransHeader.StatusID <> 3 (exclude sales)

**Key Fields:**
- TransHeader.OrderNumber, SaleDate, Description, TotalPrice, BalanceDue
- Account.CompanyName, CreditBalance
- AccountContact.FirstName, LastName, PrimaryNumber
- PaymentTerms.TermsName, GracePeriod
- Employee.FirstName, LastName
- EmployeeGroup.DivisionName

**Aging Calculation:**
- Current Age = Date() - CAST(TransHeader.SaleDate AS Date)
- Buckets: Current (< GracePeriod), 0-30, 31-60, 61-90, 91+

**Grouping:**
1. Division (EmployeeGroup.DivisionID)
2. Company (Account.ID)

---

### AR Summary Report

**Purpose:** Accounts receivable summary by company with drill-down to orders.

**Primary Data Source:** TransHeader

**Tables and Joins:**

```
TransHeader (base)
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
  LEFT JOIN PaymentTerms ON PaymentTerms.ID = Account.PaymentTermsID
  LEFT JOIN AccountContact ON AccountContact.ID = TransHeader.ContactID
  LEFT JOIN Employee ON Employee.ID = TransHeader.Salesperson1ID
  LEFT JOIN EmployeeGroup ON EmployeeGroup.ID = TransHeader.DivisionID
```

**WHERE Clause:**
- TransHeader.StatusID <> 3

**Key Difference from AR Detail:**
- Contact is TransHeader.ContactID (order contact) not Account.AccountingContactID (AP contact)

**Key Fields:**
- Same as AR Detail but uses order-specific contact

**Grouping:**
1. Division (EmployeeGroup.DivisionID)
2. Company (Account.ID)

**Drill Down Fields:**
- TransHeader.OrderNumber, SaleDate, Description, TotalPrice, BalanceDue
- Employee.FirstName + " " + Employee.LastName

---

### Deposits Made Report

**Purpose:** Breakdown of payments by deposit (Make Deposits feature).

**Primary Data Source:** Journal (Deposit entries)

**Tables and Joins:**

```
Journal AS Deposit (base)
  LEFT JOIN Payment ON Payment.DepositJournalID = Deposit.ID
  LEFT JOIN Journal ON Journal.ID = Payment.ID
  LEFT JOIN PaymentAccount ON PaymentAccount.ID = Payment.PaymentAccountID
  LEFT JOIN GLAccount AS PaymentGLAccount ON PaymentGLAccount.ID = Payment.BankAccountID
  LEFT JOIN TransHeader ON TransHeader.ID = Journal.TransactionID
  LEFT JOIN Account ON Account.ID = Journal.AccountID
  LEFT JOIN Employee ON Employee.ID = Journal.EmployeeID
  LEFT JOIN EmployeeGroup ON EmployeeGroup.ID = Deposit.DivisionID
```

**WHERE Clause:**
- Deposit.CompletedDateTime IN {?ReportDateRange}
- Journal.IsSummary = 0

**Key Fields:**
- Deposit.CompletedDateTime, ID, SummaryAmount
- PaymentAccount.AccountName
- PaymentGLAccount.AccountName
- Account.CompanyName
- TransHeader.OrderNumber, Description

**Grouping:**
1. Division (Deposit.DivisionID)
2. Deposit ID (Deposit.ID)

---

### Estimate Report

**Purpose:** Primary estimate/quotation report.

**Primary Data Source:** TransHeader

**Tables and Joins:**

```
TransHeader (base)
  INNER JOIN TransVariation ON TransVariation.TransHeaderID = TransHeader.ID
  INNER JOIN TransDetail ON TransDetail.VariationID = TransVariation.ID
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
  LEFT JOIN AccountContact ON AccountContact.ID = TransHeader.ContactID
  LEFT JOIN Address AS BillingAddress ON BillingAddress.ID = TransHeader.InvoiceAddressID
  LEFT JOIN PaymentTerms ON PaymentTerms.ID = Account.PaymentTermsID
  LEFT JOIN Employee AS Salesperson ON Salesperson.ID = TransHeader.Salesperson1ID
  LEFT JOIN Employee AS EnteredBy ON EnteredBy.ID = TransHeader.EnteredByID
  LEFT JOIN EmployeeGroup AS Division ON Division.ID = TransHeader.DivisionID
```

**WHERE Clause:**
- TransHeader.ID = {?TransHeaderID} (parameter)
- (TransDetail.ParentClassTypeID <> 10400 OR ShowChildItems = True)

**Key Fields:**
- TransHeader.EstimateNumber, CreatedDate, Description, TotalPrice, DiscountPrice, TaxesPrice, Notes
- TransVariation.VariationName, SortIndex, TotalPrice, SubTotalPrice, DiscountPrice, TaxesPrice
- TransDetail.LineItemNumber, Description, Quantity, SubTotalPrice, MeAndSonsSubTotalPrice, GoodsItemCode, HTMLShortFormat
- Account.CompanyName, TaxNumber
- AccountContact.FirstName, LastName, EmailAddress, PrimaryNumber
- Address.StreetAddress1, StreetAddress2, City, State, PostalCode
- Employee.FirstName, LastName, EmailAddress, PrimaryNumber
- PaymentTerms.MessageText

**Grouping:**
1. Estimate (TransHeader.ID)
2. Variation (TransVariation.ID)
3. Line Item (TransDetail.LineItemNumber)

**Child Items Filter:**
- Top-level items: TransDetail.ParentClassTypeID = 10400 (variations)
- Child items: TransDetail.ParentClassTypeID = 10100 (line items)

---

### GL Listing Report

**Purpose:** General ledger entries for orders, customers, activities, or accounts.

**Primary Data Source:** Ledger

**Tables and Joins:**

```
Ledger (base)
  LEFT JOIN GLAccount ON GLAccount.ID = Ledger.GLAccountID
  LEFT JOIN TransHeader ON TransHeader.ID = Ledger.TransactionID
  LEFT JOIN Account ON Account.ID = Ledger.AccountID
  LEFT JOIN Journal ON Journal.ID = Ledger.JournalID
  LEFT JOIN EmployeeGroup ON EmployeeGroup.ID = Ledger.DivisionID
```

**WHERE Clause:**
- Ledger.OnBalanceSheet = 1 OR ?ShowOffBalanceSheet = 1
- Optional filters on GLAccountID, TransactionID, AccountID, DivisionID, ActivityTypeText

**Key Fields:**
- Ledger.EntryDateTime, Amount, OnBalanceSheet
- GLAccount.AccountName
- TransHeader.OrderNumber
- Account.CompanyName
- Journal.ActivityTypeText
- EmployeeGroup.DivisionName

**Debit/Credit Display:**
- If Amount > 0: Display in Debit column
- If Amount < 0: Display -(Amount) in Credit column

**Grouping:**
1. GLAccount.AccountName
2. Ledger.OnBalanceSheet

---

### Historical AR Report

**Purpose:** Historical accounts receivable report using GL as source (not current TransHeader values).

**Primary Data Source:** GL (view of Ledger)

**Tables and Joins:**

```
GL (base)
  LEFT JOIN TransHeader ON TransHeader.ID = GL.TransactionID
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
  LEFT JOIN PaymentTerms ON PaymentTerms.ID = Account.PaymentTermsID
  LEFT JOIN AccountContact ON AccountContact.ID = Account.AccountingContactID
  LEFT JOIN Employee ON Employee.ID = TransHeader.Salesperson1ID
  LEFT JOIN EmployeeGroup ON EmployeeGroup.ID = TransHeader.DivisionID
```

**WHERE Clause:**
- GL.GLAccountID = 14
- GL.EntryDateTime <= ?ReportDate
- TransHeader.StatusID <> 3

**Key Difference from AR Detail:**
- Uses GL.Amount (historical) instead of TransHeader.BalanceDue (current)
- Balance = SUM(GL.Amount) where GL.EntryDateTime <= ?ReportDate

**Key Fields:**
- Same as AR Detail but amounts from GL, not TransHeader

**Grouping:**
1. Division (TransHeader.DivisionID)
2. Company (Account.ID)

---

### Invoice Report

**Purpose:** Primary invoice report.

**Primary Data Source:** TransHeader

**Tables and Joins:**

```
TransHeader (base)
  INNER JOIN TransDetail ON TransDetail.VariationID = TransVariation.ID
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
  LEFT JOIN AccountContact ON AccountContact.ID = TransHeader.ContactID
  LEFT JOIN Address AS BillingAddress ON BillingAddress.ID = TransHeader.InvoiceAddressID
  LEFT JOIN PaymentTerms ON PaymentTerms.ID = Account.PaymentTermsID
  LEFT JOIN Employee AS Salesperson ON Salesperson.ID = TransHeader.Salesperson1ID
  LEFT JOIN Employee AS EnteredBy ON EnteredBy.ID = TransHeader.EnteredByID
  LEFT JOIN EmployeeGroup AS Division ON Division.ID = TransHeader.DivisionID
  LEFT JOIN Journal ON Journal.TransactionID = TransHeader.ID
  LEFT JOIN Payment ON Payment.ID = Journal.ID
  LEFT JOIN AccountContact AS APContact ON APContact.ID = Account.[APContactID]
  LEFT JOIN Address AS ShippingAddress ON ShippingAddress.ID = TransHeader.ShipToAddressID
```

**WHERE Clause:**
- TransHeader.ID = {?TransHeaderID} (parameter)
- (TransDetail.ParentClassTypeID <> 10000 OR ShowChildItems = True)

**Key Fields:**
- Same as Estimate Report
- Additional: Payment information from Journal/Payment joins

**Grouping:**
1. Order (TransHeader.ID)
2. Line Item (TransDetail.LineItemNumber)

**Child Items Filter:**
- Top-level items: TransDetail.ParentClassTypeID = 10000 (variations)
- Child items: TransDetail.ParentClassTypeID = 10100 (line items)

---

### Orders Placed Between Report

**Purpose:** Orders placed, built, sold, closed, or voided within date range.

**Primary Data Source:** Ledger

**Tables and Joins:**

```
Ledger AS GLData (base)
  INNER JOIN TransHeader ON TransHeader.ID = GLData.TransHeaderID
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
  LEFT JOIN AccountContact ON AccountContact.ID = TransHeader.ContactID
  LEFT JOIN Employee AS Salesperson ON Salesperson.ID = TransHeader.Salesperson1ID
  LEFT JOIN EmployeeGroup AS Division ON Division.ID = GLData.DivisionID
  LEFT JOIN Store ON Store.ID = GLData.StoreID
```

**WHERE Clause (varies by ?ReportStatusID):**
- StatusID = 1 (WIP): GLData.EntryDateTime IN {?StartDate TO ?EndDate} AND (GLAccountID IN 11-12 OR GLClassificationType IN 4000-4999)
- StatusID = 2 (Built): GLData.EntryDateTime IN {?StartDate TO ?EndDate} AND (GLAccountID <> 12 OR GLClassificationType IN 4000-4999)
- StatusID = 3 (Sale): GLData.EntryDateTime IN {?StartDate TO ?EndDate} AND GLClassificationType IN 4000-4999
- StatusID = 4 (Closed): TransHeader.ClosedDate IN {?StartDate TO ?EndDate} AND GLClassificationType IN 4000-4999
- StatusID = 9 (Voided): TransHeader.VoidedDate IN {?StartDate TO ?EndDate}

**Key Fields:**
- TransHeader.OrderNumber, Description, StatusText, TotalPrice, BalanceDue
- TransHeader date fields (OrderCreatedDate, BuiltDate, SaleDate, ClosedDate, VoidedDate) based on StatusID
- GLData.Amount (summed for order)
- Account.CompanyName
- Employee.FirstName, LastName

**Grouping:**
1. Division (GLData.DivisionID)
2. Salesperson (optional, based on ?GroupBySalesperson)
3. Report Date (optional, based on ?DateSorted)
4. Is Adjustment (based on whether SaleDate in date range)
5. Order (TransHeader.ID)

**Adjustment Logic:**
- @IsAdj = (TransHeader.SaleDate NOT IN ?StartDate TO ?EndDate)
- New orders: SaleDate within date range
- Adjustments: SaleDate outside date range

---

### Sales By X Report

**Purpose:** Sales totaled by various dimensions (customer, product, salesperson, etc.).

**Primary Data Source:** GL (view of Ledger)

**Tables and Joins:**

```
GL (base)
  LEFT JOIN TransHeader AS Order_Info ON Order_Info.ID = GL.TransactionID
  LEFT JOIN Employee AS Salesperson1 ON Salesperson1.ID = Order_Info.Salesperson1ID
  LEFT JOIN Employee AS Salesperson2 ON Salesperson2.ID = Order_Info.Salesperson2ID
  LEFT JOIN Employee AS Salesperson3 ON Salesperson3.ID = Order_Info.Salesperson3ID
  LEFT JOIN Employee AS EnteredBy ON EnteredBy.ID = Order_Info.EnteredByID
  LEFT JOIN MarketingListItem AS OrderOrigin ON OrderOrigin.ID = Order_Info.OrderOriginID
    WHERE OrderOrigin.MarketingListID = 13
  LEFT JOIN Address AS ZipCode ON ZipCode.ID = Account.BillingAddressID
  LEFT JOIN TransDetail AS LineItem ON LineItem.ID = GL.TransDetailID
  LEFT JOIN CustomerGoodsItem AS Product ON Product.ID = LineItem.GoodsItemID
  LEFT JOIN PricingElement AS ProductCategory ON ProductCategory.ID = Product.CategoryID
    WHERE ProductCategory.ClassTypeID = 12020
  LEFT JOIN Account AS Customer_Info ON Customer_Info.ID = GL.AccountID
  LEFT JOIN MarketingListItem AS CustomerOrigin ON CustomerOrigin.ID = Account.OriginID
    WHERE CustomerOrigin.MarketingListID = 11
  LEFT JOIN MarketingListItem AS CustomerIndustry ON CustomerIndustry.ID = Account.IndustryID
    WHERE CustomerIndustry.MarketingListID = 10
  LEFT JOIN GLAccount AS IncomeAccount ON IncomeAccount.ID = GL.GLAccountID
  LEFT JOIN EmployeeGroup AS Division ON Division.ID = GL.DivisionID
  LEFT JOIN Store ON Store.ID = GL.StoreID
```

**WHERE Clause:**
- GL.EntryDateTime BETWEEN ?StartDate AND ?EndDate
- GL.GLClassificationType BETWEEN 4000 AND 4999
- GL.OffBalanceSheet = 0

**Supporting Command (SQL):**
```sql
DECLARE @StartDate datetime = {?fx_RangeStart_Date};
DECLARE @EndDate datetime = {?fx_RangeEnd_Date};

SELECT TH.ID,
       SUM(GL.Amount) AS dollars,
       COUNT(GL.ID) AS Freq
FROM dbo.Ledger GL
LEFT JOIN dbo.TransHeader TH ON GL.TransactionID = TH.ID
WHERE GL.EntryDateTime BETWEEN @StartDate AND @EndDate
  AND GL.GLClassificationType BETWEEN 4000 AND 4999
  AND GL.OffBalanceSheet = 0
GROUP BY TH.ID
```

**Key Fields:**
- GL.Amount (summed for sales total)
- GL.EntryDateTime
- TransHeader.OrderNumber, Description, StatusText, TotalPrice, BalanceDue, SaleDate
- Account.CompanyName
- Product.ItemName
- ProductCategory.ElementName
- GLAccount.AccountName
- Employee.FirstName, LastName
- MarketingListItem.ItemName (for origins and industry)
- Address.PostalCode

**Grouping (varies by ?OptionalSort):**
1. Division (GL.DivisionID)
2. Selected dimension:
   - CustomerOrigin: Customer_Info.OriginID
   - CustomerIndustry: Customer_Info.IndustryID
   - OrderOrigin: Order_Info.OrderOriginID
   - PostalCode: ZipCode.PostalCode
   - Product: LineItem.GoodsItemID
   - ProductCategory: ProductCategory.ID
   - IncomeAccount: GL.GLAccountID
   - CustomerVolume: Command.dollars
   - CustomerFrequency: Command.Freq
   - Salesperson1/2/3: Order_Info.Salesperson1ID/2ID/3ID
   - EnteredBy: Order_Info.EnteredByID

**Summary Fields:**
- Number of New Orders: COUNT(DISTINCT GL.TransHeaderID WHERE TransHeader.SaleDate IN date range)
- Number of Adjusted Orders: COUNT(DISTINCT GL.TransHeaderID WHERE TransHeader.SaleDate NOT IN date range)
- Total Sales: SUM(GL.Amount)

---

### Sales Report

**Purpose:** Sales activity from GL with adjustments.

**Primary Data Source:** GL (view of Ledger)

**Tables and Joins:**

```
GL (base)
  INNER JOIN TransHeader AS Order_Info ON Order_Info.ID = GL.TransactionID
  LEFT JOIN Account AS Customer_Info ON Customer_Info.ID = GL.AccountID
  LEFT JOIN Employee AS Salesperson ON Salesperson.ID = Order_Info.Salesperson1ID
  LEFT JOIN EmployeeGroup AS Division ON Division.ID = GL.DivisionID
  LEFT JOIN Store ON Store.ID > 0
```

**WHERE Clause:**
- GL.EntryDateTime BETWEEN ?StartDate AND ?EndDate
- GL.GLClassificationType BETWEEN 4000 AND 4999

**Key Fields:**
- GL.Amount (summed), EntryDateTime
- TransHeader.OrderNumber, Description, StatusText, TotalPrice, BalanceDue, SaleDate
- Account.CompanyName
- Employee.FirstName, LastName
- EmployeeGroup.DivisionName

**Grouping:**
1. Division (GL.DivisionID)
2. Salesperson (optional, based on ?GroupBySalesperson)
3. Sale/Adjustment (@IsAdj: whether SaleDate IN date range)
4. Entry Date Time (optional, based on ?OptionalSort)
5. Customer (optional, based on ?OptionalSort)
6. Order (optional, based on ?OptionalSort)

**Adjustment Logic:**
- @IsAdj = (TransHeader.SaleDate NOT IN ?StartDate TO ?EndDate)

---

### WIP and Built Report

**Purpose:** Production management by order status.

**Primary Data Source:** TransHeader

**Tables and Joins:**

```
TransHeader (base)
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
  LEFT JOIN AccountContact ON AccountContact.ID = TransHeader.ContactID
  LEFT JOIN Station ON Station.ID = TransHeader.StationID
  LEFT JOIN Employee ON Employee.ID = TransHeader.Salesperson1ID
  LEFT JOIN EmployeeGroup ON EmployeeGroup.ID = TransHeader.DivisionID
```

**WHERE Clause:**
- TransHeader.StatusID = ?StatusID (1 for WIP, 2 for Built, or both)
- TransHeader.TransactionType = 1 OR (TransactionType = 6 if IncludeServiceTickets)

**Key Fields:**
- TransHeader.OrderNumber, StatusText, DueDate, CreatedDate
- Account.CompanyName
- AccountContact contact fields
- Station.StationName
- Employee.FirstName, LastName
- EmployeeGroup.DivisionName

**Grouping:**
1. Division (TransHeader.DivisionID)
2. Order Status (TransHeader.StatusID)
3. Salesperson (optional, based on ?GroupBySalesperson)
4. Order Station (optional, based on ?GroupByStation)

**Summary:**
- Number of Orders: COUNT(DISTINCT TransHeader.ID)
- SubTotal: SUM(TransHeader.SubTotalPrice)

---

### WIP By Line Item Report

**Purpose:** Production management by line item status.

**Primary Data Source:** TransDetail

**Tables and Joins:**

```
TransHeader (base)
  INNER JOIN TransDetail ON TransDetail.TransHeaderID = TransHeader.ID
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
  LEFT JOIN AccountContact ON AccountContact.ID = TransHeader.ContactID
  LEFT JOIN Station ON Station.ID = TransDetail.StationID
  LEFT JOIN Employee AS Salesperson ON Salesperson.ID = TransHeader.Salesperson1ID
  LEFT JOIN Employee AS AssignedTo ON AssignedTo.ID = TransDetail.AssignedToID
  LEFT JOIN EmployeeGroup ON EmployeeGroup.ID = TransHeader.DivisionID
```

**WHERE Clause:**
- TransHeader.StatusID = ?StatusID (1 for WIP, 2 for Built, or both)
- TransHeader.TransactionType = 1 OR (TransactionType = 6 if IncludeServiceTickets)
- (TransDetail.VariationID = TransDetail.ParentID OR IncludeChildLineItems)
- TransDetail.IsComplete <> 1 (unless ShowCompleted)

**Key Fields:**
- TransHeader.OrderNumber, StatusText, DueDate, CreatedDate
- TransDetail.LineItemNumber, Description, ItemName, Quantity, SubTotalPrice, StationID, AssignedToID, IsComplete
- Account.CompanyName
- Station.StationName
- Employee.FirstName, LastName (for both Salesperson and AssignedTo)
- EmployeeGroup.DivisionName

**Grouping:**
1. Division (TransHeader.DivisionID)
2. Order Status (TransHeader.StatusID)
3. Line Item Station (optional, default) OR Salesperson (optional)

**Child Items Filter:**
- Top-level items: TransDetail.VariationID = TransDetail.ParentID
- Child items: TransDetail.VariationID <> TransDetail.ParentID

**Summary:**
- Number of Line Items: COUNT(DISTINCT TransDetail.ID)
- SubTotal: SUM(TransDetail.SubTotalPrice)

---

### Work Order Report

**Purpose:** Production work order with parts list and shipping information.

**Primary Data Source:** TransHeader

**Tables and Joins:**

```
TransHeader AS OrderLevel (base)
  INNER JOIN TransDetail AS LineItemLevel ON LineItemLevel.TransHeaderID = OrderLevel.ID
  LEFT JOIN Address AS ShippingAddress ON ShippingAddress.ID = OrderLevel.ShippingAddressID
  LEFT JOIN Account AS ShipToCompany ON ShipToCompany.ID = OrderLevel.ShippingCompanyID
  LEFT JOIN AccountContact AS ShippingContact ON ShippingContact.ID = OrderLevel.ShippingContactID
  LEFT JOIN Account AS CustomerInfo ON CustomerInfo.ID = OrderLevel.AccountID
  LEFT JOIN AccountContact AS ContactInfo ON ContactInfo.ID = OrderLevel.ContactID
  LEFT JOIN Employee AS Salesperson ON Salesperson.ID = OrderLevel.Salesperson1ID
  LEFT JOIN Employee AS EnteredBy ON EnteredBy.ID = OrderLevel.EnteredByID
  LEFT JOIN EmployeeGroup AS Division ON Division.ID = OrderLevel.DivisionID
```

**Subreport: Parts List**

```
TransPart AS Part_Link
  LEFT JOIN Part ON Part.ID = Part_Link.PartID
  LEFT JOIN GoodsItemPartLink ON GoodsItemPartLink.ID = Part_Link.PartLinkID
  LEFT JOIN Station AS PartsStation ON PartsStation.ID = Part.StationID
WHERE Part_Link.TransHeaderID = ?TransDetail.ID
```

**Subreport: Shipment Details (using ShipmentDetail view)**

```
ShipmentDetail
  LEFT JOIN Shipments ON Shipments.TransHeaderID = ShipmentDetail.TransHeaderID
  LEFT JOIN TransDetail ON TransDetail.ID = ShipmentDetail.LineItemID
  LEFT JOIN Account ON Account.ID = Shipments.AccountID
  LEFT JOIN AccountContact ON AccountContact.ID = Shipments.ContactID
  LEFT JOIN ShippingMethod ON ShippingMethod.ID = Shipments.CarrierID
  LEFT JOIN AddressLink ON AddressLink.ID = Shipments.ShipToAddressLinkID
  LEFT JOIN Address ON Address.ID = AddressLink.AddressID
WHERE ShipmentDetail.TransHeaderID = ?TransHeader.ID
```

**Key Fields:**
- TransHeader.OrderNumber, OrderDate, Description
- TransDetail.LineItemNumber, Description, GoodsItemCode, Quantity, SubTotalPrice, HTMLLongFormat
- Account.CompanyName
- Employee.FirstName, LastName
- Part.ItemName, PartType, StationID
- Shipment.ShipmentNumber
- ShippingMethod carrier info

**Grouping:**
1. Division (TransHeader.DivisionID)
2. Order (TransHeader.ID)
3. Line Item (TransDetail.ID)

**Subreport Grouping:**
- Parts: Station.StationName
- Shipments: Shipments.ShipmentNumber, then TransDetail.LineItemNumber

---

## Common Join Patterns

### Pattern: TransHeader to Account
```
TransHeader (base)
  LEFT JOIN Account ON Account.ID = TransHeader.AccountID
```
**Used in:** All order/estimate reports
**Purpose:** Get customer information for order/estimate

### Pattern: TransHeader to Contact
```
TransHeader (base)
  LEFT JOIN AccountContact ON AccountContact.ID = TransHeader.ContactID
```
**Used in:** Invoice, Estimate, AR Summary, WIP reports
**Purpose:** Get order-specific contact (not billing/AP contact)

### Pattern: TransHeader to Employee (Salesperson)
```
TransHeader (base)
  LEFT JOIN Employee ON Employee.ID = TransHeader.Salesperson1ID
```
**Used in:** All order/estimate reports
**Purpose:** Get salesperson name for order

### Pattern: TransHeader to Division
```
TransHeader (base)
  LEFT JOIN EmployeeGroup ON EmployeeGroup.ID = TransHeader.DivisionID
```
**Used in:** All reports with division grouping
**Purpose:** Get division name for grouping/filtering

### Pattern: Account to PaymentTerms
```
Account (base or joined)
  LEFT JOIN PaymentTerms ON PaymentTerms.ID = Account.PaymentTermsID
```
**Used in:** AR Detail, AR Summary, Invoice, Estimate
**Purpose:** Get payment terms for invoicing/AR aging

### Pattern: TransHeader to TransDetail (via TransVariation)
```
TransHeader (base)
  INNER JOIN TransVariation ON TransVariation.TransHeaderID = TransHeader.ID
  INNER JOIN TransDetail ON TransDetail.VariationID = TransVariation.ID
```
**Used in:** Invoice, Estimate
**Purpose:** Get line items for order/estimate (full hierarchy)

### Pattern: TransHeader to TransDetail (direct)
```
TransHeader (base)
  INNER JOIN TransDetail ON TransDetail.TransHeaderID = TransHeader.ID
```
**Used in:** WIP By Line Item, Work Order
**Purpose:** Get line items without variation grouping

### Pattern: GL to TransHeader
```
GL (base)
  INNER/LEFT JOIN TransHeader ON TransHeader.ID = GL.TransactionID
```
**Used in:** Sales Report, Sales By X, Historical AR, Orders Placed Between
**Purpose:** Get order details for GL entries (financial reporting)

### Pattern: Ledger to GL Components
```
Ledger (base)
  LEFT JOIN GLAccount ON GLAccount.ID = Ledger.GLAccountID
  LEFT JOIN TransHeader ON TransHeader.ID = Ledger.TransactionID
  LEFT JOIN Account ON Account.ID = Ledger.AccountID
  LEFT JOIN Journal ON Journal.ID = Ledger.JournalID
```
**Used in:** GL Listing, Historical AR
**Purpose:** Complete GL entry context (account, order, customer, activity)

### Pattern: Journal to Payment
```
Journal AS Deposit (base)
  LEFT JOIN Payment ON Payment.DepositJournalID = Deposit.ID
  LEFT JOIN Journal ON Journal.ID = Payment.ID
  LEFT JOIN PaymentAccount ON PaymentAccount.ID = Payment.PaymentAccountID
```
**Used in:** Deposits Made Report
**Purpose:** Get payment details for deposit journal entries

### Pattern: TransDetail to Part
```
TransDetail (base or joined)
  LEFT JOIN TransPart ON TransPart.TransDetailID = TransDetail.ID
  LEFT JOIN Part ON Part.ID = TransPart.PartID
```
**Used in:** Work Order (subreport)
**Purpose:** Get parts list for line item

### Pattern: TransDetail to Station
```
TransDetail (base or joined)
  LEFT JOIN Station ON Station.ID = TransDetail.StationID
```
**Used in:** WIP By Line Item
**Purpose:** Get current production station for line item

### Pattern: TransHeader to Shipment
```
TransHeader (base)
  LEFT JOIN Shipments ON Shipments.TransHeaderID = TransHeader.ID
  LEFT JOIN AddressLink ON AddressLink.ID = Shipments.ShipToAddressLinkID
  LEFT JOIN Address ON Address.ID = AddressLink.AddressID
```
**Used in:** Work Order (subreport)
**Purpose:** Get shipping information for order

---

## Cross-Reference with Inferred Analysis

### Wiki-Documented Reports with Inferred Counterparts

| Wiki Report | Inferred File | Status | Discrepancies |
|-------------|---------------|--------|---------------|
| Sales Report | sales_report.md | Exists | **MAJOR DISCREPANCY**: Wiki documents GL view as primary source; inferred analysis assumed TransHeader directly. Wiki is authoritative. |
| Invoice Report | invoice_report.md | Exists | Minor: Inferred missed Payment joins |
| Estimate Report | estimate_report.md | Exists | Aligned |
| AR Detail Report | ar_detail_report.md | Exists | Aligned |
| AR Summary Report | ar_summary_report.md | Exists | Aligned |
| WIP and Built | wip_and_built_report.md | Exists | Aligned |
| WIP By Line Item | wip_by_line_item_report.md | Exists | Aligned |
| Work Order | work_order_report.md | Exists | Minor: Inferred analysis incomplete on subreports |
| GL Listing | gl_listing_report.md | Exists | Aligned |
| Deposits Made | deposits_made_report.md | Exists | Aligned |
| Historical AR | historical_ar_report.md | Exists | Aligned |
| Orders Placed Between | orders_placed_between_report.md | Exists | Aligned |
| Sales By X | sales_by_x_report.md | Exists | Minor: Inferred analysis incomplete on Command SQL |

### Inferred-Only Reports (No Wiki Documentation)

The following ~23 reports in `output/reports/` have **NO** wiki documentation available for validation. Their join patterns were inferred from field names and may not match actual Crystal Report implementations:

1. account_activity_report.md
2. account_list_report.md
3. aging_report.md
4. artwork_status_report.md
5. commission_detail_report.md
6. commission_summary_report.md
7. customer_list_report.md
8. daily_sales_report.md
9. employee_list_report.md
10. employee_time_report.md
11. gl_account_balance_report.md
12. inventory_report.md
13. item_price_list_report.md
14. order_status_report.md
15. part_usage_report.md
16. payment_detail_report.md
17. payroll_report.md
18. production_schedule_report.md
19. profit_margin_report.md
20. purchase_order_report.md
21. quote_report.md
22. shipping_manifest_report.md
23. station_activity_report.md

**Note:** These inferred patterns should be validated against actual .rpt files or production Crystal Report SQL if available.

---

## Source Files

**Wiki Report Documentation (authoritative):**
- `output/wiki/reports/ar_detail_report_standard.md`
- `output/wiki/reports/ar_summary_report_standard.md`
- `output/wiki/reports/deposits_made_report_standard.md`
- `output/wiki/reports/estimate_report_standard.md`
- `output/wiki/reports/gl_listing_report_standard.md`
- `output/wiki/reports/historical_ar_report_standard.md`
- `output/wiki/reports/invoice_report_standard.md`
- `output/wiki/reports/orders_placed_between_report_standard.md`
- `output/wiki/reports/sales_by_x_report_standard.md`
- `output/wiki/reports/sales_report_standard.md`
- `output/wiki/reports/wip_and_built_report_standard.md`
- `output/wiki/reports/wip_by_line_item_report_standard.md`
- `output/wiki/reports/work_order_report_standard.md`

**Inferred Report Analysis (for comparison):**
- `output/reports/*.md` (36 total files)

---

*Reference created: 2026-02-08*
*For Control ERP Database Mapping Project - Phase 3: Wiki Knowledge Formalization*
