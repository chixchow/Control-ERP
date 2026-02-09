---
phase: 02-schema-infrastructure
plan: 02
subsystem: schema-documentation
tags: [classtypeid, domains, reference-data, skill-building]

requires:
  - phase: 02
    plan: 01
    what: Schema files and FK relationship documentation
  - phase: 03
    plan: 01
    what: Wiki crawl data with ClassTypeID mappings

provides:
  - artifact: "Corrected domains.md with accurate table count (187)"
  - artifact: "Standalone ClassTypeID reference (350+ entries)"
  - artifact: "Single authoritative source for polymorphic type identifiers"

affects:
  - phase: 02
    plan: 03
    reason: "Query pattern documentation can reference ClassTypeID lookup"
  - phase: 04
    plan: all
    reason: "Skill assembly will use ClassTypeID reference as core component"

tech-stack:
  added: []
  patterns:
    - "Polymorphic foreign key pattern (ParentID + ParentClassTypeID)"
    - "Reference document organization by ID range categories"

key-files:
  created:
    - path: "output/skill/references/classtypeid_reference.md"
      purpose: "Complete ClassTypeID lookup table (350+ entries organized by category)"
      lines: 570
  modified:
    - path: "output/domains.md"
      purpose: "Corrected table count from 181 to 187"
    - path: "output/skill/references/field_values.md"
      purpose: "Replaced ClassTypeID subset with reference pointer"

key-decisions:
  - decision: "Create standalone ClassTypeID reference instead of embedding in field_values.md"
    rationale: "350+ entries is too large for mixed-content file; dedicated reference improves findability"
    impact: "Establishes pattern for large lookup table documentation"

  - decision: "Organize ClassTypeID entries by ID range categories (System, Entities, Transactions, etc.)"
    rationale: "Logical grouping mirrors Control's design intent and aids navigation"
    impact: "Users can find relevant IDs by functional area without scanning entire list"

  - decision: "Include quick reference section with 25+ most common ClassTypeIDs"
    rationale: "80/20 rule - most queries use same 20-30 IDs repeatedly"
    impact: "Reduces lookup time for common operations"

duration: "4m 41s"
completed: "2026-02-09"
---

# Phase 02 Plan 02: Domain Classification and ClassTypeID Reference

**One-liner:** Corrected domain classification table count and compiled complete 350+ ClassTypeID reference from wiki documentation into standalone, skill-consumable lookup table.

---

## Performance

- **Duration:** 4 minutes 41 seconds
- **Started:** 2026-02-09T04:18:05Z
- **Completed:** 2026-02-09T04:22:46Z
- **Tasks completed:** 2/2
- **Files created:** 1
- **Files modified:** 2

---

## Accomplishments

### Task 1: Fixed domains.md table count discrepancy (CORE-05)

**Problem:** domains.md overview said "181 active tables" and summary statistics showed "179 tables", but 187 schema files exist in `output/schemas/`.

**Resolution:**
- Verified all 187 schema files are represented in domains.md (including PrCA.* and Util.* tables using dot notation)
- Updated overview text to accurately state "187 tables"
- Updated summary statistics TOTAL row from 179 to 187
- No restructuring needed - existing 16-domain classification was already complete

**Verification:**
```bash
# Schema files: 187
ls -1 output/schemas/*.md | wc -l

# Tables in domains.md: 187 (all accounted for)
# Notation differences handled:
#   - PrCA_Board_Data.md → PrCA.Board.Data (in domains.md)
#   - Util_MacroMessageType.md → Util.MacroMessageType (in domains.md)
```

### Task 2: Compiled standalone ClassTypeID reference (CORE-06)

**Problem:** ClassTypeID data was scattered across 3 locations:
1. Wiki extract (database_integration_knowledge.md) - 350+ entries but hard to navigate
2. field_values.md - 8 entries (incomplete)
3. MEMORY.md notes - 3 specific FLS-related entries

**Solution:** Created dedicated `classtypeid_reference.md` with:

- **Quick Reference section:** 25+ most frequently used ClassTypeIDs with usage context
- **Complete table:** All 350+ entries from wiki documentation
- **Organized by category:** 12 functional groupings (System, Entities, Transactions, Products, Payments, Activities, etc.)
- **FLS-specific notes:** GoodsItemClassTypeID 12000 (Product) and TimeCard 20050/20051 patterns
- **Usage examples:** SQL patterns for polymorphic foreign key queries
- **Cross-references:** Links to field_values.md and wiki source

**Structure:**
```
## Quick Reference (25+ common IDs)
## FLS Banners Specific Notes
## Complete ClassTypeID Table
   - System & Configuration (< 2000)
   - Entities (2000-9999)
   - Transactions (10000-10999)
   - Vendor Transactions (11000-11999)
   - Products & Pricing (12000-12999)
   - Inventory & Shipping (12100-13999)
   - Parameters & Variables (14000-14999)
   - Templates & Reports (15000-17999)
   - Payments & GL (20000-20999)
   - Activities & Journal (20050-21999)
   - User Fields & Queries (22000-22999)
   - Rules & Automation (23000-23999)
   - Replication & System (24000-25999)
   - Production & Scheduling (26000-29999)
   - Education & Service (30000+)
   - Payroll (35000+)
## Usage Examples
## Related Documentation
```

**field_values.md update:**
- Replaced incomplete 8-entry ClassTypeID section with reference pointer to classtypeid_reference.md
- Added 9-entry quick lookup table for most common IDs
- Preserved all other sections (TransactionType, StatusID, Boolean flags, etc.)

