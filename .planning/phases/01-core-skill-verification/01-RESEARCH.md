# Phase 1: Core Skill Verification - Research

**Researched:** 2026-02-08
**Domain:** ERP skill file verification and knowledge documentation structure
**Confidence:** HIGH

## Summary

This phase is about reviewing and verifying two existing documents -- `skills/control-erp-core/control-erp-core-SKILL.md` (the core business rules skill) and `documentation/control-erp-knowledge-log.md` (the knowledge log) -- rather than building new software. The "stack" is the documents themselves and the data model they describe. Research focused on identifying contradictions between all truth sources (core skill, knowledge log, CLAUDE.md, MEMORY.md), cataloging completeness gaps in the core skill, and assessing the knowledge log's structural fitness for CORE-02.

The core skill is substantially complete and well-structured. The knowledge log is thorough but has structural inconsistencies (unresolved TODOs that were actually resolved elsewhere in the same document, StatusText label mismatches between sections). The most significant finding is a set of concrete contradictions between the four truth sources that must be resolved to achieve the "single source of truth" success criterion.

**Primary recommendation:** Systematically resolve the 8 identified contradictions between truth sources, fill 4 completeness gaps in the core skill (Types 3/4/10, StatusID 28 for Type 7), and restructure knowledge log Section 7 to remove stale TODO markers.

## Standard Stack

This phase involves no libraries or code. The "stack" is:

### Core
| Artifact | Location | Purpose | Authority Level |
|----------|----------|---------|----------------|
| control-erp-core-SKILL.md | `skills/control-erp-core/` | Authoritative business rules for all other skills | PRIMARY -- single source of truth |
| control-erp-knowledge-log.md | `documentation/` | Chronological record of all validated discoveries | REFERENCE -- feeds skill updates |
| CLAUDE.md | project root | Project instructions, references core skill for business rules | DIRECTIVE -- points to core skill |
| MEMORY.md | `~/.claude/projects/.../memory/` | Session-persistent memory, quick-reference gotchas | CACHE -- must stay consistent with core skill |

### Authority Hierarchy (Established by CLAUDE.md)
CLAUDE.md explicitly states: "The authoritative source for all Control ERP business rules -- including TransactionType mappings, StatusID values, price field selection, and query patterns -- is **skills/control-erp-core/control-erp-core-SKILL.md**. Do not duplicate those mappings here; always defer to the core skill."

This means the resolution direction for contradictions is: **Core Skill wins** (update the others), unless the knowledge log has more recent validated data (then update the core skill).

## Architecture Patterns

### Pattern 1: Single Source of Truth
**What:** Business rules live in exactly one place (the core skill). All other documents reference it.
**When to use:** Always -- this is the established pattern.
**Structure:**
```
Core Skill (authoritative)
  ^
  |-- CLAUDE.md (references, never duplicates)
  |-- MEMORY.md (caches key gotchas, references core for full rules)
  |-- Knowledge Log (chronological discoveries, feeds INTO core skill updates)
  |-- Sales Skill (depends on core, references it)
  |-- Financial Skill (depends on core, references it)
```

### Pattern 2: Knowledge Log Entry Structure
**What:** Each discovery entry must include date, query/method, finding, and source.
**Current structure (from knowledge log):** Mostly follows this pattern with tables containing Date, Finding, Confirmation Method, and Notes columns.
**Gap:** Not every entry has a formal "query" column -- some discoveries came from wiki reading or user confirmation rather than SQL queries. The structure should accommodate multiple source types (SQL query, wiki page, user confirmation, Control report comparison).

### Pattern 3: Correction-Over-Deletion
**What:** When a finding is corrected, the old value is preserved with a correction note rather than deleted.
**Current structure:** Section 13 (Corrections Log) tracks this well with Old Value, New Value, and Why columns.
**This pattern is working correctly and should be preserved.**

