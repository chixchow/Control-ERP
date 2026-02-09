# Phase 12: Production Workflow - Context

**Gathered:** 2026-02-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Users can track artwork approvals, monitor station workload, and analyze production dwell time -- through a new control-erp-production skill. This is a read-layer skill; no database modifications.

</domain>

<decisions>
## Implementation Decisions

### Artwork pipeline depth
- Default view: summary counts by status (e.g., "5 pending approval, 3 in revision, 12 approved")
- Drill-down available to see individual artwork items per status
- Include revision count per artwork item
- Always show customer name + order number on every artwork row
- Track turnaround time with averages (submission to approval)
- Support filtering by artist (ArtworkPlayer)
- Discover actual artwork statuses from database (_ArtworkStatus table) -- do not hardcode
- Audience: both production team and sales team use artwork data
- Stuck items: Claude determines threshold based on average turnaround data

### Station workload view
- Default metric: WIP count + dollar value per stage/station
- FLS station model: **dual order-level and line-item-level stations**
  - Order-level stations = high-level milestones (e.g., Approved, Production)
  - Line-item stations = granular production routing (each line moves through department stages independently)
  - Stage numbering 0-6 applied to most stations; multiple stations can exist at the same stage
- Queries must be **context-aware**: dual-level stations (like "Approved") return both orders and line items; line-item-only stations (like "West Side Production") return only line items
- Default view: stage-level summary (Stage 0, 1, 2...) with station drill-down per stage
- Purpose: departments can see what's in their queue and what's coming next
- Station list is user-definable -- must be discovered from database, not hardcoded
- Bottleneck detection: **both count + dwell time** combined as a "backed up" indicator

### Process efficiency (station dwell time)
- Employees do NOT clock on/off individual jobs -- no per-job TimeCard data
- Skip employee TimeCard queries entirely (not relevant to production goals)
- The real time metric is **station dwell time**: how long each line item sits in each station, calculated from station transition timestamps
- Business goal: 1-business-day turnaround for orders of a certain size
- Default view: overall average dwell time per station/stage
- Drill-down: dwell time broken out by order size/type to identify where large vs small orders slow down
- Focus: identifying slow stages for efficiency improvement

### Output & formatting
- Summaries show counts and totals; order numbers only appear in drill-down detail views
- Currency: rounded to dollars in summaries ($1,234), full precision in detail views ($1,234.56)
- Default to active/open items only -- completed work shown only on explicit request

### Claude's Discretion
- Stuck artwork threshold calculation method
- Exact station discovery queries and classification logic
- Dwell time calculation approach (which timestamp fields track station transitions)
- How to determine order-level vs line-item-level station classification

</decisions>

<specifics>
## Specific Ideas

- Departments need to see "what's coming and what they have to work on" -- the station view serves as a department queue
- The 1-business-day turnaround goal is the key driver for dwell time analysis -- station dwell data should help answer "which process requires attention for efficiency improvement"
- Station queries for dual-level stations (e.g., "show me everything Approved") must naturally return both orders and line items without the user specifying which level
- When a station is line-item-only (e.g., "West Side Production"), the system should only show line items without error or confusion

</specifics>

<deferred>
## Deferred Ideas

### Write-layer artwork improvements (future milestone)
- **Artwork description field**: Add a searchable description at the ArtworkGroup level so users can search by customer + design description (e.g., "XYZ flags for Company ABC")
- **Design reuse/linking**: Ability to assign previously produced art to new orders, carrying associated notes (PMS colors, swatch selections) to the new order
- **Color swatch chain tracing**: Link the chain of PMS match request -> swatch order (order 123456) -> selection (Swatch C) -> production order (123457) -> reorder (133900) so all color context is accessible from any order in the chain
- **Consistent artwork descriptions**: System-enforced requirement for artwork descriptions on every ArtworkGroup

### Read-layer artwork search (may fit Phase 12 if data exists)
- Search artwork by customer + description text (if ArtworkGroup already has description data populated)
- Show alert when a design has been previously produced, with most recent order number
- Surface color/PMS notes from related orders

</deferred>

---

*Phase: 12-production-workflow*
*Context gathered: 2026-02-09*
