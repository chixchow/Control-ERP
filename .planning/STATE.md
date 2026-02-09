# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-02-09)

**Core value:** Any FLS team member can ask a business question in plain English and get an accurate, formatted answer from their ERP data.
**Current focus:** Milestone v1.1 -- Read-Layer Build-Out

## Current Position

Phase: 10 of 13 (Customer Intelligence) -- IN PROGRESS
Plan: 1 of 2 complete
Status: In progress
Last activity: 2026-02-09 -- Completed 10-01-PLAN.md (Customer Profile Lookup)

Progress: [███░░░░░░░] 30% (v1.1) -- 3 of 10 plans complete

## Performance Metrics

**v1.0 Velocity:**
- Total plans completed: 15
- Average duration: 4min11s
- Total execution time: 1.04 hours

**v1.1 Velocity:**
- Total plans completed: 3
- Average duration: 3min56s
- Total execution time: 0.20 hours

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
- Customer skill main file stays under 600 lines (Plan 10-02 may extract to references/ if >1,000)

### Pending Todos

None.

### Blockers/Concerns

- Phases 11-12 require MCP database access for FLS-specific discovery queries (warehouse configuration, station hierarchy, artwork status usage)

## Session Continuity

Last session: 2026-02-09T18:37Z
Stopped at: Completed 10-01-PLAN.md (Customer Profile Lookup)
Resume file: None

**v1.0 Status:** SHIPPED (8 phases, 15 plans, 35/35 requirements)
**v1.1 Scope:** 14 requirements across 5 phases (9-13), 5 domains
