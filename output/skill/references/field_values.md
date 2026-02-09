# Control ERP Field Values Reference

Complete reference for all coded/enumerated field values used throughout the Control ERP database.

---

## TransactionType (TransHeader.TransactionType)

| Value | Type | FLS Banners Usage | Description |
|-------|------|-------------------|-------------|
| 1 | **Order/Sale** | **PRIMARY sales record** | Uses OrderNumber + InvoiceNumber (same value), has SaleDate |
| 2 | Web/Online Order | Separate workflow | Uses EstimateNumber, StatusID=13, SaleDate always NULL |
| 3 | (unused) | NOT USED at FLS | — |
| 4 | (unused) | NOT USED at FLS | — |
| 5 | (unused) | NOT USED at FLS | — |
| 6 | Service Ticket | Rarely used | 1 record in 2025 |
| 7 | Vendor PO | Child of Order | Purchasing materials for a specific job |
| 8 | Vendor Bill Payment | Vendor payments | Bill payment records with PurchaseOrderNumber |
| 9 | Vendor Bill Received | Vendor invoices | Bill received from vendor for materials |

**Transaction Chain at FLS Banners:**
```
Order (Type 1) ──[DefaultOrderID]──→ Vendor PO (Type 7) ──[DefaultOrderID]──→ Vendor Bill (Type 8/9)
  [REVENUE record]                       [COST record]                            [COST record]
```

**Usage:**
```sql
-- Get all sales/orders
SELECT * FROM dbo.TransHeader WHERE TransactionType = 1 AND IsActive = 1 AND StatusID NOT IN (9)

-- Get sales for a customer
SELECT OrderNumber, SaleDate, SubTotalPrice, TotalPrice
FROM dbo.TransHeader WHERE TransactionType = 1 AND AccountID = @AccountID AND StatusID NOT IN (9)

-- Get vendor purchase orders
SELECT * FROM dbo.TransHeader WHERE TransactionType = 7 AND IsActive = 1
```

---

## StatusID (TransHeader.StatusID)

### Order-Specific StatusIDs (TransactionType = 1) — Validated for FLS Banners

| Value | Status | SaleDate? | ClosedDate? | VoidedDate? | Description |
|-------|--------|-----------|-------------|-------------|-------------|
| 1 | New/Draft | No | No | No | Newly created, not yet sold |
| 3 | Sold/In Progress | Yes | No | No | Sale made, not yet closed |
| 4 | **Completed** | Yes | Yes | No | Fully closed (109,864 records) |
| 9 | Voided/Cancelled | Some | No | Yes | Cancelled/voided estimate |

### Other StatusIDs in Use Across All Types

| Value | Approximate Meaning | Used By |
|-------|---------------------|---------|
| 6 | In Progress | Various |
| 11 | Pending | Various |
| 12 | Ordered/Submitted | POs |
| 13 | Received/Processed | Type 2 (Web Orders) |
| 14 | Partially received | POs |
| 28 | Completed (PO) | Type 7 (Vendor POs) |
| 29 | Closed (PO) | POs |

**Note:** FLS Banners does NOT use StatusID 1600/1700 for orders. Those are from the standard Control ERP schema but are not applicable to FLS's workflow.

**Usage:**
```sql
-- Active, non-voided orders (revenue records)
SELECT * FROM dbo.TransHeader WHERE TransactionType = 1 AND StatusID NOT IN (9) AND IsActive = 1

-- Completed orders only
SELECT * FROM dbo.TransHeader WHERE TransactionType = 1 AND StatusID = 4 AND IsActive = 1

-- Open/pipeline orders (not yet completed)
SELECT * FROM dbo.TransHeader WHERE TransactionType = 1 AND StatusID IN (1, 3) AND IsActive = 1
```

To discover all StatusIDs in use:
```sql
SELECT TransactionType, StatusID, COUNT(*) AS [Count]
FROM dbo.TransHeader WHERE IsActive = 1
GROUP BY TransactionType, StatusID
ORDER BY TransactionType, StatusID
```

---

## ClassTypeID (Entity Type Identifier)

ClassTypeID is the cornerstone of Control ERP's polymorphic design. It identifies what TYPE of entity a record is, enabling the `ParentID + ParentClassTypeID` polymorphic foreign key pattern.

**For the complete ClassTypeID reference (350+ entries), see [ClassTypeID Reference](classtypeid_reference.md).**

### Most Common ClassTypeIDs