### Anti-Patterns to Avoid
- **Scattershot StatusID references:** StatusID values currently appear in at least 4 different places in the knowledge log (Sections 1, 2, 7, and 13) with slight inconsistencies. Consolidate to Section 2 as authoritative, reference it elsewhere.
- **Stale TODOs alongside resolved answers:** Section 7 has `StatusID ? = Built` and `StatusID ? = Closed` with a TODO to confirm, but Section 2 already contains the answers (2=Built, 4=Closed). This creates false uncertainty.
- **Partial StatusID listings in MEMORY.md:** MEMORY.md lists some StatusIDs per type but omits others (e.g., Type 1 omits 0=New and 2=Built; Type 7 omits 6=Open and 26=Approved). Either list them all or don't list them.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Contradiction detection | Manual side-by-side reading | Systematic checklist (provided below) | There are 4 source files, each 200-1200 lines. Spot-checking misses things. |
| Knowledge log restructuring | Rewrite from scratch | Targeted updates to specific sections | The log is 1,229 lines of validated discoveries. Rewriting risks losing detail. |
| StatusID reference | Multiple partial tables in different files | Single comprehensive table in core skill, referenced everywhere else | Partial duplicates inevitably diverge |

**Key insight:** This phase is about editorial consistency across documents, not about generating new content. The data is already validated. The work is reconciliation.

## Common Pitfalls

### Pitfall 1: Contradictions Between Truth Sources
**What goes wrong:** Four files contain overlapping business rules with subtle differences. Future sessions may read any of them and get slightly different information.
**Why it happens:** The knowledge was discovered iteratively over multiple sessions. Early findings were corrected later, but not all files were updated consistently.
**How to avoid:** Resolve every contradiction identified below, establishing the core skill as the single source.
**Warning signs:** Any place where StatusID, TransactionType, or field name appears in more than one file with different values.

**Specific contradictions found (HIGH confidence -- verified by reading all 4 files):**

| # | Contradiction | Source A | Source B | Resolution |
|---|--------------|----------|----------|------------|
| 1 | **MEMORY.md says "Project is NOT a git repo"** | MEMORY.md: "No .git directory exists" | Git status: master branch, 5 commits | Update MEMORY.md -- project IS a git repo now |
| 2 | **Type 1 StatusID completeness in MEMORY.md** | MEMORY.md: "1=WIP, 3=Sale, 4=Closed, 9=Voided" | Core skill: "0=New, 1=WIP, 2=Built, 3=Sales, 4=Closed, 9=Voided" | MEMORY.md omits 0=New and 2=Built; update to match core skill |
| 3 | **Type 7 StatusID completeness in MEMORY.md** | MEMORY.md: lists 25, 27, 28, 29, 30, 31 | Core skill & knowledge log: adds 6=Open, 26=Approved | MEMORY.md omits 6=Open and 26=Approved; update to match |
| 4 | **Type 7 StatusID 28 meaning** | Core skill: "28=Open" | Knowledge log Section 2: "28=Closed"; MEMORY.md: "28=Closed" | Knowledge log and MEMORY.md agree on 28=Closed. The wiki reference also says 28=Closed. **Core skill is wrong here** -- update core skill to 28=Closed |
| 5 | **Knowledge log Section 1 vs Section 2 StatusText labels** | Section 1: Type 1 "1=New, 3=Sold, 4=Completed" | Section 2: "0=New, 1=WIP, 3=Sale, 4=Closed" | Section 2 was validated from wiki and is correct. Section 1 has stale labels from early discovery |
| 6 | **Knowledge log Section 7 unresolved TODO** | Section 7: "StatusID ? = Built" and "StatusID ? = Closed" with TODO | Section 2: 2=Built, 4=Closed (resolved from wiki) | Section 7 has stale markers; update with resolved values |
| 7 | **Core skill omits Types 3, 4, 10** | Core skill: Types 3, 4, 5 listed as "UNUSED" | Knowledge log: Type 3=Recurring Order (StatusID 21), Type 4=Credit Memo (StatusID 20/9), Type 10=Vendor Credit Memo (StatusID 24) | Core skill should include Types 3, 4, 10 with their StatusIDs even though FLS has zero/few records. "UNUSED at FLS" is more accurate than "UNUSED". |
| 8 | **StatusID 3 naming: "Sale" vs "Sales"** | Core skill TransactionType table: "3=Sales" | Core skill StatusID section: "3=Sale" | Minor inconsistency -- pick one and use it consistently. Wiki uses "Sale" (singular). |

### Pitfall 2: Knowledge Log Structural Issues
**What goes wrong:** The knowledge log has grown organically to 1,229 lines. Some sections have internal inconsistencies where early entries were corrected in later sections but the original text was not updated.
**Why it happens:** The correction-over-deletion pattern was applied at the section level but not within sections. Section 7 still shows `?` for values that Section 2 resolved.
**How to avoid:** Cross-reference within the knowledge log itself during verification pass.