---

## Task Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | `11dac15` | fix(02-02): correct table count in domains.md |
| 2 | `7d202df` | feat(02-02): create standalone ClassTypeID reference with 350+ entries |

---

## Files Created

### output/skill/references/classtypeid_reference.md (570 lines)

Complete ClassTypeID lookup reference with:
- 350+ ClassTypeID entries from wiki documentation
- Quick reference section (25+ most common)
- FLS Banners specific usage notes
- Organized by ID range categories for easy navigation
- SQL usage examples for polymorphic FK pattern
- Cross-references to related documentation

**Key benefit:** Single authoritative source for ClassTypeID data, resolving the scattered 3-location documentation problem.

---

## Files Modified

### output/domains.md

- **Line 5:** Changed "181 active tables" → "187 tables"
- **Line 384:** Changed "TOTAL: 179" → "TOTAL: 187"
- **Impact:** Accurately reflects schema file count; no functional changes to domain classifications

### output/skill/references/field_values.md

- **ClassTypeID section:** Replaced incomplete 8-entry list with:
  - Reference pointer to classtypeid_reference.md
  - 9-entry quick lookup table
  - Explanation of polymorphic FK pattern
- **Preserved sections:** TransactionType, StatusID, Boolean flags, Payment methods, etc. (no data loss)
- **Impact:** Users now have complete ClassTypeID reference instead of incomplete subset

---

## Decisions Made

### 1. Standalone ClassTypeID reference instead of embedding in field_values.md

**Context:** Wiki extract contains 350+ ClassTypeID entries; field_values.md had only 8.

**Options considered:**
- **Option A:** Add all 350+ entries to field_values.md
- **Option B:** Create standalone classtypeid_reference.md

**Decision:** Option B (standalone reference)

**Rationale:**
- field_values.md is a mixed-content reference (TransactionType, StatusID, boolean flags, etc.)
- Adding 350+ entries would make it unwieldy (500+ lines)
- ClassTypeID is conceptually distinct - it's a polymorphic type system, not just "field values"
- Dedicated file allows better organization (categories, quick reference, examples)

**Impact:**
- Establishes pattern for large lookup tables (dedicated files for 100+ entries)
- Improves findability - users know where to look for ClassTypeID data
- Allows classtypeid_reference.md to evolve independently

### 2. Organize by ID range categories instead of alphabetical

**Context:** 350+ entries need logical organization.

**Options considered:**
- **Option A:** Alphabetical by Object Type
- **Option B:** Numerical order (raw ClassTypeID sequence)
- **Option C:** Functional categories by ID ranges

**Decision:** Option C (functional categories)

**Rationale:**
- Control's ClassTypeID design uses ranges intentionally (10000-10999 = Transactions, 20000-20999 = Payments, etc.)
- Functional grouping mirrors how developers think ("I need a payment ClassTypeID" → look in 20000-20999 range)
- Preserves Control's architectural intent

**Impact:**
- Users can navigate by domain knowledge instead of memorizing arbitrary numbers
- Reveals patterns in Control's design (e.g., all history tables are base ID + 1)

### 3. Include quick reference section with 25+ common IDs

**Context:** Most queries use same 20-30 ClassTypeIDs repeatedly.

**Decision:** Add quick reference section at top with most frequently used IDs plus usage context.

**Rationale:**
- 80/20 rule: ~20 ClassTypeIDs account for 80% of queries
- Users shouldn't scroll through 350 entries for common lookups
- "Common Usage" column provides context for when to use each ID

**Impact:**
- Reduces lookup time for routine queries
- Educates users on typical usage patterns
- Quick reference can serve as learning tool for new developers

---

## Deviations from Plan

None - plan executed exactly as written.

---

## Issues Encountered

None. Plan execution was straightforward:
1. Python script verified all 187 tables accounted for in domains.md
2. Wiki extract Section 2 contained complete ClassTypeID data ready for extraction
3. No data quality issues or missing entries

---

## Next Phase Readiness

### Blockers
None.

### Concerns
None.

### Recommendations

**For Phase 02 Plan 03 (Query Pattern Documentation):**
- Reference classtypeid_reference.md when documenting polymorphic FK join patterns
- Example: "To join TransDetail to TransHeader, use ParentID + ParentClassTypeID (10000)"

**For Phase 04 (Skill Assembly):**
- classtypeid_reference.md is ready for inclusion in skill package
- Consider adding to SKILL.md frontmatter as a key reference file

**For Future Phases:**
- ClassTypeID reference can be expanded with:
  - Frequency analysis (which IDs are actually used at FLS vs theoretical wiki list)
  - Common join patterns by ClassTypeID
  - Troubleshooting guide (missing ClassTypeID foreign keys)

---

## Verification Results

All success criteria met:

- ✅ domains.md groups every table into a functional domain with accurate total count (187)
- ✅ ClassTypeID reference (350+ entries) exists in structured, skill-consumable format
- ✅ Quick-reference section contains 25+ most commonly used ClassTypeIDs
- ✅ field_values.md defers to the new reference (no duplicate incomplete subset)
- ✅ Single authoritative source for ClassTypeID data established

**CORE-05 (domains.md table count):** VERIFIED - Text matches actual schema file count
**CORE-06 (ClassTypeID reference):** COMPLETE - Standalone reference file created with all 350+ entries
