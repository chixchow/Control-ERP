# Phase 11: Inventory Management - Research

**Researched:** 2026-02-09
**Domain:** Control ERP inventory system (MS SQL), skill authoring
**Confidence:** HIGH (all findings verified via live database queries)

## Summary

This research covers everything needed to build a `control-erp-inventory` skill for FLS Banners. The investigation queried all inventory-related tables live, discovered the FLS warehouse configuration, mapped the Part-to-Inventory relationship structure, validated the PO-to-Part cost lookup chain, inventoried unit configurations, and confirmed GL inventory account mappings.

**The single most important finding:** FLS inventory data is in poor shape -- 214 inventory records have negative QuantityOnHand, part-level valuation (QuantityOnHand * AverageCost) produces a meaningless negative $2.1M, while the GL NodeID 10414 shows $651K. This validates the CONTEXT.md decision to use GL for valuation and treat purchasing data (Type 7 POs) as the reliable primary data layer.

**Primary recommendation:** Build a two-tier skill where purchasing intelligence (PO history, vendor costs, order frequency) is the reliable primary tier, and inventory quantities/reorder alerts are the secondary tier with clear data quality warnings.

---

## Standard Stack

This is a skill-authoring phase, not a code phase. The "stack" is the skill file format and MCP SQL query patterns.

### Core
| Component | Purpose | Why Standard |
|-----------|---------|--------------|
| SKILL.md with YAML frontmatter | Skill file consumed by Claude | Established pattern across all control-erp skills |
| MCP mssql tools | Live database queries | Only database access method available |
| SQL query templates in skill | Verified patterns Claude copies/adapts | Proven in financial, sales, customer skills |

### Skill File Pattern (from existing skills)
```yaml
---
name: control-erp-inventory
description: [NL trigger description]
---
```
- Depends on `control-erp-core` for business rules
- Overview section with table architecture
- Query templates with SQL blocks
- Field value references inline
- Natural language routing guidance

---

## Architecture Patterns

### Recommended Skill Structure
```
skills/control-erp-inventory/
  control-erp-inventory-SKILL.md    # Main skill file
```

No `references/` subfolder needed -- existing skills (financial, customers, sales) are single-file. The skill should follow that pattern unless it exceeds ~1,000 lines.

### Pattern 1: Two-Tier Data Confidence
**What:** Separate queries into Primary (reliable PO data) and Secondary (caveated inventory quantities)
**When to use:** Every inventory query should indicate which tier it draws from
**Example structure in skill:**
```
## PRIMARY TIER: Purchasing Intelligence (Reliable)
[PO history, vendor costs, order frequency queries]

## SECONDARY TIER: Inventory Quantities (Caveated)
[Stock levels, reorder alerts -- with warnings about data quality]
```

### Pattern 2: Summary-First with Drill-Down
**What:** Quick answer first (part name, available qty, on-hand qty, unit), drill-down available
**When to use:** Stock level checks, reorder reports
**Example:**
```
Summary: "China Poly 2.4m-45g: 121,931 on hand, 121,606 available (325 reserved) - Sq. Foot"
Drill-down: warehouse breakdown, reorder point, last PO, average cost, GL account
```

### Pattern 3: Default Filter Chain
**What:** Progressive filtering: IsActive=1 -> TrackInventory=1 -> context-dependent qty filter
**When to use:** All inventory queries
**Critical:**
- Stock checks: exclude zero-qty parts (WHERE QuantityOnHand > 0)
- Reorder checks: include zero-qty parts (zero = NEEDS reorder)
- Always filter to non-summary Inventory rows (IsGroup=0 AND IsDivisionSummary=0)

