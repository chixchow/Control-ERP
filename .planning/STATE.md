# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-09)

**Core value:** Any FLS team member can ask a business question in plain English and get an accurate, formatted answer from their ERP data.
**Current focus:** Milestone v1.1 -- Read-Layer Build-Out

## Current Position

Phase: 11 of 13 (Inventory Management) -- IN PROGRESS
Plan: 1 of 2 complete
Status: In progress
Last activity: 2026-02-09 -- Completed 11-01-PLAN.md (Inventory Skill Foundation)

Progress: [█████░░░░░] 50% (v1.1) -- 5 of 10 plans complete

## Performance Metrics

**v1.0 Velocity:**
- Total plans completed: 15
- Average duration: 4min11s
- Total execution time: 1.04 hours

**v1.1 Velocity:**
- Total plans completed: 5
- Average duration: 3min38s
- Total execution time: 0.31 hours

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

### Pending Todos

None.

### Blockers/Concerns

- Phase 11: Warehouse configuration discovered (Warehouse 10 = default) -- blocker cleared
- Phase 12: Station hierarchy and artwork status usage still require live database discovery

## Session Continuity

Last session: 2026-02-09T19:10Z
Stopped at: Completed 11-01-PLAN.md (Inventory Skill Foundation) -- Phase 11 1/2 complete
Resume file: None

**v1.0 Status:** SHIPPED (8 phases, 15 plans, 35/35 requirements)
**v1.1 Scope:** 14 requirements across 5 phases (9-13), 5 domains
