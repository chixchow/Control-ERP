# Phase 7: Test Suite Execution - Research

**Researched:** 2026-02-08
**Domain:** ERP skill validation / test execution / scorecard generation
**Confidence:** HIGH

## Summary

Phase 7 formalizes the execution of a 21-test suite against the Control ERP StoreData database, producing a milestone scorecard at `validation/milestone1-scorecard.md`. The critical finding from this research is that **all 21 tests have already been executed and passed** in `output/phase2-test-results.md` (Rev 2, 2026-02-07) -- Phase 7's job is to re-execute them in a formal, documented framework that satisfies the six TEST requirements and produces the milestone gate artifact.

The test suite is already fully defined in `phase2-test-suite.md` with 7 tiers (Revenue Fundamentals, Product Intelligence, Customer Intelligence, Estimate/Pipeline, Web Import, Salesperson, Edge Cases), exact expected values, tolerances, and red-flag indicators. The skills under test are `control-erp-core`, `control-erp-sales`, `control-erp-financial`, and `control-erp-glossary`. All validation and skill fixes from prior phases (TransDetailParam filter fix, EstimateCreatedDate fix, etc.) are already incorporated.

**Primary recommendation:** Re-execute all 21 tests using the existing test suite definition, capture results in the formal scorecard format at `validation/milestone1-scorecard.md`, and document the gotcha validations explicitly to satisfy TEST-06. The two plans (07-01 and 07-02) map cleanly to Tiers 1-3 + Tiers 4-7 with scorecard compilation.

## Standard Stack

This phase does not introduce new libraries or tools. It uses existing infrastructure.

### Core
| Tool | Version | Purpose | Why Standard |
|------|---------|---------|--------------|
| MS SQL MCP Server | Current | Execute SQL queries against StoreData | Already configured and used across all prior phases |
| Markdown | N/A | Scorecard and results documentation | Consistent with all project documentation |

### Supporting
| Artifact | Location | Purpose | When to Use |
|----------|----------|---------|-------------|
| Test suite definition | `phase2-test-suite.md` | 21 test cases with expected values | Defines what to test |
| Prior test results | `output/phase2-test-results.md` | Rev 2 results (21/21 PASS) | Baseline comparison |
| Core skill | `skills/control-erp-core/control-erp-core-SKILL.md` | Business rules reference | Must be loaded before any query |
| Sales skill | `skills/control-erp-sales/control-erp-sales-SKILL.md` | Product/revenue patterns | Must be loaded for product queries |
| Financial skill | `skills/control-erp-financial/control-erp-financial-SKILL.md` | GL/payment reference | Referenced for completeness |
| Glossary skill | `skills/control-erp-glossary/control-erp-glossary-SKILL.md` | Terminology routing | Referenced for completeness |

### Alternatives Considered
| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Re-running live queries | Cross-reference existing results | Cross-reference was used in Phase 4 due to Mac MCP constraint; live execution is preferred for Phase 7 if MCP is available |
| Manual scorecard | Automated test runner | Not worth building -- 21 tests is manageable manually and tests require human judgment (Tier 7 edge cases) |

## Architecture Patterns

### Recommended Scorecard Structure
```
validation/
└── milestone1-scorecard.md    # Final output -- formal pass/fail scorecard
```

### Pattern 1: Test-Suite-Driven Validation
**What:** Execute each test case from the test suite definition, compare results to expected values, document pass/fail with evidence.
**When to use:** Every test in Tiers 1-6 (quantitative tests).
**Approach:**
1. Read the test definition from `phase2-test-suite.md`
2. Construct the SQL query following the skill's documented patterns
3. Execute against StoreData via MCP
4. Compare result to expected value within stated tolerance
5. Record PASS/FAIL with actual value and any variance

### Pattern 2: Skill-Behavior Validation
**What:** Assess whether skills provide correct guidance, not just correct numbers -- tests where the "answer" is a behavior, not a number.
**When to use:** Tier 7 edge cases (Tests 7.1, 7.2, 7.3).
**Approach:**
1. Read the test scenario
2. Verify the skill documentation would lead to correct behavior
3. Document which skill section provides the guidance
4. Record PASS if skill correctly handles the ambiguity/trap

