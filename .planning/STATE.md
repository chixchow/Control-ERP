# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-09)

**Core value:** Any FLS team member can ask a business question in plain English and get an accurate, formatted answer from their ERP data.
**Current focus:** Milestone v1.1 -- Read-Layer Build-Out

## Current Position

Phase: 12 of 13 (Production Workflow)
Plan: 1 of 2 complete
Status: In progress
Last activity: 2026-02-10 -- Completed 12-01-PLAN.md (Production Skill with Artwork Pipeline)

Progress: [███████░░░] 70% (v1.1) -- 7 of 10 plans complete

## Performance Metrics

**v1.0 Velocity:**
- Total plans completed: 15
- Average duration: 4min11s
- Total execution time: 1.04 hours

**v1.1 Velocity:**
- Total plans completed: 7
- Average duration: 3min18s
- Total execution time: 0.39 hours

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Extend control-erp-financial (not create separate financial-deep skill) -- 80% of new content builds on existing queries, safeguard: extract to references/ if >1,200 lines
- Fold reports catalog into control-erp-glossary (not standalone skill) -- 36 reports is ~100-150 lines, too thin for standalone, glossary is already a routing skill
- Phases 9+10 parallelizable (Financial and Customer have zero dependencies)
- Phases 11-12 need live database discovery (warehouse config, station hierarchy)
- AR aging uses SaleDate; AP aging uses DueDate -- different semantics per transaction type
- Skill file extracted to references/ at 1,270 lines; main file reduced to 888 lines with references/financial-analysis.md (412 lines)
- COGS is aggregate at GL level; product-line margin uses two-query approach
- Payment forecast deferred to analytics milestone; NL routing redirects to historical cash flow
- CompanyName not AccountName (Account table has no AccountName column)
- AR balance = SUM(TransHeader.BalanceDue), not Account.CreditBalance (opposite semantics)
- Walk-In account (ID=10, AccountNumber=0) excluded from customer analytics
- Customer skill kept integrated at 1,073 lines (not extracted to references, below 1,200-line threshold)
- Customer ranking/segmentation/at-risk completed in single comprehensive commit (more efficient than artificial separation)
- RFM NTILE ordering: DESC for Recency, ASC for Frequency/Monetary (verifier caught inversion)
- Two-tier confidence model for inventory: Primary (reliable PO data) vs Secondary (caveated inventory quantities)
- Default to Warehouse 10 for inventory queries (has 98% of stock vs Apparel WH with 3 parts)
- Progressive filtering for inventory: IsActive -> TrackInventory -> context-dependent qty filter
- GL NodeID 10414 is authoritative for inventory valuation ($651K), not part-level sum
- VendorTransDetail polymorphic link: try CatalogItem path (12076) first, then direct Part (12014)
- Months of supply calculation marked as SECONDARY TIER (depends on incomplete data)
- th.ID DESC for PO ordering (not SaleDate which is unreliable for Type 7)
- Artwork stuck threshold: 7 days (3x average 54-hour turnaround), adjustable per user request
- StatusID=8 ("Unknown") excluded from actionable artwork pipeline -- 25,274 groups represent orders without formal artwork workflow (intentional)
- Artwork turnaround filter: ArtworkApprovalDT > GroupCreatedDT (excludes cloned-order stale dates)
- Production skill kept integrated at 418 lines (not extracted to references, below 1,200-line threshold)

### Pending Todos

None.

### Blockers/Concerns

- Phase 11: COMPLETE -- blocker cleared
- Phase 12: Artwork pipeline (PROD-01) complete; station workload and dwell time (PROD-02/03) ready to build in Plan 02

## Session Continuity

Last session: 2026-02-10T00:22Z
Stopped at: Completed 12-01-PLAN.md (Production Skill with Artwork Pipeline) -- Phase 12 Plan 1 of 2
Resume file: None

**v1.0 Status:** SHIPPED (8 phases, 15 plans, 35/35 requirements)
**v1.1 Scope:** 14 requirements across 5 phases (9-13), 5 domains
