---
name: control-erp-sales
description: Query FLS Banners sales and revenue data from Control ERP. Handles product-level reporting using DyeSub Print container variables (FP_ProductDescription for categories, FP_ProductID for SKUs), table cover identification, customer revenue analysis, and sales trend reporting. Depends on control-erp-core for business rules. Use when the user asks about sales, revenue, product performance, customer sales history, or any revenue-related question.
---

# Control ERP Sales — Product & Revenue Intelligence

**Depends on:** `control-erp-core` (always read core first for business rules)

---

## PRODUCT ARCHITECTURE AT FLS BANNERS

FLS Banners uses **container products** — products in Control that act as configurable shells housing many actual product lines through variables. Understanding this is essential for any product-level reporting.

### Container 1: DyeSub Print (Majority of Revenue)

DyeSub Print is NOT a product. It is a configurable framework. **Never report "DyeSub Print" as a product name in output.** Always drill into the variables.

**Key Variables:**

| Variable | VariableID | Column | Purpose |
|----------|-----------|--------|---------|
| `FP_ProductDescription` | **11053** | `ValueAsStr25` | **Product Category** — Swing Flags, Feather Flags, SEG, Fab Frames, Banners, etc. |
| `FP_ProductID` | **11052** | `ValueAsStr25` | **Product SKU** — specific product identifier with size/options |

**Two levels of detail — always clarify with user if ambiguous:**
- **Category level** → GROUP BY `FP_ProductDescription` (VariableID 11053)
- **SKU level** → GROUP BY or filter by `FP_ProductID` (VariableID 11052)

**Known Categories (2025 validated against Control "Sales by Product" report):**

| Category | 2025 Revenue | Orders |
|----------|-------------|--------|
| Swing Flags | $615,203 | 490 |
| Feather Flags | $485,666 | 544 |
| Banners | $259,212 | 296 |
| SEG | $69,728 | 91 |
| Tear Drop Flags | $53,041 | 61 |
| Banner Stands | $45,505 | 116 |
| Fab Frames | $40,474 | 65 |
| Custom Flag | $39,209 | 101 |
| Golf Flag 14x20 | $38,821 | 151 |
| Table Runner | $35,733 | 112 |
| Pillow Case Frames | $33,553 | 60 |
| Pop Up Banners | $28,841 | 16 |
| Printed Dyesub Paper | $20,619 | 11 |
| Golf Flag 5x8 | $13,485 | 102 |
| Backdrop | $5,823 | 20 |
| Trombone cover | $2,480 | 6 |
| By The Yard | $1,690 | 5 |
| Golf Cart Flag | $1,408 | 7 |
| Bell Covers | $943 | 4 |
| Swoopper Flags | $843 | 3 |
| CUSTOM BURGEE | $0 | 2 |
| **DyeSub Print Total** | **$1,793,445** | — |

**DyeSub Print = 58.7% of total FLS revenue.** This is the core of the business.

### Container 2: DyeLux-Full Print Table Cover ($536K in 2025)

**Important:** TC_ variables (TC_FabricCategory, TC_ProductName) are NOT saved to TransDetailParam. They exist only in the product configurator UI.

**Identify by GoodsItemID (preferred) or Description:**
```sql
-- By GoodsItemID (authoritative — matches Control reports exactly)
WHERE td.GoodsItemID = 10026

-- By Description (legacy — undercounts by ~$57K)
WHERE td.Description LIKE '%Dyelux%Table Cover%'
   OR td.Description LIKE '%FULL%Table Cover%'
```

### All Products — GoodsItemID Mapping

Every TransDetail row has a `GoodsItemID` linking to the Product table (`GoodsItemClassTypeID = 12000`). Use `Product.ItemName` for authoritative product grouping — this matches Control's "Sales by Product" report exactly.