### Pitfall 3: Over-Editing the Knowledge Log
**What goes wrong:** Attempting to "clean up" the knowledge log removes the discovery narrative -- the corrections and how understanding evolved is itself valuable context.
**Why it happens:** Natural instinct to make documents "clean" conflicts with the log's purpose as a chronological record.
**How to avoid:** Keep the correction-over-deletion pattern. Only fix factual errors and stale markers. Never remove corrected entries from Section 13.

### Pitfall 4: Missing SaleDate IS NOT NULL in Core Skill Standard Filters
**What goes wrong:** The "Standard Query Filters" section of the core skill shows `WHERE th.IsActive = 1 AND th.StatusID != 9` as the minimum filters, but does NOT mention `SaleDate IS NOT NULL` even though the revenue query formula at the top of the file includes it.
**Why it happens:** The Standard Filters section is a general-purpose filter, while the revenue formula is specific. But since most queries will involve revenue, the omission could lead to including un-sold orders in revenue totals.
**How to avoid:** Add a note in Standard Filters that revenue queries must additionally include `AND SaleDate IS NOT NULL`, or add a "Revenue-Specific Filters" subsection.

## Code Examples

Not applicable -- this phase produces markdown documents, not code. However, the verification process should check that all SQL examples in the core skill are syntactically correct and follow the documented rules. Key SQL patterns to verify:

### Revenue Query (already in core skill, verified correct)
```sql
-- THE canonical revenue query
SELECT SUM(SubTotalPrice) AS Revenue
FROM TransHeader
WHERE TransactionType = 1
    AND IsActive = 1
    AND SaleDate IS NOT NULL
    AND YEAR(SaleDate) = @Year
```

### TransDetailParam JOIN (already in core skill, verified correct)
```sql
-- Correct: No IsActive or ParentClassTypeID filter
INNER JOIN TransDetailParam tdp ON tdp.ParentID = td.ID
    AND tdp.VariableID = @VariableID
```

### Patterns that should be verified present in the core skill
- Header-to-Detail JOIN with `td.IsActive = 1`
- Header-to-Customer via `Account.ID` (using `CompanyName`, not `AccountName`)
- Estimate date filtering via `EstimateCreatedDate`
- Web import deduplication via ROW_NUMBER

All of these are currently present and correctly documented in the core skill.

## State of the Art

Not applicable in the traditional sense (no library versions). The "state" is the current accuracy of the skill files.

| Previous State | Current State | When Changed | Impact |
|---------------|---------------|--------------|--------|
| Three conflicting TransactionType definitions | Single validated reference in core skill | 2025-02-07 | All other skills now defer to core |
| TransDetailParam with IsActive filter | No filter (75% more data) | 2025-02-07 | DyeSub Print revenue went from $464K to $1.79M |
| TotalPrice for revenue | SubTotalPrice for revenue | 2025-02-07 | 99.98% accuracy match |
| OrderCreatedDate for filtering | SaleDate (Type 1) / EstimateCreatedDate (Type 2) | 2025-02-07 | Correct date semantics |

**All major corrections are already captured in the skill.** This phase is about consistency verification, not discovery.

## Completeness Assessment

### Core Skill (CORE-01) Completeness Check

| Section | Present? | Complete? | Notes |
|---------|----------|-----------|-------|
| Revenue Query Formula | Yes | Yes | Validated to 99.98% |
| TransactionType Reference | Yes | **Partial** | Missing Types 3 (Recurring), 4 (Credit Memo), 10 (Vendor Credit Memo) from wiki data |
| Natural Language Mapping | Yes | Yes | Comprehensive |
| Standard Query Filters | Yes | **Partial** | Revenue queries need `SaleDate IS NOT NULL` explicitly mentioned |
| Date Filtering Patterns | Yes | Yes | Both Type 1 and Type 2 covered |
| Schema Quick Reference | Yes | Yes | Core hierarchy, JOINs, value storage |
| Pricing Fields Reference | Yes | Yes | Both header and detail levels |
| StatusID Reference | Yes | **Partial** | Type 7 StatusID 28 says "Open" but should be "Closed"; Types 3/4/10 StatusIDs missing |
| High-Volume Tables | Yes | Yes | Performance tips included |
| GL/Ledger Architecture | Yes | Yes | View vs table, sign conventions |
| Multi-Division Context | Yes | Yes | Company vs Apparel |
| History Tables | Yes | Yes | Brief but sufficient |
| Web Import | Yes | Yes | Deduplication pattern included |

