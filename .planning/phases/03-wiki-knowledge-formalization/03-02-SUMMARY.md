# Phase 03 Plan 02: Wiki Reference Formalization Summary

**One-liner:** Formalized three wiki-sourced developer references: CHAPI/SQL Bridge architecture (351 lines, 7 functions, 17 stored procs), Crystal Reports SQL patterns (825 lines, 13 reports, authoritative joins), and macro automation system (552 lines, 95 message types, 10 action types).

---

## Plan Metadata

- **Phase:** 03-wiki-knowledge-formalization
- **Plan:** 02
- **Execution Date:** 2026-02-08
- **Duration:** 7 minutes 15 seconds
- **Status:** Complete
- **Tasks Completed:** 2/2
- **Subsystem:** Documentation & Knowledge Extraction
- **Tags:** #chapi #sql-bridge #crystal-reports #macros #wiki #reference-docs

---

## Objectives Achieved

**Objective:** Formalize three wiki-sourced reference documents (CHAPI architecture, Crystal Reports SQL patterns, macro automation system) for downstream phase consumption.

**Results:**
1. **CHAPI Architecture Reference** - Consolidated 18+ wiki files into single developer reference
2. **Crystal Reports SQL Patterns** - Extracted authoritative join patterns from 13 wiki-documented standard reports
3. **Macro Automation Reference** - Quick-lookup table for 95 message types, 10 RuleAction types, 14 categories

**Downstream Consumers:**
- Phase 5 (Write-Path Analysis) needs CHAPI for understanding safe write operations
- Phase 8 (Documentation Enhancement) needs Crystal Reports for DOC-04 (report query patterns)
- Phase 6 (Glossary Creation) needs macro terminology

---

## Tasks Completed

### Task 1: Consolidate CHAPI Architecture Reference

**Output:** `output/wiki/references/chapi-architecture.md` (351 lines)

**Source Files Consolidated:**
- Primary: chapi_url_endpoint_listener.md, chapi.md, sql_bridge.md, database_integration_knowledge.md
- Secondary: 17 stored procedure files (signatures extracted, not full T-SQL)

**Sections Created:**
1. Overview - What CHAPI is, when to use it
2. HTTP Endpoint - URL format (`http://{server}:12556/chapi/sqlmacro?name={ProcName}`), port 12556, GET/POST
3. SQL Bridge Core Functions - All 7 functions with signatures:
   - NextID(ClassTypeID, Count) → Get next ID(s)
   - NextNumber(SpecialTypeName, Count) → Get next number
   - Lock(ID, ClassTypeID) → Lock record for editing
   - Unlock(ID, ClassTypeID) → Release lock
   - Refresh(ID, ClassTypeID, eTag) → Notify Control to reload
   - RefreshEx(ID, ClassTypeID, eTag, MacroMessageTypes, SessionID) → Refresh with macro triggers
   - Recompute(ID, ClassTypeID, Notes) → Recalculate derived fields
4. Import Stored Procedures Catalog - All 17 procedures: import_company, import_contact, import_order, import_estimate, import_line_item, import_payment, import_address, import_shipping_information, import_shipping_information_detail, import_part, import_part_category, insert_part_on_a_line_item, convert_estimate_to_order, add_note, customer_list_import, quickbooks_import, import_text_file_to_cols
5. Operation Patterns - INSERT/UPDATE/DELETE patterns with examples
6. Safety Rules - 10 critical rules for write operations
7. Source Files - Full traceability to wiki sources

**Key Value:** Developers now have single authoritative reference for all CHAPI operations instead of navigating 18+ wiki pages.

**Commit:** `643117a` - docs(03-02): consolidate CHAPI architecture reference

---

### Task 2: Extract Crystal Reports SQL Patterns and Macro Reference

**Output Part A:** `output/wiki/references/crystal-reports-sql-patterns.md` (825 lines)

**Reports Documented (13 total):**
1. AR Detail Report - A/R by company with aging buckets
2. AR Summary Report - A/R summary with drill-down
3. Deposits Made Report - Payment breakdown by deposit
4. Estimate Report - Primary estimate/quotation
5. GL Listing Report - General ledger entries
6. Historical AR Report - Historical A/R from GL
7. Invoice Report - Primary invoice
8. Orders Placed Between Report - Orders by status change date
9. Sales By X Report - Sales by dimension (customer, product, etc.)
10. Sales Report - Sales activity from GL
11. WIP and Built Report - Production by order
12. WIP By Line Item Report - Production by line item
13. Work Order Report - Work order with parts/shipping

**Content Per Report:**
- Primary data source (table/view)
- Complete JOIN syntax with table linkages
- WHERE clause patterns
- Key fields displayed
- Grouping structure
- Aging/calculation logic where applicable

