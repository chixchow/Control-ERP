---
phase: 13
plan: 01
subsystem: skill-routing
tags: [crystal-reports, routing, glossary, gap-tracking]
requires: [Phase 2 report analysis, all 6 domain skills]
provides: [Crystal Reports catalog, gap log, complete coverage assessment]
affects: [Future skill development priorities]
tech-stack:
  added: []
  patterns: [LLM-first routing, honest gap acknowledgment]
key-files:
  created:
    - skills/control-erp-glossary/gap-log.md
  modified:
    - skills/control-erp-glossary/control-erp-glossary-SKILL.md
key-decisions:
  - decision: "Route to control-erp-customers for 'top customers' queries (not sales)"
    rationale: "Customer skill has comprehensive ranking with YoY, Pareto, segmentation vs. sales Template 6"
    context: "Phase 13-01 Crystal Reports Catalog"
  - decision: "Mark 5 reports as 'Not yet covered' vs attempting synthesis"
    rationale: "Honest gap acknowledgment prevents unvalidated answers"
    context: "Phase 13-01 Gap Log"
  - decision: "No TransactionType values cited from report_summary.md"
    rationale: "Report analysis files contain incorrect inferred SQL (Type 3=Order); use core skill mappings only"
    context: "Phase 13-01 execution, pitfall avoidance"
duration: 2min18s
completed: 2026-02-10
---

# Phase 13 Plan 01: Crystal Reports Catalog + Gap Log Summary

**One-liner:** Added 36-report catalog to glossary with coverage status mapped to 6 domain skills (47% replaced), created gap log tracking 7 uncovered domains and 5 uncovered reports

## Performance

- **Execution time:** 2min18s
- **Tasks completed:** 2 of 2 (100%)
- **Commits:** 2 task commits + 1 metadata commit
- **Files modified:** 1 (glossary skill)
- **Files created:** 1 (gap-log.md)
- **Lines added:** 123 lines (88 to glossary, 35 gap log)

## Accomplishments

### Crystal Reports Catalog Integration
Added comprehensive CRYSTAL REPORTS CATALOG section to control-erp-glossary skill (88 lines) positioned between ORDER LIFECYCLE and NATURAL LANGUAGE ROUTING TABLE sections. Catalog organizes all 36 Crystal Reports by category:
- Financial Reports: 5 (all replaced by control-erp-financial)
- WIP Reports: 4 (3 replaced, 1 partial by control-erp-production)
- Sales Reports: 14 (6 replaced, 8 partial by control-erp-sales + control-erp-customers)
- Estimate Reports: 4 (1 replaced by control-erp-customers, 2 partial by control-erp-core, 1 uncovered)
- Inventory & Product Reports: 6 (4 partial by control-erp-inventory, 2 uncovered)
- Other Reports: 3 (1 partial by control-erp-customers, 2 uncovered)

Coverage summary: 17 reports (47%) fully replaced by skills, 14 reports (39%) partially covered, 5 reports (14%) not yet covered.

### LLM-First Guidance
Added clear user guidance establishing LLM query answers as primary approach, Crystal Reports as fallback for:
1. Uncovered data domains
2. Complex formatting requirements (invoices, work orders with embedded images)
3. Explicit user request to run a Crystal Report

Includes disclaimer that report details were inferred from filenames/schema analysis (not extracted from binary .rpt files).

### Gap Log for Future Development
Created skills/control-erp-glossary/gap-log.md (35 lines) tracking uncovered domains with example queries, table existence validation, and priority levels:
- **7 uncovered domains:** Payroll/TimeCard, Commissions, Tax Compliance, Service Tickets, Shipping Detail, Product Configuration, Est vs Actual Cost
- **5 uncovered reports:** Est. vs Act. Cost Summary, Product Detail, Product Parameters, Shipped By Date, Work Order with Proof
- **Update policy:** Only log domains with database tables and recurring business need (prevents noise)

### Coverage Assessment Quality
Honest gap acknowledgment prevents attempting unvalidated synthesis:
- "Partially covered" status acknowledges skill can answer some but not all of report's purpose
- "Not yet covered" explicitly directs users to run Crystal Report in Control
- No incorrect TransactionType values cited from report_summary.md (validated against core skill mappings)

## Task Commits

| Task | Commit | Description | Files |
|------|--------|-------------|-------|
| 1 | 847f0c4 | Add Crystal Reports catalog to glossary | control-erp-glossary-SKILL.md |
| 2 | 8ffd13b | Create gap log for uncovered domains | gap-log.md |

## Files Created

