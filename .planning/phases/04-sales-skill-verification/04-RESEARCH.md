# Phase 4: Sales Skill Verification - Research

**Researched:** 2026-02-08
**Domain:** Sales skill verification and validation against Control ERP reports
**Confidence:** HIGH

## Summary

Phase 4 is a **verification and validation** phase, not a build phase. The `control-erp-sales-SKILL.md` already exists with 368 lines of documented product architecture, 9 query templates, natural language mappings, and revenue figures validated to 99.98% accuracy. A comprehensive test suite (`phase2-test-suite.md`) was already executed with a **21/21 perfect score** (`output/phase2-test-results.md`). The work here is to formally verify that the skill meets the 6 SALES requirements (SALES-01 through SALES-06), identify any gaps or undocumented edge cases, and produce a structured verification report.

The key risk is **false confidence** -- the test results look perfect, but the validation was done in the same session that built the skill. This phase should independently verify the claims, check for product groups that might be missing from the reconciliation, and confirm that the "Other Products/Services" bucket ($256K) does not contain misclassified products that should have their own categories.

**Primary recommendation:** Structure the phase as two plans: (1) audit the skill's product identification patterns for completeness and accuracy, and (2) validate all query templates against live data with expected values documented for each, producing a formal verification report tied to SALES-01 through SALES-06.

## Standard Stack

This phase involves no libraries or code to install. The "stack" is:

### Core
| Artifact | Location | Purpose | Status |
|----------|----------|---------|--------|
| control-erp-sales-SKILL.md | `skills/control-erp-sales/` | Sales skill being verified | EXISTS (368 lines) |
| control-erp-core-SKILL.md | `skills/control-erp-core/` | Dependency -- business rules | VERIFIED (Phase 1) |
| phase2-test-suite.md | project root | 21-test validation suite | EXISTS (already executed) |
| phase2-test-results.md | `output/` | Test results (21/21 PASS) | EXISTS (Rev 2) |
| control-erp-validation-results.md | `validation/` | Original validation data | EXISTS |

### Ground Truth Sources
| Source | Location | What It Provides |
|--------|----------|-----------------|
| Control "Sales by Product" report | Run from Control UI | Product-level revenue totals (170-page PDF) |
| MSSQL MCP server | Live database | Direct query validation |
| Known 2025 income | $3,053,541.85 | Revenue baseline target |

### Key Known Values (Ground Truth)
| Metric | Value | Source | Confidence |
|--------|-------|--------|------------|
| Total 2025 revenue | $3,052,952.52 (query) / $3,053,541.85 (known) | Validated both ways | HIGH |
| DyeSub Print total | $1,793,445 | Validated against Control report ($1,793,442, $3 variance) | HIGH |
| DyeLux Table Cover | $479,184 | Query result; Control report says $537,043 (different grouping) | MEDIUM |
| Garments/Apparel | $224,430 | Query result; Control report says $261,379 (different grouping) | MEDIUM |
| Total order count (2025) | 4,172 | Exact match | HIGH |
| Web imports (deduplicated) | 429 orders, $150,750 | Exact match | HIGH |

## Architecture Patterns

### Pattern 1: Verification Report Structure
**What:** Each SALES requirement gets its own verification section with: requirement text, what was checked, query used, expected vs actual result, PASS/FAIL determination, and any gaps found.
**When to use:** For all 6 SALES requirements.
**Example structure:**
```markdown
### SALES-01: DyeSub Print Container Variable Architecture
**Requirement:** control-erp-sales skill complete with DyeSub Print container variable architecture
**Verified by:** Reading skill file + running query template 3
**Expected:** 21 categories, total $1,793,445, VariableID 11053 for FP_ProductDescription
**Actual:** [run query, document result]
**Status:** PASS/FAIL
**Gaps:** [any missing documentation]
```

### Pattern 2: Two-Level Verification
**What:** Check both the documentation (does the skill say the right things?) AND the queries (do they produce correct results when run?).
**When to use:** Every SALES requirement needs both levels.
**Why:** The skill could document the right architecture but have a query bug, or could have working queries but missing documentation.

