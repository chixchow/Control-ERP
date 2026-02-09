# Phase 9: Financial Depth - Context

**Gathered:** 2026-02-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Extend the existing control-erp-financial skill with AR aging, AP tracking, P&L breakdown, and cash flow queries. Users can answer analytical financial questions through natural language. This builds on the validated financial foundation from v1.0 Phase 5 — no new skill files, just deeper content in the existing skill.

</domain>

<decisions>
## Implementation Decisions

### AR/AP Detail Level
- AR aging default view: customer breakdown with balance split across aging buckets (not summary-only)
- AR aging buckets: Standard 4-bucket — Current/30/60/90+
- AP view mirrors AR format: vendor breakdown with aging buckets (Current/30/60/90+)
- Both AR and AP support single-customer/vendor drill-down — "show me AR for ABC Signs" returns just that entity with invoice-level detail

### P&L Structure
- Two P&L modes available: traditional format (default) and product-line breakdown (on request)
- Traditional P&L: Revenue → COGS → Gross Margin → Operating Expenses (by category) → Net Income — full depth to bottom line
- Product-line P&L: Revenue, COGS, and gross margin split by product category (DyeSub, Vinyl, Wrap, etc.)
- Comparison periods supported: month-over-month and year-over-year with delta columns
- Default date range when unspecified: year-to-date

### Cash Flow Presentation
- Bank balance query returns: total cash position at top, then individual account balances below (checking, savings, etc.)
- Cash flow shows categorized in/out: customer payments, vendor payments, payroll, etc. — not just net totals
- Time-series view supported: monthly columns when date range spans multiple months
- Default date range when unspecified: year-to-date (consistent with P&L default)

### Output Formatting
- Large result sets: show full list always — no truncation or top-N limiting
- Currency: full precision with two decimals, comma separators, dollar sign ($1,234.56)
- Totals row: include when useful (AR/AP aging lists) but not where redundant (P&L already shows net income)
- Negative numbers: accounting parentheses convention — ($1,234.56), not -$1,234.56

### Claude's Discretion
- Table column widths and alignment
- How to label GL account categories in P&L (use GLAccount tree structure)
- Cash flow category names (derive from GL classification types)
- How to handle periods with no data in time-series views
- Invoice-level detail format for drill-down queries

</decisions>

<specifics>
## Specific Ideas

- AR/AP should feel like Control's A_R Detail report — customer-level with aging columns
- P&L and cash flow both default to year-to-date for consistency — users shouldn't need to specify the same date range convention each time
- Full list output for all queries — FLS team wants to see everything, not summaries

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 09-financial-depth*
*Context gathered: 2026-02-09*