| GoodsItemID | Product (ItemName) | 2025 Revenue | 2024 Revenue |
|------------|-------------------|-------------|-------------|
| 10013 | **Dyesub Print** (container — drill into FP_ vars) | $1,792,275 | $1,803,555 |
| 10026 | DyeLux-Full Print Table Cover | $536,285 | $507,778 |
| 4244 | Decorated Garments | $261,379 | $273,303 |
| 4246 | Garments - Embroidered | $125,839 | $124,300 |
| 10031 | Display Hardware | $101,718 | $86,708 |
| 10024 | Shipping | $84,084 | $90,676 |
| 4243 | Pop-Up Tent Top | $54,392 | $44,799 |
| 4192 | Misc | $26,213 | $31,210 |
| 4230 | Digital Print | $24,610 | $17,054 |
| 4012 | Design | $9,263 | $8,198 |
| 10010 | Custom Table Cover | $8,910 | $12,644 |
| 4191 | Custom Printed Banner | $7,638 | $4,583 |
| 4071 | Custom Coroplast Sign | $4,413 | $5,560 |
| 4223 | Golf Flags | $2,244 | $29,820 |
| 4198 | Table Cover Carry Bag | $2,167 | $2,716 |
| 4126 | Name/Number | $1,561 | $1,143 |
| 4200 | Table Covers | $1,515 | $1,402 |
| 4250 | Stock Flags | $826 | $709 |
| 10028 | Color Match Swatch | $275 | $875 |
| 4231 | DyeLux Print | $70 | $5,213 |
| 4197 | Shipment (legacy) | $299 | $3,527 |
| 4218 | Message Flags | $482 | $383 |
| 4127 | STD Knit Poly Flag | $71 | $358 |
| 4205 | Misc Banner | $473 | — |
| 4209 | DyeLux Full Color Table Covers | — | $2,148 |
| 4210 | DyeLux Front Panel Table Covers | $198 | — |
| 4213 | DyeLux Full Cov Round Table Covers | $132 | $132 |
| 4217 | Flag Hardware | — | $792 |
| 4220 | Golf Flag Hardware and Accessories | — | $156 |
| 10029 | Dyesub Print Rework | $0 | $0 |
| 10033 | Coupon | -$303 | -$351 |
| 10032 | Online Undefined SKU | -$1,137 | -$1,061 |

**Note:** `GoodsItemClassTypeID` in TransDetail is always **12000** (not 49). Join: `Product.ID = td.GoodsItemID AND Product.ClassTypeID = td.GoodsItemClassTypeID`.

---

## QUERY TEMPLATES

### 1. Total Revenue for a Period
```sql
SELECT
    YEAR(SaleDate) AS SaleYear,
    SUM(SubTotalPrice) AS Revenue,
    COUNT(*) AS OrderCount
FROM TransHeader
WHERE TransactionType = 1
    AND IsActive = 1
    AND SaleDate IS NOT NULL
GROUP BY YEAR(SaleDate)
ORDER BY SaleYear DESC
```

### 2. Monthly Revenue Trend
```sql
SELECT
    MONTH(th.SaleDate) AS SaleMonth,
    SUM(th.SubTotalPrice) AS Revenue,
    COUNT(*) AS OrderCount
FROM TransHeader th
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) = @Year
GROUP BY MONTH(th.SaleDate)
ORDER BY SaleMonth
```

### 3. Revenue by DyeSub Product Category
```sql
SELECT
    tdp.ValueAsStr25 AS ProductCategory,
    COUNT(DISTINCT th.ID) AS OrderCount,
    COUNT(DISTINCT td.ID) AS LineItemCount,
    SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN TransDetail td
    ON th.ID = td.TransHeaderID AND td.IsActive = 1
INNER JOIN TransDetailParam tdp
    ON tdp.ParentID = td.ID
    AND tdp.VariableID = 11053  -- FP_ProductDescription
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND YEAR(th.SaleDate) = @Year
GROUP BY tdp.ValueAsStr25
ORDER BY Revenue DESC
```
**⚠️ Do NOT add IsActive or ParentClassTypeID filters to the TransDetailParam JOIN.**