**Level 1 -- Documentation audit:**
- Does the skill file document all required patterns?
- Are the VariableIDs correct (11053, 11052)?
- Are the product groups complete?
- Are the caveats accurate?

**Level 2 -- Query validation:**
- Run each query template against live data
- Compare results to known values
- Check internal consistency (do monthly totals sum to annual?)

### Pattern 3: Revenue Reconciliation Waterfall
**What:** The full product breakdown should account for 100% of detail-level revenue with no unexplained residual.
**When to use:** SALES-04 validation.
**Structure:**
```
Header-level total: $3,052,953
  - DyeSub Print (container): $1,793,445 (58.7%)
  - DyeLux Table Cover: $479,184 (15.7%)
  - Garments/Apparel: $224,430 (7.3%)
  - Other Table Covers: $68,997 (2.3%)
  - Design/Artwork: $73,804 (2.4%)
  - Shipping: $65,714 (2.2%)
  - Tents/Pop-Ups: $61,191 (2.0%)
  - Banners hardware: $13,184 (0.4%)
  - SEG hardware: $5,695 (0.2%)
  - Flags hardware: $4,852 (0.2%)
  - Other Products/Services: $256,563 (8.4%)
  = Detail-level total: $3,047,058
  + Header-level adjustment: ~$5,895
  = Header-level total
```

