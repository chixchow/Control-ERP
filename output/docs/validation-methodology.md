# Validation Methodology (DOC-06)

This document describes the validation methodology used in Milestone 1. It enables future milestones to reproduce the same verification process for new skills and query templates.

The methodology was developed and proven during Milestone 1 with a 21-test scorecard achieving 21/21 PASS against FLS Banners' actual 2025 financial data. Revenue accuracy reached 99.98% of the known income figure ($3,053,541.85 from QuickBooks).

---

## Methodology Steps

1. **Establish Known Baseline**

   Get a known-correct number from an authoritative source external to the skill being tested. This is the ground truth that query results are measured against.

   Examples from Milestone 1:
   - FLS 2025 income = $3,053,541.85 (from QuickBooks)
   - Control "Sales by Product" DyeSub Print = $1,793,442 (from built-in Control report)
   - Web orders = 429 deduplicated (from prior validated CSV export)

   The baseline must come from a source the skill did not influence. If no external baseline exists, use an internally consistent cross-check (e.g., monthly totals must sum to annual total).

2. **Execute MCP Queries**

   Run SQL against StoreData via MCP mssql tools. Capture raw results with timestamps in a results file (e.g., `output/phase2-test-results.md`).

   Key rules:
   - Load the skill being tested before writing queries
   - Use exact skill templates (do not improvise SQL)
   - Record the full query text, result set, and execution timestamp
   - If MCP is unavailable, cross-reference against prior validated results and note the method

3. **Cross-Reference Validation**

   Compare query results against the known baseline. Calculate variance as both absolute value and percentage.

   When live MCP is unavailable (e.g., Mac environment without Windows MCP access), cross-reference against prior validated results from a session where MCP was available. Document the method as "cross-reference validation" and cite the original results file.

4. **Internal Consistency Checks**

   Verify that related queries agree with each other. These catch errors even when no external baseline exists.

   Standard consistency checks:
   - Monthly totals sum to annual total (tolerance: within $5)
   - Product categories sum to total revenue (tolerance: within 1% header-detail gap)
   - Top 10 customer sum is less than or equal to total revenue
   - Detail-level and header-level totals are within acceptable gap (~$7K / 0.2%)
   - Year-over-year figures are plausible (not identical, not wildly different)
   - Order counts are consistent across all queries using the same filters

5. **Gotcha Validation**

   For each known gotcha documented in `output/docs/known-gotchas.md`, demonstrate that:
   - The wrong approach produces incorrect results (quantify the error)
   - The correct approach (from skills) produces correct results
   - The skill file explicitly warns against the wrong approach (cite file and line)

   This proves the skills actively prevent known errors, not just document correct patterns.

6. **Gate Evaluation**

   Apply pass/fail gate criteria to determine whether the milestone passes.

   Milestone 1 gate criteria:
   - At least 19/21 tests PASS
   - Zero FAIL on Tier 1 tests (revenue fundamentals)
   - All known gotchas validated with skill citations

   Future milestones should define gate criteria before test execution (not after).

---

## Tolerance Definitions

| Metric Type | Tolerance | Example |
|-------------|-----------|---------|
| Revenue totals | +/-1% of known baseline | $3,052,952.52 vs $3,053,541.85 (0.02% variance) |
| Record counts | Exact match or documented reason for variance | 4,172 orders (exact across all queries) |
| Customer rankings | Same top 10 members, +/-1 position swap | FLASH Visual Media #1 ($430,578) |
| Product percentages | +/-2% of category share | DyeSub 58.7% +/- 1.2% |
| Internal consistency | Exact (sum of parts = total), or documented reason | 12 monthly totals = annual within $5 |
| Cross-query agreement | Same metric from different queries within 0.1% | Order count 4,172 in Tests 1.1, 1.4, 3.3 |

When a variance exceeds tolerance, document the reason (e.g., "$3.50 variance in monthly sum due to voided orders with SaleDate") and assess whether it indicates a real error or an expected data characteristic.

---

## Test Tier Structure