### Anti-Patterns to Avoid
- **Using Part.QuantityOnHand instead of Inventory table:** Part has quantity fields but they are NOT the canonical source. Always use Inventory table with ClassTypeID = 12200.
- **Using QuantityOnHand * AverageCost for valuation:** Produces wildly incorrect results (-$2.1M). Use GL NodeID 10414 balance for accounting valuation.
- **Forgetting to filter out summary/group Inventory rows:** Each part has 4 Inventory records (one per warehouse + summaries). Queries must filter IsGroup=0 AND IsDivisionSummary=0 for actual warehouse rows, or WarehouseID=10 for the default warehouse.
- **Assuming VendorTransDetail.ItemID links directly to Part:** It links to CatalogItem (ClassTypeID 12076) or Part (ClassTypeID 12014) polymorphically. Must check ItemClassTypeID.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Inventory valuation | SUM(Qty * AvgCost) | GL NodeID 10414 balance | Part-level data produces -$2.1M vs GL $651K |
| Part category names | Self-join on Part table | JOIN PricingElement ON Part.CategoryID = PricingElement.ID WHERE ClassTypeID=12035 | Categories are PricingElement records, not Part records |
| Unit display text | Custom lookup | Part.DisplayUnitText (already resolved) | Part stores pre-resolved display unit name |
| Last price paid | Complex subquery | Simple TOP 1 from VendorTransDetail via CatalogItem/Part join | Well-defined chain, verified working |

---

## Common Pitfalls

### Pitfall 1: Inventory Table Has 4 Rows Per Part (Not 1)
**What goes wrong:** Queries return duplicated/quadrupled quantities
**Why it happens:** FLS has 2 divisions (Company, Apparel) x 2 warehouses each = 4 Inventory records per part:
1. WarehouseID=10 "Default Warehouse for Company" (IsGroup=0, IsDivisionSummary=0) -- **USE THIS ONE**
2. WarehouseID=11 "Company WH Summary" (IsGroup=1, IsDivisionSummary=1) -- summary, skip
3. WarehouseID=10000 "Apparel" (IsGroup=1, IsDivisionSummary=1) -- summary, skip
4. WarehouseID=10001 "Apparel WH" (IsGroup=0, IsDivisionSummary=0) -- Apparel division warehouse

**How to avoid:** Always include `AND i.IsGroup = 0 AND i.IsDivisionSummary = 0` in WHERE clause, or filter by WarehouseID=10 for default warehouse only.
**Warning signs:** Total quantities are exactly 2x or 4x expected values.

### Pitfall 2: Negative Quantities Are Common
**What goes wrong:** Parts show negative on-hand or available quantities
**Why it happens:** FLS hasn't been maintaining inventory accurately. 214 records have negative QuantityOnHand.
**How to avoid:** Don't treat negative quantities as errors in queries -- they're real data reflecting the current state. Flag them as "data quality issue" in output.
**Warning signs:** N/A -- this is expected.

### Pitfall 3: AverageCost Is Per-Inventory-Unit, Not Per-Display-Unit
**What goes wrong:** Costs look wildly wrong (e.g., $654/grommet, $304/yard of fabric)
**Why it happens:** AverageCost is per inventory unit (Roll, Each), but some items have display units different from inventory units. A roll of transfer paper at $1,035/roll is reasonable; displayed as "meter" it looks wrong.
**How to avoid:** Always show costs with their unit context. Note when DisplayUnitText differs from inventory UnitText.
**Warning signs:** Costs that seem unreasonably high or low for the item type.

### Pitfall 4: VendorTransDetail Polymorphic Link
**What goes wrong:** Part name lookup fails; ItemID doesn't match Part.ID
**Why it happens:** VendorTransDetail.ItemClassTypeID determines the target table:
- 12014 = Part table (84K rows, mostly garment/apparel outsource items)
- 12076 = CatalogItem table (6.5K rows, raw materials with vendor catalog linkage)
- NULL = freeform line items (39K rows, no link)
**How to avoid:** Always check ItemClassTypeID. For inventory parts with PO history, the CatalogItem path (12076) is the primary one. Chain: VendorTransDetail -> CatalogItem -> Part.
**Warning signs:** NULL PartName in query results when joining directly to Part.

