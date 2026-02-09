---
phase: 01-core-skill-verification
verified: 2026-02-09T04:30:00Z
status: gaps_found
score: 5/7 must-haves verified
gaps:
  - truth: "A user asking 'What is Type 7 StatusID 28?' gets 'Closed' from every source consistently -- core skill, knowledge log, and MEMORY.md all agree"
    status: partial
    reason: "Core skill and MEMORY.md correctly say 28=Closed. Knowledge log Section 2 correctly says 28=Closed. BUT knowledge log Sections 1, vendor chain, and vendor workflow still say '28 (Open)' or '28 = Open' -- 3 stale references contradict the Section 2 definitive reference within the same file."
    artifacts:
      - path: "documentation/control-erp-knowledge-log.md"
        issue: "Line 39: 'All StatusID=28 (Open)' -- label should be Closed. Line 51: 'StatusID 28 = Open' -- should be Closed. Line 397: 'StatusID 28 (Open)' -- should be Closed."
    missing:
      - "Change 'StatusID=28 (Open)' to 'StatusID=28 (Closed)' on line 39"
      - "Change 'StatusID 28 = Open' to 'StatusID 28 = Closed' on line 51"
      - "Change 'StatusID 28 (Open)' to 'StatusID 28 (Closed)' on line 397"
  - truth: "The knowledge log Section 7 has no stale TODO markers -- StatusID 2=Built and 4=Closed are filled in, not question marks"
    status: partial
    reason: "The Section 7 StatusID TABLE (lines 377-384) correctly shows 0=New, 1=WIP, 2=Built, 3=Sale, 4=Closed, 9=Voided. TODO warning block was removed. But the ASCII workflow DIAGRAM (lines 340-357) still has 'StatusID ? = Built' (line 345) and 'StatusID = Closed' without the number 4 (line 350). Also line 347 says '3 = Sales' (plural) instead of '3 = Sale' (singular)."
    artifacts:
      - path: "documentation/control-erp-knowledge-log.md"
        issue: "Line 345: 'StatusID ? = Built' -- should be 'StatusID 2 = Built'. Line 350: 'StatusID = Closed' -- should be 'StatusID 4 = Closed'. Line 347: 'StatusID 3 = Sales' -- should be 'StatusID 3 = Sale' (singular)."
    missing:
      - "Change 'StatusID ? = Built' to 'StatusID 2 = Built' on line 345"
      - "Change 'StatusID = Closed' to 'StatusID 4 = Closed' on line 350"
      - "Change 'StatusID 3 = Sales' to 'StatusID 3 = Sale' on line 347 (consistency with core skill)"
---

# Phase 1: Core Skill Verification Report

**Phase Goal:** The control-erp-core skill is verified complete and the knowledge log is structured for future reference
**Verified:** 2026-02-09T04:30:00Z
**Status:** gaps_found
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | A user asking "What StatusIDs exist for Type 2 estimates?" gets 11=Pending, 12=Lost, 13=Converted, 14=Voided from the core skill alone | VERIFIED | Core skill lines 254-260: Type 2 StatusID table with all 4 values correctly listed. TransactionType table line 40 also includes lifecycle summary. |
| 2 | A user asking "What is Type 7 StatusID 28?" gets "Closed" from every source consistently | PARTIAL | Core skill (line 45): 28=Closed -- correct. MEMORY.md (line 28): 28=Closed -- correct. Knowledge log Section 2 (line 87): 28=Closed -- correct. BUT knowledge log lines 39, 51, and 397 still say "28 (Open)" or "28 = Open". Three internal contradictions remain in the knowledge log. |
| 3 | Types 3 (Recurring Order), 4 (Credit Memo), and 10 (Vendor Credit Memo) appear in the core skill TransactionType table with wiki-sourced StatusIDs and a note about zero FLS records | VERIFIED | Core skill line 41: Type 3 Recurring Order, StatusID 21, zero records at FLS. Line 42: Type 4 Credit Memo, StatusID 20=Credit Memo, 9=Voided, zero records at FLS. Line 48: Type 10 Vendor Credit Memo, StatusID 24, not validated at FLS. Only Type 5 remains UNUSED (1 match for "UNUSED" in file). |
| 4 | The knowledge log Section 7 has no stale TODO markers -- StatusID 2=Built and 4=Closed are filled in, not question marks | PARTIAL | The Section 7 StatusID TABLE (lines 377-384) is fully correct: 0=New, 1=WIP, 2=Built, 3=Sale, 4=Closed, 9=Voided. No TODO/FIXME markers found anywhere in file. BUT the ASCII workflow diagram still has `StatusID ? = Built` (line 345), `StatusID = Closed` without number (line 350), and `StatusID 3 = Sales` (plural, line 347). |
| 5 | MEMORY.md StatusID listings match the core skill exactly, and outdated claims about the project have been removed | VERIFIED | Type 1: MEMORY.md "0=New, 1=WIP, 2=Built, 3=Sale, 4=Closed, 9=Voided" matches core skill exactly. Type 7: MEMORY.md "6=Open, 25=Requested, 26=Approved, 27=Ordered, 28=Closed, 29=Cancelled, 30=Rejected, 31=Received" matches core skill exactly. "NOT a git repo" claim: grep returns no matches -- removed. |
| 6 | The knowledge log has a future-entry template showing the required four fields: date, query/method, finding, and source | VERIFIED | Lines 17-23: "### Entry Template" section with table showing Date, Finding, Method, Source columns. Format matches plan specification. |
| 7 | The core skill Standard Query Filters section mentions that revenue queries must include SaleDate IS NOT NULL | VERIFIED | Line 113-117: "Revenue-Specific Filters" subsection with `AND th.SaleDate IS NOT NULL -- Exclude orders not yet invoiced` and explanatory text. 4 total occurrences of "SaleDate IS NOT NULL" in the file (lines 22, 115, 158, 249). |

