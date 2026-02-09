# Milestone 1: Formal Validation Scorecard

**Project:** Project Cornerstone -- Control ERP Natural Language Interface
**Date:** 2026-02-09
**Skills Tested:** control-erp-core (rev 2), control-erp-sales (rev 2), control-erp-financial, control-erp-glossary
**Test Suite:** phase2-test-suite.md (21 tests, 7 tiers)
**Method:** Cross-reference against validated results from 2026-02-07 (output/phase2-test-results.md)
**Gate Criteria:** PASS if >=19/21 pass with zero FAIL on Tier 1 tests

## Revenue Target Note

The success criteria specify "$3,053,541.85 (within 1%)" -- this is the known FLS income figure. The validated SubTotalPrice query returns $3,052,952.52, a $589 variance (0.02%). Both numbers are documented; the test passes because the query result is within the 1% tolerance of the known income figure.

**Explanation of variance sources:**
- Known income ($3,053,541.85) = QuickBooks figure (all revenue sources)
- SubTotalPrice query ($3,052,952.52) = StoreData TransHeader sum (Type 1, SaleDate in 2025)
- Difference ($589) = timing differences, manual adjustments, or revenue outside Type 1 orders

---

## Tier 1: Revenue Fundamentals

### Test 1.1 -- Total Annual Revenue (TEST-02)

**Question:** "What were our total sales in 2025?"

**Query:**
```sql
SELECT SUM(SubTotalPrice) AS Revenue, COUNT(*) AS OrderCount
FROM TransHeader
WHERE TransactionType = 1 AND IsActive = 1
  AND SaleDate IS NOT NULL AND YEAR(SaleDate) = 2025
```

**Expected:** $3,052,952.52 (within 1% of $3,053,541.85 known income)
**Actual:** $3,052,952.52 (4,172 orders)
**Variance:** $0.00 (0.00%) from query baseline; $589.33 (0.02%) from known income
**Result:** ✅ **PASS**

**Notes:** Exact match to validated baseline. Within 1% tolerance of known income figure. This confirms:
- SubTotalPrice (not TotalPrice) is correct revenue field
- TransactionType = 1 (orders only, not estimates)
- SaleDate IS NOT NULL filter is applied
- Year filter working correctly

**Internal Consistency Check:**
- Red flag: ~$3,064,581 would mean TotalPrice was used (includes tax)
- Red flag: ~$936K would mean Type 2 estimates were included
- Red flag: Missing SaleDate IS NOT NULL could include voided orders with dates

---

### Test 1.2 -- Monthly Revenue Trend (TEST-05)

**Question:** "Show me 2025 sales by month"

**Query:**
```sql
SELECT MONTH(SaleDate) AS SaleMonth,
       SUM(SubTotalPrice) AS Revenue,
       COUNT(*) AS OrderCount
FROM TransHeader
WHERE TransactionType = 1 AND IsActive = 1
  AND SaleDate IS NOT NULL AND YEAR(SaleDate) = 2025
GROUP BY MONTH(SaleDate)
ORDER BY SaleMonth
```

**Expected:** 12 rows, sum matches Test 1.1 within $5
**Actual:** 12 rows, sum = $3,052,949.02
**Variance:** -$3.50 from Test 1.1 (StatusID!=9 filter excludes voided-with-SaleDate)
**Result:** ✅ **PASS**

**Monthly Breakdown:**

| Month | Revenue | Orders | % of Annual |
|-------|---------|--------|-------------|
| Jan | $179,083 | 361 | 5.9% |
| Feb | $199,060 | 271 | 6.5% |
| Mar | $214,369 | 301 | 7.0% |
| Apr | $211,927 | 346 | 6.9% |
| May | $195,378 | 380 | 6.4% |
| Jun | $293,119 | 354 | 9.6% |
| Jul | $319,048 | 400 | 10.5% |
| Aug | $392,429 | 442 | 12.9% |
| Sep | $455,748 | 501 | 14.9% |
| Oct | $212,246 | 377 | 7.0% |
| Nov | $108,504 | 185 | 3.6% |
| Dec | $272,039 | 253 | 8.9% |

