# Phase 2: Schema Infrastructure - Research

**Researched:** 2026-02-08
**Domain:** Database schema documentation, FK relationship mapping, domain classification, ClassTypeID reference
**Confidence:** HIGH

## Summary

This phase verifies and completes four deliverables that already substantially exist: schema documentation (187 files), FK relationship mapping, domain classification, and a ClassTypeID reference. Research investigated the current state of each artifact by examining actual file contents, comparing table coverage across documents, and assessing completeness against the requirements.

The primary finding is that the schema files (CORE-03), FK relationships (CORE-04), and domain classification (CORE-05) are already substantially complete and require validation and minor fixes rather than significant new work. The largest gap is CORE-06: the ClassTypeID reference exists as raw data embedded in a wiki extract (356 entries in `output/wiki/extracts/database_integration_knowledge.md`) but has not been compiled into a standalone, skill-consumable reference document. The existing skill's `field_values.md` only contains 8 ClassTypeIDs out of 356.

**Primary recommendation:** This phase is primarily a verification and gap-filling exercise. The main new artifact to create is a standalone ClassTypeID reference extracted from the wiki data. Schema, relationship, and domain files need spot-check verification and minor corrections, not reconstruction.

## Standard Stack

This phase involves no code libraries. The "stack" is the MSSQL database and markdown documentation files.

### Core
| Tool | Purpose | Why Standard |
|------|---------|--------------|
| MSSQL MCP server | Query INFORMATION_SCHEMA, sys.foreign_keys, sample data | Direct database access for verification |
| Markdown files | All output artifacts | Existing format, human and skill-consumable |
| Python (if needed) | Data transformation scripts | Only if bulk ClassTypeID extraction needs automation |

### Data Sources
| Source | Location | Content | Confidence |
|--------|----------|---------|------------|
| 187 schema files | `output/schemas/*.md` | Table columns, types, nullable, row counts, FKs, sample data | HIGH -- verified complete |
| relationships.md | `output/relationships.md` | 293 lines, explicit FKs + inferred relationships + join patterns | HIGH -- reviewed in full |
| domains.md | `output/domains.md` | 384 lines, 16 domains, 181+ tables classified | HIGH -- reviewed in full |
| Wiki ClassTypeID data | `output/wiki/extracts/database_integration_knowledge.md` | 356 ClassTypeID entries with object type and table mapping | HIGH -- counted and verified |
| Existing skill field_values.md | `output/skill/references/field_values.md` | 8 ClassTypeIDs only, plus TransactionType and StatusID refs | HIGH -- reviewed |

## Architecture Patterns

### Existing Output Structure
```
output/
  schemas/           # 187 .md files, one per table
  relationships.md   # Explicit FKs + inferred relationships
  domains.md         # 16 functional domains
  wiki/extracts/     # Raw wiki knowledge including ClassTypeIDs
  skill/references/  # Skill-consumable reference files
```

### Pattern 1: Schema File Format (Established)
**What:** Each table has a markdown file with consistent sections: Overview (row count, PK), Columns (table with name/type/length/nullable), Foreign Keys (table or "none"), Notes (if needed), Sample Data.
**Verified consistent across:** Account.md, TransHeader.md, TimeCard.md, TransDetailParam.md, Warehouse.md, _ArtworkBand.md, Part.md, Employee.md, Payment.md, Shipments.md, SelectionList.md, PrCA_Job_Data.md
**Format:**
```markdown
# TableName

## Overview
- **Row Count**: N
- **Primary Key**: ID

## Columns
| Column Name | Data Type | Max Length | Nullable |
|-------------|-----------|-----------|----------|
| ... | ... | ... | ... |

## Foreign Keys
| FK Name | Column | References |
|---------|--------|------------|
| ... | ... | ... |

## Notes (optional)
- Gotchas, ClassTypeID notes, spatial column warnings

## Sample Data
- **ColumnName**: value
```

### Pattern 2: Domain Classification Format (Established)
**What:** Tables grouped by functional domain with row counts and descriptions. Each domain has a summary of key relationships.
**16 domains identified:** Accounts & Contacts, Orders & Transactions, Products & Pricing, Inventory & Parts, Finance & GL, Payroll & Time, Artwork & Proofing, Production & Workflow, Shipping, Marketing & CRM, System & Config, Tax & Compliance, Service, Education/Training, Rules & Automation, Miscellaneous/Lookup.