### 4. Revenue by Specific SKU (FP_ProductID)
```sql
SELECT
    tdp_sku.ValueAsStr25 AS ProductSKU,
    tdp_cat.ValueAsStr25 AS ProductCategory,
    COUNT(DISTINCT th.ID) AS OrderCount,
    SUM(td.Quantity) AS TotalUnits,
    SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN TransDetail td
    ON th.ID = td.TransHeaderID AND td.IsActive = 1
INNER JOIN TransDetailParam tdp_sku
    ON tdp_sku.ParentID = td.ID
    AND tdp_sku.VariableID = 11052  -- FP_ProductID
LEFT JOIN TransDetailParam tdp_cat
    ON tdp_cat.ParentID = td.ID
    AND tdp_cat.VariableID = 11053  -- FP_ProductDescription
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND YEAR(th.SaleDate) = @Year
    AND tdp_sku.ValueAsStr25 LIKE '%' + @SKU + '%'
GROUP BY tdp_sku.ValueAsStr25, tdp_cat.ValueAsStr25
ORDER BY Revenue DESC
```

### 5. Customer Revenue for a Specific Product
```sql
SELECT
    a.CompanyName,
    td.Description,
    COUNT(DISTINCT th.ID) AS OrderCount,
    SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
INNER JOIN TransDetail td
    ON th.ID = td.TransHeaderID AND td.IsActive = 1
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND YEAR(th.SaleDate) = @Year
    AND a.CompanyName LIKE '%' + @CustomerName + '%'
GROUP BY a.CompanyName, td.Description
ORDER BY Revenue DESC
```

### 6. Top Customers by Revenue
```sql
SELECT TOP (@N)
    a.CompanyName,
    a.AccountNumber,
    COUNT(DISTINCT th.ID) AS OrderCount,
    SUM(th.SubTotalPrice) AS Revenue,
    MAX(th.SaleDate) AS LastSaleDate
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) = @Year
GROUP BY a.CompanyName, a.AccountNumber
ORDER BY Revenue DESC
```

### 7. Revenue by Salesperson
```sql
SELECT
    ISNULL(e.FirstName + ' ' + e.LastName, 'Unassigned') AS Salesperson,
    COUNT(DISTINCT th.ID) AS OrderCount,
    SUM(th.SubTotalPrice) AS Revenue
FROM TransHeader th
LEFT JOIN Employee e ON th.SalesPerson1ID = e.ID
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) = @Year
GROUP BY e.FirstName, e.LastName
ORDER BY Revenue DESC
```

### 8. Full Revenue by Product (Authoritative — matches Control reports)
```sql
SELECT
    CASE
        WHEN td.GoodsItemID = 10013
            THEN 'DyeSub: ' + ISNULL(tdp_fp.ValueAsStr25, '(uncategorized)')
        ELSE p.ItemName
    END AS Product,
    COUNT(DISTINCT td.ID) AS LineItems,
    SUM(td.Quantity) AS TotalQuantity,
    SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN TransDetail td
    ON th.ID = td.TransHeaderID AND td.IsActive = 1
INNER JOIN Product p
    ON td.GoodsItemID = p.ID AND p.ClassTypeID = td.GoodsItemClassTypeID
LEFT JOIN TransDetailParam tdp_fp
    ON tdp_fp.ParentID = td.ID
    AND tdp_fp.VariableID = 11053
    AND td.GoodsItemID = 10013
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND YEAR(th.SaleDate) = @Year
GROUP BY
    CASE
        WHEN td.GoodsItemID = 10013
            THEN 'DyeSub: ' + ISNULL(tdp_fp.ValueAsStr25, '(uncategorized)')
        ELSE p.ItemName
    END
ORDER BY Revenue DESC
```
**This replaces the old Description pattern-matching approach. Joins to Product via GoodsItemID for exact grouping. DyeSub Print (GoodsItemID=10013) drills into FP_ProductDescription categories. All other products use Product.ItemName directly.**