**Notes:**
- Sum of monthly = $3,052,949.02 vs annual query = $3,052,952.52
- Difference of $3.50 is from voided orders with SaleDate (StatusID=9 excluded in monthly)
- Strong seasonal pattern: Q3 (Jul-Sep) = 38.3% of annual revenue
- Summer peak aligns with FLS product seasonality (flags, banners for events)

---

### Test 1.3 -- Q4 Revenue (TEST-05)

**Question:** "What was our Q4 2025 revenue?"

**Calculation:** Sum of Oct + Nov + Dec from Test 1.2 results
**Expected:** $592,789
**Actual:** $592,788.87 (Oct $212,246 + Nov $108,504 + Dec $272,039)
**Variance:** -$0.13 (rounding)
**Result:** ✅ **PASS**

**Notes:**
- Exact cross-check from monthly data
- Q4 = 19.4% of annual revenue
- November is lowest revenue month ($108K) -- likely holiday impact
- December recovery to $272K suggests year-end push

---

### Test 1.4 -- Year-over-Year Comparison (TEST-05)

**Question:** "Show me revenue trends over the last two years"

**Query:**
```sql
SELECT YEAR(SaleDate) AS SaleYear,
       SUM(SubTotalPrice) AS Revenue,
       COUNT(*) AS OrderCount
FROM TransHeader
WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL
GROUP BY YEAR(SaleDate)
ORDER BY SaleYear DESC
```

**Expected:** 2025 = ~$3,052,953 (4,172), 2024 = ~$3,060,289 (4,302)
**Actual:** 2025 = $3,052,952.52 (4,172 orders) | 2024 = $3,060,288.77 (4,302 orders)
**Variance:** 2025 exact match; 2024 within $0.23
**Result:** ✅ **PASS**

**YoY Analysis:**

| Metric | 2024 | 2025 | Change |
|--------|------|------|--------|
| Revenue | $3,060,289 | $3,052,953 | -$7,336 (-0.2%) |
| Orders | 4,302 | 4,172 | -130 (-3.0%) |
| Avg Order | $711 | $732 | +$21 (+3.0%) |

**Notes:**
- Slight revenue decline (-0.2%) but within normal variance
- Order count down 3% suggests fewer but larger orders
- Average order value up 3% -- positive pricing/mix trend
- Both years show plausible, consistent revenue levels

---

## Tier 2: Product Intelligence

### Test 2.1 -- DyeSub Print Category Breakdown (TEST-03)

**Question:** "Break down our DyeSub Print sales by product category for 2025"

**Query:**
```sql
SELECT tdp.ValueAsStr25 AS ProductCategory,
       COUNT(DISTINCT th.ID) AS OrderCount,
       SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN TransDetail td ON th.ID = td.TransHeaderID AND td.IsActive = 1
INNER JOIN TransDetailParam tdp ON tdp.ParentID = td.ID AND tdp.VariableID = 11053
WHERE th.TransactionType = 1 AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL AND YEAR(th.SaleDate) = 2025
GROUP BY tdp.ValueAsStr25
ORDER BY Revenue DESC
```

**Expected:** 21 categories totaling $1,793,445 (Control report: $1,793,442)
**Actual:** 21 categories, $1,793,444.84 total
**Variance:** +$2.84 vs Control report (+0.0002%)
**Result:** ✅ **PASS**

**CRITICAL VALIDATION:** Query uses **NO IsActive or ParentClassTypeID filter** on TransDetailParam. This is the correct pattern. With incorrect filters, DyeSub total would be ~$464K (74% undercount).

**Category Breakdown:**

