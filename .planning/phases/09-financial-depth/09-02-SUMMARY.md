---
phase: 09-financial-depth
plan: 02
subsystem: financial
tags: [P&L, cash-flow, bank-balance, product-line-margin, YoY-comparison, MoM-comparison, extraction]
depends_on:
  requires: [phase-05-financial, 09-01]
  provides: [P&L-YoY-comparison, P&L-MoM-comparison, product-line-margin, bank-balance, undeposited-funds, cash-flow-summary, monthly-cash-flow, NL-routing-P&L-cash-flow, references-financial-analysis]
  affects: [analytics-milestone]
tech_stack:
  added: []
  patterns: [period-comparison-CASE-WHEN, two-query-margin-approach, cumulative-vs-period-balance, references-extraction]
key_files:
  created:
    - skills/control-erp-financial/references/financial-analysis.md
  modified:
    - skills/control-erp-financial/control-erp-financial-SKILL.md
decisions:
  - COGS is aggregate at GL level; product-line margin uses two-query approach (revenue breakdown + aggregate COGS)
  - Bank balance is cumulative (no date filter); cash flow is period-based (with date filter)
  - Payment forecast deferred to analytics milestone; NL routing redirects to historical cash flow
  - File extracted to references/ at 1,270 lines (threshold: 1,200); main file reduced to 888 lines
metrics:
  duration: 6m44s
  completed: 2026-02-09
---

# Phase 9 Plan 02: P&L Analysis and Cash Flow Summary

P&L comparison queries (YoY, MoM), product-line margin as two-query approach (revenue breakdown + aggregate COGS with limitation note), bank balance, undeposited funds, categorized cash flow summary, and monthly time series -- all extracted to references/financial-analysis.md after exceeding 1,200-line threshold.

## What Was Done

### Task 1: P&L Analysis with Comparison Periods (FIN-06)
Added P&L Analysis section with four query templates:

1. **P&L Year-over-Year Comparison** -- Uses YEAR() and DATEPART(DAYOFYEAR) for apples-to-apples YTD comparison with prior year. CASE WHEN for CurrentYear/PriorYear columns.

2. **P&L Month-over-Month Comparison** -- DECLARE variables for current/prior month boundaries. Same CASE WHEN pattern for CurrentMonth/PriorMonth columns.

3. **Product-Line Revenue Breakdown (Query A)** -- Revenue by GLAccount.AccountName where GLClassificationType = 4000. Uses SUM(-Amount). YTD default.

4. **Aggregate COGS (Query B)** -- Separate query for total COGS (GLClassificationType = 5001). Cannot be broken by product at GL level.

Documented COGS limitation prominently. Added 4 NL routing entries for P&L comparisons and product profitability.

### Task 2: Cash Flow and Bank Balance (FIN-07)
Added Cash Flow and Bank Balances section with four query templates:

1. **Bank Balance** -- Cumulative (NO date filter), GLClassificationType = 1000, exclude ZoomTex (10522-10525). Total row at presentation.

2. **Undeposited Funds** -- GLClassificationType = 1007, exclude ZoomTex. Shown separately from bank balance.

3. **Cash Flow Summary** -- Categorized cash in/out for YTD. Tracks only bank account movements (GLAccountID IN 90, 10412) to avoid double-counting undeposited-to-bank. Categories: Customer Deposits, Vendor Payments, Payroll, Manual/Adjustments, Other.

4. **Monthly Cash Flow Time Series** -- CashIn/CashOut/NetCashFlow by month for trend analysis.

Added 5 NL routing entries including "payment forecast" redirect to historical cash flow.

### Task 3: Line Count Management and Extraction
File reached 1,270 lines (over 1,200 threshold). Extracted four sections to `references/financial-analysis.md`:
- AR Detail with Customer Breakdown (from plan 09-01)
- AP Detail with Vendor Breakdown (from plan 09-01)
- P&L Analysis (from this plan)
- Cash Flow and Bank Balances (from this plan)

Main file reduced to 888 lines with brief summaries and `See references/financial-analysis.md` routing. NL interpretation table kept in main file for routing. No content duplication.

## Decisions Made

| Decision | Rationale |
|----------|-----------|
| Two-query approach for product-line margin | COGS GL accounts are aggregate; revenue has per-product NodeIDs but COGS does not |
| Bank balance = cumulative, cash flow = period | Different financial concepts require different query patterns |
| Payment forecast deferred with NL redirect | Historical cash flow implemented; forecasting requires analytics capabilities |
| Extract at 1,270 lines to references/ | 1,200-line threshold per STATE.md decision; main file reduced to 888 lines |
| Keep NL routing in main file | Routing stays close to the skill entry point; detail goes to references |

## Deviations from Plan

None -- plan executed exactly as written.

## Metrics

- **File growth:** 1,038 lines -> 1,270 lines (pre-extraction) -> 888 lines (post-extraction)
- **References file:** 412 lines (new, contains all extracted query templates)
- **New NL routing entries:** 9 total (4 P&L + 5 cash flow)
- **New query templates:** 8 total (4 P&L + 4 cash flow)
- **Duration:** 6m44s

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | af68e83 | feat(09-02): add P&L analysis with comparison periods (FIN-06) |
| 2 | 66628a0 | feat(09-02): add cash flow and bank balance queries (FIN-07) |
| 3 | d2ff520 | refactor(09-02): extract analytical queries to references/financial-analysis.md |

## Next Phase Readiness

Phase 9 (Financial Depth) is now complete. All four requirements (FIN-04 through FIN-07) are covered across plans 01 and 02:
- FIN-04: AR aging with customer breakdown
- FIN-05: AP tracking with vendor breakdown
- FIN-06: P&L comparison (YoY, MoM) and product-line margin
- FIN-07: Bank balance, cash flow summary, monthly time series (historical only; forecast deferred)

Phase 10 (Customer Intelligence) can proceed independently -- no dependencies on Phase 9 outputs.