### 9. Year-over-Year Product Comparison (2 years)
```sql
SELECT
    CASE
        WHEN td.GoodsItemID = 10013
            THEN 'DyeSub: ' + ISNULL(tdp_fp.ValueAsStr25, '(uncategorized)')
        ELSE p.ItemName
    END AS Product,
    SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN td.SubTotalPrice END) AS [PriorYear_Revenue],
    SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN td.Quantity END) AS [PriorYear_Qty],
    SUM(CASE WHEN YEAR(th.SaleDate) = @Year THEN td.SubTotalPrice END) AS [CurrentYear_Revenue],
    SUM(CASE WHEN YEAR(th.SaleDate) = @Year THEN td.Quantity END) AS [CurrentYear_Qty],
    -- % Change column: (Current - Prior) / Prior
    CASE
        WHEN SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN td.SubTotalPrice END) IS NULL
            OR SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN td.SubTotalPrice END) = 0
            THEN NULL
        ELSE (SUM(CASE WHEN YEAR(th.SaleDate) = @Year THEN td.SubTotalPrice END)
            - SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN td.SubTotalPrice END))
            / SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN td.SubTotalPrice END)
    END AS [YoY_Change_Pct]
FROM TransHeader th
INNER JOIN TransDetail td
    ON th.ID = td.TransHeaderID AND td.IsActive = 1
INNER JOIN Product p
    ON td.GoodsItemID = p.ID AND p.ClassTypeID = td.GoodsItemClassTypeID
LEFT JOIN TransDetailParam tdp_fp
    ON tdp_fp.ParentID = td.ID
    AND tdp_fp.VariableID = 11053
    AND td.GoodsItemID = 10013
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) IN (@Year, @Year - 1)
GROUP BY
    CASE
        WHEN td.GoodsItemID = 10013
            THEN 'DyeSub: ' + ISNULL(tdp_fp.ValueAsStr25, '(uncategorized)')
        ELSE p.ItemName
    END
ORDER BY ISNULL(SUM(CASE WHEN YEAR(th.SaleDate) = @Year THEN td.SubTotalPrice END), 0)
       + ISNULL(SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN td.SubTotalPrice END), 0) DESC
```

**For 3+ year comparisons**, add additional CASE columns per year and a % change column between each consecutive pair. Example for 2023/2024/2025:
```
Product | 2023 $ | 2023 Qty | 23v24 % | 2024 $ | 2024 Qty | 24v25 % | 2025 $ | 2025 Qty
```
Each `% Change = (LaterYear - EarlierYear) / EarlierYear`. NULL when prior year is 0 or absent.

### 10. Year-over-Year Monthly Trend
```sql
SELECT
    MONTH(th.SaleDate) AS SaleMonth,
    SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN th.SubTotalPrice END) AS [PriorYear_Revenue],
    COUNT(DISTINCT CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN th.ID END) AS [PriorYear_Orders],
    SUM(CASE WHEN YEAR(th.SaleDate) = @Year THEN th.SubTotalPrice END) AS [CurrentYear_Revenue],
    COUNT(DISTINCT CASE WHEN YEAR(th.SaleDate) = @Year THEN th.ID END) AS [CurrentYear_Orders],
    CASE
        WHEN SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN th.SubTotalPrice END) IS NULL
            OR SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN th.SubTotalPrice END) = 0
            THEN NULL
        ELSE (SUM(CASE WHEN YEAR(th.SaleDate) = @Year THEN th.SubTotalPrice END)
            - SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN th.SubTotalPrice END))
            / SUM(CASE WHEN YEAR(th.SaleDate) = @Year - 1 THEN th.SubTotalPrice END)
    END AS [YoY_Change_Pct]
FROM TransHeader th
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) IN (@Year, @Year - 1)
GROUP BY MONTH(th.SaleDate)
ORDER BY SaleMonth
```

### 11. Monthly Income Statement (Closeout-Based — Matches Printed Report)

**⚠️ Use this template when the user asks about income statement figures, monthly P&L, or needs numbers that match the printed FLS income statement.** This uses GL Ledger entries bounded by monthly Closeout timestamps — NOT TransHeader.SaleDate.