### Pitfall 5: GLAccount Uses AccountGroupID, Not ParentID
**What goes wrong:** GL hierarchy queries fail with execution errors
**Why it happens:** GLAccount's parent field is `AccountGroupID`, not `ParentID` as in most other tables.
**How to avoid:** Use `AccountGroupID` for GL hierarchy navigation.

---

## FLS Warehouse Configuration (DISCOVERED)

FLS has a simple warehouse setup with 2 divisions:

| ID | Name | Type | IsDefault | IsGroup | Division | Purpose |
|----|------|------|-----------|---------|----------|---------|
| -1 | . | 0 | No | No | -1 | System placeholder (inactive) |
| **10** | **Default Warehouse for Company** | 0 (Standard) | **Yes** | No | Company (10) | **Primary warehouse -- use this** |
| 11 | Company WH Summary | 1 (Group) | No | Yes | Company (10) | Division summary (auto-calculated) |
| 10000 | Apparel | 1 (Group) | Yes | Yes | Apparel (10000) | Apparel division summary |
| 10001 | Apparel WH | 0 (Standard) | Yes | No | Apparel (10000) | Apparel division warehouse |

**Key findings:**
- Only 2 real warehouses: Default (ID=10) and Apparel WH (ID=10001)
- Default Warehouse (ID=10) has 273 parts with stock; Apparel WH has only 3
- No Kanban or Stockroom warehouses configured (no ProFITS module)
- For practical purposes, all meaningful inventory is in Warehouse ID=10
- Warehouse breakdown is rarely useful given this configuration -- default to totals

**Recommendation for skill:** Default to Warehouse ID=10 (Company default). Only mention Apparel WH when user explicitly asks about warehouse breakdown or apparel division.

---

## Inventory Table Schema (ClassTypeID 12200)

37 columns. Key fields for queries:

### Quantity Fields
| Field | Type | Description | FLS Data Quality |
|-------|------|-------------|-----------------|
| QuantityBilled | float | In GL (accounted for) | 224 negative records |
| QuantityReceivedOnly | float | Received but not billed | Sparse |
| QuantityOnHand | float | Billed + Received (physically present) | 214 negative records, 552 with qty > 0 |
| QuantityReserved | float | Reserved for pending orders | Mostly 0 |
| QuantityAvailable | float | OnHand - Reserved | **Primary "what's available" field** |
| QuantityOnOrder | float | On open Purchase Orders | Mostly 0 |
| QuantityExpected | float | Available + OnOrder | Rarely used |
| QuantityOnOrderWIP | float | On order for WIP orders | Rarely used |

### Cost/Config Fields
| Field | Type | Description |
|-------|------|-------------|
| AverageCost | float | Weighted average cost per inventory unit |
| AssetAccountID | int | GL inventory asset account (60=Unclassified, 10414=Inventory) |
| WarehouseID | int | FK to Warehouse table |
| PartID | int | FK to Part table |
| ReorderPoint | float | Qty threshold for reorder alert (74 parts configured) |
| ReOrderQuantity | float | Suggested reorder qty (74 parts configured) |
| YellowNotificationPoint | float | Yellow alert threshold |
| RedNotificationPoint | float | Red alert threshold |
| Location | varchar(255) | Physical location in warehouse (mostly NULL at FLS) |
| IsGroup | bit | True = summary/rollup row (filter out for queries) |
| IsDivisionSummary | bit | True = division summary row (filter out for queries) |
| GroupID | int | Parent group Inventory ID |

### Inventory Statistics (FLS)
- Total rows: 27,268 (27,267 with ClassTypeID 12200)
- Active rows: 27,264
- With QuantityOnHand > 0: 552
- With ReorderPoint configured: 74
- GL balance (NodeID 10414): **$651,403.34**

---

