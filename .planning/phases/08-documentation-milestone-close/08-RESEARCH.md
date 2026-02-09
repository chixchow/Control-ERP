# Phase 8: Documentation & Milestone Close - Research

**Researched:** 2026-02-09
**Domain:** Technical documentation, milestone verification, knowledge formalization
**Confidence:** HIGH

## Summary

Phase 8 is a pure documentation and verification phase. All source materials already exist -- four skill files (core, sales, financial, glossary), 11 reference files, 36 Crystal Report analysis docs, a 1011-line validation scorecard, 12 wiki extracts, 3 formalized wiki references, and a knowledge log. The task is to synthesize these materials into six deliverable documents (DOC-01 through DOC-06), cross-check all 35 requirements, and close the milestone.

This is not a research-heavy phase. There are no libraries to evaluate, no patterns to discover, and no technology decisions to make. The research focus is: what exactly exists today, where is it, what gaps remain, and how should the documents be organized to serve the primary audience (future Claude sessions loading skills).

**Primary recommendation:** Create documentation by synthesizing existing materials, not by querying new data. Every fact needed is already captured in skill files, the scorecard, or the knowledge log. The Crystal Reports catalog is the only document requiring a database query (ReportMenuItem table), and even that has already been partially documented in output/report_summary.md.

## Standard Stack

Not applicable -- this phase produces markdown documentation files. No libraries, frameworks, or tools required beyond the existing project structure.

### Existing Materials Inventory

| Material | Location | Lines | Purpose |
|----------|----------|-------|---------|
| control-erp-core SKILL | `skills/control-erp-core/control-erp-core-SKILL.md` | 347 | Business rules, TransactionType, StatusID, pricing, joins |
| control-erp-sales SKILL | `skills/control-erp-sales/control-erp-sales-SKILL.md` | 370 | Product architecture, 9 query templates, NL routing |
| control-erp-financial SKILL | `skills/control-erp-financial/control-erp-financial-SKILL.md` | 856 | GL/Ledger, AR, AP, P&L, payments, closeout |
| control-erp-glossary SKILL | `skills/control-erp-glossary/control-erp-glossary-SKILL.md` | 491 | FLS terminology, Control terms, routing table |
| Milestone 1 Scorecard | `validation/milestone1-scorecard.md` | 1011 | 21 tests, 5 gotchas, gate evaluation |
| Phase 2 Test Results | `output/phase2-test-results.md` | 274 | Raw MCP query results from 2026-02-07 |
| NL Query Patterns | `output/skill/references/nl_query_patterns.md` | 962 | Natural language to SQL translations |
| Query Patterns | `output/skill/references/query_patterns.md` | 892 | SQL cookbook by domain |
| Common Joins | `output/skill/references/common_joins.md` | 541 | JOIN templates for every table combo |
| Schema Reference | `output/skill/references/schema_reference.md` | 1084 | Complete table/column reference by domain |
| ClassTypeID Reference | `output/skill/references/classtypeid_reference.md` | 570 | 350+ ClassTypeID entries |
| Field Values | `output/skill/references/field_values.md` | 367 | Enumerated field value reference |
| Business Rules Validation | `output/skill/references/business_rules_validation.md` | 156 | Revenue formula validation methodology |
| Advanced Analytics | `output/skill/references/advanced_analytics.md` | 583 | BI queries (CLV, RFM, etc.) |
| Stored Procedures | `output/skill/references/stored_procedures.md` | 384 | SP templates |
| Quick Reference | `output/skill/references/quick_reference.md` | 214 | Cheat sheet |
| Relationships | `output/skill/references/relationships.md` | 326 | FK and inferred relationships |
| Report Summary | `output/report_summary.md` | ~100 | 36 Crystal Reports overview |
| Report Join Patterns | `output/report_join_patterns.md` | varies | Join patterns from reports |
| Individual Reports | `output/reports/*.md` | 36 files | Per-report analysis docs |
| CHAPI Architecture | `output/wiki/references/chapi-architecture.md` | 351 | CHAPI/SQLBridge reference |
| Crystal Reports SQL | `output/wiki/references/crystal-reports-sql-patterns.md` | 825 | Authoritative join patterns |
| Macro Automation | `output/wiki/references/macro-automation-reference.md` | 552 | 95 message types, 10 action types |
| Wiki Extracts | `output/wiki/extracts/*.md` | 12 files | Domain knowledge from wiki crawl |
| Knowledge Log | `documentation/control-erp-knowledge-log.md` | varies | Validated discovery log |
| Schema Docs | `output/schemas/*.md` | 187 files | Per-table schema documentation |
| ReportMenuItem Schema | `output/schemas/ReportMenuItem.md` | 75 | Report menu table structure |

