# Roadmap: Project Cornerstone — Milestone 1

## Overview

Milestone 1 formalizes and validates the substantial existing work on the Control ERP natural language interface. The core skill, sales skill, 187 table schemas, and wiki knowledge base are already built -- this milestone verifies accuracy against Control reports, fills documentation gaps, creates a glossary skill, executes a formal test suite, and produces the documentation that makes the system maintainable and extensible for Milestone 2. Eight phases cluster naturally from the 35 requirements, with three initial phases that can run fully in parallel.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Core Skill Verification** - Verify control-erp-core skill completeness and structure the knowledge log
- [ ] **Phase 2: Schema Infrastructure** - Validate schema docs, FK mappings, domain classification, and ClassTypeID reference
- [x] **Phase 3: Wiki Knowledge Formalization** - Formalize wiki crawl outputs into organized, skill-consumable references
- [ ] **Phase 4: Sales Skill Verification** - Verify control-erp-sales skill and document all product identification patterns
- [ ] **Phase 5: Financial Foundation** - Document GL/Ledger architecture, system accounts, and payment logic in the financial skill
- [ ] **Phase 6: Glossary Skill Creation** - Build control-erp-glossary with FLS business language and Control technical terminology
- [ ] **Phase 7: Test Suite Execution** - Execute all validation tests with formal scorecard proving accuracy against Control reports
- [ ] **Phase 8: Documentation & Milestone Close** - Produce maintainability documentation and close the milestone

## Parallelism

```
          +---> Phase 1 (Core) ----+---> Phase 4 (Sales) --+---> Phase 6 (Glossary) --+
          |                        |                        |                          |
START ----+---> Phase 2 (Schema) --+---> Phase 5 (Financial)+---> Phase 7 (Tests) -----+---> Phase 8 (Docs & Close)
          |                        |                                                   |
          +---> Phase 3 (Wiki) ----+---------------------------------------------------+
```

- **Wave 1** (parallel): Phases 1, 2, 3 -- no dependencies between them
- **Wave 2** (parallel): Phases 4, 5 -- both need Phase 1 done; independent of each other
- **Wave 3** (parallel): Phases 6, 7 -- both need Phases 1+4 done; independent of each other
- **Wave 4** (sequential): Phase 8 -- needs everything done

## Phase Details

### Phase 1: Core Skill Verification
**Goal**: The control-erp-core skill is verified complete and the knowledge log is structured for future reference
**Depends on**: Nothing (first wave)
**Requirements**: CORE-01, CORE-02
**Success Criteria** (what must be TRUE):
  1. `control-erp-core-SKILL.md` contains validated TransactionType mappings (Types 1, 2, 6, 7, 8, 9), StatusID values per type, and pricing logic (SubTotalPrice, SaleDate filtering) -- a user asking "What StatusIDs exist for Type 2 estimates?" gets an accurate answer from the skill alone
  2. Knowledge log at `documentation/control-erp-knowledge-log.md` is structured with date, query, finding, and source for every validated discovery -- a new session can review the log and understand what was learned and when
  3. Any contradictions between the skill file and CLAUDE.md/MEMORY.md are resolved (single source of truth)
**Plans:** 1 plan

Plans:
- [x] 01-01-PLAN.md — Resolve contradictions across truth sources, fill core skill gaps, structure knowledge log

### Phase 2: Schema Infrastructure
**Goal**: Schema documentation, FK relationships, domain classification, and ClassTypeID reference are verified complete and consumable by downstream skills
**Depends on**: Nothing (first wave)
**Requirements**: CORE-03, CORE-04, CORE-05, CORE-06
**Success Criteria** (what must be TRUE):
  1. All 187+ table schema files in `output/schemas/` contain columns, data types, nullable flags, and row counts -- spot-checking any 5 tables produces complete schema info
  2. `output/relationships.md` contains both explicit FK relationships (from sys.foreign_keys) and inferred relationships (from naming patterns like AccountID -> Account.ID)
  3. `output/domains.md` groups every table into a functional domain with a description of what that domain covers
  4. ClassTypeID reference (350+ entries from wiki) exists in a structured, skill-consumable format -- not a raw wiki dump but an organized lookup table
