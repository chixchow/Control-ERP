# Phase 3: Wiki Knowledge Formalization - Research

**Researched:** 2026-02-08
**Domain:** Content verification, quality audit, reference document formalization
**Confidence:** HIGH

## Summary

Phase 3 is a documentation formalization phase, not a software engineering phase. The wiki has already been crawled (1,769 pages per crawl state), 12 knowledge extracts have been produced, and extensive reference material exists in organized directories. The work here is to verify completeness, reconcile discrepancies between requirements and actual outputs, audit extract quality, and formalize three specific reference documents (CHAPI architecture, Crystal Reports SQL patterns, macro automation system).

Research examined every relevant file on disk: crawl state, KNOWLEDGE_BASE.md, INDEX.md, CATALOG.md, all 12 extracts, the reference/ directory tree (125 files across 6 subdirectories), the Crystal Report analysis files in output/reports/ (36 reports), and the wiki's reports subdomain (13 standard report documentation pages with actual table/join details).

**Primary recommendation:** Structure the phase into two plans: (1) completeness verification and discrepancy reconciliation, (2) formalization of CHAPI, Crystal Reports SQL, and macro automation references. All the source material already exists -- the work is audit, cross-reference, and consolidation.

## Standard Stack

This phase requires no external libraries or frameworks. It is a content audit and documentation formalization phase.

### Tools Needed
| Tool | Purpose | Why |
|------|---------|-----|
| File reading/comparison | Verify crawl state vs disk files | Reconcile page count discrepancies |
| Python (stdlib only) | Optional validation scripts | Count pages, verify extract coverage, check links |
| Markdown | Output format | All deliverables are .md files |

### No Libraries Required
This phase works exclusively with existing files on disk. No npm install, no pip install, no dependencies.

## Architecture Patterns

### Existing Directory Structure (already established)
```
output/wiki/
├── KNOWLEDGE_BASE.md          # Master index (282 lines)
├── INDEX.md                   # Full page listing by subdomain (~4000+ lines)
├── CATALOG.md                 # All pages categorized with summaries (18,667 lines)
├── TOPICS.json                # Machine-readable topic index (23,455 lines)
├── categories/                # 16 category files
├── extracts/                  # 12 knowledge extract files (508 KB total)
├── reference/                 # 125 exact code/SQL files (913 KB total)
│   ├── stored_procedures/     # 17 files - full T-SQL source
│   ├── sql_bridge/            # 10 files - SQL Bridge core docs
│   ├── sql_functions/         # 3 files
│   ├── sql_queries/           # 71 files - diagnostic/utility SQL
│   ├── chapi/                 # 18 files - CHAPI service docs
│   └── sslip/                 # 6 files
├── control/                   # 1,146 raw crawled pages
├── reports/                   # 24 raw files (46 crawl state entries, see discrepancy below)
└── support/                   # 577 raw crawled pages
```

### Deliverable Pattern
Phase 3 deliverables should be **consolidated reference documents** that synthesize information from multiple sources (extracts + reference files + raw pages) into developer-consumable format. Not raw dumps, not duplicates of existing extracts -- curated, cross-referenced, and actionable.

### Anti-Patterns to Avoid
- **Re-crawling the wiki:** The crawl is complete. Do not re-run crawl_wiki.py unless a specific gap is identified.
- **Duplicating extract content:** Don't copy extracts into new files. Reference them, cross-reference them, or synthesize them.
- **Replacing existing files:** The existing KNOWLEDGE_BASE.md, CATALOG.md, extracts/, and reference/ are assets. Formalization adds to them, doesn't replace them.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Page count verification | Manual file counting | Python script against _crawl_state.json + disk listing | Crawl state has 1,769 entries; disk has different counts per subdomain |
| Extract coverage audit | Reading every extract by hand | Systematic section-header scan across all 12 extracts | 13,413 total lines across 12 files |
| Cross-reference validation | Manual comparison | Side-by-side check of wiki reports pages vs output/reports/ analysis | 13 standard report wiki pages have actual join info |

## Common Pitfalls