**Total existing documentation: ~14,000+ lines across 260+ files.**

## Architecture Patterns

### Document Organization

Phase 8 produces six deliverables mapped to six requirements. Based on content analysis and the CONTEXT.md decisions:

```
DOC-01: Skill Architecture      → output/docs/skill-architecture.md (NEW)
DOC-02: Query Pattern Reference  → output/docs/query-pattern-reference.md (NEW)
DOC-03: Business Process Docs    → Embedded in DOC-01 or standalone (Claude's discretion)
DOC-04: Crystal Reports Catalog  → output/docs/crystal-reports-catalog.md (NEW)
DOC-05: Known Gotchas            → output/docs/known-gotchas.md (NEW)
DOC-06: Validation Methodology   → output/docs/validation-methodology.md (NEW)

Plus:
35-requirement checklist         → output/docs/milestone1-requirements-checklist.md (NEW)
```

**Rationale for `output/docs/` location:** The primary audience is future Claude sessions. These documents serve as onboarding materials -- loaded alongside skills when a session needs to understand the system architecture, find a query pattern, or verify results. They complement (not duplicate) the skill files.

### Pattern 1: Synthesis, Not Duplication

**What:** Each document synthesizes and cross-references existing materials rather than duplicating content from skill files.

**When to use:** Always. The CONTEXT.md explicitly states "reference existing skill templates rather than duplicating -- single source of truth, no duplication."

**Example:**
```markdown
## Revenue Queries

### "What were our total sales in 2025?"
- **Skill:** control-erp-sales, Template 1
- **Key rules:** SubTotalPrice, SaleDate IS NOT NULL, TransactionType = 1 (see control-erp-core, Revenue Query Formula)
- **Validated result:** $3,052,952.52 (Test 1.1, within 0.02% of $3,053,541.85 known income)
```

Not:
```markdown
## Revenue Queries
SELECT SUM(SubTotalPrice) AS Revenue
FROM TransHeader
WHERE TransactionType = 1 AND IsActive = 1 AND SaleDate IS NOT NULL...
```

### Pattern 2: Machine-Readable Structure

**What:** Use consistent heading hierarchy, tables, and explicit labels that Claude can parse efficiently.

**When to use:** All documents. CONTEXT.md says "optimize for machine readability, structured data, explicit rules."

**Key conventions:**
- Tables for reference data (not prose)
- Numbered lists for sequential processes
- Code blocks only for SQL patterns not already in skills
- Cross-references as explicit file paths

### Pattern 3: Flat Gotcha List

**What:** Gotchas as a flat numbered list with wrong/correct code patterns.

**When to use:** DOC-05 specifically. CONTEXT.md says "Gotchas as a flat numbered list with wrong/correct patterns -- no severity grouping."

### Pattern 4: Checklist Table for Requirements

**What:** Simple table: Requirement ID | Description | Status (PASS/FAIL) | Evidence File.

**When to use:** Milestone close document. CONTEXT.md says "35-requirement cross-check as simple checklist table."

## Don't Hand-Roll

Problems that look simple but have existing solutions:

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Query pattern catalog | Write new SQL examples | Reference existing skill templates (9 in sales, 15+ in financial, 30+ in references/) | Single source of truth; skills are the authoritative patterns |
| Crystal Reports catalog | Parse .rpt files | Query ReportMenuItem table + reference `output/report_summary.md` + `output/reports/*.md` | .rpt extraction deferred to M2; metadata is sufficient for M1 |
| Gotcha list | Research new gotchas | Compile from scorecard gotcha section (5) + MEMORY.md + CLAUDE.md | All gotchas already discovered and validated |
| Validation methodology | Invent new methodology | Document what was actually done in Phase 7 (cross-reference against MCP results) | Methodology already proven; just needs documentation |
| Business process flow | Research order lifecycle | Combine core skill (StatusID flow), financial skill (GL entries), and wiki extract (orders_accounting_knowledge.md) | All stages documented in existing materials |