| Value | Entity | Table | Usage |
|-------|--------|-------|-------|
| 2000 | Account | Account | Customer/vendor records |
| 3000 | Contact | AccountContact | Contact person |
| 10000 | TransHeader | TransHeader | Order/Estimate/PO header |
| 10100 | TransDetail | TransDetail | Line item |
| 12000 | Product | CustomerGoodsItem | Product (FLS uses this) |
| 12014 | Part | Part | Part/material |
| 14030 | TransDetailParam | TransDetailParam | Line item parameter |
| 20050 | Master TimeCard | TimeCard | Clock in/out record |
| 20051 | Detail TimeCard | TimeCard | Station time detail |

### How ClassTypeID Works

Every major entity has a `ClassTypeID` column that identifies its type. Child/related records use `ParentID + ParentClassTypeID` to point back to any entity type:

```sql
-- AccountContact belongs to Account (ClassTypeID 2000)
SELECT * FROM dbo.AccountContact WHERE ParentID = @AccountID AND ParentClassTypeID = 2000

-- PhoneNumber can belong to Account OR Contact
SELECT * FROM dbo.PhoneNumber WHERE ParentID = @AccountID AND ParentClassTypeID = 2000

-- TransDetail belongs to TransHeader
SELECT * FROM dbo.TransDetail WHERE ParentID = @TransHeaderID AND ParentClassTypeID = 10000

-- TransDetailParam belongs to TransDetail
SELECT * FROM dbo.TransDetailParam WHERE ParentID = @TransDetailID AND ParentClassTypeID = 10100
```

**For all 350+ ClassTypeID values and detailed usage examples, see [ClassTypeID Reference](classtypeid_reference.md).**

---

## GoodsItemClassTypeID (TransDetail.GoodsItemClassTypeID)

TransDetail uses a polymorphic link to identify what product or part is being sold on a line item.

### Standard Control ERP Values
| Value | Links To | Description |
|-------|----------|-------------|
| 49 | dbo.Product | Product-type line item |
| 30 | dbo.Part | Part-type line item |

### FLS Banners Actual Usage
| Value | Description |
|-------|-------------|
| **12000** | Catalog item — the ONLY GoodsItemClassTypeID used at FLS Banners |

**Important:** FLS Banners does NOT use GoodsItemClassTypeID 49 (Product) or 30 (Part) on TransDetail line items. All line items use 12000. Use `TransDetail.Description` for the product/item name.

**Usage:**
```sql
-- Line items for an estimate (use ParentClassTypeID = 10000 for TransHeader)
SELECT td.SeqID, td.Description, td.Quantity, td.UnitPrice,
       td.MeAndSonsSubTotalPrice AS LineTotal
FROM dbo.TransDetail td
WHERE td.ParentID = @TransHeaderID AND td.ParentClassTypeID = 10000
ORDER BY td.SeqID

-- Revenue by product (using Description as the product identifier)
SELECT td.Description AS ProductName,
       COUNT(DISTINCT th.ID) AS OrderCount,
       SUM(td.MeAndSonsSubTotalPrice) AS Revenue
FROM dbo.TransDetail td
JOIN dbo.TransHeader th ON td.ParentID = th.ID AND td.ParentClassTypeID = 10000
WHERE th.TransactionType = 1 AND th.IsActive = 1 AND th.StatusID NOT IN (9)
  AND YEAR(th.SaleDate) = @Year
GROUP BY td.Description
ORDER BY Revenue DESC
```

---

## Boolean Flags

### Account Boolean Fields

| Column | Meaning |
|--------|---------|
| IsActive | Record is active (not archived/deleted) |
| IsClient | Account is a customer |
| IsProspect | Account is a prospect (potential customer) |
| IsVendor | Account is a vendor/supplier |
| IsSystem | System-generated account (do not modify) |

```sql
-- Active customers only
SELECT * FROM dbo.Account WHERE IsActive = 1 AND IsClient = 1

-- Active vendors
SELECT * FROM dbo.Account WHERE IsActive = 1 AND IsVendor = 1

-- Prospects not yet converted to clients
SELECT * FROM dbo.Account WHERE IsActive = 1 AND IsProspect = 1 AND IsClient = 0
```

### Other IsActive Tables
Many tables have an `IsActive` column. Always filter on it unless you specifically need inactive records:
- Account, Product, Part, Employee, Station, GLAccount, Warehouse, PricingPlan, and others

---

## DivisionID (FLS Banners Divisions)

FLS Banners operates two divisions:

| Division | Description |
|----------|-------------|
| Company Division | Signs, banners, and large-format printing |
| Apparel Division | Apparel/clothing printing |

To discover division IDs:
```sql
SELECT ID, Name FROM dbo.DivisionData
```

**Tables with DivisionID:** Account, TransHeader, Inventory, Journal, Ledger, EmployeeGroup, and others.

---