### Pattern 3: Gotcha Validation (TEST-06)
**What:** Explicitly verify that known gotchas are correctly handled by documenting what WRONG result would occur if the gotcha were not avoided.
**When to use:** Dedicated section of scorecard for TEST-06.
**Approach:**
For each gotcha:
1. Document the trap (what a naive query would do)
2. Document the wrong result it would produce
3. Document the correct approach (from skill)
4. Show the correct result
5. Calculate the impact of the gotcha (magnitude of error)

Known gotchas to validate:
- TransDetailParam IsActive filter: Wrong = $464K DyeSub, Correct = $1,793K (+286%)
- SubTotalPrice vs TotalPrice: Wrong = $3,064,581, Correct = $3,052,953 (~$11K over)
- SaleDate vs OrderCreatedDate: Wrong = misses records where dates differ
- EstimateCreatedDate for Type 2: Wrong = NULL results (OrderCreatedDate is always NULL on Type 2)
- SaleDate IS NOT NULL filter: Required to exclude incomplete Type 1 records

### Anti-Patterns to Avoid
- **Re-running tests without loading skills first:** The point of testing is to validate that the skills lead to correct queries. Each test should simulate "reading the skill, then writing the query."
- **Rounding tolerance abuse:** Don't loosen tolerances to force PASS. If a test is within 1% of expected, it passes. If it's 3% off, investigate.
- **Ignoring the validation results v1:** The older validation results in `validation/control-erp-validation-results.md` show WRONG values (DyeSub = $464K). These are useful as "before fix" evidence but must NOT be treated as current expected values.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Test case definitions | New test suite | `phase2-test-suite.md` (existing) | 21 tests already defined with expected values, tolerances, and red flags |
| Expected revenue values | Re-derive from scratch | Known validated values from Phase 4 verification | $3,052,952.52 (SubTotalPrice), $1,793,445 (DyeSub), 4,172 orders -- all validated |
| Query templates | Write queries from memory | Templates documented in `control-erp-sales-SKILL.md` | 9 templates already verified in Phase 4 |
| Scorecard format | Custom format | Template from `phase2-test-suite.md` bottom section | Already has the table structure, totals row, and gate criteria |

**Key insight:** Phase 7 is execution and documentation, not design. Every test case, expected value, query pattern, and scorecard template already exists. The value is in formally running it and producing the artifact.

## Common Pitfalls

### Pitfall 1: Using Stale Validation Results as "Current"
**What goes wrong:** The older `validation/control-erp-validation-results.md` shows DyeSub Print = $464K (WRONG, pre-filter-fix). If this is mistakenly used as a reference, tests would show false failures.
**Why it happens:** Two validation result files exist with similar names.
**How to avoid:** Always reference `output/phase2-test-results.md` (Rev 2, 2026-02-07) as the baseline. The `validation/control-erp-validation-results.md` file is historical and contains pre-fix (incorrect) data.
**Warning signs:** Any DyeSub number near $464K or Swing Flags near $172K indicates the old/wrong filter is being applied.

### Pitfall 2: TransactionType 2 Confusion
**What goes wrong:** The older validation results call Type 2 "Web/Online Order" -- this was a misidentification. Type 2 is actually "Estimate/Quote" (the core skill was corrected in Phase 1).
**Why it happens:** `validation/control-erp-validation-results.md` was generated before skill corrections.
**How to avoid:** The current `control-erp-core-SKILL.md` has the correct mapping (Type 2 = Estimate/Quote). Use the skill file, not old validation results.
**Warning signs:** If Type 2 is being described as "Web Order" anywhere in the scorecard output.

### Pitfall 3: MCP Availability on Mac
**What goes wrong:** The Mac development environment may not be able to connect to the Windows MCP server for live SQL execution.
**Why it happens:** MCP MSSQL server runs on the Windows machine; Mac environment has limited connectivity.
**How to avoid:** Two paths:
  1. **Live execution (preferred):** If MCP is available, run all 21 tests live.
  2. **Cross-reference fallback:** If MCP is unavailable, cross-reference existing test results from `output/phase2-test-results.md` (same approach used in Phase 4), noting in the scorecard that results are from the 2026-02-07 execution, and recommend live re-validation from Windows when convenient.