### Pitfall 1: Reports Subdomain Page Count Discrepancy
**What goes wrong:** Requirement WIKI-01 says 1,769 pages. Crawl state JSON confirms 1,769 downloaded entries (1,146 + 46 + 577). But the reports/ directory on disk only has 24 files (not 46). The KNOWLEDGE_BASE.md also says "46 pages" for reports subdomain.
**Why it happens:** The crawl state has both `reports/page_name` and `reports/pages:page_name` entries for the same content. The `pages:` prefix entries are DokuWiki namespace variants that may resolve to the same underlying content. The actual unique content is 23 report pages + 1 home page = 24 files on disk.
**How to avoid:** When verifying WIKI-01, count crawl state entries (1,769) as the authoritative number. Note that 22 of the 46 reports entries are `pages:` namespace duplicates. Total unique pages on disk = 1,146 + 24 + 577 = 1,747, but crawl state correctly records 1,769 download attempts.
**Resolution approach:** Document the `pages:` namespace deduplication in the verification output. The crawl is complete -- all 1,769 URLs were visited and content downloaded. The 22 `pages:` entries are just DokuWiki's namespace aliasing, not missing content.

### Pitfall 2: Extract Count Discrepancy (6 vs 12)
**What goes wrong:** REQUIREMENTS.md says "6 knowledge extracts" but there are 12 on disk.
**Why it happens:** The original REQUIREMENTS.md was written based on the initial 6 extracts (database_integration, orders_accounting, production_inventory, crm_payroll, sql_queries, howto_troubleshooting). Six more were added later (cfl_formula_language, macros_automation, parts, products_pricing, udfs_custom_fields, email_notifications) during a second extraction pass.
**How to avoid:** Treat the requirement as met and exceeded. The 12 extracts are a superset of the required 6. The original 6 named in WIKI-02 all exist. Verification should confirm all 12 exist and cover their domains, not reduce to 6.
**Resolution approach:** Update the verification to note "12 extracts produced (6 required, 6 additional)" and verify all 12 for quality.

### Pitfall 3: Crystal Reports Analysis is Inferred, Not Parsed
**What goes wrong:** The 36 report analysis files in `output/reports/` contain INFERRED SQL and joins -- they explicitly state "SQL queries and table references were inferred from the report filename and database schema knowledge. The .rpt file uses SAP Crystal Reports' proprietary encrypted OLE compound document format and cannot be parsed without the Crystal Reports SDK."
**Why it happens:** .rpt files are encrypted/proprietary binary format that cannot be reliably parsed.
**How to avoid:** For WIKI-04 (Crystal Reports SQL patterns extracted from wiki), use the wiki's reports subdomain pages as the AUTHORITATIVE source. These 13 `*_standard.md` pages contain the ACTUAL table linkages, joins, and filter information documented by Cyrious. Cross-reference these against the inferred analysis in `output/reports/` to validate or correct the inferences.
**Key data:** The wiki reports subdomain has 13 pages documenting standard reports with actual table/join details:
  1. ar_detail_report_standard
  2. ar_summary_report_standard
  3. deposits_made_report_standard
  4. estimate_report_standard
  5. gl_listing_report_standard
  6. historical_ar_report_standard
  7. invoice_report_standard
  8. orders_placed_between_report_standard
  9. sales_by_x_report_standard
  10. sales_report_standard
  11. wip_and_built_report_standard
  12. wip_by_line_item_report_standard
  13. work_order_report_standard

### Pitfall 4: CHAPI Documentation is Already Comprehensive
**What goes wrong:** Over-scoping the CHAPI formalization work by treating it as if CHAPI docs need to be created from scratch.
**Why it happens:** Not recognizing that `reference/chapi/` already has 18 files and `extracts/database_integration_knowledge.md` already covers SQL Bridge + CHAPI architecture in detail.
**How to avoid:** The CHAPI formalization task should CONSOLIDATE existing material into a single developer-facing reference, not research CHAPI from scratch. Source material:
  - `reference/chapi/` (18 files, 40 KB) -- installation, configuration, endpoint, troubleshooting
  - `reference/sql_bridge/` (10 files, 90 KB) -- SQL Bridge core functions, ClassTypeID table
  - `reference/stored_procedures/` (17 files, 539 KB) -- full T-SQL for import procedures
  - `extracts/database_integration_knowledge.md` (1,155 lines) -- synthesized architecture docs

### Pitfall 5: Macro Documentation is Already Complete
**What goes wrong:** Treating WIKI-05 as requiring new research when the macro extract already contains the complete 95 message type reference and all RuleAction types.
**Why it happens:** Not recognizing that `extracts/macros_automation_knowledge.md` (1,172 lines) already has:
  - All 95 Macro Message Types (IDs 0-94) with names and NeedsAltID flags
  - All 14 macro categories with entity/table mappings
  - All 10 action types with ClassTypeIDs
  - Complete trigger event reference for all entity types
  - 11 real-world automation patterns with SQL examples
  - Database table schemas (RuleMacro, RuleAction, RuleActivity)