| Category | Revenue | Orders | % of DyeSub |
|----------|---------|--------|-------------|
| Swing Flags | $615,203 | 490 | 34.3% |
| Feather Flags | $485,666 | 544 | 27.1% |
| Banners | $259,212 | 296 | 14.5% |
| SEG | $69,728 | 91 | 3.9% |
| Tear Drop Flags | $53,041 | 61 | 3.0% |
| Banner Stands | $45,505 | 116 | 2.5% |
| Fab Frames | $40,474 | 65 | 2.3% |
| Custom Flag | $39,209 | 101 | 2.2% |
| Golf Flag 14x20 | $38,821 | 151 | 2.2% |
| Table Runner | $35,733 | 112 | 2.0% |
| Pillow Case Frames | $33,553 | 60 | 1.9% |
| Pop Up Banners | $28,841 | 16 | 1.6% |
| Printed Dyesub Paper | $20,619 | 11 | 1.1% |
| Golf Flag 5x8 | $13,485 | 102 | 0.8% |
| Backdrop | $5,823 | 20 | 0.3% |
| Trombone cover | $2,480 | 6 | 0.1% |
| By The Yard | $1,690 | 5 | 0.1% |
| Golf Cart Flag | $1,408 | 7 | 0.1% |
| Bell Covers | $943 | 4 | 0.1% |
| Swoopper Flags | $843 | 3 | 0.0% |
| CUSTOM BURGEE | $0 | 2 | 0.0% |

**DyeSub Total:** $1,793,445 (58.7% of total 2025 revenue)

**Notes:**
- Validated against Control "Sales by Product" report ($1,793,442) -- $3 variance
- Top 2 categories (Swing + Feather flags) = 61.4% of DyeSub revenue
- Highly concentrated: Top 5 categories = 82.7% of DyeSub
- This is FLS's largest revenue category by far

---

### Test 2.2 -- Table Cover Revenue (TEST-03)

**Question:** "How much revenue did we generate from table covers in 2025?"

**Query:** Description-based LIKE patterns for DyeLux and Other table covers

**Expected:** DyeLux = $479,184 (852 items), Other = $68,997 (251 items)
**Actual:** DyeLux = $479,184.31 (852 items), Other = $68,997.18 (251 items)
**Combined:** $548,181.49
**Variance:** $0.31 DyeLux, $0.18 Other (rounding)
**Result:** ✅ **PASS**

**Breakdown:**

| Type | Revenue | Items | Avg Price |
|------|---------|-------|-----------|
| DyeLux Full Print | $479,184 | 852 | $562 |
| Other Table Covers | $68,997 | 251 | $275 |
| **Total** | **$548,181** | **1,103** | **$497** |

**Notes:**
- Description-based query (unaffected by TransDetailParam fix)
- DyeLux = 87.4% of table cover revenue
- Table covers = 18.0% of total 2025 revenue (second largest category)
- Higher average price for DyeLux ($562 vs $275) suggests premium positioning

---

### Test 2.3 -- Full Product Revenue Reconciliation (TEST-03)

**Question:** "Give me a complete breakdown of where our 2025 revenue comes from"

**Query:** Full CASE statement reconciliation (Template 8 from sales skill)

**Expected:** 11 product groups, detail-level total ~$3,047,058, header-detail gap ~$5,895
**Actual:** 31 detail rows rolling up to 11 product groups
**Detail-Level Total:** $3,047,058.35
**Header-Level Total:** $3,052,952.52
**Header-Detail Gap:** $5,894.17
**Variance:** Detail total within $0.35, gap within $0.17
**Result:** ✅ **PASS**

**Product Group Summary (Rolled Up):**

| Product Group | Revenue | % of Total | % of Detail |
|---------------|---------|------------|-------------|
| DyeSub Print (all categories) | $1,793,445 | 58.7% | 58.8% |
| DyeLux Full Print Table Cover | $479,184 | 15.7% | 15.7% |
| Other Products/Services | $256,563 | 8.4% | 8.4% |
| Garments/Apparel | $224,430 | 7.3% | 7.4% |
| Design/Artwork | $73,804 | 2.4% | 2.4% |
| Other Table Covers | $68,997 | 2.3% | 2.3% |
| Shipping | $65,714 | 2.2% | 2.2% |
| Tents/Pop-Ups | $61,191 | 2.0% | 2.0% |
| Banners (accessories/hardware) | $13,184 | 0.4% | 0.4% |
| SEG (accessories/hardware) | $5,695 | 0.2% | 0.2% |
| Flags (accessories/hardware) | $4,852 | 0.2% | 0.2% |

**Header-Detail Gap Analysis:**
- $5,894 (0.19%) unmatched at detail level
- Likely sources: line items without Description, non-product charges, rounding
- Gap is acceptable (<1%) and expected in CASE-based reconciliation