**Plans:** 2 plans

Plans:
- [ ] 02-01-PLAN.md — Verify schema completeness (10 tables spot-checked) and FK relationship completeness against live database
- [ ] 02-02-PLAN.md — Fix domain classification table count, compile standalone ClassTypeID reference (350+ entries)

### Phase 3: Wiki Knowledge Formalization
**Goal**: Wiki crawl outputs are verified complete and formalized into organized references that downstream skills and documentation can consume
**Depends on**: Nothing (first wave)
**Requirements**: WIKI-01, WIKI-02, WIKI-03, WIKI-04, WIKI-05
**Success Criteria** (what must be TRUE):
  1. Wiki crawl completeness verified -- 1,769 pages across control/reports/support subdomains are accounted for with catalog/index
  2. All 6 knowledge extracts cover their stated domains and contain actionable information (not just raw page dumps)
  3. CHAPI architecture documented with HTTP endpoint details, SQLBridge functions, and import stored procedure catalog -- a developer reading the docs could understand how to call CHAPI
  4. Crystal Reports SQL patterns extracted and cross-referenced with actual report analysis in `output/reports/`
  5. Macro automation system documented with trigger events (95 types) and RuleAction types
**Plans:** 2 plans

Plans:
- [x] 03-01-PLAN.md — Verify wiki crawl completeness (1,769 pages) and audit all 12 extract quality
- [x] 03-02-PLAN.md — Formalize CHAPI architecture, Crystal Reports SQL patterns, and macro automation references

### Phase 4: Sales Skill Verification
**Goal**: The control-erp-sales skill is verified complete with all product identification patterns documented and revenue queries that match Control reports
**Depends on**: Phase 1 (core skill must be verified before validating sales)
**Requirements**: SALES-01, SALES-02, SALES-03, SALES-04, SALES-05, SALES-06
**Success Criteria** (what must be TRUE):
  1. `control-erp-sales-SKILL.md` documents the DyeSub Print container variable architecture (FP_ProductDescription categories, FP_ProductID SKUs) with working query examples
  2. Table Cover identification (Description pattern matching, GoodsItemID=10026) and non-container products (Shipping, Artwork/Setup, Hardware, Custom Items) are documented with working queries
  3. Product category revenue breakdown validated against Control "Sales by Product" report -- DyeSub Print = $1,793K (58.7%)
  4. Customer revenue queries (top customers, CLV basics) return results consistent with Control reports
  5. Sales trend queries (monthly, quarterly, YoY) return internally consistent totals that sum to annual figures
**Plans**: TBD

Plans:
- [ ] 04-01: Review sales skill product architecture and identification patterns
- [ ] 04-02: Validate product revenue, customer revenue, and trend queries against Control

### Phase 5: Financial Foundation
**Goal**: GL/Ledger architecture, system account NodeIDs, and payment GL offset logic are documented in the financial skill with enough detail that financial queries in Milestone 2 have a solid foundation
**Depends on**: Phase 1 (core skill context needed for financial foundations)
**Requirements**: FIN-01, FIN-02, FIN-03
**Success Criteria** (what must be TRUE):
  1. `control-erp-financial-SKILL.md` documents GL view vs Ledger table structure, sign conventions, and GLClassificationType reference -- a user asking "How does GL work in Control?" gets a clear answer
  2. GL system account NodeIDs documented (11=WIP, 12=Built, 14=AR, 21=Orders Due, 24=Customer Deposits, 91=Undeposited Funds) with their purpose in the order lifecycle
  3. Payment GL offset logic documented -- how WIP/Built deposits vs Sale payments flow through GL accounts, with example journal entries
**Plans**: TBD

Plans:
- [ ] 05-01: Review and complete financial skill GL documentation
- [ ] 05-02: Document payment GL offset logic with examples