### Anti-Patterns to Avoid
- **Treating the existing 21/21 score as sufficient:** The test suite was executed in the same session that built the skill. Independent re-verification is needed.
- **Skipping the "Other" bucket analysis:** $256K (8.4%) in "Other Products/Services" could contain product groups that should be categorized. At minimum, sample the largest items.
- **Not checking the DyeLux/Garment discrepancy:** The Control report shows $537K for DyeLux and $261K for Garments, but our queries show $479K and $224K respectively. This ~$90K combined discrepancy needs explanation (different grouping logic -- our Description-based matching vs Control's product entity grouping).
- **Running queries without documenting expected values first:** Write down what you expect BEFORE running the query. This prevents confirmation bias.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Product revenue validation | New ad-hoc queries from scratch | The 9 existing query templates in the skill | They are already validated. Re-run them, don't reinvent. |
| Test suite structure | A new test framework | The existing `phase2-test-suite.md` format | 21 tests already defined with expected values, tolerances, and red flags. |
| Revenue formula | Any alternative to the canonical formula | `SUM(SubTotalPrice) WHERE TransactionType=1 AND IsActive=1 AND SaleDate IS NOT NULL` | Validated to 99.98%. Do not deviate. |
| TransDetailParam query | Adding IsActive or ParentClassTypeID filters | No filters on TransDetailParam (just VariableID) | With filters: $464K (WRONG). Without: $1,793K (CORRECT). This is the single most critical gotcha in the entire project. |

**Key insight:** This phase re-verifies existing work, it does not create new queries or patterns. The skill templates are the subject of verification, not tools to be built.

## Common Pitfalls

### Pitfall 1: TransDetailParam Filter Trap (CRITICAL)
**What goes wrong:** Adding `IsActive = 1` or `ParentClassTypeID = 10100` to TransDetailParam queries silently drops 75% of valid records.
**Why it happens:** These filters seem logical -- "of course we want active records for the right parent type." But Control sets IsActive=false on TransDetailParam after product configuration, and the ParentClassTypeID filter is unnecessary since VariableID is sufficient.
**How to avoid:** The sales skill already documents this correctly. Verification should confirm the skill's query templates do NOT have these filters. Check every TransDetailParam JOIN in the skill file.
**Warning signs:** DyeSub Print revenue showing ~$464K instead of ~$1,793K. Flags hardware showing ~$871K instead of ~$5K.
**Verification query:** Check that Query Template 3 in the skill produces $1,793,445 for DyeSub total.

### Pitfall 2: Control Report vs Query Result Discrepancy
**What goes wrong:** The Control "Sales by Product" report groups products by Product entity (GoodsItemID), while our reconciliation query uses a hybrid of TransDetailParam variables (for DyeSub) and Description pattern matching (for everything else). These groupings do not align perfectly.
**Why it happens:** Control knows which Product entity each line item belongs to. Our queries match by text patterns and variable values, which can classify differently at the margins.
**How to avoid:** Document the known discrepancies explicitly. DyeSub Print matches to within $3 (because it uses the same variable architecture that Control uses internally). Other products may show larger variances.
**Known discrepancies:**
- DyeLux Table Cover: query $479K vs report $537K (Control may include items we classify as "Other Table Covers")
- Decorated Garments: query $224K vs report $261K (Control's product entity may capture items our Description patterns miss)
- Grand total: query $3,052,953 vs report $3,053,479 ($526 variance, negligible)

### Pitfall 3: Header vs Detail Revenue Gap
**What goes wrong:** Summing TransDetail.SubTotalPrice gives ~$7K less than TransHeader.SubTotalPrice for the same period.
**Why it happens:** Header-level adjustments (discounts, rounding, pricing overrides) are not distributed to line items.
**How to avoid:** Always use TransHeader.SubTotalPrice for total revenue. Use TransDetail.SubTotalPrice only for product-level breakdowns, and acknowledge the gap.
**The skill already documents this correctly in the "Important Caveats" section.**

### Pitfall 4: Seasonal Skew in Validation
**What goes wrong:** Running a monthly query at a specific point in the year and concluding the pattern is wrong because one month looks unusual.
**Why it happens:** FLS Banners is highly seasonal. Swing Flags: Jul-Sep = 78% of annual revenue. Nov is typically the lowest month.
**How to avoid:** Validate full-year totals first, then monthly patterns. Accept that seasonal variation is expected and documented.

### Pitfall 5: "Other Products/Services" as a Black Box
**What goes wrong:** $256K (8.4% of revenue) is classified as "Other Products/Services" in the reconciliation query. This is a catch-all for line items that don't match any Description pattern and don't have FP_ variables.
**Why it happens:** The CASE statement in Template 8 catches everything that doesn't match specific patterns.
**How to avoid:** The verification should sample the "Other" bucket to understand what is in it. Are there product groups that should be broken out? Are there misclassified items?
**Verification query:** Run Template 8 and examine the "Other" rows, sorting by SubTotalPrice DESC to see the largest items.

### Pitfall 6: MCP Availability
**What goes wrong:** The MSSQL MCP server may not be available when the executor runs this phase (Mac environment vs Windows SQL Server).
**Why it happens:** MCP tools require the server connection to be active.
**How to avoid:** Design the plan so that: (1) documentation audit tasks can be done without MCP, (2) query validation tasks clearly specify the query to run and expected result, (3) if MCP is unavailable, the plan documents what needs to be verified and the expected values, allowing verification to be completed when MCP becomes available.

## Code Examples

These are the key queries that must be validated. They already exist in the skill file; they are reproduced here for the planner's reference with the expected results.

### DyeSub Print Category Revenue (Template 3 from skill)
```sql
-- Source: control-erp-sales-SKILL.md, Query Template 3
-- Expected: 21 categories, total $1,793,445
-- CRITICAL: No IsActive or ParentClassTypeID filter on TransDetailParam
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
    AND YEAR(th.SaleDate) = 2025
GROUP BY tdp.ValueAsStr25
ORDER BY Revenue DESC
```

### Full Revenue Reconciliation (Template 8 from skill)
```sql
-- Source: control-erp-sales-SKILL.md, Query Template 8
-- Expected: All product groups totaling ~$3,047,058 (detail-level)
-- The full CASE statement is in the skill file (lines 229-308)
```

### Top Customers (Template 6 from skill)
```sql
-- Source: control-erp-sales-SKILL.md, Query Template 6
-- Expected top 3: FLASH Visual Media ($430K), Propvinyls ($320K), PAC BannerWorks ($218K)
SELECT TOP 10
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
    AND YEAR(th.SaleDate) = 2025
GROUP BY a.CompanyName, a.AccountNumber
ORDER BY Revenue DESC
```

### Monthly Revenue Trend (Template 2 from skill)
```sql
-- Source: control-erp-sales-SKILL.md, Query Template 2
-- Expected: 12 rows, sum = $3,052,949
-- Peak month: September ($455,748), Trough: November ($108,504)
SELECT
    MONTH(th.SaleDate) AS SaleMonth,
    SUM(th.SubTotalPrice) AS Revenue,
    COUNT(*) AS OrderCount
FROM TransHeader th
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) = 2025
GROUP BY MONTH(th.SaleDate)
ORDER BY SaleMonth
```

## State of the Art

This is the evolution of the sales skill through the project:

| Stage | DyeSub Print Revenue | Key Issue | Status |
|-------|---------------------|-----------|--------|
| Original validation (v1) | $464,342 | IsActive + ParentClassTypeID filters on TransDetailParam | WRONG -- fixed |
| Corrected validation (v2) | $1,793,445 | Filters removed, validated against Control report | CURRENT |
| Control report value | $1,793,442 | Official "Sales by Product" report | GROUND TRUTH |
| Variance | $3 | Negligible | VERIFIED |

| Stage | Overall Revenue | Status |
|-------|----------------|--------|
| Query result | $3,052,952.52 | CURRENT |
| Known income | $3,053,541.85 | GROUND TRUTH |
| Variance | $589.33 (0.02%) | VERIFIED |

**Key evolution:** The most significant correction was removing the IsActive and ParentClassTypeID filters from TransDetailParam queries. This changed DyeSub Print from 15% to 58.7% of total revenue -- a complete shift in business understanding.

## Verification Approach

### Plan 1: Product Architecture and Identification Pattern Audit

This plan audits the skill documentation (not the queries) for completeness:

**SALES-01 checklist:**
- [ ] DyeSub Print container architecture documented
- [ ] FP_ProductDescription VariableID 11053 documented
- [ ] FP_ProductID VariableID 11052 documented
- [ ] All 21 known categories listed with revenue
- [ ] ValueAsStr25 column specified
- [ ] No IsActive/ParentClassTypeID filter warning present
- [ ] Working query example provided

**SALES-02 checklist:**
- [ ] Table Cover identification via Description pattern matching documented
- [ ] GoodsItemID=10026 alternative documented
- [ ] TC_ variable limitation explained (not saved to TransDetailParam)
- [ ] DyeLux vs Other Table Cover distinction documented

**SALES-03 checklist:**
- [ ] Non-container product groups listed (Garments, Design, Shipping, Tents, etc.)
- [ ] Description LIKE patterns documented for each
- [ ] Revenue figures provided
- [ ] "Other Products/Services" bucket explained

**Additional audit:**
- [ ] Check for any product groups in the "Other" bucket that should be broken out
- [ ] Verify Description patterns do not overlap (could cause double-counting)
- [ ] Check the DyeLux/Garment discrepancy vs Control report is documented

### Plan 2: Query Validation and Revenue Verification

This plan runs the actual queries and validates results:

**SALES-04 (Product Revenue):**
- Run Template 3 (DyeSub categories) -- expect $1,793,445 total
- Run Template 8 (full reconciliation) -- expect detail total ~$3,047,058
- Cross-check against Control "Sales by Product" report values
- Document any discrepancies with explanations

**SALES-05 (Customer Revenue):**
- Run Template 6 (top 10 customers) -- expect FLASH Visual Media at #1 ($430K)
- Run Template 5 (specific customer) -- expect PAC BannerWorks results
- Verify customer count and revenue totals are consistent

**SALES-06 (Sales Trends):**
- Run Template 2 (monthly trend) -- expect 12 rows, sum = $3,052,949
- Run Template 1 (annual) -- expect $3,052,952.52
- Run Template 9 (YoY) -- expect 2024 and 2025 rows
- Verify: monthly sum = annual total (within rounding)
- Verify: Q4 = Oct + Nov + Dec

**Internal consistency checks:**
- Monthly totals sum to annual total
- DyeSub categories sum to DyeSub total
- All product groups sum to detail-level total
- Header-detail gap is ~$5,895 (documented)

## Open Questions

1. **What is in the "Other Products/Services" bucket ($256K)?**
   - What we know: 2,560 line items totaling $256K that don't match any Description pattern and don't have FP_ variables.
   - What's unclear: Whether any of these should be broken out into named categories.
   - Recommendation: The verification should sample the top items in this bucket. If any single product type represents >$25K, consider adding it as a named category.

2. **DyeLux Table Cover discrepancy ($479K vs $537K)**
   - What we know: Our Description-based query returns $479K. Control's report shows $537K.
   - What's unclear: Whether Control includes items under its "DyeLux" product entity that our `LIKE '%Dyelux%Table Cover%'` pattern misses. The $58K gap could be table covers with different Description text that still reference the DyeLux product.
   - Recommendation: Run a query filtering by GoodsItemID=10026 instead of Description pattern to see if it captures more. Document the difference.

3. **Are there new products in 2025 not covered by existing patterns?**
   - What we know: The skill was built with 2025 data, so it should capture 2025 products.
   - What's unclear: Whether any products were added late in 2025 that might not be in the validated category list.
   - Recommendation: Check if any FP_ProductDescription values exist that are not in the documented 21 categories.

4. **MCP availability for query execution**
   - What we know: The planner is on Mac; MSSQL MCP requires Windows connection.
   - What's unclear: Whether MCP will be available when the executor runs.
   - Recommendation: Design Plan 2 so queries and expected values are fully documented. If MCP is unavailable, the plan can be executed later. Plan 1 (documentation audit) can run without MCP.

## Sources

### Primary (HIGH confidence)
- `skills/control-erp-sales/control-erp-sales-SKILL.md` -- read in full (368 lines, 9 query templates, 21 product categories)
- `skills/control-erp-core/control-erp-core-SKILL.md` -- read in full (347 lines, revenue formula, business rules)
- `output/phase2-test-results.md` -- read in full (21/21 PASS, Rev 2 with corrected TransDetailParam filters)
- `phase2-test-suite.md` -- read in full (21 tests with expected values, tolerances, and red flags)
- `validation/control-erp-validation-results.md` -- read in full (original validation data, pre-filter-fix values)
- `.planning/REQUIREMENTS.md` -- read in full (SALES-01 through SALES-06 requirements)
- `.planning/ROADMAP.md` -- read in full (Phase 4 success criteria)
- `.planning/phases/01-core-skill-verification/01-VERIFICATION.md` -- read in full (Phase 1 verification approach)
- `.planning/phases/01-core-skill-verification/01-01-PLAN.md` -- read in full (Phase 1 plan structure)
- `.planning/phases/01-core-skill-verification/01-RESEARCH.md` -- read in full (Phase 1 research approach)

### Secondary (MEDIUM confidence)
- `output/reports/Sales___By_Product.md` -- inferred report structure (Crystal Reports cannot be parsed)
- `output/reports/Sales___By_Product_Category.md` -- inferred report structure
- `output/report_join_patterns.md` -- inferred join patterns from Crystal Reports analysis
- `output/report_summary.md` -- 36 Crystal Reports cataloged

### Notes on Confidence
- All revenue figures cited are from direct database queries, validated against known income ($3,053,541.85)
- DyeSub Print total ($1,793,445) was validated against the Control "Sales by Product" report ($1,793,442)
- Crystal Report structures are inferred (reports are binary OLE format); the actual SQL used by Control reports is not available
- The discrepancy between our query groupings and Control's report groupings for non-DyeSub products is a known, documented issue

## Metadata

**Confidence breakdown:**
- Sales skill completeness: HIGH -- the skill was thoroughly built and tested (21/21 PASS)
- Revenue validation data: HIGH -- validated to 99.98% against known income
- DyeSub Print accuracy: HIGH -- $3 variance against Control report
- Non-DyeSub grouping accuracy: MEDIUM -- Description-based patterns may not match Control's product entity grouping exactly
- "Other" bucket composition: LOW -- not yet analyzed in detail
- MCP availability: UNKNOWN -- depends on runtime environment

**Research date:** 2026-02-08
**Valid until:** 30 days (stable domain -- product architecture unlikely to change)
