---
name: control-erp-inventory
description: Check stock levels, find parts, monitor reorder points, look up purchase order history and vendor costs, analyze material consumption. Use when user asks about inventory, stock, parts, reorder, warehouse, material, purchase orders, POs, vendor costs, "what's in stock", "what does X cost", or supplier queries. Depends on control-erp-core for business rules.
---

# Control ERP Inventory Management -- Stock, Reorder & Purchasing Intelligence

**Depends on:** `control-erp-core` (always read core first for business rules)

---

## Overview

This skill enables complete inventory intelligence for FLS Banners with a **two-tier confidence model**:

### Two-Tier Data Model

**Primary Tier (Reliable):** Purchasing intelligence -- PO history, vendor costs, order frequency. Type 7 PO data is well-maintained and reliable. Use this tier for cost lookups and vendor analysis.

**Secondary Tier (Caveated):** Inventory quantities -- stock levels, reorder alerts. FLS is migrating inventory data from Google Sheets to Control; quantities may not reflect reality. Always caveat inventory quantity results.

### Key Statistics

- **392 active tracked parts** (TrackInventory=1)
- **74 parts with reorder points configured** (only these appear in reorder alerts)
- **2 warehouses**: Default Warehouse (ID=10) has all meaningful stock, Apparel WH (ID=10001) has only 3 parts
- **GL inventory valuation**: $651K (NodeID 10414) vs. part-level calculation -$2.1M (invalid, don't use)

---

## ⚠️ CRITICAL WARNINGS

> **WARNING: Data Quality and Query Patterns**
>
> 1. **Inventory data quality:** FLS inventory is being migrated from Google Sheets. Quantities may be inaccurate. Always warn users that stock levels should be verified physically. Purchasing data (PO history) is reliable.
>
> 2. **Inventory table has 4 rows per part:** Must always filter `IsGroup = 0 AND IsDivisionSummary = 0` to get actual warehouse rows, not summaries. Without this filter, quantities are doubled or quadrupled.
>
> 3. **Use Inventory table, not Part table quantities:** Part has quantity fields but they are NOT canonical. Always use `Inventory` table with `ClassTypeID = 12200`.
>
> 4. **AverageCost is per inventory unit, not per display unit:** When display units differ from inventory units (50 parts), costs need context. Always show the unit alongside cost.
>
> 5. **Never use QuantityOnHand * AverageCost for total valuation:** Produces -$2.1M (meaningless). Use GL NodeID 10414 for accounting valuation ($651K).
>
> 6. **Default filter chain:** `Part.IsActive = 1 AND Part.TrackInventory = 1` then context-dependent: stock checks exclude zero-qty, reorder checks include zero-qty.
>
> 7. **Negative quantities are expected:** 214 records have negative QuantityOnHand. Flag them as data quality issues, don't treat as errors.

---

## TABLE ARCHITECTURE

| Table | Rows | Purpose | Key Columns |
|-------|------|---------|-------------|
| **Part** | 7,684 (392 active tracked) | Parts catalog | ItemName, SKU, TrackInventory, AccrueCosts, CategoryID, UnitID, UnitCost, DisplayUnitText, UseInvUnitsForDisplay, PartType |
| **Inventory** | 27,268 | Stock levels per warehouse | PartID, WarehouseID, QuantityAvailable, QuantityOnHand, QuantityReserved, QuantityOnOrder, AverageCost, ReorderPoint, ReOrderQuantity, IsGroup, IsDivisionSummary, ClassTypeID=12200 |
| **PricingElement** | (varies) | Part categories | ElementName, ClassTypeID=12035 (for part categories) |
| **_Unit** | 55 | Unit of measure reference | UnitText, UnitType, ConversionUnit |
| **Warehouse** | 5 | Warehouse definitions | Name, IsDefault, IsGroup, DivisionDataID |
| **VendorTransDetail** | 129,701 | PO line items | ItemID, ItemClassTypeID (12014=Part, 12076=CatalogItem), UnitPrice, Quantity, UnitText, TransHeaderID |
| **CatalogItem** | 581 | Vendor catalog links | PartID, VendorID, Cost, VendorPartName, IsDefault |
| **TransHeader** | 232,243 | PO headers (Type 7) | TransactionType=7, AccountID (vendor), TransactionNumber, StatusID |
| **PartUsageCard** | 286,713 | Consumption history | PartID, Amount, Cost, PostDate, TransHeaderID, WarehouseID, IsVoided |
| **InventoryLog** | 906,845 | Movement history | PartID, InventoryID, QuantityOnHand, ClassTypeID (20531=adjust, 20520=update, 20530=receive) |

### FLS Warehouse Configuration

- **Warehouse 10 (Default Warehouse for Company)**: Primary warehouse, 273 parts with stock. **Use this for all queries by default.**
- **Warehouse 10001 (Apparel WH)**: Only 3 parts with stock. Mention only if user asks about warehouse breakdown or apparel.
- Warehouses 11 and 10000 are group/summary records -- always filter out.

### Inventory Formula

From Control wiki:
```
Billed + Received Only = On Hand
On Hand - Reserved = Available
Available + On Order = Expected
```

### Inventory Tracking Modes

1. **No Tracking** (TrackInventory=0) -- 4,487 active parts, skip these
2. **Level Tracking Only** (TrackInventory=1, AccrueCosts=0) -- 193 active parts, quantities only, no GL entries
3. **Full Accrual** (TrackInventory=1, AccrueCosts=1) -- 199 active parts, full GL integration via NodeID 10414

### Part Category Reference (Top 10)

| CategoryID | Category Name | Part Count |
|------------|---------------|------------|
| 41230 | Blank Table Covers | 112 |
| 40026 | Inks | 27 |
| 41221 | DH - Feather Flag | 21 |
| 41961 | DH - Tent Spare Parts | 16 |
| 41224 | Fabrics | 15 |
| 41339 | Packing Materials | 14 |
| 41717 | FR 66 Knit | 12 |
| 41972 | DH - Golf | 12 |
| 41229 | Knit Poly | 11 |
| 41868 | DH - Pillow Case Frame | 10 |

### Part-to-Inventory Join Pattern

Canonical join to get actual warehouse row (not summaries):

```sql
-- Canonical Part -> Inventory join (gets actual warehouse row, not summaries)
SELECT p.ItemName, i.QuantityOnHand, i.QuantityAvailable, i.AverageCost
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10
WHERE p.IsActive = 1 AND p.TrackInventory = 1
```

**Do NOT use Part.InventoryID** for joining -- only 29 of 392 tracked parts have it populated (legacy field).

---

## STOCK LEVEL QUERIES (INV-01)

### Check Inventory for a Specific Part (by name search)

```sql
SELECT
    p.ItemName,
    p.SKU,
    pe.ElementName AS Category,
    i.QuantityAvailable,
    i.QuantityOnHand,
    i.QuantityReserved,
    i.QuantityOnOrder,
    i.AverageCost,
    p.DisplayUnitText AS Unit,
    CASE WHEN p.UseInvUnitsForDisplay = 0
         THEN p.DisplayUnitText + ' (inventory tracked in: ' + u.UnitText + ')'
         ELSE p.DisplayUnitText END AS UnitContext,
    i.ReorderPoint,
    i.Location
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
LEFT JOIN _Unit u ON p.UnitID = u.UnitID
WHERE p.IsActive = 1
    AND p.TrackInventory = 1
    AND p.ItemName LIKE '%search_term%'
ORDER BY p.ItemName
```

**Result formatting:**
- Summary format: "**[ItemName]**: [QuantityAvailable] available ([QuantityOnHand] on hand, [QuantityReserved] reserved) -- [Unit]"
- If QuantityAvailable is negative, add warning: "(negative quantity -- data quality issue, verify physically)"
- If UseInvUnitsForDisplay = 0, show both display and inventory units
- If ReorderPoint > 0 and QuantityAvailable <= ReorderPoint, add: "BELOW REORDER POINT"

### What's In Stock (broad overview)

Show top 25 parts with stock, ordered by QuantityOnHand DESC:

```sql
SELECT TOP 25
    p.ItemName,
    pe.ElementName AS Category,
    i.QuantityAvailable,
    i.QuantityOnHand,
    p.DisplayUnitText AS Unit,
    i.AverageCost
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
WHERE p.IsActive = 1
    AND p.TrackInventory = 1
    AND i.QuantityOnHand > 0
ORDER BY i.QuantityOnHand DESC
```

**Note:** 552 inventory records have QuantityOnHand > 0 across 392 tracked parts. For broad queries, show top 25 to avoid overwhelming output.

### Parts by Category

```sql
SELECT
    pe.ElementName AS Category,
    COUNT(*) AS PartCount,
    SUM(CASE WHEN i.QuantityOnHand > 0 THEN 1 ELSE 0 END) AS WithStock,
    SUM(CASE WHEN i.QuantityAvailable < 0 THEN 1 ELSE 0 END) AS NegativeQty
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
WHERE p.IsActive = 1 AND p.TrackInventory = 1
GROUP BY pe.ElementName
ORDER BY PartCount DESC
```

### Part Detail Card (drill-down for a specific part)

For when user wants full detail on one part:

```sql
SELECT
    p.ID AS PartID,
    p.ItemName,
    p.SKU,
    p.BarCode,
    pe.ElementName AS Category,
    p.DisplayUnitText AS DisplayUnit,
    CASE WHEN p.UseInvUnitsForDisplay = 0 THEN u.UnitText ELSE p.DisplayUnitText END AS InventoryUnit,
    p.UseInvUnitsForDisplay,
    p.TransferUnitText AS PurchaseUnit,
    p.TransferUnitQuantity AS QtyPerPurchaseUnit,
    i.QuantityAvailable,
    i.QuantityOnHand,
    i.QuantityBilled,
    i.QuantityReceivedOnly,
    i.QuantityReserved,
    i.QuantityOnOrder,
    i.QuantityExpected,
    i.AverageCost,
    i.ReorderPoint,
    i.ReOrderQuantity,
    i.YellowNotificationPoint,
    i.RedNotificationPoint,
    i.Location,
    p.UnitCost AS DefaultUnitCost,
    p.Vendor,
    p.VendorPartNumber,
    CASE WHEN p.AccrueCosts = 1 THEN 'Full Accrual (GL integrated)' ELSE 'Level Tracking Only' END AS TrackingMode
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
LEFT JOIN _Unit u ON p.UnitID = u.UnitID
WHERE p.ID = @PartID
```

---

## REORDER MONITORING (INV-02)

### Parts Below Reorder Point

```sql
SELECT
    p.ItemName,
    pe.ElementName AS Category,
    i.QuantityAvailable,
    i.QuantityOnHand,
    i.ReorderPoint,
    i.ReOrderQuantity AS SuggestedReorderQty,
    i.QuantityOnOrder,
    p.DisplayUnitText AS Unit,
    i.AverageCost,
    i.AverageCost * i.ReOrderQuantity AS EstimatedReorderCost
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
WHERE p.IsActive = 1
    AND p.TrackInventory = 1
    AND i.ReorderPoint > 0
    AND i.QuantityAvailable <= i.ReorderPoint
ORDER BY (i.QuantityAvailable - i.ReorderPoint) ASC
```

**Note:** Only 74 parts have ReorderPoint configured. Include zero-quantity parts in reorder checks (zero = definitely needs reorder). Sort by most urgent (biggest deficit) first.

**Formatting guidance:**
- For each part: "[ItemName]: [QuantityAvailable] available vs [ReorderPoint] reorder point -- REORDER [SuggestedReorderQty] [Unit] (est. $[EstimatedReorderCost])"
- If QuantityOnOrder > 0, note: "(already [QuantityOnOrder] on order)"
- If QuantityAvailable is negative, flag urgently

### Reorder Summary by Category

```sql
SELECT
    pe.ElementName AS Category,
    COUNT(*) AS PartsNeedingReorder,
    SUM(i.ReOrderQuantity * i.AverageCost) AS TotalEstimatedCost
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
WHERE p.IsActive = 1
    AND p.TrackInventory = 1
    AND i.ReorderPoint > 0
    AND i.QuantityAvailable <= i.ReorderPoint
GROUP BY pe.ElementName
ORDER BY PartsNeedingReorder DESC
```

### Reorder Coverage Note

Reorder alerts depend on accurate ReorderPoint configuration in Control. Only 74 of 392 tracked parts have reorder points set. Parts without reorder points won't appear in alerts even if at zero stock. Recommend user work with Control admin to configure reorder points for critical parts.

---

## PURCHASING INTELLIGENCE -- PRIMARY TIER (Reliable)

**Purchasing data from Type 7 POs is well-maintained and reliable.** These queries are the primary source for cost lookups and vendor intelligence. When a user asks "what does X cost", use the last price paid from PO history.

### Last Price Paid for a Part (via CatalogItem -- primary path)

Most raw materials with vendor catalog linkage use this path:

```sql
SELECT TOP 1
    p.ItemName AS PartName,
    vd.UnitPrice AS LastPricePaid,
    vd.UnitText AS PriceUnit,
    vd.Quantity AS LastQtyOrdered,
    vd.TotalPrice AS LastTotalCost,
    a.CompanyName AS Vendor,
    th.TransactionNumber AS PONumber
FROM VendorTransDetail vd
JOIN CatalogItem ci ON vd.ItemID = ci.ID AND vd.ItemClassTypeID = 12076
JOIN Part p ON ci.PartID = p.ID
JOIN TransHeader th ON vd.TransHeaderID = th.ID
JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 7
    AND p.ID = @PartID
ORDER BY th.ID DESC
```

**Guidance:** Try CatalogItem path (12076) first. If no results, try direct Part path (12014). 160 tracked parts have PO history via CatalogItem; 118 via direct Part link.

### Last Price Paid for a Part (direct Part link -- secondary path)

Garments and outsource items link directly:

```sql
SELECT TOP 1
    p.ItemName AS PartName,
    vd.UnitPrice AS LastPricePaid,
    vd.UnitText AS PriceUnit,
    vd.Quantity AS LastQtyOrdered,
    vd.TotalPrice AS LastTotalCost,
    a.CompanyName AS Vendor,
    th.TransactionNumber AS PONumber
FROM VendorTransDetail vd
JOIN Part p ON vd.ItemID = p.ID AND vd.ItemClassTypeID = 12014
JOIN TransHeader th ON vd.TransHeaderID = th.ID
JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 7
    AND p.ID = @PartID
ORDER BY th.ID DESC
```

### PO History for a Part (last 10 orders)

```sql
SELECT TOP 10
    th.TransactionNumber AS PONumber,
    a.CompanyName AS Vendor,
    vd.Quantity,
    vd.UnitPrice,
    vd.TotalPrice,
    vd.UnitText
FROM VendorTransDetail vd
JOIN CatalogItem ci ON vd.ItemID = ci.ID AND vd.ItemClassTypeID = 12076
JOIN Part p ON ci.PartID = p.ID
JOIN TransHeader th ON vd.TransHeaderID = th.ID
JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 7
    AND p.ID = @PartID
ORDER BY th.ID DESC
```

### Price History for a Part (trend over time)

```sql
SELECT TOP 20
    vd.UnitPrice,
    vd.Quantity,
    vd.UnitText,
    a.CompanyName AS Vendor,
    th.TransactionNumber AS PONumber,
    th.ID AS POSequence
FROM VendorTransDetail vd
JOIN CatalogItem ci ON vd.ItemID = ci.ID AND vd.ItemClassTypeID = 12076
JOIN Part p ON ci.PartID = p.ID
JOIN TransHeader th ON vd.TransHeaderID = th.ID
JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 7
    AND p.ID = @PartID
ORDER BY th.ID DESC
```

**Note:** Use th.ID DESC for ordering POs by recency (auto-increment, always reliable). SaleDate may not work reliably for Type 7 records.

### Top Vendors by PO Count

```sql
SELECT TOP 10
    a.CompanyName AS Vendor,
    COUNT(DISTINCT th.ID) AS POCount,
    SUM(vd.TotalPrice) AS TotalSpend
FROM VendorTransDetail vd
JOIN TransHeader th ON vd.TransHeaderID = th.ID
JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 7
    AND th.StatusID NOT IN (29)  -- exclude cancelled
GROUP BY a.CompanyName
ORDER BY POCount DESC
```

### Most Frequently Ordered Parts (last 12 months)

```sql
SELECT TOP 25
    p.ItemName,
    pe.ElementName AS Category,
    COUNT(DISTINCT th.ID) AS POCount,
    SUM(vd.TotalPrice) AS TotalSpend,
    MAX(vd.UnitPrice) AS LastUnitPrice,
    p.DisplayUnitText AS Unit,
    i.QuantityAvailable,
    i.QuantityOnHand
FROM VendorTransDetail vd
JOIN CatalogItem ci ON vd.ItemID = ci.ID AND vd.ItemClassTypeID = 12076
JOIN Part p ON ci.PartID = p.ID
JOIN TransHeader th ON vd.TransHeaderID = th.ID
LEFT JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200 AND i.IsGroup = 0 AND i.IsDivisionSummary = 0 AND i.WarehouseID = 10
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
WHERE th.TransactionType = 7
    AND p.IsActive = 1 AND p.TrackInventory = 1
GROUP BY p.ItemName, pe.ElementName, p.DisplayUnitText, i.QuantityAvailable, i.QuantityOnHand
ORDER BY POCount DESC
```

### VendorTransDetail Polymorphic Link Reference

| ItemClassTypeID | Meaning | Count | Join Target |
|-----------------|---------|-------|-------------|
| 12076 | CatalogItem | 6,523 | VendorTransDetail.ItemID -> CatalogItem.ID -> CatalogItem.PartID -> Part.ID |
| 12014 | Part (direct) | 84,191 | VendorTransDetail.ItemID -> Part.ID |
| NULL | Freeform | 38,987 | No link (manual line items, skip) |

### PO Status Reference (Type 7)

| StatusID | Status | Count |
|----------|--------|-------|
| 28 | Closed | 12,338 |
| 29 | Cancelled | 164 |
| 27 | Ordered | 12 |
| 25 | Requested | 1 |

---

## MATERIAL CONSUMPTION & USAGE (INV-03)

### Average Monthly Consumption for a Part

```sql
SELECT
    p.ItemName,
    COUNT(*) AS UsageEvents,
    SUM(puc.Amount) AS TotalConsumed,
    SUM(puc.Amount) / 12.0 AS MonthlyAverage,
    p.DisplayUnitText AS Unit
FROM PartUsageCard puc
JOIN Part p ON puc.PartID = p.ID
WHERE puc.IsVoided = 0
    AND puc.PostDate >= DATEADD(MONTH, -12, GETDATE())
    AND p.ID = @PartID
GROUP BY p.ItemName, p.DisplayUnitText
```

**Note:** PartUsageCard has 286K non-voided records spanning 2006-2026, covering 1,127 distinct parts. This is the best source for consumption analysis. PartConsumptionJournal is empty at FLS.

### Top Consumed Parts (last 12 months)

```sql
SELECT TOP 25
    p.ItemName,
    pe.ElementName AS Category,
    SUM(puc.Amount) AS TotalConsumed,
    COUNT(*) AS UsageEvents,
    SUM(puc.Amount) / 12.0 AS MonthlyAverage,
    p.DisplayUnitText AS Unit,
    i.QuantityAvailable,
    i.QuantityOnHand
FROM PartUsageCard puc
JOIN Part p ON puc.PartID = p.ID
LEFT JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200 AND i.IsGroup = 0 AND i.IsDivisionSummary = 0 AND i.WarehouseID = 10
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
WHERE puc.IsVoided = 0
    AND p.IsActive = 1
    AND puc.PostDate >= DATEADD(MONTH, -12, GETDATE())
GROUP BY p.ItemName, pe.ElementName, p.DisplayUnitText, i.QuantityAvailable, i.QuantityOnHand
ORDER BY TotalConsumed DESC
```

### Usage History for a Part (recent events)

```sql
SELECT TOP 20
    puc.PostDate,
    puc.Amount,
    puc.Cost,
    p.DisplayUnitText AS Unit,
    puc.Description,
    th.TransactionNumber AS OrderNumber
FROM PartUsageCard puc
JOIN Part p ON puc.PartID = p.ID
LEFT JOIN TransHeader th ON puc.TransHeaderID = th.ID
WHERE puc.IsVoided = 0
    AND p.ID = @PartID
ORDER BY puc.PostDate DESC
```

### Months of Supply Remaining (consumption-based)

For parts with consumption history, calculate how long current stock will last:

```sql
SELECT
    p.ItemName,
    i.QuantityAvailable AS CurrentStock,
    p.DisplayUnitText AS Unit,
    SUM(puc.Amount) / 12.0 AS MonthlyConsumption,
    CASE WHEN SUM(puc.Amount) > 0
         THEN i.QuantityAvailable / (SUM(puc.Amount) / 12.0)
         ELSE NULL END AS MonthsOfSupply
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200 AND i.IsGroup = 0 AND i.IsDivisionSummary = 0 AND i.WarehouseID = 10
JOIN PartUsageCard puc ON puc.PartID = p.ID AND puc.IsVoided = 0
    AND puc.PostDate >= DATEADD(MONTH, -12, GETDATE())
WHERE p.IsActive = 1 AND p.TrackInventory = 1
    AND i.QuantityAvailable > 0
GROUP BY p.ItemName, i.QuantityAvailable, p.DisplayUnitText
HAVING SUM(puc.Amount) > 0
ORDER BY CASE WHEN SUM(puc.Amount) > 0
         THEN i.QuantityAvailable / (SUM(puc.Amount) / 12.0)
         ELSE 9999 END ASC
```

**Note:** This is a SECONDARY TIER query -- both inventory quantities and consumption records may be incomplete. Caveat results accordingly. Sort by fewest months of supply (most urgent) first.

---

## INVENTORY VALUATION (INV-03)

### Total Inventory Value (Accounting -- GL-based, RECOMMENDED)

```sql
SELECT SUM(Amount) AS InventoryBalance
FROM GL
WHERE GLAccountID = 10414
```

**Result at FLS:** $651,403.34 (as of research date). This matches the balance sheet and is the authoritative inventory valuation.

**Guidance:** When user asks "what's our inventory worth" or "inventory value", use this GL query. The GL balance is the single source of truth for inventory valuation. Do NOT sum part-level costs.

### Inventory GL Sub-Account Breakdown

```sql
SELECT
    ga.ID AS NodeID,
    ga.AccountName,
    SUM(g.Amount) AS Balance
FROM GL g
JOIN GLAccount ga ON g.GLAccountID = ga.ID
WHERE ga.GLClassificationType = 1003
    AND ga.ClassTypeID = 8001  -- leaf accounts only
GROUP BY ga.ID, ga.AccountName
HAVING SUM(g.Amount) <> 0
ORDER BY SUM(g.Amount) DESC
```

Shows breakdown across all inventory-type GL accounts (10414=Inventory, plus product-specific accounts like Banner Supplies, Inks, etc.).

### Part-Level Valuation (operational estimate, use with caveats)

```sql
SELECT
    p.ItemName,
    i.QuantityBilled,
    i.AverageCost,
    i.QuantityBilled * i.AverageCost AS EstimatedValue,
    p.DisplayUnitText AS Unit,
    CASE WHEN p.AccrueCosts = 1 THEN 'Full Accrual' ELSE 'Level Only' END AS TrackingMode
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10
WHERE p.IsActive = 1 AND p.TrackInventory = 1
    AND p.AccrueCosts = 1
    AND i.QuantityBilled <> 0
ORDER BY i.QuantityBilled * i.AverageCost DESC
```

**Note:** Use QuantityBilled (not QuantityOnHand) for valuation per wiki formula. Only show AccrueCosts=1 parts (these are GL-integrated). This is an operational estimate; the GL total is authoritative.

---

## NATURAL LANGUAGE INTERPRETATION

| User Says | Route To | Section |
|-----------|----------|---------|
| "check inventory for X" / "what's in stock for X" / "stock level for X" | Stock Level Check | 4a |
| "what's in stock" / "show me inventory" / "inventory overview" | Broad Stock Overview | 4b |
| "parts by category" / "inventory by category" | Parts by Category | 4c |
| "tell me everything about part X" / "part detail for X" | Part Detail Card | 4d |
| "what needs reordering" / "reorder report" / "low stock" | Reorder Alert | 5a |
| "reorder by category" / "reorder summary" | Reorder Summary | 5b |
| "what does X cost" / "how much is X" / "price of X" | Last Price Paid | 6a/6b |
| "PO history for X" / "purchase orders for X" | PO History | 6c |
| "price history for X" / "price trend" | Price History | 6d |
| "top vendors" / "who do we buy from" | Top Vendors | 6e |
| "most ordered parts" / "what do we buy most" | Most Ordered Parts | 6f |
| "how much X do we use" / "consumption for X" / "usage rate" | Monthly Consumption | 7a |
| "top consumed parts" / "most used materials" | Top Consumed Parts | 7b |
| "usage history for X" / "when did we last use X" | Usage History | 7c |
| "how long will X last" / "months of supply" | Months of Supply | 7d |
| "what's our inventory worth" / "inventory value" | GL Inventory Value | 8a |
| "inventory value breakdown" / "inventory by GL account" | GL Sub-Account | 8b |
| "part-level valuation" / "value of each part" | Part-Level Estimate | 8c |
| "warehouse breakdown" / "apparel inventory" | Run default queries with WarehouseID filter adjusted | General |

### Cross-Skill Routing

- **"AR for vendor X" / "what do we owe vendor"** → Redirect to control-erp-financial (AP queries)
- **"revenue from X" / "sales of X"** → Redirect to control-erp-sales or control-erp-customers
- **"GL inventory account" / "inventory on balance sheet"** → Can answer with 8a/8b, or redirect to control-erp-financial for full GL context

---

## DATA QUALITY & MIGRATION STATUS

**Current State (2026):**
- FLS is migrating inventory from Google Sheets to Control
- Current inventory data quality: 214 records with negative QuantityOnHand, part-level valuation diverges significantly from GL
- Purchasing data (Type 7 POs) is reliable and well-maintained
- Recommendation: Verify stock levels physically for critical decisions; use PO data for cost/vendor questions

**Post-Migration:**
- Inventory quantity queries will become primary tier
- Reorder monitoring will become reliable
- Part-level valuation will align with GL accounts

---