The 21-test suite is organized into 7 tiers of increasing specificity. Lower tiers must pass before higher tiers are meaningful.

| Tier | Name | Test Count | What It Validates |
|------|------|-----------|-------------------|
| 1 | Revenue Fundamentals | 4 | Core revenue formula accuracy (annual, monthly, quarterly, YoY) |
| 2 | Product Intelligence | 4 | Product identification patterns (DyeSub categories, table covers, reconciliation, trends) |
| 3 | Customer Intelligence | 3 | Customer queries (top 10, specific lookup, order count) |
| 4 | Estimate & Pipeline | 4 | Estimate lifecycle (conversion rate, pending, pipeline value, lost quotes) |
| 5 | Web Import Intelligence | 2 | Web order identification and deduplication |
| 6 | Salesperson Performance | 1 | Revenue by salesperson with cross-check to total |
| 7 | Behavioral Assessment | 3 | Skill documentation quality (ambiguity handling, trap prevention, field guidance) |

Reference: `validation/milestone1-scorecard.md` for full scorecard with all 21 test results.
Reference: `phase2-test-suite.md` for test definitions.

---

## Reproducing a Test

Step-by-step example using Test 1.1 (Annual Revenue):

1. **Load skill:** Load `skills/control-erp-core/control-erp-core-SKILL.md`

2. **Identify template:** Revenue Query Formula (lines 14-29 of the core skill). This provides the canonical query structure including SubTotalPrice, TransactionType = 1, SaleDate IS NOT NULL, and IsActive = 1.

3. **Execute query via MCP:** Use the mssql MCP tools to run the query against StoreData. Record the raw result and timestamp.

4. **Compare result to baseline:** Query returned $3,052,952.52 (4,172 orders). Known income from QuickBooks is $3,053,541.85.

5. **Calculate variance:** $589.33 difference = 0.02%. Tolerance is +/-1%. Result is within tolerance.

6. **Check internal consistency:** Verify this result against Test 1.2 (monthly sum = $3,052,949.02, $3.50 variance explained by voided orders with SaleDate) and Test 1.4 (YoY 2025 figure = $3,052,952.52, exact match).

7. **Result:** PASS

To reproduce any other test, follow the same six steps using the test definition from `phase2-test-suite.md` and the corresponding skill template.

---

## Key Artifacts

| Artifact | Location | Purpose |
|----------|----------|---------|
| Test suite definition | `phase2-test-suite.md` | 21 tests in 7 tiers with expected results |
| Scorecard | `validation/milestone1-scorecard.md` | Executed results (21/21 PASS) with full detail |
| Raw MCP results | `output/phase2-test-results.md` | Query outputs from 2026-02-07 live session |
| Revenue validation | `output/skill/references/business_rules_validation.md` | Revenue formula proof and validation |
| Known gotchas | `output/docs/known-gotchas.md` | 13 gotchas with wrong/correct/impact/citation |
| Skill architecture | `output/docs/skill-architecture.md` | Dependency graph and skill interaction model |

---

## Applying This Methodology to Future Milestones

For Milestone 2 and beyond:

1. **Define tests before building.** Write the test suite (expected results, baselines, tolerances) before implementing new skills. This prevents retrofitting tests to match results.

2. **Establish new baselines.** Each new domain (AR aging, inventory levels, payroll) needs its own authoritative baseline from Control reports or QuickBooks.

3. **Extend gotcha validation.** As new gotchas are discovered during skill development, add them to `output/docs/known-gotchas.md` and include gotcha validation tests in the next milestone scorecard.

4. **Maintain internal consistency.** Every new query should cross-check against at least one existing validated result. For example, an AR aging query should sum to the AR balance from the GL query.

5. **Set gate criteria.** Define the pass/fail threshold before execution. Milestone 1 used 19/21 minimum with zero Tier 1 failures. Future milestones may adjust based on test count and risk profile.

---

*Methodology developed during Milestone 1 execution (Phases 7-8). Scorecard: 21/21 PASS. Gate: PASS (100%).*