**How to avoid:** Verification should CONFIRM the extract is complete and accurate, not redo the work. The formalization step should create a clean, well-structured reference from the existing extract.

## Code Examples

No code is needed for this phase. All work is content audit and document creation.

### Verification Script Pattern (if needed)
```python
# Pattern for verifying crawl completeness
import json, os

WIKI_DIR = '/Users/cain/projects/control-db-map/output/wiki'

# Load crawl state
with open(f'{WIKI_DIR}/_crawl_state.json') as f:
    state = json.load(f)

# Count by subdomain
for subdomain in ['control', 'reports', 'support']:
    state_count = sum(1 for k in state['downloaded'] if k.startswith(f'{subdomain}/'))
    disk_count = len(os.listdir(f'{WIKI_DIR}/{subdomain}'))
    # Note: reports/ has pages/ subdirectory that contains duplicates
    print(f'{subdomain}: state={state_count}, disk={disk_count}')
```

### Cross-Reference Pattern (Crystal Reports)
```python
# Pattern for cross-referencing wiki report docs with inferred analysis
wiki_reports = [f for f in os.listdir(f'{WIKI_DIR}/reports/')
                if f.endswith('_standard.md')]
inferred_reports = [f for f in os.listdir('output/reports/')
                    if f.endswith('.md')]
# Match wiki-documented reports to inferred analysis files
# Wiki has authoritative join info; inferred analysis may need correction
```

## State of the Art

This section documents what exists NOW vs what the requirements expect.

### Current State vs Requirements

| Requirement | Expected State | Actual State | Gap |
|-------------|---------------|--------------|-----|
| WIKI-01: 1,769 pages crawled | 1,769 pages across 3 subdomains | Crawl state confirms 1,769 entries. Disk files: 1,747 unique (22 `pages:` duplicates in reports) | NONE - crawl is complete. Discrepancy is namespace aliasing, not missing content |
| WIKI-02: 6 knowledge extracts | 6 extracts covering named domains | 12 extracts exist (superset of required 6) | EXCEEDED - 12 vs 6. All original 6 present |
| WIKI-03: CHAPI architecture | Developer-facing CHAPI docs | 18 CHAPI reference files + database_integration extract with full SQL Bridge docs | EXISTS but needs CONSOLIDATION into single reference |
| WIKI-04: Crystal Reports SQL | SQL patterns from wiki | 13 wiki report pages with actual table/join info + 36 inferred analysis files + report_join_patterns.md (inferred) | EXISTS in wiki but needs CROSS-REFERENCE with inferred analysis |
| WIKI-05: Macro automation | 95 trigger events + RuleAction types | macros_automation extract has complete 95 message types + all action types + 11 patterns | EXISTS and is comprehensive. Needs VERIFICATION only |

### What Needs To Be Done (Ordered by Effort)

1. **Verify crawl completeness** (LOW effort): Confirm 1,769 crawl state entries, document the `pages:` namespace deduplication, verify all 3 subdomains have expected file counts.

2. **Audit extract quality** (MEDIUM effort): Review each of the 12 extracts for:
   - Actionable content (not raw page dumps)
   - Coverage of stated domain
   - Correct SQL/code examples
   - Cross-references to related extracts

3. **Formalize CHAPI reference** (MEDIUM effort): Consolidate from 18 reference files + database_integration extract into a single developer-facing document covering:
   - HTTP endpoint (port 12556, URL format, parameter passing)
   - SQL Bridge core functions (nextid, nextnumber, lock, unlock, refresh, recompute)
   - Insert/Update/Delete operation patterns
   - Import stored procedure catalog (17 procedures)
   - CHAPI service installation/configuration

4. **Formalize Crystal Reports SQL reference** (MEDIUM effort):
   - Extract actual table/join info from 13 wiki `*_standard.md` pages
   - Cross-reference with inferred analysis in `output/reports/`
   - Note where wiki-documented joins differ from inferences
   - Create authoritative join pattern reference

5. **Verify macro automation reference** (LOW effort): Confirm macros_automation_knowledge.md contains:
   - All 95 message types (IDs 0-94) -- VERIFIED present
   - All RuleAction types with ClassTypeIDs -- VERIFIED present
   - Trigger events per entity type -- VERIFIED present
   - Possibly create a clean quick-reference summary

## Detailed Findings

### Finding 1: Crawl State vs Disk File Count
**Confidence:** HIGH (directly verified against files)