```sql
-- Step 1: Get monthly closeout boundaries
-- Replace @Year/@Month with target period
DECLARE @Year INT = 2026, @Month INT = 1

DECLARE @PriorCloseout DATETIME, @TargetCloseout DATETIME

-- Prior month closeout (start boundary, exclusive)
SELECT TOP 1 @PriorCloseout = ModifiedDate
FROM Closeout
WHERE CloseoutType = 2  -- Monthly
  AND IsActive = 1
  AND MONTH(ModifiedDate) = CASE WHEN @Month = 1 THEN 12 ELSE @Month - 1 END
  AND YEAR(ModifiedDate) = CASE WHEN @Month = 1 THEN @Year - 1 ELSE @Year END
ORDER BY ModifiedDate DESC

-- Target month closeout (end boundary, inclusive)
SELECT TOP 1 @TargetCloseout = ModifiedDate
FROM Closeout
WHERE CloseoutType = 2  -- Monthly
  AND IsActive = 1
  AND MONTH(ModifiedDate) = @Month
  AND YEAR(ModifiedDate) = @Year
ORDER BY ModifiedDate DESC

-- Step 2: Income statement from Ledger
SELECT
    gl.AccountName,
    gl.PathName2 AS Category,
    SUM(l.Amount) * -1 AS Income
FROM Ledger l
JOIN GLAccount gl ON l.GLAccountID = gl.ID
WHERE l.EntryDateTime > @PriorCloseout
  AND l.EntryDateTime <= @TargetCloseout
  AND l.IsActive = 1
  AND (gl.PathName2 LIKE '%Product Sales%'
       OR gl.PathName2 LIKE '%Income/Sales%')
GROUP BY gl.AccountName, gl.PathName2
ORDER BY Income DESC
```

**Since the MCP readonly tool doesn't support DECLARE, use this single-query version:**
```sql
SELECT
    gl.AccountName,
    gl.PathName2 AS Category,
    SUM(l.Amount) * -1 AS Income
FROM Ledger l
JOIN GLAccount gl ON l.GLAccountID = gl.ID
WHERE l.EntryDateTime > (
        SELECT TOP 1 ModifiedDate FROM Closeout
        WHERE CloseoutType = 2 AND IsActive = 1
          AND MONTH(ModifiedDate) = CASE WHEN @Month = 1 THEN 12 ELSE @Month - 1 END
          AND YEAR(ModifiedDate) = CASE WHEN @Month = 1 THEN @Year - 1 ELSE @Year END
        ORDER BY ModifiedDate DESC
      )
  AND l.EntryDateTime <= (
        SELECT TOP 1 ModifiedDate FROM Closeout
        WHERE CloseoutType = 2 AND IsActive = 1
          AND MONTH(ModifiedDate) = @Month
          AND YEAR(ModifiedDate) = @Year
        ORDER BY ModifiedDate DESC
      )
  AND l.IsActive = 1
  AND (gl.PathName2 LIKE '%Product Sales%'
       OR gl.PathName2 LIKE '%Income/Sales%')
GROUP BY gl.AccountName, gl.PathName2
ORDER BY Income DESC
```

**Key rules:**
- Revenue in Ledger is stored as **negative** (credit entries). Multiply by **-1** for display.
- Use `>` for the start boundary (exclusive) and `<=` for the end boundary (inclusive) — this matches how Control defines the period.
- Filter income accounts via `gl.PathName2 LIKE '%Product Sales%' OR gl.PathName2 LIKE '%Income/Sales%'` — this captures all revenue categories including Accessories & Displays.
- **Validated:** January 2026 = $147,836.16 (exact match to printed income statement).
- For Closeout table details and timing patterns, see `control-erp-core` skill → "Closeout / Period Boundaries" section.

**GL Account → Income Statement Category Mapping:**

| GL PathName2 | Income Statement Categories |
|-------------|---------------------------|
| `Revenue/Product Sales/Product Sales - Digital` | DyeSub Print, Table Covers, Flags, SEG, etc. |
| `Revenue/Product Sales/Garments` | Embroidery, Screenprint |
| `Revenue/Product Sales` | Shipping, Services, Promotion |
| `Income/Sales` | Artwork-Design |
| `Income/Sales/Accessories and Displays` | Banner Stand, Carry Bags |

---

## OUTPUT FORMATTING RULES

**These rules apply to ALL multi-year reports — sales, income statements, GL, or any financial comparison.**

**Year-over-year comparison format (mandatory for multi-year):**
- One row per product/entity with each year's data side by side
- Never use separate tables per year or long-format (year as a row dimension)
- Include a **% Change column** between each consecutive pair of years
- Formula: `% Change = (LaterYear - EarlierYear) / EarlierYear`
- Display as percentage (e.g., +25.6%, -3.2%). NULL/blank when prior year is 0 or absent.