### Pattern 3: ClassTypeID Reference (Target Format)
**What:** A standalone lookup table mapping every ClassTypeID to its object type, database table, and usage category.
**Source data format (from wiki extract):**
```markdown
| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 2000 | Company | | Account |
| 10000 | Order/Estimate/Template/Service Tickets | | TransHeader |
```
**Target:** This exact data in a standalone file, organized by category (entities, transactions, payments, activities, system), with a quick-reference section for the ~25 most commonly used IDs.

### Anti-Patterns to Avoid
- **Rebuilding from scratch:** The schema files are complete. Do not re-extract them from the database unless spot-checks reveal errors.
- **Duplicating ClassTypeID data:** Create ONE authoritative ClassTypeID reference, not multiple partial lists scattered across documents. Currently the data appears in 3 places (relationships.md with 7 entries, field_values.md with 8 entries, wiki extract with 356 entries).
- **Inconsistent table naming:** Schema files use underscores for dotted table names (PrCA_Board_Data.md), domains.md uses dots (PrCA.Board.Data). Choose one convention and document the mapping.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| ClassTypeID extraction | Manual copy-paste from wiki | Programmatic extraction from `database_integration_knowledge.md` | 356 entries, error-prone to copy manually |
| Schema verification | Re-query all 187 tables | Spot-check 10-15 tables, query INFORMATION_SCHEMA for any found inconsistent | 187 tables would take hours; spot-checking is the success criterion |
| Table count reconciliation | Manual comparison | `ls output/schemas/ \| wc -l` vs grep on domains.md | Already done in research: 187 schema files, all accounted for |

**Key insight:** This phase is verification, not creation. The artifacts exist and are ~95% complete. The planner should structure tasks around validation spot-checks and gap-filling, not reconstruction.

## Common Pitfalls

### Pitfall 1: Table Name Convention Mismatch (Dots vs Underscores)
**What goes wrong:** Schema files use underscores (PrCA_Board_Data.md) because dots are problematic in filenames. The domains.md and relationships.md use dots (PrCA.Board.Data) matching the actual SQL Server table name. This can cause confusion when cross-referencing.
**Why it happens:** Filesystem constraints force underscore substitution.
**How to avoid:** Document the convention explicitly. When referencing these tables in the ClassTypeID reference or relationship docs, use the SQL Server dot notation with a note about the filename convention.
**Affected tables:** PrCA.Board.Data, PrCA.Board.Enum.SortType, PrCA.Board.MyBoard, PrCA.Board.StationLink, PrCA.Job.Data, PrCA.Job.TransactionLink, PrCA.Schema, Util.MacroMessageType, Util.Numbers (9 tables)

### Pitfall 2: Geography/Spatial Columns in Queries
**What goes wrong:** The TimeCard table has `LatLonStart` and `LatLonEnd` geography columns that crash `SELECT *` queries.
**Why it happens:** MS SQL geography type cannot be serialized to standard text output.
**How to avoid:** Schema files already document this (TimeCard.md has a note). Verification queries must exclude geography columns. INFORMATION_SCHEMA queries work fine since they describe metadata, not data.
**Warning signs:** Query errors mentioning "geography" or "spatial" data type.

### Pitfall 3: INFORMATION_SCHEMA vs describe_table
**What goes wrong:** The `describe_table` MCP function uses STRING_AGG which fails on some SQL Server versions.
**Why it happens:** STRING_AGG requires SQL Server 2017+.
**How to avoid:** Always use `SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = 'X'` as documented in CLAUDE.md.
**Warning signs:** STRING_AGG errors.

### Pitfall 4: Table Count Discrepancy
**What goes wrong:** domains.md header says "181 active tables" but there are 187 schema files and the domains.md body actually lists ~187 tables (some are listed as "Lookup tables" within domain descriptions, not as separate rows).
**Why it happens:** The underscore-prefixed lookup tables (_ArtworkBand, _ArtworkStatus, etc.) and the Util/PrCA tables were counted differently.
**How to avoid:** When verifying, count schema files (187) as the authoritative count. The domains.md text should be updated to match.
**What needs fixing:** The "181 active tables" text in domains.md overview should be corrected to match the actual table count.

### Pitfall 5: ClassTypeID Scattered Across Multiple Documents
**What goes wrong:** ClassTypeID values exist in 3 places with different levels of completeness (7 in relationships.md, 8 in field_values.md, 356 in wiki extract). A user looking in one place gets incomplete information.
**Why it happens:** Incremental documentation -- relationships.md was written early with known values, wiki crawl later discovered the full set.
**How to avoid:** Create one authoritative ClassTypeID reference file. Other documents should reference it, not maintain their own subset.