**Warning signs:** MCP connection errors at the start of execution.

### Pitfall 4: Revenue Baseline Mismatch
**What goes wrong:** The success criteria say "Total sales 2025 returns $3,053,541.85 (within 1%)" but the validated SubTotalPrice query returns $3,052,952.52. The $589 variance is known and documented.
**Why it happens:** $3,053,541.85 is the "known actual income" from FLS's own books. $3,052,952.52 is what the SubTotalPrice query returns. The difference ($589.33, 0.02%) is within tolerance.
**How to avoid:** The scorecard should note both numbers and explain the variance. The test PASSES because $3,052,952.52 is within 1% of $3,053,541.85.
**Warning signs:** If someone expects exact match to $3,053,541.85 and calls it a failure.

### Pitfall 5: Tier 7 Tests Require Judgment
**What goes wrong:** Tests 7.1-7.3 are behavioral/judgment tests, not numeric. Trying to run them as SQL queries misses the point.
**Why it happens:** Natural tendency to automate everything.
**How to avoid:** For each Tier 7 test, evaluate the SKILL documentation and document which section provides the correct guidance. These are "does the skill lead a user to the right answer?" tests, not "does the query return the right number?" tests.

## Code Examples

These are the key SQL patterns that will be executed during testing. All are from the validated skills.

### Revenue Baseline (TEST-02)
```sql
-- Source: control-erp-core-SKILL.md, lines 16-23
SELECT SUM(SubTotalPrice) AS Revenue
FROM TransHeader
WHERE TransactionType = 1
    AND IsActive = 1
    AND SaleDate IS NOT NULL
    AND YEAR(SaleDate) = 2025
-- Expected: $3,052,952.52 (within 1% of $3,053,541.85)
```

### DyeSub Product Breakdown (TEST-03)
```sql
-- Source: control-erp-sales-SKILL.md, Template 3
SELECT
    tdp.ValueAsStr25 AS ProductCategory,
    COUNT(DISTINCT th.ID) AS OrderCount,
    SUM(td.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN TransDetail td ON th.ID = td.TransHeaderID AND td.IsActive = 1
INNER JOIN TransDetailParam tdp ON tdp.ParentID = td.ID
    AND tdp.VariableID = 11053  -- FP_ProductDescription
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) = 2025
GROUP BY tdp.ValueAsStr25
ORDER BY Revenue DESC
-- Expected: ~21 categories, total ~$1,793,445
-- CRITICAL: NO IsActive or ParentClassTypeID filter on TransDetailParam
```

### Top 10 Customers (TEST-04)
```sql
-- Source: control-erp-sales-SKILL.md, Template 5
SELECT TOP 10
    a.CompanyName,
    COUNT(DISTINCT th.ID) AS OrderCount,
    SUM(th.SubTotalPrice) AS Revenue
FROM TransHeader th
INNER JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 1
    AND th.IsActive = 1
    AND th.SaleDate IS NOT NULL
    AND YEAR(th.SaleDate) = 2025
GROUP BY a.CompanyName
ORDER BY Revenue DESC
-- Expected: #1 = FLASH Visual Media ($430,578)
```

### Gotcha Demonstration: TransDetailParam Filter Impact
```sql
-- WRONG (with IsActive filter) -- DO NOT USE:
-- SELECT ... FROM TransDetailParam tdp WHERE tdp.IsActive = 1 AND tdp.ParentClassTypeID = 10100
-- Result: DyeSub = $464,342 (WRONG)

-- CORRECT (no IsActive filter):
-- SELECT ... FROM TransDetailParam tdp WHERE tdp.VariableID = 11053
-- Result: DyeSub = $1,793,445 (CORRECT, matches Control report)

-- Impact: 286% revenue undercount ($1.3M missing)
```

## State of the Art