### Phase 6: Glossary Skill Creation
**Goal**: A control-erp-glossary skill exists that maps FLS business language and Control-specific terminology to database entities and queries
**Depends on**: Phase 1, Phase 4 (needs core and sales terminology established)
**Requirements**: GLOSS-01, GLOSS-02, GLOSS-03
**Success Criteria** (what must be TRUE):
  1. `skills/control-erp-glossary/` contains a SKILL.md with YAML frontmatter that Claude can load as a skill
  2. FLS product line terminology mapped (blanket, SEG, Fab Frame, DyeLux, table throw, step and repeat, etc.) to database queries and entities -- a user saying "find blanket orders" gets routed to the right query
  3. Control-specific technical terms defined (CHAPI, SSLIP, CFL, SQLBridge, ClassTypeID, GoodsItemClassTypeID) with how each relates to the database -- a user asking "What is a SSLIP?" gets a clear, useful answer
**Plans**: TBD

Plans:
- [ ] 06-01: Create glossary skill structure and populate terminology

### Phase 7: Test Suite Execution
**Goal**: All validation tests formally executed with a pass/fail scorecard proving that the skills produce accurate results against known Control report outputs
**Depends on**: Phase 1, Phase 4 (skills must be verified before formal testing)
**Requirements**: TEST-01, TEST-02, TEST-03, TEST-04, TEST-05, TEST-06
**Success Criteria** (what must be TRUE):
  1. Full test suite executed with pass/fail results documented in a formal scorecard at `validation/milestone1-scorecard.md`
  2. Revenue baseline test passes -- "Total sales 2025" returns $3,053,541.85 (within 1% tolerance)
  3. Product breakdown test passes -- DyeSub Print categories match Control "Sales by Product" report totals
  4. Customer revenue test passes -- Top 10 customers by revenue match Control report
  5. Known-gotcha tests pass -- TransDetailParam without IsActive filter, SubTotalPrice not TotalPrice, SaleDate not OrderCreatedDate all verified correct
**Plans**: TBD

Plans:
- [ ] 07-01: Execute revenue baseline, product breakdown, and customer revenue tests
- [ ] 07-02: Execute date range, gotcha, and remaining tests; compile scorecard

### Phase 8: Documentation & Milestone Close
**Goal**: Complete maintainability documentation exists and all 35 requirements are verified complete, closing Milestone 1
**Depends on**: All previous phases
**Requirements**: DOC-01, DOC-02, DOC-03, DOC-04, DOC-05, DOC-06
**Success Criteria** (what must be TRUE):
  1. Skill architecture document shows dependency graph, what each skill covers (core, sales, financial, glossary), and how they interact
  2. Query pattern reference catalogs all validated SQL patterns with the natural language trigger phrases that invoke them
  3. Business process documentation covers the order lifecycle (estimate -> order -> production -> shipping -> invoice) with table references at each stage
  4. Crystal Reports catalog lists which reports exist, their parameters, and guidance on when to use them vs. custom queries
  5. Known gotchas and validation methodology documented -- a future maintainer reading these would avoid the IsActive/TransDetailParam trap, TotalPrice vs SubTotalPrice trap, and SaleDate vs OrderCreatedDate trap
**Plans**: TBD

Plans:
- [ ] 08-01: Document skill architecture, query patterns, and business processes
- [ ] 08-02: Document Crystal Reports catalog, gotchas, and validation methodology
- [ ] 08-03: Cross-check all 35 requirements and close milestone

## Progress

**Execution Order:**
Wave 1 (parallel): 1, 2, 3 -> Wave 2 (parallel): 4, 5 -> Wave 3 (parallel): 6, 7 -> Wave 4: 8

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Core Skill Verification | 1/1 | Complete | 2026-02-08 |
| 2. Schema Infrastructure | 0/2 | Not started | - |
| 3. Wiki Knowledge Formalization | 2/2 | Complete | 2026-02-08 |
| 4. Sales Skill Verification | 0/2 | Not started | - |
| 5. Financial Foundation | 0/2 | Not started | - |
| 6. Glossary Skill Creation | 0/1 | Not started | - |
| 7. Test Suite Execution | 0/2 | Not started | - |
| 8. Documentation & Milestone Close | 0/3 | Not started | - |