### Pitfall 6: TransDetailParam IsActive Trap
**What goes wrong:** Filtering IsActive=true on TransDetailParam suppresses valid data (revenue drops from $1,793K to $464K).
**Why it happens:** Control sets IsActive=false after product configuration; the data is still valid.
**How to avoid:** This is already documented in MEMORY.md and the core skill. Schema verification should NOT add "use IsActive = 1" notes to TransDetailParam.md.

## Code Examples

### Verification Query: Schema Spot-Check
```sql
-- Verify a schema file's column list matches the database
SELECT COLUMN_NAME, DATA_TYPE, CHARACTER_MAXIMUM_LENGTH, IS_NULLABLE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Account'
ORDER BY ORDINAL_POSITION
```

### Verification Query: Row Count Check
```sql
-- Verify row count for a table (compare against schema file)
SELECT COUNT(*) AS RowCount FROM dbo.Account
```

### Verification Query: FK Completeness Check
```sql
-- Get ALL foreign keys in the database for comparison with relationships.md
SELECT
    fk.name AS FK_Name,
    tp.name AS ParentTable,
    cp.name AS ParentColumn,
    tr.name AS ReferencedTable,
    cr.name AS ReferencedColumn
FROM sys.foreign_keys fk
JOIN sys.foreign_key_columns fkc ON fk.object_id = fkc.constraint_object_id
JOIN sys.tables tp ON fkc.parent_object_id = tp.object_id
JOIN sys.columns cp ON fkc.parent_object_id = cp.object_id AND fkc.parent_column_id = cp.column_id
JOIN sys.tables tr ON fkc.referenced_object_id = tr.object_id
JOIN sys.columns cr ON fkc.referenced_object_id = cr.object_id AND fkc.referenced_column_id = cr.column_id
ORDER BY tp.name, cp.name
```

### Verification Query: Table Count from Database
```sql
-- Count all user tables in the database
SELECT COUNT(*) AS TableCount
FROM sys.tables
WHERE type = 'U' AND name NOT LIKE '%_bak'
```

### ClassTypeID Quick Reference Format (Target)
```markdown
## Quick Reference: Most Used ClassTypeIDs

| ClassTypeID | What It Is | Table | Common Usage |
|-------------|-----------|-------|--------------|
| 2000 | Company/Account | Account | Customer/vendor lookup |
| 3000 | Contact | AccountContact | Contact person |
| 3500 | Employee | Employee | Staff member |
| 10000 | Transaction Header | TransHeader | Order/estimate/PO |
| 10100 | Transaction Detail | TransDetail | Line item |
| 10300 | Transaction Part | TransPart | Part used on line item |
| 12000 | Product | Product (CustomerGoodsItem) | Product definition |
| 12014 | Part | Part | Material/supply |
| 20050 | Master Time Card | TimeCard | Clock in/out parent |
| 20051 | Detail Time Card | TimeCard | Station time detail |
```

## State of the Art

This phase deals with existing documentation artifacts, not evolving technology. No "state of the art" changes apply.

| Current State | Needed State | Gap Size |
|--------------|-------------|----------|
| 187 schema files, all complete | Verified via spot-checks | SMALL -- verify, don't rebuild |
| relationships.md with explicit + inferred FKs | Same, possibly verify FK count vs database | SMALL -- one verification query |
| domains.md with 16 domains, ~181-187 tables | Fix table count text, verify all tables classified | SMALL -- text correction + verification |
| 356 ClassTypeIDs in wiki extract (embedded) | Standalone ClassTypeID reference file | MEDIUM -- extraction and formatting |
| 7-8 ClassTypeIDs in relationships.md/field_values.md | Reference new standalone file | SMALL -- update references |

## Detailed Gap Analysis

### CORE-03: Schema Documentation
**Status:** COMPLETE pending spot-check verification
**Evidence:**
- All 187 schema files exist in `output/schemas/`
- Automated check found zero files missing columns, row counts, or FK sections
- Format is consistent across all files examined (12 files spot-checked)
- Row counts present, data types documented, nullable flags included
**Remaining work:** Spot-check 5+ tables by querying the database and comparing to schema files (this IS the success criterion)