## Part Table -- Inventory-Relevant Fields

76 columns total. Key inventory-related fields:

| Field | Type | Description |
|-------|------|-------------|
| ItemName | nvarchar(50) | Part name (searchable) |
| TrackInventory | bit | **Critical filter** -- only 392 active parts have this set |
| AccrueCosts | bit | Full GL accrual mode (199 active tracked parts) vs cost accounting (193) |
| CategoryID | int | FK to PricingElement (ClassTypeID 12035) |
| InventoryID | int | Legacy link to Inventory (only 29 populated -- DO NOT rely on this) |
| UnitID | int | Inventory unit (FK to _Unit) |
| UnitCost | float | Default unit cost |
| PartType | int | All 392 tracked parts are type 0 (Material) |
| SKU | nvarchar(50) | Stock keeping unit code |
| BarCode | nvarchar(50) | Barcode |
| Vendor | varchar(40) | Vendor name (empty for all tracked parts at FLS) |
| VendorPartNumber | varchar(40) | Vendor part number |
| AssetAccountID | int | GL inventory asset account |
| ExpenseAccountID | int | GL expense account for this part |

### Unit Display Fields (Critical for Context)
| Field | Type | Description |
|-------|------|-------------|
| UseInvUnitsForDisplay | bit | True = display same as inventory unit |
| DisplayUnitID | int | Display unit (FK to _Unit) |
| DisplayUnitText | varchar(25) | Pre-resolved display unit name |
| DisplayConversionFormula | text | Formula to convert inventory -> display units |
| TransferUnitID | int | Transfer/purchase unit |
| TransferUnitQuantity | float | Qty per transfer unit (e.g., 3600 grommets per carton) |
| TransferUnitText | varchar(50) | Transfer unit name |

**50 parts** have different display units from inventory units (UseInvUnitsForDisplay=0). Example: inventory tracked in Rolls, displayed in Meters.

### Part Statistics (FLS)
- Total parts: 7,684
- Active parts: 4,879
- TrackInventory=1: 454 (392 active)
- AccrueCosts=1: 199 of 392 active tracked
- Parts with InventoryID set: only 29 (legacy field, do NOT use for joins)

### Part Unit Distribution (Active + Tracked)
| Unit | Count |
|------|-------|
| Each | 256 |
| Yard | 53 |
| Meter | 19 |
| Carton | 17 |
| Sqr. Meter | 12 |
| Roll | 11 |
| Foot | 8 |
| Liter | 6 |
| Kilogram | 4 |
| Inch | 3 |
| Sqr. Foot | 3 |

---

## Part-to-Inventory Relationship

**Relationship: 1 Part -> 4 Inventory rows** (one per warehouse/summary combination)

```
Part (ID=1555, "Banner - 36 in. White")
  -> Inventory (ID=1555,  WH=10,    IsGroup=0, IsDivisionSummary=0, GroupID=16512) -- REAL warehouse row
  -> Inventory (ID=16512, WH=11,    IsGroup=1, IsDivisionSummary=1, GroupID=NULL)  -- Company summary
  -> Inventory (ID=22633, WH=10000, IsGroup=1, IsDivisionSummary=1, GroupID=NULL)  -- Apparel summary
  -> Inventory (ID=22634, WH=10001, IsGroup=0, IsDivisionSummary=0, GroupID=22633) -- Apparel WH row
```

**Join pattern:**
```sql
-- Canonical Part -> Inventory join (gets actual warehouse row, not summaries)
SELECT p.ItemName, i.QuantityOnHand, i.QuantityAvailable, i.AverageCost
FROM Part p
JOIN Inventory i ON i.PartID = p.ID
    AND i.ClassTypeID = 12200
    AND i.IsGroup = 0
    AND i.IsDivisionSummary = 0
    AND i.WarehouseID = 10  -- Default warehouse (where stock actually lives)
WHERE p.IsActive = 1 AND p.TrackInventory = 1
```

