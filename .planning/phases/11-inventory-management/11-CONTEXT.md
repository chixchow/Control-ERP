# Phase 11: Inventory Management - Context

**Gathered:** 2026-02-09
**Status:** Ready for planning

<domain>
## Phase Boundary

New skill for stock levels, reorder monitoring, and warehouse planning through natural language queries. Includes purchasing intelligence (PO history, vendor costs) as a reliable data layer alongside inventory data that is currently incomplete.

**Critical context:** FLS does not currently make effective use of Control's inventory system. Stock data lives in Google Sheets and is being migrated to Control this year. Inventory quantities and usage data may be inaccurate. Purchasing data (Type 7 POs) is reliable. The skill must work with current data quality while being ready to become more useful post-migration.

</domain>

<decisions>
## Implementation Decisions

### Stock level presentation
- Summary-first pattern: quick answer (part name, available qty, on-hand qty, unit), with drill-down available for full inventory card (reorder point, last received, warehouse, cost)
- Always show both Available and On-Hand quantities side by side
- Group multi-part results by category/type
- Search matching: Claude's discretion (exact for part numbers, fuzzy for names/descriptions)

### Reorder intelligence
- Two tiers with clear separation:
  - **Primary (reliable):** Purchasing intelligence — last price paid, price history, vendor comparison, order frequency, average order size, last order date
  - **Secondary (caveated):** Reorder alerts from Control's inventory fields — include but clearly warn that results depend on accurate inventory setup
- Proactively flag data quality mismatches (e.g., recent PO activity but zero inventory qty) to help identify setup gaps
- Show unit context always — display the unit AND note when alternate/warehouse units exist for a part

### Inventory valuation
- Route by intent, keep separate:
  - **"What's our inventory worth?"** → GL account balance (accounting valuation, must match balance sheet)
  - **"What does X cost?"** → Last price paid on PO (operational cost lookup)
  - Never mix these contexts in a single answer
- GL sub-account breakdown: Claude's discretion based on what discovery reveals under the inventory asset accounts
- Part-level valuation available but with accuracy warnings given current data state

### Query scope defaults
- Default filter chain: IsActive=1 → TrackInventory=true → then context-dependent qty filter
  - Stock checks: exclude zero-qty parts
  - Reorder checks: include zero-qty parts (zero = needs reorder)
- Inactive and non-tracked parts hidden by default, available on explicit request
- Broad queries ("what's in stock"): show top 25 by most active (most frequently ordered via PO history). If activity ranking is too complex, fall back to category-level summary with drill-down
- Purchasing query time windows: Claude's discretion based on query context
- Warehouse scope: Claude's discretion after discovering FLS warehouse configuration

### Claude's Discretion
- Part search matching strategy (exact vs fuzzy based on input)
- Purchasing query default time windows
- Warehouse breakdown vs totals (based on FLS warehouse count discovery)
- GL inventory sub-account depth
- Drill-down detail level for inventory card

</decisions>

<specifics>
## Specific Ideas

- "If someone wants to know what a 4x4x48 box costs, last value paid is the correct number to return"
- Units are HUGELY important in Control — inventory units, alternate units, and warehouse units all impact usage, costing, and valuation. The skill must surface unit context, not just quantities.
- FLS has a large number of parts that were never intended to be inventoried — the TrackInventory filter is essential to avoid noise
- Purchasing data (Type 7 POs) is the reliable data source; inventory quantities may not match reality until migration is complete

</specifics>

<deferred>
## Deferred Ideas

- **Google Sheets to Control inventory migration** — Dedicated phase for migrating stock data from Google Sheets into Control. Involves ensuring each part is properly set up with correct units (inventory units, alternate units, warehouse units). This is prerequisite to reliable inventory data.
- **Usage-based reorder thresholds** — Deriving reorder points from consumption history. Deferred until inventory data is reliable post-migration.

</deferred>

---

*Phase: 11-inventory-management*
*Context gathered: 2026-02-09*