**Key insight:** Phase 8 is documentation of work already done, not new discovery. Every fact is already captured somewhere. The value is in organization, cross-referencing, and making materials accessible to future sessions.

## Common Pitfalls

### Pitfall 1: Duplicating Skill Content

**What goes wrong:** Writing full SQL queries into documentation that are already in skill files. Creates two sources of truth that will drift apart.

**Why it happens:** Natural instinct to make each document self-contained.

**How to avoid:** Reference skill file + template number. Include only the NL trigger phrase and a one-line description of what the template does. If the reader needs the actual SQL, they load the skill.

**Warning signs:** Any SQL block longer than 3 lines in a DOC-* file that also exists in a skill file.

### Pitfall 2: Missing Requirement Cross-Check

**What goes wrong:** Marking a requirement PASS without citing specific evidence file and section.

**Why it happens:** 29 of 35 requirements are already marked complete in REQUIREMENTS.md. Easy to rubber-stamp.

**How to avoid:** For each requirement, cite the specific file and section that satisfies it. A future reader should be able to follow the link and verify.

**Warning signs:** Any requirement row with "PASS" but no file reference.

### Pitfall 3: Crystal Reports Catalog Scope Creep

**What goes wrong:** Attempting to extract SQL from .rpt files or parse binary report formats.

**Why it happens:** CONTEXT.md for Phase 8 explicitly says .rpt SQL extraction is deferred to Milestone 2.

**How to avoid:** Catalog uses ONLY database metadata (ReportMenuItem table) and existing `output/report_summary.md` / `output/reports/*.md` files. Note the deferral explicitly.

**Warning signs:** Any reference to opening .rpt files, running Python extraction scripts, or parsing binary formats.

### Pitfall 4: Business Process Documentation Becoming Too Long

**What goes wrong:** Order lifecycle documentation expands to cover every edge case, making it unwieldy.

**Why it happens:** The lifecycle is genuinely complex (estimate -> order -> WIP -> Built -> Sale -> Closed, with deposits, payments, GL entries at each stage).

**How to avoid:** Focus on the happy path with table references at each stage. Edge cases (voided orders, partial prepay, off-balance sheet) are already documented in the financial skill. Cross-reference rather than duplicate.