**Do NOT use Part.InventoryID** for joining -- only 29 of 392 tracked parts have this field populated.

---

## Part Category Lookup

Categories are stored in the **PricingElement** table (ClassTypeID 12035), not the Part table.

```sql
-- Get part category name
SELECT p.ItemName, pe.ElementName AS CategoryName
FROM Part p
LEFT JOIN PricingElement pe ON p.CategoryID = pe.ID AND pe.ClassTypeID = 12035
WHERE p.IsActive = 1 AND p.TrackInventory = 1
```

### Top Part Categories (Active + Tracked)
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

---

## _Unit Table (Complete Reference)

55 unit types organized by UnitType:

| UnitType | Category | Units |
|----------|----------|-------|
| 1 | Count | Each (1), Sets of 10/100/1000/10000, Piece, Sheet, Page, Impression, Roll, Package, Box, Set, Gross, Carton, Ream, Tub, Skid |
| 2 | Weight | Milligram, Gram, Kilogram, Ounce, Pound, Ton |
| 3 | Length | Inch, Foot, Yard, Meter, Centimeter, Millimeter |
| 4 | Area | Sqr. Inch/Foot/Yard/Meter/Centimeter/Millimeter |
| 5 | Volume | Cubic variants, Milliliter, Liter, Gallon, Quart, Pint, Cup, Fluid Ounce |
| 6 | Time | Second, Minute, Hour, Day |
| 7 | Rate | Square Feet per Hour |
| 0 | Other | Feet per Hour |

Each unit has a `ConversionUnit` field for converting between units of the same type.

---

## Purchase Order Data (Type 7) -- Primary Reliable Tier

### PO Statistics
- Total POs: 12,541
- Closed (StatusID=28): 12,338
- Ordered (StatusID=27): 12
- Requested (StatusID=25): 1
- Cancelled (StatusID=29): 164
- Date range: 2016-06-08 to 2026-02-06

### Top Vendors by PO Count
| Vendor | PO Count |
|--------|----------|
| SanMar | 2,134 |
| S&S Activewear | 1,239 |
| B2Signs | 618 |
| Alpha | 351 |
| Phillipp Textiles, Inc. | 343 |

### VendorTransDetail Structure
129,701 total line items across 47,197 PO references.

**ItemClassTypeID distribution:**
| ClassTypeID | Meaning | Count | Link Target |
|-------------|---------|-------|-------------|
| 12014 | Part | 84,191 | VendorTransDetail.ItemID -> Part.ID |
| NULL | Freeform | 38,987 | No link (manual line items) |
| 12076 | CatalogItem | 6,523 | VendorTransDetail.ItemID -> CatalogItem.ID -> CatalogItem.PartID -> Part.ID |

### "Last Price Paid" Query Chain

For items with CatalogItem link (12076) -- raw materials with vendor catalog:
```sql
-- Last price paid for a specific part (via CatalogItem)
SELECT TOP 1
    p.ItemName AS PartName,
    vd.UnitPrice AS LastPricePaid,
    vd.Quantity AS LastQtyOrdered,
    vd.UnitText AS PriceUnit,
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

For items with direct Part link (12014) -- garments, outsource:
```sql
-- Last price paid for a part (direct Part link)
SELECT TOP 1
    p.ItemName AS PartName,
    vd.UnitPrice AS LastPricePaid,
    vd.Quantity AS LastQtyOrdered,
    vd.UnitText AS PriceUnit,
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

**Coverage:** 160 tracked parts have PO history via CatalogItem path; 118 via direct Part path. Combined coverage is significant but not complete.

### CatalogItem Table
581 items (580 active), linking 519 distinct parts to 73 distinct vendors.