The crawl state JSON has exactly 1,769 `downloaded` entries:
- `control/`: 1,146 entries, 1,146 files on disk -- MATCH
- `reports/`: 46 entries, 24 files on disk -- EXPLAINED by `pages:` namespace duplication
- `support/`: 577 entries, 577 files on disk -- MATCH

The 22 "extra" crawl state entries in `reports/` are DokuWiki namespace aliases (e.g., both `reports/sales_report_standard` and `reports/pages:sales_report_standard` point to the same content). The crawl correctly downloaded both URL forms but they resolve to the same page content. Disk files correctly deduplicate to 24 unique files.

Additionally, the `reports/` directory contains a `pages/` subdirectory with duplicate copies, further confirming this is namespace aliasing.

### Finding 2: Extract Inventory (12 files, not 6)
**Confidence:** HIGH (directly verified against files)

All 12 extracts with line counts and sizes:

| Extract | Lines | Size | Original 6? |
|---------|-------|------|-------------|
| database_integration_knowledge.md | 1,155 | 50 KB | YES |
| orders_accounting_knowledge.md | 816 | 30 KB | YES |
| production_inventory_knowledge.md | 796 | 31 KB | YES |
| crm_payroll_system_knowledge.md | 838 | 37 KB | YES |
| sql_queries_reference.md | 1,480 | 49 KB | YES |
| howto_troubleshooting_knowledge.md | 980 | 31 KB | YES |
| cfl_formula_language_knowledge.md | 1,650 | 68 KB | No (additional) |
| macros_automation_knowledge.md | 1,172 | 46 KB | No (additional) |
| parts_knowledge.md | 1,259 | 48 KB | No (additional) |
| products_pricing_knowledge.md | 1,268 | 48 KB | No (additional) |
| udfs_custom_fields_knowledge.md | 1,172 | 47 KB | No (additional) |
| email_notifications_knowledge.md | 827 | 34 KB | No (additional) |

Total: 13,413 lines, ~519 KB

### Finding 3: CHAPI Source Material Inventory
**Confidence:** HIGH (directly verified against files)

Material available for CHAPI formalization:

**reference/chapi/ (18 files, 40 KB):**
- `chapi.md` -- Overview
- `chapi_home.md` -- Home page
- `chapi_url_endpoint_listener.md` -- **KEY**: HTTP endpoint on port 12556, URL format: `http://{server}:12556/chapi/sqlmacro?name={ProcName}&{param}={value}`, example stored procedure
- `chapi_integration_reverse_proxy.md` -- Reverse proxy setup
- 14 installation/configuration pages (service creation, DB connection, passwords, troubleshooting)

**reference/sql_bridge/ (10 files, 90 KB):**
- `sql_bridge.md` -- **KEY**: Master reference with Lock/Unlock/NextID/NextNumber/Refresh/RefreshEx/Recompute + all 94 Macro Message Types
- `database_table_by_object_type_and_classtypeid.md` -- Complete 350+ ClassTypeID mapping
- 4 example files (time clock, scheduled payments, notes, auto-SKU)
- 4 utility/troubleshooting files

**reference/stored_procedures/ (17 files, 539 KB):**
- Full T-SQL source for: import_company, import_contact, import_order, import_estimate, import_line_item, import_payment, import_address, import_shipping_information, import_shipping_information_detail, import_part, import_part_category, insert_part_on_a_line_item, convert_estimate_to_order, add_note, customer_list_import, quickbooks_import, import_text_file_to_cols

**extracts/database_integration_knowledge.md (1,155 lines):**
- SQL Bridge architecture and all core functions
- Complete ClassTypeID mapping table (350+ entries)
- All 95 Macro Message Types
- Operation patterns (insert, update, delete)
- Key rules for safe database writes

### Finding 4: Crystal Reports Wiki Pages with Actual SQL
**Confidence:** HIGH (directly verified against files)

The reports subdomain has 13 `*_standard.md` pages that document **actual** Crystal Report structure with real table names, joins, filters, and display fields. These are authoritative (written by Cyrious/SAP) unlike the inferred analysis in `output/reports/`.

Example from `sales_report_standard.md`:
- Primary data source: GL view (not TransHeader directly)
- Joins: GL -> TransHeader (GL.TransactionID = TransHeader.ID), GL -> Account (GL.AccountID = Account.ID), TransHeader -> Employee (Salesperson1ID), GL -> EmployeeGroup (DivisionID)
- Filter: EntryDateTime between dates, GLClassificationType between 4000 and 4999
- **This differs from the inferred analysis** which assumed the report uses TransHeader directly