| Prior State | Current State | When Changed | Impact |
|-------------|---------------|--------------|--------|
| TransDetailParam filtered with IsActive=1 | No IsActive filter on TransDetailParam | 2026-02-07 (Rev 2) | DyeSub: $464K -> $1,793K |
| EstimateCreatedDate not documented | EstimateCreatedDate used for Type 2 date filtering | 2026-02-07 (Rev 2) | Lost quotes test: PARTIAL -> PASS |
| Type 2 = "Web Order" (wrong) | Type 2 = "Estimate/Quote" (correct) | 2026-02-08 (Phase 1) | Corrected TransactionType mapping |
| Validation done v1 (wrong filters) | Validation done Rev 2 (correct filters) | 2026-02-07 | All 21/21 tests PASS |

**Deprecated/outdated:**
- `validation/control-erp-validation-results.md` -- Contains pre-fix results with wrong TransDetailParam filters. Historical only. Do not use as reference for current testing.
- `validation/control-erp-validation-prompt-v2.md` -- The validation prompt that produced the wrong results. Historical only.

## Mapping: Requirements to Tests

This maps each TEST requirement to specific tests from the suite.

| Requirement | Tests | What It Validates |
|-------------|-------|-------------------|
| **TEST-01** | All 21 (1.1-7.3) | Full test suite executed with scorecard |
| **TEST-02** | 1.1 | Revenue baseline: $3,052,952.52 within 1% of $3,053,541.85 |
| **TEST-03** | 2.1, 2.2, 2.3, 2.4 | DyeSub categories match Control report |
| **TEST-04** | 3.1, 3.2, 3.3 | Customer revenue top 10 match |
| **TEST-05** | 1.2, 1.3, 1.4 | Monthly, quarterly, YoY consistency |
| **TEST-06** | 2.1 (TransDetailParam), 1.1 (SubTotalPrice), 1.2 (SaleDate) + explicit gotcha section | Known gotchas verified correct |

### Scorecard Gate Criteria
From `phase2-test-suite.md`: **PASS if >=19/21 pass with zero FAIL on Tier 1 tests.**
Prior result: 21/21 PASS (perfect score).

## Plan Split Recommendation

Based on the existing plan names from the roadmap:

### Plan 07-01: Execute Revenue, Product, and Customer Tests (Tiers 1-3)
- Tests 1.1-1.4 (Revenue Fundamentals) -- satisfies TEST-02, TEST-05
- Tests 2.1-2.4 (Product Intelligence) -- satisfies TEST-03
- Tests 3.1-3.3 (Customer Intelligence) -- satisfies TEST-04
- Total: 11 tests
- These are the quantitative core tests with exact expected values

### Plan 07-02: Execute Remaining Tests and Compile Scorecard (Tiers 4-7)
- Tests 4.1-4.4 (Estimate/Pipeline)
- Tests 5.1-5.2 (Web Import)
- Test 6.1 (Salesperson)
- Tests 7.1-7.3 (Edge Cases)
- Explicit gotcha validation section -- satisfies TEST-06
- Compile formal scorecard at `validation/milestone1-scorecard.md` -- satisfies TEST-01
- Total: 10 tests + scorecard compilation

## Expected Values Reference

Consolidated from `output/phase2-test-results.md` (Rev 2, 2026-02-07):