Key fields:
- PartID -> Part.ID
- VendorID -> Account.ID (the vendor account)
- Cost, UnitCost (catalog pricing)
- PackageSize, PackageUnitID
- VendorPartName, PartNumber (vendor's own identifiers)
- VendorPriority (preferred vendor ranking)
- IsDefault (default catalog item for this part)

---

## PartUsageCard Table -- Material Consumption History

286,713 non-voided records spanning 2006-10-26 to 2026-02-09 (today), covering 1,127 distinct parts.

**This is the most data-rich table for consumption analysis.**

Key fields:
| Field | Type | Description |
|-------|------|-------------|
| PartID | int | FK to Part |
| Amount | float | Quantity consumed |
| Cost | float | Cost of consumption |
| PostDate | datetime | When consumption was recorded |
| TransHeaderID | int | Order this consumption is for |
| TransDetailID | int | Line item this consumption is for |
| WarehouseID | int | Warehouse consumed from |
| StationID | int | Production station |
| EmployeeID | int | Who recorded it |
| Description | varchar(100) | How it was generated (e.g., "Manually Entered via Order 133720") |
| IsVoided | bit | Void flag |

**Query pattern for consumption rate:**
```sql
-- Average monthly consumption for a part (last 12 months)
SELECT p.ItemName,
    COUNT(*) AS UsageEvents,
    SUM(puc.Amount) AS TotalConsumed,
    SUM(puc.Amount) / 12.0 AS MonthlyAverage
FROM PartUsageCard puc
JOIN Part p ON puc.PartID = p.ID
WHERE puc.IsVoided = 0
    AND puc.PostDate >= DATEADD(MONTH, -12, GETDATE())
    AND p.ID = @PartID
GROUP BY p.ItemName
```

---

## PartConsumptionJournal Table

**EMPTY at FLS.** Zero non-voided records. This table is not useful for this skill.

---

## InventoryLog Table -- Movement History

906,845 total records spanning 2007-07-30 to 2026-02-09, covering 680 distinct inventory IDs and 1,321 distinct parts.

### ClassTypeID Distribution (Event Types)
| ClassTypeID | Meaning | Count |
|-------------|---------|-------|
| 20531 | Inventory adjustment/consumption (reservation/unreservation) | 543,630 |
| 20520 | Inventory update (quantity recalculation) | 360,414 |
| 20530 | Inventory receive/return | 2,743 |
| 20536 | Inventory transfer | 53 |
| 20500 | Unknown | 2 |
| 20600 | Unknown | 1 |

Key fields:
- PartID, InventoryID -- which part/inventory record
- QuantityOnHand, QuantityAvailable -- snapshot at time of event
- UnitCost, Cost -- cost info
- FromWarehouseID, ToWarehouseID -- for transfers
- TransDetailID, TransPartID -- linked order items
- ModifiedDate -- when the event occurred

---

## PartInventoryConversion Table

2,290 records (2,289 active), covering 1,986 distinct parts.

This table defines how production consumption formulas convert between units. Most records at FLS have NULL ConversionFormula and ConsumptionFormula (simple 1:1 default conversions).

Not directly needed for stock-level or reorder queries, but relevant if users ask "how much material does product X consume?"

---

## GL Inventory Accounts

### All Accounts with GLClassificationType = 1003 (Inventory)

| NodeID | Account Name | AccountGroupID | Purpose |
|--------|-------------|----------------|---------|
| **10414** | **Inventory** | 31 | **Primary inventory asset -- accrued parts** |
| **60** | **Unclassified Inventory** | 31 | Off-balance sheet cost tracking (expensed parts) |
| 10052 | Coro Supplies | 10051 | Product-specific |
| 10053 | Banner Supplies | 10051 | Product-specific |
| 10054 | Edge Supplies | 10050 | Product-specific |
| 10055 | Inkjet Supplies | 10050 | Product-specific |
| 10056 | Mount and Lamin Supplies | 10050 | Product-specific |
| 10066 | Acrylic Supplies | 10051 | Product-specific |
| 10067 | Magnetic Supplies | 10051 | Product-specific |
| 10068 | RTA Supplies | 10051 | Product-specific |
| 10069 | Wood and Alum Supplies | 10051 | Product-specific |
| 10071 | Ship and Pack Supplies | 10070 | Product-specific |
| 10072 | SubContract Supplies | 10070 | Product-specific |
| 10088 | Decal Stock Inventory | 10087 | Product-specific |
| 10089 | UV Ink Inventory | 10087 | Product-specific |
| 10268 | Finished Good Inventory - Feather Flag | 10267 | Product-specific |
| 10418 | Work In Progress Inventory | 31 | WIP tracking |
| 6000 | Computed Inventory | NULL | Non-GL computed (ClassificationType 9002) |

### Inventory AssetAccountID Distribution
| AssetAccountID | Account | Inventory Rows | Part Rows |
|----------------|---------|----------------|-----------|
| 60 | Unclassified Inventory | 5,821 | 151 |
| 10414 | Inventory | 18 | 241 |
| 105 | Wages Payable (!) | 10 | -- |
| 6000 | Computed Inventory | 3 | -- |

**Key insight:** Most Inventory rows point to AssetAccountID=60 (off-balance sheet), while most Part records point to 10414 (on-balance sheet). The Inventory.AssetAccountID may not match Part.AssetAccountID. For valuation queries, use the GL view against NodeID 10414.

### Inventory Valuation Queries

**Accounting valuation (recommended):**
```sql
-- Total inventory value from GL (matches balance sheet)
SELECT SUM(Amount) AS InventoryBalance
FROM GL
WHERE GLAccountID = 10414
-- Result: $651,403.34
```

**Part-level valuation (wiki-recommended formula, use with caveats):**
```sql
-- Part-level valuation using QuantityBilled (not OnHand)
SELECT p.ItemName,
    i.QuantityBilled * i.AverageCost AS InventoryValue,
    i.AssetAccountID
FROM Part p
JOIN Inventory i ON p.ID = i.PartID AND i.ClassTypeID = 12200
    AND i.IsGroup = 0 AND i.IsDivisionSummary = 0 AND i.WarehouseID = 10
WHERE p.TrackInventory = 1 AND p.AccrueCosts = 1
```

**Do NOT use QuantityOnHand * AverageCost** -- produces -$2.1M (meaningless).

---

## Inventory Tracking Modes (from Wiki)

Three modes, important for understanding which parts affect GL:

1. **No Tracking** (TrackInventory=0) -- Most FLS parts (4,487 of 4,879 active)
2. **Level Tracking Only** (TrackInventory=1, AccrueCosts=0) -- 193 active parts. Tracks quantities but NO GL entries. Off-balance sheet.
3. **Full Accrual** (TrackInventory=1, AccrueCosts=1) -- 199 active parts. Full GL integration. Uses NodeID 10414.

**Inventory Formula (from wiki):**
```
Billed + Received Only = On Hand
On Hand - Reserved = Available
Available + On Order = Expected
```

---

## Code Examples

### INV-01: Stock Level Check for a Part
```sql
-- Check inventory for a specific part (by name search)
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
         THEN p.DisplayUnitText + ' (inventory unit: ' + u.UnitText + ')'
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

### INV-02: Reorder Alert Report
```sql
-- Parts below reorder point (include zero-qty)
SELECT
    p.ItemName,
    pe.ElementName AS Category,
    i.QuantityAvailable,
    i.QuantityOnHand,
    i.ReorderPoint,
    i.ReOrderQuantity AS SuggestedReorderQty,
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
    AND i.ReorderPoint > 0
    AND i.QuantityAvailable <= i.ReorderPoint
ORDER BY (i.QuantityAvailable - i.ReorderPoint) ASC
```

### INV-03: Last Price Paid (via CatalogItem)
```sql
-- Last price paid for a part (primary path via CatalogItem)
SELECT TOP 1
    p.ItemName,
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

### INV-03: PO History for a Part
```sql
-- Purchase order history for a part (last 10 orders)
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

### Broad Query: Top 25 Most Active Parts
```sql
-- Most active parts by PO frequency (last 12 months)
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
    AND th.SaleDate >= DATEADD(MONTH, -12, GETDATE())
GROUP BY p.ItemName, pe.ElementName, p.DisplayUnitText, i.QuantityAvailable, i.QuantityOnHand
ORDER BY POCount DESC
```

---

## Existing Skill Patterns (for format consistency)

All existing skills follow this pattern:
1. YAML frontmatter with `name` and `description` (NL trigger terms)
2. `**Depends on:** control-erp-core` line
3. Table architecture section with row counts
4. Key field references inline (not separate reference files)
5. SQL query blocks with comments
6. Natural language routing section
7. Sign conventions / gotcha sections where applicable
8. Single file (no references/ subfolder)

The `description` field is critical for routing -- it should contain all the natural language triggers: "inventory", "stock", "what's in stock", "parts", "reorder", "warehouse", "material", "purchase order", "PO", "vendor cost", "what does X cost", "supplier".

---

## Open Questions

1. **Part category hierarchy depth**
   - What we know: Part.CategoryID -> PricingElement.ID (ClassTypeID 12035). PricingElement has ParentID for hierarchy.
   - What's unclear: How deep is the category tree? Are there subcategories that matter for grouping?
   - Recommendation: Use flat category name (ElementName) for display. Don't navigate the hierarchy unless user asks.

2. **TransHeader.SaleDate for POs**
   - What we know: TransHeader query with SaleDate filter kept failing for Type 7 records.
   - What's unclear: Whether SaleDate is reliably populated for POs or if a different date field should be used.
   - Recommendation: Use `th.ID DESC` for ordering POs by recency (auto-increment, always reliable). For date filtering, test in skill development whether SaleDate or ModifiedDate works better.

3. **Apparel division relevance**
   - What we know: Apparel WH (10001) has only 3 items with stock. 7,512 inventory records but almost all zero.
   - What's unclear: Is Apparel division actively used or legacy?
   - Recommendation: Ignore Apparel warehouse by default. Only surface when user explicitly asks about divisions or apparel.

---

## Sources

### Primary (HIGH confidence)
- Live database queries via MCP mssql tools (all findings verified against actual data)
- Wiki extract: `output/wiki/extracts/production_inventory_knowledge.md` -- inventory formula, tracking modes, warehouse types, PO workflow
- Wiki extract: `output/wiki/extracts/database_integration_knowledge.md` -- ClassTypeID reference (12014=Part, 12076=CatalogItem, 12200=Inventory, 12035=Part Category)
- Existing skill: `skills/control-erp-financial/control-erp-financial-SKILL.md` -- GL account reference, NodeID 10414, off-balance sheet context
- Existing skill: `skills/control-erp-core/control-erp-core-SKILL.md` -- Type 7 PO patterns, NL routing patterns

### Secondary (MEDIUM confidence)
- InventoryLog ClassTypeID meanings (20531=adjustment, 20520=update, 20530=receive) -- inferred from data patterns and wiki context, not explicitly documented

---

## Metadata

**Confidence breakdown:**
- Table schemas & relationships: HIGH -- verified via live queries
- FLS warehouse configuration: HIGH -- complete enumeration from database
- PO/vendor cost chain: HIGH -- verified end-to-end with sample data
- Inventory data quality assessment: HIGH -- quantified (214 negative, $651K GL vs -$2.1M calc)
- InventoryLog event types: MEDIUM -- inferred from ClassTypeID patterns
- Part category lookup: HIGH -- verified PricingElement join

**Research date:** 2026-02-09
**Valid until:** 2026-03-09 (stable domain, no expected schema changes)