**Common Join Patterns Catalog:**
- TransHeader → Account (customer info)
- TransHeader → AccountContact (order contact)
- TransHeader → Employee (salesperson)
- TransHeader → EmployeeGroup (division)
- Account → PaymentTerms (payment terms)
- TransHeader → TransDetail (line items)
- GL → TransHeader (financial reporting)
- Ledger → GL components (GLAccount, Journal, Account)
- Journal → Payment (deposit breakdown)
- TransDetail → Part (parts list)

**Cross-Reference with Inferred Analysis:**
- Validated 13 wiki-documented reports against inferred analysis
- **MAJOR DISCREPANCY FLAGGED:** Sales Report - wiki documents GL view as primary source, inferred analysis incorrectly assumed TransHeader directly
- Identified 23 inferred-only reports with NO wiki validation available
- Provides clear differentiation: authoritative (wiki) vs. inferred (needs validation)

**Output Part B:** `output/wiki/references/macro-automation-reference.md` (552 lines)

**Content:**
- **95 Macro Message Types** (IDs 0-94) - Complete table with ID, Name, NeedsAltID flag, AltID description
- **10 RuleAction Types** - ClassTypeIDs 23110-23180 (Contact Activity, Email, Report, UDF, Popup, Transaction, Chain Macro, Company Activity, Service Contract, Work Assignment, Transaction Update)
- **14 Macro Categories** - Entity type mappings (Order, Estimate, Company, Contact, Employee, Bill, PO, Service Ticket, Line Item, Part, Receivables, Receiving Doc, Recurring Order, Work Assignment)
- **Trigger Events** - Organized by category (13 Order events, 9 Estimate events, 11 Company events, etc.)
- **Database Tables** - RuleMacro (ClassTypeID 23000), RuleAction (231XX range), RuleActivity (9201/9202) with key columns
- **Execution Modes** - Automatic (Background), Automatic (Workstation), Manual, Scheduled
- **Data Sources** - 4 types including `<%TriggerID%>` placeholder usage
- **Common Action Configurations** - Email, Report, UDF, Transaction, Chain Macro

**Key Value:** Fast-lookup table for macro developers - no need to navigate comprehensive extract or wiki pages.

**Commit:** `cfba096` - docs(03-02): extract crystal reports SQL patterns and macro reference

---

## Deviations from Plan

None - plan executed exactly as written.

---

## Key Files Created

| File | Lines | Purpose | Consumers |
|------|-------|---------|-----------|
| `output/wiki/references/chapi-architecture.md` | 351 | CHAPI/SQL Bridge developer reference | Phase 5 (Write-Path Analysis) |
| `output/wiki/references/crystal-reports-sql-patterns.md` | 825 | Authoritative Crystal Report join patterns | Phase 8 (DOC-04 report query patterns) |
| `output/wiki/references/macro-automation-reference.md` | 552 | Macro system quick-reference | Phase 6 (Glossary), Phase 8 (Automation docs) |

**Total output:** 1,728 lines of formalized reference documentation

---

## Decisions Made

| Decision | Rationale | Impact |
|----------|-----------|--------|
| CHAPI reference excludes full T-SQL from stored procedures | This is a reference doc, not a code repository. Included signatures and purpose only. | Keeps document focused and consumable. Full T-SQL remains in wiki source files. |
| Crystal Reports includes cross-reference section | Plan explicitly requires comparison with inferred analysis to flag discrepancies | Identified major discrepancy in Sales Report (GL vs TransHeader) - authoritative source now clear |
| Macro reference uses quick-reference format | Large comprehensive extract exists; this doc optimizes for fast lookup | Developers can find message type IDs, action ClassTypeIDs, and trigger events in seconds |

---

## Technical Insights

### CHAPI/SQL Bridge Architecture

**Critical Safety Pattern Discovered:**
The Lock → Modify → Refresh → Unlock sequence is mandatory for all UPDATE operations. Skipping any step causes:
- No Lock: Concurrent edit overwrites
- No Unlock: Record locked indefinitely
- No Refresh: Control displays stale data, may overwrite on next edit