This finding means the inferred `output/reports/` analysis has accuracy limitations. The wiki pages should be treated as ground truth where they exist.

### Finding 5: Macro Automation Documentation Completeness
**Confidence:** HIGH (directly verified against file content)

`macros_automation_knowledge.md` (1,172 lines) contains:
- All 95 Macro Message Types (IDs 0-94) with names, NeedsAltID flags, and AltID descriptions -- COMPLETE
- All 14 macro categories (Order, Estimate, Company, Contact, Employee, Bill, PO, Service Ticket, Line Item, Part, Receivables, Receiving Doc, Recurring Order, Work Assignment) -- COMPLETE
- All RuleAction types with ClassTypeIDs (Email=23100, Popup=23200, UDF=23300, Transaction=23400, Report Crystal=23500, Report Action=23600, Report SQL=23700, API Call=23800, Status Change=23900, Station Change=24000) -- COMPLETE
- Trigger events for all entity types (Order: 14 events, Estimate: 8 events, Company: 9 events, etc.) -- COMPLETE
- RuleMacro table schema (32+ columns) -- COMPLETE
- RuleAction table schema -- COMPLETE
- RuleActivity table schema -- COMPLETE
- RecurringActivity configuration -- COMPLETE
- 11 real-world automation patterns with full SQL -- COMPLETE

WIKI-05 success criteria are already met by this extract. Formalization only needs to verify and potentially create a clean quick-reference format.

## Open Questions

1. **Quality of "other" category pages (425 pages)**
   - What we know: 425 pages fell into "other" (uncategorized) in the catalog
   - What's unclear: Whether any of these contain important information that should be in an extract
   - Recommendation: Spot-check a sample (10-20 pages) during verification to confirm nothing critical is miscategorized. Do not attempt to recategorize all 425.

2. **Release notes pages (193 pages) coverage**
   - What we know: 193 pages are categorized as release_notes, covering Control versions 4.3 through 6.1
   - What's unclear: Whether any version-specific features documented only in release notes are missing from the extracts
   - Recommendation: Low priority. Release notes are not a required extract domain. Note their existence in the verification output.

3. **Inferred report analysis accuracy**
   - What we know: 36 reports in `output/reports/` have inferred SQL. At least one (Sales Report) shows incorrect assumptions when compared to wiki docs.
   - What's unclear: How many of the other 23 non-wiki-documented reports have accurate inferences
   - Recommendation: Cross-reference the 13 wiki-documented reports against their inferred counterparts. For the remaining 23 reports without wiki documentation, note them as "inferred only" with appropriate confidence flags.

## Sources

### Primary (HIGH confidence)
- `/Users/cain/projects/control-db-map/output/wiki/_crawl_state.json` -- Crawl state with all 1,769 entries verified
- `/Users/cain/projects/control-db-map/output/wiki/KNOWLEDGE_BASE.md` -- Master index (282 lines)
- `/Users/cain/projects/control-db-map/output/wiki/extracts/` -- All 12 extract files examined
- `/Users/cain/projects/control-db-map/output/wiki/reference/` -- All 6 subdirectories examined
- `/Users/cain/projects/control-db-map/output/wiki/reports/` -- All 24 files examined
- `/Users/cain/projects/control-db-map/output/reports/` -- 36 report analysis files examined
- `/Users/cain/projects/control-db-map/.planning/REQUIREMENTS.md` -- Requirements WIKI-01 through WIKI-05
- `/Users/cain/projects/control-db-map/.planning/ROADMAP.md` -- Phase 3 success criteria

### Not Needed
- No web searches required (all material is local)
- No Context7 queries required (no external libraries)
- No official documentation needed (domain-specific internal project)

## Metadata

**Confidence breakdown:**
- Crawl completeness (WIKI-01): HIGH -- verified crawl state JSON against disk files
- Extract inventory (WIKI-02): HIGH -- all 12 files verified with line counts and sizes
- CHAPI documentation (WIKI-03): HIGH -- source material inventory confirmed (18 + 10 + 17 reference files + extract)
- Crystal Reports SQL (WIKI-04): HIGH -- wiki reports pages verified as authoritative source; inferred analysis confirmed as supplementary
- Macro automation (WIKI-05): HIGH -- complete 95 message type table verified in extract

**Research date:** 2026-02-08
**Valid until:** No expiration (internal documentation project, no external dependencies)