### Knowledge Log (CORE-02) Completeness Check

| Section | Present? | Structured? | Issues |
|---------|----------|-------------|--------|
| 1. Transaction Types | Yes | Table format | StatusText labels don't match Section 2 |
| 2. Status IDs | Yes | Table format | Complete and well-structured |
| 3. Pricing Fields | Yes | Table format | Well-structured |
| 4. Product Architecture | Yes | Table + narrative | Well-structured |
| 5. Schema & JOIN Patterns | Yes | Table + code | Well-structured |
| 6. User Defined Fields | Yes | Table format | Well-structured |
| 7. Business Workflow Rules | Yes | Flowchart + narrative | Stale TODO markers for already-resolved questions |
| 8. General Ledger | Yes | Narrative + tables + code | Comprehensive |
| 9. Accounts Receivable | Yes | Tables + code | Comprehensive |
| 10. Crystal Reports | Yes | **Stub** | Only 1 report cataloged |
| 11. CHAPI | Yes | **Stub** | Empty table |
| 12. Open Questions | Yes | Table format | 4 unresolved items -- all are opportunities, not blockers |
| 13. Corrections Log | Yes | Table format | Complete and well-maintained |

**Knowledge log structural assessment:** Each entry does have date and finding, but the "query" and "source" fields are not consistently formatted. Some entries use "Confirmed by" column, others describe the method inline. The requirement says "date, query, finding, and source for every validated discovery." The format is close but not uniform.

## Open Questions

Things that could not be fully resolved during research:

1. **Type 7 StatusID 28: "Open" vs "Closed"**
   - What we know: The core skill says 28=Open. The knowledge log (sourced from wiki) says 28=Closed. MEMORY.md says 28=Closed.
   - What's unclear: Whether the core skill's "28=Open" was a typo or reflected a different source. The knowledge log cites wiki as source for 28=Closed.
   - Recommendation: Trust the wiki-sourced value (28=Closed). Update core skill. This should be verified against live data during execution: `SELECT DISTINCT StatusID, StatusText FROM TransHeader WHERE TransactionType = 7 AND StatusID = 28`

2. **Knowledge log entry format standardization**
   - What we know: CORE-02 requires "date, query, finding, and source." Current entries have these but in varying formats.
   - What's unclear: How strict the formatting needs to be. Some entries are narrative-style; others are table-style.
   - Recommendation: Don't reformat existing entries (too much churn). Add a template at the top of the knowledge log for future entries. Focus on ensuring all entries at least HAVE the four required elements, even if formatting varies.

3. **Types 3, 4, 10 -- FLS usage**
   - What we know: Wiki documents these types. Knowledge log confirms zero FLS records for Types 3 and 5. Type 4 and Type 10 have wiki-sourced StatusIDs but haven't been validated against FLS data.
   - What's unclear: Whether FLS has any Type 4 or Type 10 records in the database.
   - Recommendation: Include Types 3, 4, 10 in core skill with wiki-sourced StatusIDs, marked as "0 records at FLS" or "not validated at FLS" as appropriate.

## Sources

### Primary (HIGH confidence)
- `skills/control-erp-core/control-erp-core-SKILL.md` -- read in full (341 lines)
- `documentation/control-erp-knowledge-log.md` -- read in full (1,229 lines)
- `CLAUDE.md` -- read in full (focus on business rules section)
- `~/.claude/projects/.../memory/MEMORY.md` -- read in full (53 lines)
- `skills/control-erp-sales/control-erp-sales-SKILL.md` -- read in full (368 lines)
- `skills/control-erp-financial/control-erp-financial-SKILL.md` -- read in full (576 lines)
- `.planning/REQUIREMENTS.md` -- read in full (requirements CORE-01, CORE-02)
- `.planning/ROADMAP.md` -- read in full (phase definition, success criteria)

### Secondary (MEDIUM confidence)
- Git status output -- confirms project is a git repo (contradicting MEMORY.md)

### Tertiary (LOW confidence)
- None. All research was based on direct file reading.

## Metadata

**Confidence breakdown:**
- Contradictions identified: HIGH -- all identified by direct comparison of source files
- Completeness assessment: HIGH -- systematic section-by-section review
- Resolution recommendations: HIGH for most, MEDIUM for Type 7 StatusID 28 (should be verified against live data)

**Research date:** 2026-02-08
**Valid until:** Indefinite -- this research covers static document analysis, not evolving technology