**ID Generation:**
- Never use SQL IDENTITY or manually assigned IDs
- Always call NextID(ClassTypeID, Count) before INSERT
- NextNumber() for human-readable numbers (order#, estimate#, etc.)
- Failure to use NextID() causes key violations and data loss

**HTTP Endpoint:**
- Port 12556 (standard, may be customized)
- URL format: `http://{server}:12556/chapi/sqlmacro?name={ProcName}&{param}={value}`
- GET or POST supported
- Returns JSON

### Crystal Reports SQL Patterns

**Primary Source Validation Critical:**
The Sales Report discrepancy (wiki: GL view, inferred: TransHeader) highlights the importance of wiki documentation. The GL view filters `OffBalanceSheet = 0` and `GLClassificationType BETWEEN 4000 AND 4999`, which would be missed in TransHeader-only approach.

**Common Pattern: GL-Based Financial Reports**
Multiple reports (Sales Report, Sales By X, Historical AR, Orders Placed Between) use Ledger/GL as primary source rather than TransHeader. This pattern:
- Captures adjustments to prior orders
- Reflects historical state (not current TransHeader values)
- Requires JOIN back to TransHeader for order details

**Adjustment Logic:**
Reports distinguish "new" vs "adjusted" orders by comparing TransHeader.SaleDate to report date range. If SaleDate outside range, GL entry is an adjustment.

### Macro Automation System

**Message Type Patterns:**
- IDs 0-4: General (Undefined, TierObj events)
- IDs 5-13: Order events
- IDs 14-20: Estimate events
- IDs 21-30: Company/Contact events
- IDs 31-59: Mixed (Payment, Employee, Service Ticket, Line Item, Inventory, Training, Work Assignment, GL)
- IDs 60-73: Status/Station "To" and "From" change events (14 pairs)
- IDs 74-94: Purchase Order, Bill, Receiving Doc, Account Stage, Shipment events

**NeedsAltID Pattern:**
22 of 95 message types require an alternate ID (e.g., "OrderStationToChange" needs new StationID). These are primarily:
- Status change "To/From" events (need StatusID)
- Station change "To/From" events (need StationID)
- Company new order/estimate/service ticket (need new TransHeaderID)
- Shipment events (need ShipmentID)

**ClassTypeID Ranges:**
- RuleMacro: 23000
- RuleAction: 23110-23180 (10 types)
- RuleActivity: 9201 (Recurring), 9202 (Recurring Macro)

---

## Validation & Quality

### Must-Have Truths Verified

✅ **Truth 1:** A developer reading chapi-architecture.md can understand how to call CHAPI -- HTTP endpoint URL format, SQL Bridge functions, and available import stored procedures
- HTTP endpoint fully documented with port 12556, URL format, GET/POST semantics
- All 7 SQL Bridge functions cataloged with signatures and purpose
- All 17 import stored procedures cataloged

✅ **Truth 2:** Crystal Reports SQL patterns from 13 wiki standard report pages are extracted with actual table names, joins, and filters -- authoritative source, not inferred
- All 13 reports documented with complete JOIN syntax
- WHERE clauses captured
- Cross-reference section distinguishes authoritative (wiki) vs inferred
- Sales Report discrepancy flagged

✅ **Truth 3:** Macro automation reference contains all 95 message types, all RuleAction types, and trigger events in a clean quick-reference format
- All 95 message types (IDs 0-94) in complete table
- All 10 RuleAction types with ClassTypeIDs (23110-23180)
- Trigger events organized by category
- Quick-reference formatting for fast lookup

### Must-Have Artifacts Verified

✅ **output/wiki/references/chapi-architecture.md**
- Exists: Yes
- Min lines (150): Yes (351 lines)
- Contains "12556": Yes (3 occurrences)
- Provides: Consolidated CHAPI developer reference

✅ **output/wiki/references/crystal-reports-sql-patterns.md**
- Exists: Yes
- Min lines (100): Yes (825 lines)
- Contains "TransHeader": Yes (144 occurrences)
- Provides: Authoritative Crystal Report join patterns from wiki

✅ **output/wiki/references/macro-automation-reference.md**
- Exists: Yes
- Min lines (80): Yes (552 lines)
- Contains "RuleAction": Yes (3 occurrences)
- Contains "95" or "IDs 0-94": Yes
- Provides: Complete macro system quick-reference

### Key Links Verified

✅ **chapi-architecture.md consolidates 18 CHAPI reference files**
- Pattern `sqlmacro.*name=.*ProcName` documented in HTTP Endpoint section
- Stored procedure catalog references all 17 import procedures

✅ **chapi-architecture.md catalogs import stored procedures**
- Pattern `import_company|import_contact|import_order` present
- All 17 procedures listed with purpose and key parameters

✅ **crystal-reports-sql-patterns.md extracts from wiki report pages**
- Pattern `FROM.*JOIN` captured in all 13 reports
- Cross-reference section validates against inferred analysis

✅ **macro-automation-reference.md formatted from comprehensive extract**
- Pattern `231\d{2}` (ClassTypeIDs) present in RuleAction types table
- All 10 action types listed with ClassTypeIDs 23110-23180

---

## Next Phase Readiness

### Phase 5 (Write-Path Analysis) Dependencies Met

**Dependency:** CHAPI architecture understanding
**Status:** ✅ Complete
**Deliverable:** `output/wiki/references/chapi-architecture.md` provides complete CHAPI/SQL Bridge reference including:
- HTTP endpoint usage
- SQL Bridge function signatures
- Lock → Modify → Refresh → Unlock pattern
- Import stored procedure catalog
- Safety rules for write operations

**Phase 5 can now:** Map write paths using authoritative CHAPI documentation instead of inferring from field names.

### Phase 8 (Documentation Enhancement) Dependencies Met

**Dependency 1 (DOC-04):** Crystal Reports query patterns
**Status:** ✅ Complete
**Deliverable:** `output/wiki/references/crystal-reports-sql-patterns.md` provides:
- 13 wiki-documented standard reports
- Authoritative JOIN patterns
- WHERE clause patterns
- Common join pattern catalog
- Cross-reference with inferred analysis

**Dependency 2 (Automation Docs):** Macro terminology
**Status:** ✅ Complete
**Deliverable:** `output/wiki/references/macro-automation-reference.md` provides macro system quick-reference

**Phase 8 can now:** Reference authoritative Crystal Report patterns for DOC-04 and use macro reference for automation documentation.

### Phase 6 (Glossary Creation) Dependencies Met

**Dependency:** Macro terminology
**Status:** ✅ Complete
**Deliverable:** `output/wiki/references/macro-automation-reference.md` provides:
- 95 macro message types
- 10 RuleAction types
- Trigger event terminology
- Execution mode terminology

**Phase 6 can now:** Use macro reference to define automation-related glossary terms.

---

## Lessons Learned

### What Worked Well

1. **Structured consolidation approach** - Reading primary sources first, then secondary sources, created clear information hierarchy
2. **Cross-reference validation** - Comparing wiki (authoritative) vs inferred (guessed) identified major Sales Report discrepancy
3. **Quick-reference formatting** - Macro reference optimized for fast lookup rather than comprehensive narrative
4. **Source file traceability** - All three references list source files for future maintenance

### Improvements for Next Time

1. **Early validation of source file counts** - Plan assumed 6 wiki extracts but 12 exist on disk (blocker noted in STATE.md)
2. **Pre-check Crystal Report file counts** - Plan listed 13 standard reports; verified all exist before starting extraction
3. **Token efficiency** - Reading all 13 Crystal Report files consumed significant tokens; could optimize by reading in smaller batches

### Technical Discoveries

1. **CHAPI port 12556** - Standard port for HTTP endpoint (may be customized during installation)
2. **GL view as primary source** - Multiple financial reports use GL/Ledger as primary source, not TransHeader
3. **Macro Message Type patterns** - Clear numerical organization by event category (0-4 General, 5-13 Order, 14-20 Estimate, etc.)
4. **NeedsAltID for "To/From" events** - Status/Station change events that specify target/source require alternate ID

---

## Statistics

- **Plans completed:** 2/3 in phase
- **Tasks completed:** 2/2 in this plan
- **Files created:** 3
- **Lines of documentation:** 1,728
- **Source files consolidated:** 30+ wiki files
- **Execution time:** 7 minutes 15 seconds
- **Commits:** 2
  - `643117a` - CHAPI architecture reference
  - `cfba096` - Crystal Reports and Macro references

---

## Commits

| Hash | Type | Scope | Message | Files |
|------|------|-------|---------|-------|
| 643117a | docs | 03-02 | consolidate CHAPI architecture reference | output/wiki/references/chapi-architecture.md |
| cfba096 | docs | 03-02 | extract crystal reports SQL patterns and macro reference | output/wiki/references/crystal-reports-sql-patterns.md, output/wiki/references/macro-automation-reference.md |

---

## Dependencies

### Requires (Built Upon)

- Phase 3 Plan 1 (03-01): Wiki knowledge extraction - Provided source extracts and reference files

### Provides (Delivers)

- **CHAPI/SQL Bridge Reference** - HTTP endpoint, SQL Bridge functions, import procedures, safety rules
- **Crystal Reports SQL Patterns** - Authoritative join patterns from 13 wiki-documented standard reports
- **Macro Automation Reference** - 95 message types, 10 RuleAction types, trigger events, quick-reference format

### Affects (Downstream Consumers)

- **Phase 5 (Write-Path Analysis)** - Uses CHAPI reference for safe write operation understanding
- **Phase 8 (Documentation Enhancement)** - Uses Crystal Reports patterns (DOC-04) and macro reference
- **Phase 6 (Glossary Creation)** - Uses macro terminology for automation glossary terms

---

## Tech Stack

### Added
- None (documentation only)

### Patterns Established
- **Reference Document Format** - Structured sections, source file traceability, quick-reference tables
- **Cross-Reference Validation** - Compare authoritative (wiki) vs inferred (analysis) to flag discrepancies
- **Quick-Reference Optimization** - Optimize for fast lookup rather than comprehensive narrative

---

*Summary completed: 2026-02-08*
*Total execution time: 7 minutes 15 seconds*
*Status: All must-haves satisfied, all downstream dependencies met*