### CORE-04: FK Relationships
**Status:** COMPLETE pending verification query
**Evidence:**
- relationships.md (293 lines) contains explicit FKs organized by domain
- Inferred relationships section covers 25+ common column patterns
- Key join patterns documented (transaction chain, account-to-contact, artwork workflow, etc.)
**Remaining work:** Run the sys.foreign_keys query and compare count/content to relationships.md. Verify no new FKs have been added since initial extraction.

### CORE-05: Domain Classification
**Status:** COMPLETE with minor text corrections needed
**Evidence:**
- domains.md (384 lines) classifies tables into 16 functional domains
- All 187 tables are accounted for (9 PrCA/Util tables use dot notation matching SQL Server names)
- Each domain has description, table list with row counts, and key relationships
**Remaining work:**
1. Fix "181 active tables" count in overview to match actual 187 (or clarify the discrepancy if some are lookup/enum tables counted separately)
2. Verify the summary statistics row counts are still approximately accurate

### CORE-06: ClassTypeID Reference
**Status:** Raw data exists, needs formatting into standalone reference
**Evidence:**
- `output/wiki/extracts/database_integration_knowledge.md` contains 356 ClassTypeID entries in Section 2
- Data includes ClassTypeID value, object type name, system data flag, and database table mapping
- "Frequently Used" quick reference section (24 entries) already organized
- Existing skill's field_values.md has only 8 ClassTypeIDs
- relationships.md has only 7 ClassTypeIDs
**Remaining work:**
1. Extract Section 2 from wiki extract into standalone file (e.g., `output/classtypeid_reference.md`)
2. Organize by category (entities, transactions, products, payments, activities, system, etc.)
3. Add quick-reference section for the ~25 most-used IDs
4. Cross-reference with actual ClassTypeID values found in the database
5. Update relationships.md and field_values.md to reference the new standalone file

## Open Questions

1. **Should row counts be refreshed?**
   - What we know: Schema files have row counts from when they were extracted. Counts may have changed since then.
   - What's unclear: How stale the counts are (could be days or weeks old).
   - Recommendation: Spot-check a few table counts. If significantly different (>10%), consider refreshing all. If close, note the extraction date and move on.

2. **Where should the ClassTypeID reference live?**
   - Option A: `output/classtypeid_reference.md` (alongside other output files)
   - Option B: `output/skill/references/classtypeid_reference.md` (in the skill package)
   - Option C: Both (one authoritative, one copy in skill)
   - Recommendation: Create at `output/skill/references/classtypeid_reference.md` since the requirement says "skill-consumable format." Update the existing `field_values.md` to reference it rather than maintaining a duplicate subset.

3. **ClassTypeID 49 vs 12000 discrepancy**
   - What we know: The wiki maps ClassTypeID 49 to "Product" but FLS Banners uses GoodsItemClassTypeID 12000 on TransDetail. The skill's field_values.md already documents this FLS-specific behavior.
   - What's unclear: Whether this is an FLS customization or a Control version change.
   - Recommendation: Document both in the reference. The wiki data represents the Control platform; FLS-specific overrides should be noted separately.

## Sources

### Primary (HIGH confidence)
- Direct file inspection of 12 schema files in `output/schemas/`
- Direct review of `output/relationships.md` (293 lines, full content)
- Direct review of `output/domains.md` (384 lines, full content)
- Direct review of `output/wiki/extracts/database_integration_knowledge.md` (ClassTypeID section, 356 entries counted)
- Direct review of `output/skill/references/field_values.md` (ClassTypeID and related sections)
- Automated completeness scan of all 187 schema files (checked for columns, row count, FK sections)
- Cross-referencing between schema files, domains.md, and relationships.md

### Secondary (MEDIUM confidence)
- CLAUDE.md and MEMORY.md for query patterns and known gotchas
- Phase 1 research for cross-reference on truth-source hierarchy

### Tertiary (LOW confidence)
- None. All research was based on direct file inspection.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all artifacts directly inspected, no external dependencies
- Architecture: HIGH -- established patterns verified across multiple files
- Pitfalls: HIGH -- all pitfalls identified from direct observation of actual data or documented in MEMORY.md
- Gap analysis: HIGH -- quantified gaps with specific evidence (187 files checked, 356 ClassTypeIDs counted)

**Research date:** 2026-02-08
**Valid until:** 2026-03-08 (30 days -- these are documentation artifacts, not rapidly changing libraries)
