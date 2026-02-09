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

<!-- Plan 02 adds: purchasing intelligence (PO history, last price paid, vendor costs), material consumption analysis, inventory valuation, and NL routing table -->

---