| Test | Expected Value | Tolerance | Source |
|------|---------------|-----------|--------|
| 1.1 | $3,052,952.52 | +/-1% ($30,530) | Phase 2 Rev 2, validated to 99.98% of known income |
| 1.2 | 12 months, sum = $3,052,949 | Sum +/-2% of 1.1 | Monthly trend (StatusID!=9 excludes $3.50 voided) |
| 1.3 | $592,789 | +/-2% | Oct+Nov+Dec sum |
| 1.4 | 2024=$3,060,289, 2025=$3,052,953 | +/-2% each | YoY comparison |
| 2.1 | 21 categories, total $1,793,445 | +/-2% per category | Validated vs Control report ($3 variance) |
| 2.2 | DyeLux $479,184 + Other $68,997 = $548,181 | +/-2% | Description-based identification |
| 2.3 | 11 groups, detail total ~$3,047,058 | +/-2% per group | Full reconciliation |
| 2.4 | $615,203 annual, seasonal Jul-Sep=78% | +/-2% annual | Monthly Swing Flag trend |
| 3.1 | #1 FLASH Visual Media $430,578 | Plausible list | Top 10 customers |
| 3.2 | 46 product lines for PAC BannerWorks | Has results | Customer detail |
| 3.3 | 4,172 orders | +/-5% | Total order count |
| 4.1 | 56.1% converted, 41.3% lost | +/-3% each | Conversion rate (all-time) |
| 4.2 | 326 pending | >0, <1000 | Pending estimates (may shift) |
| 4.3 | $1,635,895 | >$0 | Pipeline value (may shift) |
| 4.4 | $3,268,561 (244 quotes) | Plausible | Lost quotes 2025 (EstimateCreatedDate) |
| 5.1 | 429 orders, $150,750 | +/-5% count, +/-2% revenue | Deduplicated web orders |
| 5.2 | 4.94% revenue, 10.28% count | +/-1% each | Web percentage |
| 6.1 | Sum = $3,052,952.52 | +/-2% | Salesperson revenue |
| 7.1 | Clarify or note ambiguity | Behavioral | Feather flag disambiguation |
| 7.2 | Don't combine estimates + revenue | Behavioral | Estimate trap |
| 7.3 | Use OrderCreatedDate for "created" | Behavioral | Date field distinction |

**Note on Tests 4.2 and 4.3:** Pending estimate count and pipeline value are point-in-time values that may change as new estimates are created or existing ones convert. The test validates the query approach, not the exact number. Any reasonable value (>0, <1000 count; >$0 pipeline) passes.

## Open Questions

1. **MCP Availability**
   - What we know: Phase 4 used cross-reference method because Mac could not connect to Windows MCP server. Phase 2 test results were from live execution on 2026-02-07.
   - What's unclear: Whether MCP is available for this session.
   - Recommendation: Attempt live execution first. If MCP is unavailable, use cross-reference fallback (document in scorecard). Both approaches produce valid scorecards -- live is preferred.

2. **Revenue Target Number**
   - What we know: Success criteria says "$3,053,541.85 (within 1%)" but the validated SubTotalPrice query returns $3,052,952.52. Both numbers are well-established.
   - What's unclear: Nothing -- this is resolved. The $589 variance (0.02%) is within the 1% tolerance.
   - Recommendation: Document both numbers in the scorecard with the explanation.

3. **Tests 4.2/4.3 Volatility**
   - What we know: Pending estimates (326) and pipeline value ($1,635,895) are point-in-time. New estimates may have been created since 2026-02-07.
   - What's unclear: Current exact numbers.
   - Recommendation: These tests validate query correctness, not exact values. Any reasonable result passes.

## Sources

### Primary (HIGH confidence)
- `phase2-test-suite.md` -- 21 test definitions with expected values (authored for this project)
- `output/phase2-test-results.md` -- Rev 2 results, 21/21 PASS, executed 2026-02-07
- `skills/control-erp-core/control-erp-core-SKILL.md` -- Validated business rules
- `skills/control-erp-sales/control-erp-sales-SKILL.md` -- Validated product/revenue patterns
- `.planning/phases/04-sales-skill-verification/04-VERIFICATION.md` -- Phase 4 verification report
- `.planning/STATE.md` -- Current project state and blockers

### Secondary (MEDIUM confidence)
- `.planning/ROADMAP.md` -- Plan split guidance (07-01 and 07-02 descriptions)

### Tertiary (LOW confidence)
- `validation/control-erp-validation-results.md` -- Historical only, contains pre-fix (wrong) values
- `validation/control-erp-validation-prompt-v2.md` -- Historical only

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all tools/artifacts are existing and verified
- Architecture: HIGH -- scorecard format is defined, test suite exists, requirements map cleanly to tests
- Pitfalls: HIGH -- all gotchas are well-documented from prior phases with specific wrong/correct values
- Expected values: HIGH -- all from validated Phase 2 Rev 2 results

**Research date:** 2026-02-08
**Valid until:** 2026-03-08 (30 days -- stable domain, no library version concerns)