## Warehouse IDs (FLS Banners)

| Warehouse ID Range | Division |
|--------------------|----------|
| 10, 11 | Company Division |
| 10000, 10001 | Apparel Division |

```sql
SELECT ID, Name FROM dbo.Warehouse WHERE IsActive = 1
```

---

## Artwork Status (_ArtworkStatus)

The artwork proofing workflow uses status values from the `_ArtworkStatus` lookup table.

```sql
SELECT ID, Name FROM dbo._ArtworkStatus ORDER BY ID
```

Common statuses (verify with query above):
- Not Started
- In Progress
- Awaiting Approval
- Approved
- Completed
- Cancelled

---

## Artwork Lookup Tables

| Table | Purpose | Sample Query |
|-------|---------|--------------|
| _ArtworkBand | Priority bands | `SELECT ID, Name FROM dbo._ArtworkBand` |
| _ArtworkCollectionType | Collection types | `SELECT ID, Name FROM dbo._ArtworkCollectionType` |
| _ArtworkCommentType | Comment categories | `SELECT ID, Name FROM dbo._ArtworkCommentType` |
| _ArtworkLogType | Log entry types | `SELECT ID, Name FROM dbo._ArtworkLogType` |
| _ArtworkPriority | Priority levels | `SELECT ID, Name FROM dbo._ArtworkPriority` |
| _ArtworkRole | Player roles | `SELECT ID, Name FROM dbo._ArtworkRole` |
| _ArtworkStatus | Workflow statuses | `SELECT ID, Name FROM dbo._ArtworkStatus` |

---

## Payment Methods

Payment method values are stored in the Payment table. Discover available methods:
```sql
SELECT DISTINCT PaymentMethod, COUNT(*) AS [Count]
FROM dbo.Payment
GROUP BY PaymentMethod
ORDER BY [Count] DESC
```

---

## SelectionList and SelectionListItem

Control ERP uses SelectionList/SelectionListItem as a general-purpose lookup value store. Many dropdown fields in the UI are backed by these tables.

```sql
-- Find all selection lists
SELECT ID, Name FROM dbo.SelectionList ORDER BY Name

-- Get items for a specific list
SELECT sli.Value, sli.Description
FROM dbo.SelectionListItem sli
WHERE sli.SelectionListID = @ListID
ORDER BY sli.SeqID
```

---

## UserFieldDef (Custom Fields)

FLS Banners has 562 custom field definitions. These are user-defined fields attached to various entity types.

```sql
-- Find custom fields for a specific entity type
SELECT ID, Name, FieldType, ParentClassTypeID
FROM dbo.UserFieldDef
WHERE ParentClassTypeID = 2000  -- Account custom fields
ORDER BY Name

-- Common ParentClassTypeID values for custom fields:
-- 2000 = Account fields
-- 10000 = TransHeader fields
-- 10100 = TransDetail fields
-- 32 = Employee fields
```

---

## Units (_Unit)

The `_Unit` table defines measurement units used for quantities, pricing, and inventory.

```sql
SELECT ID, Name, Abbreviation FROM dbo._Unit ORDER BY Name
```

---

## PricingLevel

Pricing tiers assigned to accounts to determine their pricing.

```sql
SELECT ID, Name FROM dbo.PricingLevel ORDER BY ID
```

Accounts reference their pricing level via `Account.PricingLevelID`.

---

## TimeCard ClassTypeID Detail

TimeCard records come in two flavors:

| ClassTypeID | Type | Description |
|-------------|------|-------------|
| 20050 | Clock In/Out Parent | The overall shift record with clock-in and clock-out times |
| 20051 | Station Time Detail | Time spent at a specific production station within a shift |

```sql
-- Get shift records (clock in/out)
SELECT EmployeeID, StartTime, EndTime
FROM dbo.TimeCard
WHERE ClassTypeID = 20050 AND StartTime >= @Date

-- Get station time detail
SELECT EmployeeID, StationID, StartTime, EndTime
FROM dbo.TimeCard
WHERE ClassTypeID = 20051 AND StartTime >= @Date

-- Join both for complete picture
SELECT
    parent.EmployeeID,
    parent.StartTime AS ShiftStart,
    parent.EndTime AS ShiftEnd,
    detail.StationID,
    s.Name AS StationName,
    detail.StartTime AS StationStart,
    detail.EndTime AS StationEnd
FROM dbo.TimeCard parent
JOIN dbo.TimeCard detail ON detail.ParentID = parent.ID AND detail.ClassTypeID = 20051
LEFT JOIN dbo.Station s ON detail.StationID = s.ID
WHERE parent.ClassTypeID = 20050 AND parent.StartTime >= @Date
```