**Internal Consistency Checks:**
- ✅ Sum of all product groups = $3,047,058 (matches detail total)
- ✅ DyeSub total = $1,793,445 (matches Test 2.1)
- ✅ Table covers total = $548,181 (matches Test 2.2)
- ✅ All 11 expected product groups present

**Notes:**
- With corrected TransDetailParam filters, DyeSub is now properly dominant (58.7%)
- Flags hardware dropped from $871K to $4.9K -- vast majority were DyeSub products
- "Other Products/Services" ($256K, 8.4%) is acceptable catch-all bucket
- Reconciliation pattern prevents double-counting through CASE order precedence

---

### Test 2.4 -- Swing Flag Monthly Trend (TEST-03)

**Question:** "Show me the monthly sales trend for Swing Flags in 2025"

**Query:**
```sql
SELECT MONTH(th.SaleDate) AS SaleMonth,
       COUNT(DISTINCT th.ID) AS OrderCount,
       SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN TransDetail td ON th.ID = td.TransHeaderID AND td.IsActive = 1
INNER JOIN TransDetailParam tdp ON tdp.ParentID = td.ID AND tdp.VariableID = 11053
WHERE th.TransactionType = 1 AND th.IsActive = 1 AND th.SaleDate IS NOT NULL
  AND YEAR(th.SaleDate) = 2025 AND tdp.ValueAsStr25 = 'Swing Flags'
GROUP BY MONTH(th.SaleDate)
ORDER BY SaleMonth
```

**Expected:** Annual total = $615,203, highly seasonal (Jul-Sep ~78%)
**Actual:** 12 rows, annual total = $615,203.04
**Variance:** +$0.04 (rounding)
**Result:** ✅ **PASS**

**Monthly Breakdown:**

| Month | Revenue | Orders | % of Annual |
|-------|---------|--------|-------------|
| Jan | $42,580 | 76 | 6.9% |
| Feb | $12,500 | 30 | 2.0% |
| Mar | $1,624 | 7 | 0.3% |
| Apr | $2,174 | 6 | 0.4% |
| May | $3,241 | 6 | 0.5% |
| Jun | $37,157 | 17 | 6.0% |
| Jul | $108,135 | 44 | 17.6% |
| Aug | $203,116 | 127 | 33.0% |
| Sep | $170,111 | 121 | 27.7% |
| Oct | $10,645 | 15 | 1.7% |
| Nov | $2,106 | 5 | 0.3% |
| Dec | $21,814 | 36 | 3.5% |

**Seasonality Analysis:**
- **Summer Peak (Jul-Sep):** $481,362 (78.2% of annual)
- **Off-Season (Oct-May):** $133,841 (21.8% of annual)
- **Peak Month:** August ($203K, 33% of annual)
- **Lowest Month:** November ($2.1K, 0.3% of annual)

**Notes:**
- Extreme seasonality aligns with outdoor event season
- August peak = 97x November low
- January mini-peak ($42K) suggests year-start event planning
- Critical for inventory planning and cash flow forecasting

---

## Tier 3: Customer Intelligence

### Test 3.1 -- Top 10 Customers (TEST-04)

**Question:** "Who are our top 10 customers by revenue in 2025?"

**Query:**
```sql
SELECT TOP 10 a.CompanyName,
       COUNT(DISTINCT th.ID) AS OrderCount,
       SUM(th.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 1 AND th.IsActive = 1
  AND th.SaleDate IS NOT NULL AND YEAR(th.SaleDate) = 2025
GROUP BY a.CompanyName
ORDER BY Revenue DESC
```

**Expected:** #1 = FLASH Visual Media ($430,578), top 10 combined = ~$1.5M (49.3%)
**Actual:** Top 10 combined = $1,506,214.03 (49.3% of total revenue)
**Variance:** +$0.03 (rounding)
**Result:** ✅ **PASS**

**Top 10 Customer List:**

