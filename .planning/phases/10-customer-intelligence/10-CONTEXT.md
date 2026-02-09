# Phase 10: Customer Intelligence - Context

**Gathered:** 2026-02-09
**Status:** Ready for planning

<domain>
## Phase Boundary

New `control-erp-customers` skill enabling customer lookup, segmentation, CLV analysis, and at-risk detection through natural language queries. Users can search for any customer and get profiles, rankings, and churn risk assessments. Revenue totals must cross-check against the core revenue baseline ($3,052,952.52).

</domain>

<decisions>
## Implementation Decisions

### Customer Profile Depth
- **Default: Executive summary** with drill-down available on request
- Summary includes business + relationship fields: company name, account #, primary contact, phone, total revenue, order count, last order date, AR balance, sales rep, account age (customer since), payment terms, IsProspect/IsClient status, last contact activity
- Detail view adds: all contacts, all addresses, all phones, account notes, full order aggregates
- Order history: aggregates in detail view by default; user can ask "show orders for [customer]" to get individual order list (last 10-20 orders with date, order #, amount, status)
- **Search: Smart match** -- try exact first, fall back to fuzzy (LIKE %term%). If multiple matches, show a pick list before returning a profile. Searches company name, contact name, and account number

### Segmentation & Ranking
- **Default ranking: Revenue** (SubTotalPrice), trailing 12 months
- Support alternative rankings on request: "top by order count", "top by margin"
- **Default list size: Top 10**, expandable ("show top 25", "show all")
- **Full RFM segmentation** available: Recency/Frequency/Monetary scoring with named segments (Champions, Loyal, At Risk, Lost, etc.)
- All time windows overridable ("top customers this year", "top customers all time")

### At-Risk Detection
- **Multi-signal risk model**: dormancy, declining spend trend, shrinking order size, late payments
- **Personalized dormancy thresholds** based on each customer's historical ordering pattern:
  - Resellers with monthly ordering cadence flagged after ~6 weeks of silence
  - Seasonal customers (e.g., Flash, PropVinyl, Wilsons Restaurant) compared against their seasonal window, not a flat threshold
  - Where a definable historic pattern exists, use it
  - **Fallback: 90-day default** for customers without enough history to establish a pattern (aligns with quarterly touch goal)
- **Revenue at risk with trend**: show trailing 12-month revenue PLUS trend direction (e.g., "$45K trailing 12mo, down 30% vs prior year")
- **Sorted by combined score**: blend of revenue impact and overdue severity -- a high-value customer slightly overdue ranks similar to a mid-value customer very overdue

### Output & Formatting
- **YoY comparisons: Always included** on every revenue figure (e.g., "$125K, +12% vs prior year")
- Format and list presentation: Claude's discretion based on query type
- Currency formatting: Claude's discretion based on context (abbreviated for lists, precise for detail)
- Cross-domain data (AR balance, open orders): Claude's discretion on what to inline vs reference to other skills

### Claude's Discretion
- Output format choice per query type (table, narrative, hybrid)
- Currency abbreviation vs full precision based on context
- Cross-domain inlining decisions (what to show inline vs "ask about X for detail")
- RFM segment boundary calculations
- Minimum order history required to establish a "pattern" for personalized dormancy

</decisions>

<specifics>
## Specific Ideas

- FLS has two clear customer types: regular-frequency orderers and as-needed orderers. The risk model must handle both
- Quarterly touch cadence is the business goal -- every client should be contacted at least quarterly
- Higher-frequency customers (especially resellers) need more aggressive drift detection since they should be ordering monthly
- Named seasonal examples: Flash, PropVinyl, Wilsons Restaurant -- these have predictable seasonal patterns
- Some customers order yearly or bi-yearly -- that's normal for them, not a risk signal
- Revenue totals must cross-check against validated core baseline ($3,052,952.52)

</specifics>

<deferred>
## Deferred Ideas

None -- discussion stayed within phase scope

</deferred>

---

*Phase: 10-customer-intelligence*
*Context gathered: 2026-02-09*