**Column layout examples:**
- 2 years: `Product | 2024 $ | 2024 Qty | 2025 $ | 2025 Qty | % Change`
- 3 years: `Product | 2023 $ | 2023 Qty | 23v24 % | 2024 $ | 2024 Qty | 24v25 % | 2025 $ | 2025 Qty`

**General rules:**
- For CSV export: same comparison layout with empty cells for items absent in a year
- Include a TOTAL row at the bottom
- Sort by revenue descending (most recent year, or sum across all years)

---

## NATURAL LANGUAGE INTERPRETATION

| User Says | Action |
|-----------|--------|
| "What were our sales in [period]?" | Template 1 or 2, header-level SubTotalPrice |
| "Sales by product" / "product breakdown" | Template 3 (category via FP_ProductDescription) |
| "Sales by product category" | Template 3 |
| "[Category name] sales" (e.g., "Feather Flag sales") | Template 3 filtered by category |
| "How many [SKU] did we sell?" | Template 4 with SKU filter |
| "Sales by SKU" / "product detail" | Template 4 grouped by FP_ProductID |
| "[Customer] sales" / "what did [customer] buy?" | Template 5 |
| "Top customers" | Template 6 |
| "Sales by rep" / "salesperson performance" | Template 7 |
| "Full product breakdown" / "where does our revenue come from?" | Template 8 (Product join + DyeSub drill-down) |
| "Compare this year to last year" | Template 9 |
| "Table cover sales" | Filter by GoodsItemID IN (10026, 10010, 4200, 4209, 4210, 4213) |
| "Income statement" / "monthly P&L" / "match the income statement" | Template 11 (Closeout-based Ledger query) |
| "Open AR" / "unpaid invoices" / "outstanding balances" | Type 1, StatusID = 3 (Sales) WHERE BalanceDue > 0 |
| "Paid orders" / "closed orders" | Type 1, StatusID = 4 (Closed) WHERE BalanceDue = 0 |
| "Orders in production" / "WIP" | Type 1, StatusID = 1 (WIP) |
| "Web orders" / "website orders" | Type 1 with Import_Order_Number UDF populated |

**Ambiguity resolution:** If the user asks for "sales by product" without specifying category vs SKU level, default to category (FP_ProductDescription) and mention that SKU-level detail is available if needed.

---

## IMPORTANT CAVEATS

1. **Header vs Detail gap:** SUM(TransDetail.SubTotalPrice) is ~$7K less than SUM(TransHeader.SubTotalPrice) for the same period due to header-level adjustments. Always note which level you're reporting from.

2. **DyeSub Print = 58.7% of revenue:** DyeSub Print categories accounted for ~$1.79M of $3.05M in 2025. This is the core of the business. Swing Flags alone = $615K (20% of total).

3. **Table covers without TC_ drill-down:** Since TC_ variables don't persist, table cover analysis is limited to Description-based grouping. You can't drill into fabric type or specific TC SKUs from transaction data.

4. **Estimates (Type 2):** 71,200 total since 2003. 56% convert to Type 1 orders (linked via EstimateNumber). Type 2 SubTotalPrice = quoted value, NOT sold value. Never include in revenue. Useful for pipeline analysis and conversion rate reporting.

5. **Web-imported orders:** $150,750 / 429 orders in 2025 (4.9% of revenue) after deduplication. These are Type 1 orders imported via CHAPI, identified by `Import_Order_Number` UDF. Must deduplicate — cloned orders share the same Import_Order_Number; only the first (lowest OrderNumber) is a true web import. Average ~$351/order — smaller B2C profile.

6. **Product join methodology:** Template 8 now joins TransDetail to Product via `GoodsItemID` / `GoodsItemClassTypeID = 12000`, matching Control's own "Sales by Product" report methodology. This eliminated the old Description pattern-matching discrepancies (DyeLux was undercounted by $57K, Garments by $163K). The "Other Products/Services" catch-all is eliminated — every dollar maps to a named Product.

---

*All query patterns validated against FLS Banners' 2024-2025 financial data. Template 8 matches Control's "Sales by Product" report methodology exactly. Template 11 validated against January 2026 income statement ($147,836.16 exact match).*