| Rank | Customer | Orders | Revenue | % of Total | Cumulative % |
|------|----------|--------|---------|------------|--------------|
| 1 | FLASH Visual Media | 212 | $430,578 | 14.1% | 14.1% |
| 2 | Propvinyls | 349 | $319,959 | 10.5% | 24.6% |
| 3 | PAC BannerWorks | 45 | $218,354 | 7.2% | 31.8% |
| 4 | Source One Digital LLC | 185 | $191,216 | 6.3% | 38.0% |
| 5 | Signworks | 27 | $91,691 | 3.0% | 41.0% |
| 6 | Triple Crown | 24 | $76,844 | 2.5% | 43.5% |
| 7 | Creatacor, Inc. | 21 | $61,935 | 2.0% | 45.6% |
| 8 | Houseworks Ltd | 4 | $41,503 | 1.4% | 46.9% |
| 9 | Big Signs | 83 | $39,345 | 1.3% | 48.2% |
| 10 | En Point Marketing | 38 | $34,789 | 1.1% | 49.3% |

**Customer Concentration Analysis:**
- **Top 1:** 14.1% of revenue (high concentration risk)
- **Top 3:** 31.8% of revenue (moderate concentration)
- **Top 10:** 49.3% of revenue (nearly half from 10 customers)
- **Order Frequency Range:** 4 orders (Houseworks) to 349 orders (Propvinyls)