**Score:** 5/7 truths fully verified, 2/7 partial (knowledge log internal contradictions)

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `skills/control-erp-core/control-erp-core-SKILL.md` | Authoritative business rules -- single source of truth for TransactionType, StatusID, pricing, and query filters | VERIFIED (347 lines, substantive, all content correct) | Contains Type 10, all StatusIDs consistent, "Sale" singular throughout, revenue filter present. Zero instances of "28=Open" or "3=Sales". |
| `documentation/control-erp-knowledge-log.md` | Chronological record of all validated discoveries with structured entries | PARTIAL (1235 lines, substantive, has Entry Template) | Entry template added (line 17). Section 7 table fixed. BUT ASCII diagram has stale question marks (lines 345, 350) and three "28=Open/28 (Open)" references remain (lines 39, 51, 397). |
| `~/.claude/projects/-Users-cain-projects-control-db-map/memory/MEMORY.md` | Session-persistent quick-reference gotchas consistent with core skill | VERIFIED (50 lines, all values match core skill) | Type 1 complete with 0=New and 2=Built. Type 7 complete with 6=Open and 26=Approved. False git repo claim removed. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| MEMORY.md StatusID section | Core skill StatusID Reference | Values must match exactly | VERIFIED | Type 1: both show 0=New, 1=WIP, 2=Built, 3=Sale, 4=Closed, 9=Voided. Type 7: both show 6=Open, 25=Requested, 26=Approved, 27=Ordered, 28=Closed, 29=Cancelled, 30=Rejected, 31=Received. |
| Knowledge log Section 2 | Core skill StatusID Reference | Both authoritative, must agree | VERIFIED | Section 2 Type 7 table (line 87): 28=Closed. Core skill Type 7 (line 45): 28=Closed. Match. |
| Core skill TransactionType table Type 7 | Core skill StatusID Reference Type 7 | Internal consistency | VERIFIED | TransactionType table (line 45): "28=Closed". StatusID Reference (line 269): "28=Closed". No contradiction. |
| Knowledge log Sections 1/7 (vendor) | Knowledge log Section 2 (definitive) | Internal consistency | FAILED | Section 2 says 28=Closed. Lines 39, 51, and 397 say "28 (Open)" or "28 = Open". Internal contradiction. |

### Requirements Coverage

| Requirement | Status | Blocking Issue |
|-------------|--------|----------------|
| CORE-01: Core skill contains validated TransactionType mappings, StatusID values, pricing logic | VERIFIED | None -- all 10 types present, StatusIDs complete per type, SubTotalPrice/SaleDate filtering documented |
| CORE-02: Knowledge log structured with date/query/finding/source | PARTIAL | Entry template added. Most entries have structured format. But ASCII diagram in Section 7 has unfixed stale values, and three "28=Open" references contradict Section 2. |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `documentation/control-erp-knowledge-log.md` | 345 | `StatusID ? = Built` (question mark placeholder) | Warning | Contradicts resolved Section 7 table showing 2=Built. Reader of diagram gets wrong information. |
| `documentation/control-erp-knowledge-log.md` | 350 | `StatusID = Closed` (missing number) | Warning | Contradicts resolved Section 7 table showing 4=Closed. |
| `documentation/control-erp-knowledge-log.md` | 347 | `StatusID 3 = Sales` (plural) | Info | Core skill standardized to "Sale" (singular). Minor inconsistency. |
| `documentation/control-erp-knowledge-log.md` | 39 | `All StatusID=28 (Open)` | Warning | Label "Open" contradicts Section 2 definitive reference saying 28=Closed. |
| `documentation/control-erp-knowledge-log.md` | 51 | `StatusID 28 = Open` | Warning | Same contradiction -- should be "Closed". |
| `documentation/control-erp-knowledge-log.md` | 397 | `StatusID 28 (Open)` | Warning | Same contradiction -- should be "Closed". |

### Human Verification Required

None -- all checks are structural/textual and were verified programmatically.

### Gaps Summary

Two must-have truths are partially achieved due to residual inconsistencies in `documentation/control-erp-knowledge-log.md`. The core skill file and MEMORY.md are fully correct and consistent with each other. The knowledge log has the right data in its definitive reference tables (Section 2 StatusIDs, Section 7 StatusID table) but retains stale labels in three locations:

1. **"28=Open" vs "28=Closed"**: The knowledge log's Section 1 TransactionType table (line 39), vendor chain (line 51), and vendor workflow (line 397) all label StatusID 28 as "Open" for Type 7 POs. Section 2's definitive reference correctly says 28=Closed. The core skill and MEMORY.md both correctly say 28=Closed. This is an internal knowledge log contradiction -- 3 lines need their label changed from "Open" to "Closed."

2. **ASCII diagram stale values**: The workflow diagram in Section 7 (lines 340-357) has `StatusID ? = Built` (line 345), `StatusID = Closed` without number (line 350), and `StatusID 3 = Sales` with plural (line 347). The immediately following StatusID table (lines 377-384) has all correct values. The diagram needs 3 small text fixes.

All gaps are in one file (`documentation/control-erp-knowledge-log.md`) and involve 6 specific line-level text edits. The core skill file is clean. MEMORY.md is clean.

---

_Verified: 2026-02-09T04:30:00Z_
_Verifier: Claude (gsd-verifier)_
