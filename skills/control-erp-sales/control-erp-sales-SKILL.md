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

### Container 2: DyeLux-Full Print Table Cover ($479K in 2025)

**Important:** TC_ variables (TC_FabricCategory, TC_ProductName) are NOT saved to TransDetailParam. They exist only in the product configurator UI.

**To identify Table Cover products, use Description pattern matching:**
```sql
-- DyeLux Full Print Table Covers
WHERE td.Description LIKE '%Dyelux%Table Cover%'
   OR td.Description LIKE '%FULL%Table Cover%'

-- All table covers (broader)
WHERE td.Description LIKE '%Table Cover%'

-- Alternative: by GoodsItemID
WHERE td.GoodsItemID = 10026  -- Main Table Cover product container
```

### Non-Container Products

Not everything flows through DyeSub Print or DyeLux. These product groups are identified by TransDetail.Description pattern matching:

| Pattern | Product Group | 2025 Revenue |
|---------|-------------|-------------|
| `%Garment%` or `%Embroidered%` | Apparel | $224,430 |
| `%Design%` or `%Artwork%` | Design services | $73,804 |
| `%Table Cover%` (non-DyeLux) | Other Table Covers | $68,997 |
| `%Shipment%` or `%Shipping%` or `%Freight%` | Shipping charges | $65,714 |
| `%Tent%` or `%Pop%Up%` | Tents/Pop-Ups | $61,191 |
| `%Banner%` (non-DyeSub) | Banner hardware/accessories | $13,184 |
| `%SEG%` (non-DyeSub) | SEG hardware/accessories | $5,695 |
| `%Flag%` (non-DyeSub) | Flags hardware/accessories | $4,852 |
| Everything else | Other products/services | $256,563 |

**Note:** With corrected TransDetailParam queries (no IsActive/ParentClassTypeID filters), DyeSub Print properly captures most flag, banner, and SEG line items. The hardware/accessories groups are now small — they represent standalone hardware sales not attached to a DyeSub Print order.

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

### 8. Full Revenue Reconciliation (All Product Groups)
```sql
SELECT
    CASE
        WHEN tdp_fp.ID IS NOT NULL
            THEN 'DyeSub Print (' + tdp_fp.ValueAsStr25 + ')'
        WHEN td.Description LIKE '%Dyelux%Table Cover%'
            OR td.Description LIKE '%FULL%Table Cover%'
            THEN 'DyeLux Full Print Table Cover'
        WHEN td.Description LIKE '%Table Cover%'
            OR td.Description LIKE '%Table Throw%'
            THEN 'Other Table Covers'
        WHEN td.Description LIKE '%Table Runner%'
            THEN 'Table Runners'
        WHEN td.Description LIKE '%Garment%'
            OR td.Description LIKE '%Embroidered%'
            THEN 'Garments/Apparel'
        WHEN td.Description LIKE '%Tent%'
            OR td.Description LIKE '%Pop%Up%'
            OR td.Description LIKE '%Popup%'
            THEN 'Tents/Pop-Ups'
        WHEN td.Description LIKE '%Shipment%'
            OR td.Description LIKE '%Shipping%'
            OR td.Description LIKE '%Freight%'
            THEN 'Shipping'
        WHEN td.Description LIKE '%Flag%'
            THEN 'Flags (accessories/hardware)'
        WHEN td.Description LIKE '%Banner%'
            THEN 'Banners (accessories/hardware)'
        WHEN td.Description LIKE '%SEG%'
            THEN 'SEG (accessories/hardware)'
        WHEN td.Description LIKE '%Design%'
            OR td.Description LIKE '%Artwork%'
            THEN 'Design/Artwork'
        ELSE 'Other Products/Services'
    END AS ProductGroup,
    COUNT(DISTINCT td.ID) AS LineItems,
    SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN TransDetail td
    ON th.ID = td.TransHeaderID AND td.IsActive = 1
LEFT JOIN TransDetailParam tdp_fp
    ON tdp_fp.ParentID = td.ID
    AND tdp_fp.VariableID = 11053
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND YEAR(th.SaleDate) = @Year
GROUP BY
    CASE
        WHEN tdp_fp.ID IS NOT NULL
            THEN 'DyeSub Print (' + tdp_fp.ValueAsStr25 + ')'
        WHEN td.Description LIKE '%Dyelux%Table Cover%'
            OR td.Description LIKE '%FULL%Table Cover%'
            THEN 'DyeLux Full Print Table Cover'
        WHEN td.Description LIKE '%Table Cover%'
            OR td.Description LIKE '%Table Throw%'
            THEN 'Other Table Covers'
        WHEN td.Description LIKE '%Table Runner%'
            THEN 'Table Runners'
        WHEN td.Description LIKE '%Garment%'
            OR td.Description LIKE '%Embroidered%'
            THEN 'Garments/Apparel'
        WHEN td.Description LIKE '%Tent%'
            OR td.Description LIKE '%Pop%Up%'
            OR td.Description LIKE '%Popup%'
            THEN 'Tents/Pop-Ups'
        WHEN td.Description LIKE '%Shipment%'
            OR td.Description LIKE '%Shipping%'
            OR td.Description LIKE '%Freight%'
            THEN 'Shipping'
        WHEN td.Description LIKE '%Flag%'
            THEN 'Flags (accessories/hardware)'
        WHEN td.Description LIKE '%Banner%'
            THEN 'Banners (accessories/hardware)'
        WHEN td.Description LIKE '%SEG%'
            THEN 'SEG (accessories/hardware)'
        WHEN td.Description LIKE '%Design%'
            OR td.Description LIKE '%Artwork%'
            THEN 'Design/Artwork'
        ELSE 'Other Products/Services'
    END
ORDER BY Revenue DESC
```

### 9. Year-over-Year Comparison
```sql
SELECT
    YEAR(th.SaleDate) AS SaleYear,
    MONTH(th.SaleDate) AS SaleMonth,
    SUM(th.SubTotalPrice) AS Revenue,
    COUNT(DISTINCT th.ID) AS OrderCount
FROM TransHeader th
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) IN (@Year, @Year - 1)
GROUP BY YEAR(th.SaleDate), MONTH(th.SaleDate)
ORDER BY SaleMonth, SaleYear
```

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
| "Full product breakdown" / "where does our revenue come from?" | Template 8 (reconciliation) |
| "Compare this year to last year" | Template 9 |
| "Table cover sales" | Template 8 filtered to table cover groups, or Description LIKE query |
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

6. **Control Report vs Query Grouping Discrepancy:** Our reconciliation query (Template 8) uses Description pattern matching for non-DyeSub products, while Control's "Sales by Product" report groups by Product entity (GoodsItemID). This causes known differences: DyeLux ($479K query vs $537K report), Garments ($224K vs $261K). DyeSub Print matches precisely ($3 variance) because both methods use the same FP_ variable architecture. Grand total variance is $526 (0.02%).

---

*All query patterns validated against FLS Banners' 2025 financial data. Revenue formula proven to 99.98% accuracy.*