**Warning signs:** Business process section exceeding ~150 lines. If it does, it should be standalone (CONTEXT.md leaves this to Claude's discretion based on length).

### Pitfall 5: nl_query_patterns.md vs Skill Files Inconsistency

**What goes wrong:** The existing `output/skill/references/nl_query_patterns.md` (962 lines) has some pre-validation content that contradicts the validated skill files (e.g., StatusID=4 for "completed" vs correct StatusID flow, Type 2 described as "Web/Online Order" rather than "Estimate/Quote").

**Why it happens:** The nl_query_patterns.md and field_values.md were written before the Phase 1 validation corrected business rules.

**How to avoid:** DOC-02 (Query Pattern Reference) should reference the validated skill file patterns, NOT the pre-validation output/skill/references files. Where DOC-02 covers NL triggers, use the NL mapping tables from core skill (lines 80-95) and sales skill (lines 329-349) as the authoritative source.

**Warning signs:** Any reference to "StatusID = 4" meaning "Completed" (correct label is "Closed"), or Type 2 described as "Web/Online Order" (correct is "Estimate/Quote").

## Code Examples

Not applicable -- this phase produces markdown documentation, not code. The relevant "patterns" are document templates.

### DOC-01 Architecture Document Template

```markdown
# Skill Architecture

## Dependency Graph

[Mermaid diagram]

## Skill Summaries

### control-erp-core
- **Purpose:** [one line]
- **Covers:** [bullet list of domains]
- **Key sections:** [top 3-5]
- **Lines:** 347

[repeat for each skill]

## How Skills Interact
[narrative walkthrough]

## Order Lifecycle
[table: Stage | StatusID | Tables Involved | GL Entries | Skill Reference]
```

### DOC-05 Gotcha Template

```markdown
# Known Gotchas and Pitfalls

1. **TransDetailParam IsActive Filter**
   - WRONG: `WHERE tdp.IsActive = 1`
   - CORRECT: No IsActive filter on TransDetailParam
   - Impact: $1.33M revenue undercount (DyeSub shows $464K instead of $1.79M)
   - Documented in: control-erp-core lines 120-125, control-erp-sales line 147
```

### Milestone Close Checklist Template

```markdown
| Req ID | Description | Status | Evidence |
|--------|-------------|--------|----------|
| CORE-01 | Core skill complete with TransactionType mappings | PASS | skills/control-erp-core/control-erp-core-SKILL.md, lines 33-48 |
```

## State of the Art

Not applicable -- this is documentation work, not technology implementation.

## Key Data Sources for Each Requirement

### DOC-01: Skill Architecture

**Sources (all HIGH confidence -- local files):**
- 4 skill files (core 347L, sales 370L, financial 856L, glossary 491L)
- YAML frontmatter in each skill contains `name`, `description`, and dependency declarations
- Core skill: "Foundation reference... Required dependency for all other control-erp skills"
- Sales skill: "Depends on: control-erp-core"
- Financial skill: "Depends on: control-erp-core"
- Glossary skill: "routing and translation tool, not a knowledge repository"
- Order lifecycle: core skill StatusID table (lines 239-252), financial skill GL Transaction Flows (lines 276-363)
- 11 reference files in `output/skill/references/` (6,079 lines total)

**Dependency graph:**
```
control-erp-core (foundation)
  ├── control-erp-sales (depends on core)
  ├── control-erp-financial (depends on core)
  └── control-erp-glossary (routes to all three)

Supporting references:
  output/skill/references/ (11 files, consumed by all skills)
  output/schemas/ (187 files, raw schema data)
  output/wiki/references/ (3 files, formalized wiki knowledge)
```

### DOC-02: Query Pattern Reference

**Sources (all HIGH confidence -- validated in Phase 7):**
- Sales skill: 9 numbered templates (lines 99-309) with NL interpretation table (lines 329-349)
- Financial skill: AR snapshot, AR aging, AP snapshot, AP aging, P&L queries, balance sheet (lines 500-732), NL interpretation table (lines 760-784)
- Core skill: Revenue query formula (lines 14-29), NL mapping table (lines 80-95), date filtering patterns (lines 130-162)
- Scorecard: All 21 test queries with validated results (1011 lines)
- Existing reference files: nl_query_patterns.md (962L), query_patterns.md (892L) -- NOTE: pre-validation, may have inconsistencies

**Organization decision (Claude's discretion per CONTEXT.md):**
Recommend organizing by use case (business question) rather than by table/domain, since the primary audience is a Claude session answering a business question. Group as:
1. Revenue & Sales (core + sales templates)
2. Product Analysis (sales templates 3, 4, 8)
3. Customer Intelligence (sales templates 5, 6)
4. Estimates & Pipeline (core Type 2 patterns)
5. Financial (AR, AP, P&L, GL from financial skill)
6. Operational (WIP, production, inventory -- pointers to M2)

### DOC-03: Business Process Documentation

**Sources (all HIGH confidence -- synthesized from validated skills):**
- Core skill StatusID table: New(0) -> WIP(1) -> Built(2) -> Sale(3) -> Closed(4) (lines 239-252)
- Financial skill GL Transaction Flows: Stage 1 (New Order), Stage 2 (Deposit), Stage 2.5 (Built), Stage 3a/3b (Sale), Bill lifecycle (lines 276-363)
- Financial skill Payment Posting Patterns: Path A (Prepaid) vs Path B (Credit/AR) (lines 368-398)
- Sales skill estimate conversion: Type 2 StatusID 11->13/12/14, EstimateNumber linkage (core lines 55-57)
- Wiki extract: orders_accounting_knowledge.md (order lifecycle from wiki perspective)

**Stages to document with table references:**
1. Estimate (TransHeader Type 2, StatusID 11)
2. Estimate Conversion (StatusID 11->13, creates Type 1)
3. New Order (TransHeader Type 1, StatusID 0, GL: WIP/Orders Due)
4. WIP (StatusID 1, production, artwork proofing)
5. Built (StatusID 2, GL: WIP->Built, FGI cost flow)
6. Sale (StatusID 3, SaleDate set, GL: Revenue recognition)
7. Closed (StatusID 4, BalanceDue = 0)
8. Shipping (Shipments table, FedEx/UPS logs)
9. Invoicing/Payment (Payment table, GL: AR or Prepayments)

### DOC-04: Crystal Reports Catalog

**Sources (HIGH confidence for metadata, MEDIUM for report details):**
- ReportMenuItem table (899 rows, 35 distinct .rpt filenames per context)
- `output/report_summary.md` (36 reports categorized, with inferred primary tables)
- `output/reports/*.md` (36 individual report analysis files)
- `output/wiki/references/crystal-reports-sql-patterns.md` (825 lines, 13 wiki-documented reports)
- Context note: DropShipInvoice.rpt referenced by 450 menu items (most-used)
- Context note: Some reports use UseSystemReport=true with SystemReportID
- Context note: .rpt SQL extraction deferred to Milestone 2

**Database query needed:** One query to ReportMenuItem to get the full catalog with menu hierarchy:
```sql
SELECT DISTINCT
    rmi.MenuItemName,
    rmi.FileName,
    rmi.UseSystemReport,
    rmi.SystemReportID,
    parent.MenuItemName AS ParentMenu,
    rmi.IsActive,
    rmi.IsSystem
FROM ReportMenuItem rmi
LEFT JOIN ReportMenuItem parent ON rmi.ParentID = parent.ID
WHERE rmi.FileName IS NOT NULL OR rmi.UseSystemReport = 1
ORDER BY parent.MenuItemName, rmi.MenuItemName
```

### DOC-05: Known Gotchas

**Sources (all HIGH confidence -- validated in Phase 7 scorecard):**
1. TransDetailParam IsActive filter ($1.33M undercount) -- scorecard Gotcha 1, core skill lines 120-125
2. SubTotalPrice vs TotalPrice ($11.6K overcount) -- scorecard Gotcha 2, core skill lines 14-29
3. SaleDate vs OrderCreatedDate (wrong periods) -- scorecard Gotcha 3, core skill lines 130-136
4. EstimateCreatedDate for Type 2 (zero results) -- scorecard Gotcha 4, core skill lines 134-136
5. SaleDate IS NOT NULL filter (WIP/Built included) -- scorecard Gotcha 5, core skill lines 113-117
6. CompanyName not AccountName (query failure) -- MEMORY.md, nl_query_patterns.md line 20
7. DueDate means different things by type (AR aging error) -- financial skill caveat 2
8. GL view vs Ledger table (missing OBS entries) -- financial skill caveat 1
9. Web order deduplication (8.4% overcount) -- scorecard Test 5.1, core skill lines 60-73
10. Header vs Detail revenue gap (~$7K) -- sales skill caveat 1
11. Type 2 in revenue totals (double-counting) -- core skill lines 50-55
12. TC_ variables not in TransDetailParam -- sales skill lines 62-63
13. pre-2017 SQL Server (no STRING_AGG) -- CLAUDE.md, SKILL.md line 21

**MEMORY.md additions:**
- TransDetailParam query rules (never filter IsActive or ParentClassTypeID)
- Type 2 date field (EstimateCreatedDate)
- SubTotalPrice for revenue
- SaleDate for date filtering
- Geography columns cause errors

### DOC-06: Validation Methodology

**Sources (all HIGH confidence -- methodology actually used):**
- Phase 7 execution: cross-reference validation against MCP results from 2026-02-07
- `phase2-test-suite.md` (248 lines) -- test suite definition with 21 tests, 7 tiers
- `validation/milestone1-scorecard.md` (1011 lines) -- executed scorecard
- `output/phase2-test-results.md` (274 lines) -- raw MCP query results
- Phase 7 VERIFICATION.md -- verifier's assessment of scorecard quality
- `output/skill/references/business_rules_validation.md` (156 lines) -- revenue formula validation

**Methodology to document:**
1. Known baseline establishment ($3,053,541.85 FLS income from QuickBooks)
2. Live MCP query execution against StoreData
3. Cross-reference validation (scorecard values vs MCP results)
4. Internal consistency checks (11 cross-checks documented)
5. Gotcha validation with quantified error magnitudes
6. Gate criteria (>=19/21 PASS, zero Tier 1 FAIL)
7. Tolerance definitions (1% for revenue, exact for counts, plausible for rankings)

## Open Questions

### 1. Pre-Validation Reference File Inconsistencies

**What we know:** The `output/skill/references/nl_query_patterns.md` and `field_values.md` were written before Phase 1 validation. They contain some incorrect information (e.g., Type 2 labeled "Web/Online Order", StatusID 4 labeled "Completed" instead of "Closed", `StatusID = 4` recommended for revenue instead of `SaleDate IS NOT NULL`).

**What's unclear:** Should DOC-02 replace these files, update them, or coexist alongside them?

**Recommendation:** DOC-02 should be the authoritative query pattern reference that supersedes the pre-validation files. The existing files in `output/skill/references/` should NOT be modified (they may have consumers), but DOC-02 should explicitly note it supersedes `nl_query_patterns.md` for validated patterns. Alternatively, leave those files as-is since they are part of the original Phase 1 `output/skill/` package (pre-Cornerstone), and the skills themselves are the authoritative source.

### 2. ReportMenuItem Query Execution

**What we know:** Context mentions 35 distinct .rpt filenames in ReportMenuItem. The existing `output/report_summary.md` documents 36 reports from the physical .rpt files on disk.

**What's unclear:** Whether the MCP connection will be available during execution to query ReportMenuItem directly.

**Recommendation:** Attempt the MCP query. If unavailable (Mac environment), use the existing `output/report_summary.md` as the basis and note that ReportMenuItem metadata enrichment is deferred. Phase 2 context notes from earlier discussions indicate this query was already done, so the 35 distinct filenames figure is reliable.

### 3. "What's Next" Section Scope

**What we know:** CONTEXT.md says "include 'What's Next' section seeding Milestone 2 ideas -- deferred ideas collected during M1 plus natural next steps."

**What's unclear:** How comprehensive this should be.

**Recommendation:** Keep it concise -- a bulleted list of M2 candidates, grouped by the v2 requirement categories already defined in REQUIREMENTS.md (Financial, Customer, Inventory, Production, HR, Reports & Analytics). Include M1 deferred items (.rpt SQL extraction, Crystal Reports deep analysis) and natural follow-ons identified during M1 execution.

## Sources

### Primary (HIGH confidence)
- All 4 skill files -- read and analyzed in full
- `validation/milestone1-scorecard.md` -- read in full (1011 lines)
- `.planning/REQUIREMENTS.md` -- all 35 v1 requirements with status
- `.planning/ROADMAP.md` -- phase structure and completion status
- `.planning/STATE.md` -- current project state
- `.planning/phases/08-documentation-milestone-close/08-CONTEXT.md` -- user decisions
- `output/report_summary.md` -- Crystal Reports overview
- `output/schemas/ReportMenuItem.md` -- report menu table structure
- All 7 phase VERIFICATION.md files (read Phase 7's)
- All phase SUMMARY.md files (sampled Phase 1 and Phase 3)

### Secondary (MEDIUM confidence)
- `output/skill/references/*.md` -- pre-validation reference files (some content outdated)
- `output/wiki/references/*.md` -- formalized wiki knowledge (verified in Phase 3)
- `CLAUDE.md` and `MEMORY.md` -- project instructions and accumulated knowledge

### Tertiary (LOW confidence)
- None. All sources are local project files with known provenance.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- no technology decisions, pure documentation
- Architecture: HIGH -- document organization follows CONTEXT.md decisions; all source materials inventoried
- Pitfalls: HIGH -- all pitfalls identified from actual project experience (pre-validation inconsistencies, scope creep, duplication risk)

**Research date:** 2026-02-09
**Valid until:** Indefinite (documentation of completed milestone)