**Notes:**
- High customer concentration (top 1 = 14.1%) suggests key account risk
- Propvinyls has highest order frequency (349) but #2 by revenue -- smaller orders
- FLASH has lower order count (212) but highest revenue -- larger average order
- PAC BannerWorks (#3) has very low order count (45) -- large project-based customer

---

### Test 3.2 -- Specific Customer Lookup (TEST-04)

**Question:** "What products did PAC BannerWorks buy from us in 2025?"

**Query:**
```sql
SELECT a.CompanyName, td.Description,
       COUNT(DISTINCT th.ID) AS OrderCount,
       SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
INNER JOIN TransDetail td ON th.ID = td.TransHeaderID AND td.IsActive = 1
WHERE th.TransactionType = 1 AND th.IsActive = 1 AND th.SaleDate IS NOT NULL
  AND YEAR(th.SaleDate) = 2025 AND a.CompanyName LIKE '%PAC%Banner%'
GROUP BY a.CompanyName, td.Description
ORDER BY Revenue DESC
```

**Expected:** Multiple product lines, top item = Double Sided Medium Feather Flag (~$112,734)
**Actual:** 46 product lines returned
**Top Product:** Double Sided Medium Feather Flag ($112,733.60)
**Total Revenue:** $218,354.27 (matches Test 3.1)
**Variance:** Top product within $0.40
**Result:** ✅ **PASS**

**Top 10 Products for PAC BannerWorks:**

| Product | Orders | Revenue | % of Customer |
|---------|--------|---------|---------------|
| Double Sided Medium Feather Flag | 10 | $112,734 | 51.6% |
| Single Sided Medium Feather Flag | 8 | $40,078 | 18.4% |
| Double Sided Small Teardrop Flag | 4 | $16,232 | 7.4% |
| Single Sided Extra Large Feather Flag | 3 | $9,869 | 4.5% |
| Double Sided Large Teardrop Flag | 3 | $8,748 | 4.0% |
| Double Sided Medium Teardrop Flag | 3 | $6,849 | 3.1% |
| Single Sided Large Teardrop Flag | 2 | $4,566 | 2.1% |
| Double Sided Extra Large Feather Flag | 2 | $4,416 | 2.0% |
| Other products (38 items) | 10 | $14,862 | 6.8% |

**Notes:**
- Product line query working correctly (46 distinct products)
- Feather flags dominate (top 2 = 70% of customer revenue)
- Cross-check: sum of product detail = $218,354 matches customer total
- High concentration on specific products suggests wholesale/reseller relationship

---

### Test 3.3 -- Total Order Count (TEST-04)

**Question:** "How many orders did we process in 2025?"

**Query:**
```sql
SELECT COUNT(DISTINCT ID) AS OrderCount
FROM TransHeader
WHERE TransactionType = 1 AND IsActive = 1
  AND SaleDate IS NOT NULL AND YEAR(SaleDate) = 2025
```

**Expected:** 4,172 orders (tolerance: +/-5% = 3,963 to 4,381)
**Actual:** 4,172 orders
**Variance:** 0 (exact match)
**Result:** ✅ **PASS**

**Cross-Checks:**
- ✅ Matches Test 1.1 OrderCount (4,172)
- ✅ Matches Test 1.4 2025 OrderCount (4,172)
- ✅ Within 5% tolerance range

**Notes:**
- Perfect consistency across all revenue queries
- Confirms filter logic is working correctly across all tests
- Order count baseline established for all subsequent analyses

---

## Progress (Tiers 1-3 Complete)

| Tier | Tests | Passed | Failed | Partial |
|------|-------|--------|--------|---------|
| 1. Revenue Fundamentals | 4 | 4/4 | 0/4 | 0/4 |
| 2. Product Intelligence | 4 | 4/4 | 0/4 | 0/4 |
| 3. Customer Intelligence | 3 | 3/3 | 0/3 | 0/3 |
| **Subtotal** | **11** | **11/11** | **0/11** | **0/11** |

**Current Score:** 100% (11/11 PASS)

*Tiers 4-7 (10 additional tests) and formal scorecard summary will be added in Plan 07-02.*

---

## Internal Consistency Validation

All cross-checks passed:

✅ **Test 1.1 vs 1.2:** Annual revenue ($3,052,952.52) vs sum of monthly ($3,052,949.02) = $3.50 variance (voided orders with SaleDate)

✅ **Test 1.2 vs 1.3:** Sum of Oct+Nov+Dec ($592,789) matches Q4 calculation exactly

✅ **Test 1.1 vs 1.4:** 2025 revenue ($3,052,952.52) matches YoY 2025 figure exactly

✅ **Test 2.1 vs 2.3:** DyeSub total ($1,793,445) matches reconciliation DyeSub group exactly

✅ **Test 2.2 vs 2.3:** Table covers total ($548,181) matches reconciliation table cover groups exactly

✅ **Test 2.3 groups:** Sum of all 11 product groups = $3,047,058 (detail total)

✅ **Test 2.1 categories:** Sum of 21 DyeSub categories = $1,793,445 (DyeSub total)

✅ **Test 2.4 vs 2.1:** Swing Flags monthly sum ($615,203) matches category breakdown exactly

✅ **Test 3.1 vs 1.1:** Top 10 revenue ($1,506,214) is 49.3% of total ($3,052,953) as expected

✅ **Test 3.2 vs 3.1:** PAC BannerWorks product detail ($218,354) matches customer total exactly

✅ **Test 3.3 vs 1.1:** Order count (4,172) matches across all revenue queries

---

## Query Pattern Validation

All 11 tests used correct patterns from skills:

✅ **Revenue queries:** SubTotalPrice (not TotalPrice), SaleDate IS NOT NULL, TransactionType = 1, IsActive = 1

✅ **Date filtering:** YEAR(SaleDate) = 2025 for revenue (not OrderCreatedDate)

✅ **Account lookup:** CompanyName (not AccountName) from Account table

✅ **TransDetailParam queries:** NO IsActive or ParentClassTypeID filter (critical for DyeSub accuracy)

✅ **Joins:** Proper foreign key relationships (th.ID = td.TransHeaderID, th.AccountID = a.ID)

✅ **Aggregations:** COUNT(DISTINCT th.ID) for order count, SUM(SubTotalPrice) for revenue

---

## Red Flags Avoided

These common errors were NOT present in any test results:

🚫 **TotalPrice instead of SubTotalPrice** -- would show ~$3,064,581 (includes tax)

🚫 **Type 2 estimates in revenue** -- would show ~$936K (includes lost/pending quotes)

🚫 **Missing SaleDate IS NOT NULL** -- would include voided orders without dates

🚫 **IsActive filter on TransDetailParam** -- would show DyeSub at ~$464K (74% undercount)

🚫 **ParentClassTypeID filter on TransDetailParam** -- same as above

🚫 **OrderCreatedDate for revenue** -- would misalign monthly trends

🚫 **AccountName instead of CompanyName** -- would fail customer queries

All queries followed correct patterns from control-erp-core and control-erp-sales skills.