### skills/control-erp-glossary/gap-log.md (35 lines)
Gap tracking file with:
- 7 uncovered domains with example queries, table validation, priority levels
- 5 uncovered Crystal Reports requiring Control execution
- Update policy preventing noise (requires table existence + recurring need)
- Last updated timestamp and domain/report counts

## Files Modified

### skills/control-erp-glossary/control-erp-glossary-SKILL.md
- **Before:** 492 lines (3 sections: Product Terminology, Technical Terminology, NL Routing)
- **After:** 578 lines (+88 lines, +17.9%)
- **Added:** CRYSTAL REPORTS CATALOG section (86 lines content + 2 separators)
- **Structure:** All 36 reports organized in 6 category tables + coverage summary table
- **Position:** Between ORDER LIFECYCLE (line 395) and NATURAL LANGUAGE ROUTING TABLE (line 538)
- **Coverage mapping:** 31 references to domain skills (control-erp-financial, production, sales, customers, inventory, core)

## Decisions Made

### Decision 1: Coverage Status Accuracy
**Context:** Mapping 36 Crystal Reports to 6 domain skills with honest assessment
**Options considered:**
1. Mark marginal coverage as "Replaced" to show skill completeness
2. Mark everything with any overlap as "Partially covered"
3. Use three-tier status: Replaced / Partially covered / Not yet covered

**Decision:** Three-tier status with strict criteria
- "Replaced by [skill]" = skill query fully answers report's business intent
- "Partially covered" = skill covers some aspect but not complete report purpose
- "Not yet covered" = no skill handles this data domain

**Rationale:** Honest gap acknowledgment prevents users from getting incomplete answers they assume are complete. Three-tier status creates clear expectations.

**Impact:** 17 reports marked "Replaced", 14 "Partially covered", 5 "Not yet covered" based on actual skill query coverage vs. report purpose.

### Decision 2: No Report Parameter Details
**Context:** Per CONTEXT.md: "Report entries show name and purpose only—no parameter details"
**Rationale:** Users already know how to run their reports; catalog is for discovery and coverage assessment, not documentation
**Implementation:** Each report entry has Name + Purpose + Coverage status only
**Impact:** Kept catalog concise (86 lines for 36 reports = 2.4 lines/report average)

### Decision 3: LLM-First Routing Philosophy
**Context:** Crystal Reports are legacy; skills are modern replacement
**Decision:** Establish LLM query answers as primary approach, Crystal Reports as fallback
**Guidance added:**
- "LLM answers first"
- Only recommend Crystal Report when: skill doesn't cover domain, complex formatting needed, or user explicitly requests report
- Skills are "modern replacement" for reports

**Rationale:** Aligns with project goal (any team member can ask business question in plain English)
**Impact:** Users default to asking Claude instead of searching for the right report in Control

## Deviations from Plan

None. Plan executed exactly as written.

## Issues Encountered

None. All verifications passed on first attempt.

## Next Phase Readiness

### Phase 13 Plan 02: Expanded NL Routing Table
**Status:** READY
**Blockers:** None
**Inputs available:**
- All 6 domain skills have NL routing tables with 13-30 entries each
- Current glossary NL table has 37 entries routing to 4 destinations (sales, financial, core, glossary)
- Missing routes to customers, inventory, production skills
- Domain boundary decisions documented in 13-RESEARCH.md Section "Overlap Resolution Rules"

**Dependencies satisfied:**
- Crystal Reports Catalog complete (needed for report discovery routing entries)
- Gap log exists (needed for "not covered" routing)

### Phase 13 Plan 03: Test Query Validation
**Status:** READY (dependent on 13-02 completion)
**Blockers:** None
**Inputs available:**
- Example test queries in 13-RESEARCH.md Section "Example 4: Test Query Set"
- 20+ queries covering sales, financial, customer, inventory, production domains
- Includes realistic language patterns (typos, casual phrasing, abbreviations)

## Recommendations

1. **Maintain gap log actively:** Every "I don't know" response should trigger gap log review—if domain has tables + recurring need, add it
2. **Re-assess coverage quarterly:** As skills evolve, reports marked "Partially covered" may become "Replaced"
3. **User feedback loop:** When users run Crystal Reports, ask why—if answer is "skill couldn't do X", that's a gap to log
4. **Consider report retirement:** 17 reports (47%) are fully replaced—FLS could hide those from Control UI to encourage LLM usage

---

**Plan 13-01 complete.** Crystal Reports catalog integrated into glossary, gap log established for uncovered domain tracking. Phase 13 Plan 02 (Expanded NL Routing) ready to execute.
