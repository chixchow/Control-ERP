# Requirements: Project Cornerstone

**Defined:** 2026-02-08
**Core Value:** Any FLS team member can ask a business question in plain English and get an accurate, formatted answer from their ERP data.

## v1 Requirements (Milestone 1: Foundation + Sales — Validation & Documentation)

Milestone 1 formalizes and validates the substantial work already completed. Skills are built but need formal test suites, documentation for maintainability, and verified accuracy against known Control report outputs.

### Core Foundation

- [x] **CORE-01**: control-erp-core skill complete with validated TransactionType mappings, StatusID reference, pricing logic, and standard query filters
- [x] **CORE-02**: Knowledge log captured in structured format — every validated discovery documented with date, query, and finding
- [x] **CORE-03**: Schema documentation complete for all 187+ tables with columns, types, nullable flags, and row counts
- [x] **CORE-04**: FK relationships mapped (explicit from sys.foreign_keys + inferred from naming patterns)
- [x] **CORE-05**: Domain classification complete — tables grouped into functional domains with descriptions
- [x] **CORE-06**: ClassTypeID reference compiled from wiki extracts (350+ entries) into skill-consumable format

### Sales Intelligence

- [x] **SALES-01**: control-erp-sales skill complete with DyeSub Print container variable architecture (FP_ProductDescription categories, FP_ProductID SKUs)
- [x] **SALES-02**: Table Cover product identification documented (Description pattern matching, GoodsItemID=10026)
- [x] **SALES-03**: Non-container product identification documented (Shipping, Artwork/Setup, Hardware, Custom Items)
- [x] **SALES-04**: Product category revenue validated against Control "Sales by Product" report
- [x] **SALES-05**: Customer revenue queries validated (top customers, CLV basics, account lookup)
- [x] **SALES-06**: Sales trend queries validated (monthly, quarterly, YoY comparison)

### Financial Foundation

- [ ] **FIN-01**: control-erp-financial skill started with GL/Ledger architecture, sign conventions, GLClassificationType reference
- [ ] **FIN-02**: GL system account NodeIDs documented (11=WIP, 12=Built, 14=AR, 21=Orders Due, 24=Customer Deposits, 91=Undeposited Funds)
- [ ] **FIN-03**: Payment GL offset logic documented (WIP/Built deposits vs Sale payments)

### Wiki Knowledge Base

- [x] **WIKI-01**: Wiki crawled — 1,769 pages across control/reports/support subdomains
- [x] **WIKI-02**: 6 knowledge extracts produced covering database integration, orders/accounting, production/inventory, CRM/payroll, SQL queries, and troubleshooting
- [x] **WIKI-03**: CHAPI architecture documented from wiki — HTTP endpoint, SQLBridge functions, import stored procedures
- [x] **WIKI-04**: Crystal Reports SQL patterns extracted from wiki
- [x] **WIKI-05**: Macro automation system documented (95 trigger events, RuleAction types)

### Glossary

- [ ] **GLOSS-01**: control-erp-glossary skill created with FLS-specific terminology mapping (business language → database entities)
- [ ] **GLOSS-02**: Product line terminology documented (blanket, SEG, Fab Frame, DyeLux, etc.)
- [ ] **GLOSS-03**: Control-specific terminology mapped (CHAPI, SSLIP, CFL, SQLBridge, ClassTypeID patterns)

### Test Suite & Validation

- [ ] **TEST-01**: Phase 2 test suite (21 test cases) executed with pass/fail scorecard
- [ ] **TEST-02**: Revenue baseline test — "Total sales 2025" returns $3,053,541.85 (±1%)
- [ ] **TEST-03**: Product breakdown test — DyeSub Print categories match Control report totals
- [ ] **TEST-04**: Customer revenue test — Top 10 customers by revenue match Control report
- [ ] **TEST-05**: Date range tests — monthly, quarterly, YoY queries return consistent totals
- [ ] **TEST-06**: Known-gotcha tests — TransDetailParam without IsActive filter, SubTotalPrice not TotalPrice, SaleDate not OrderCreatedDate

### Documentation & Maintainability

- [ ] **DOC-01**: Skill architecture documented — dependency graph, what each skill covers, how they interact
- [ ] **DOC-02**: Query pattern reference — all validated SQL patterns with natural language triggers
- [ ] **DOC-03**: Business process documentation — order lifecycle (estimate → order → production → shipping → invoice)
- [ ] **DOC-04**: Crystal Reports catalog — which reports exist, parameters, when to use vs. custom query
- [ ] **DOC-05**: Known gotchas and pitfalls documented (things future maintainers need to know)
- [ ] **DOC-06**: Validation methodology documented — how to verify skill accuracy against Control reports

## v2 Requirements (Milestone 2: Read-Layer Build-Out)

### Financial Skills
- **FIN-04**: AR aging report matching Control's built-in report
- **FIN-05**: AP tracking and vendor payment queries
- **FIN-06**: P&L reporting with product-line breakdown
- **FIN-07**: Cash flow and bank balance queries

### Customer Skills
- **CUST-01**: Customer lookup, contact info, account status queries
- **CUST-02**: Customer segmentation and CLV analysis
- **CUST-03**: Churn detection (customers inactive > N days)

### Inventory Skills
- **INV-01**: Parts and stock level queries
- **INV-02**: Reorder point monitoring
- **INV-03**: Warehouse and material planning queries

### Production Skills
- **PROD-01**: Artwork status and approval pipeline queries
- **PROD-02**: Station workload and scheduling queries
- **PROD-03**: Time tracking and labor analysis

### HR Skills
- **HR-01**: Employee, payroll, PTO queries (read-only)
- **HR-02**: Timecard reporting and labor cost analysis

### Reports & Analytics
- **RPT-01**: Crystal Report catalog skill with routing logic
- **ANLYT-01**: Sales forecasting from historical patterns
- **ANLYT-02**: Trend analysis and seasonal detection
- **DASH-01**: Interactive executive dashboard (React artifacts)

## v3 Requirements (Milestone 3: Write Operations)

- **WRITE-01**: Customer account creation through CHAPI
- **WRITE-02**: Estimate creation through CHAPI
- **WRITE-03**: Order creation through CHAPI with confirmation workflow
- **WRITE-04**: PO and receiving document creation through CHAPI
- **WRITE-05**: Inventory adjustments through CHAPI
- **WRITE-06**: Intent verification system
- **WRITE-07**: Audit trail for all write operations
- **ORCH-01**: Orchestrator skill routing to correct domain skill(s)
- **ORCH-02**: Multi-step workflow handling

## Out of Scope

| Feature | Reason |
|---------|--------|
| Direct SQL writes to database | Data integrity — all writes must go through CHAPI stored procedures |
| Real-time data streaming | Control doesn't support CDC; queries are point-in-time reads |
| Multi-company support | FLS is single-company; no multi-tenant abstraction needed |
| Mobile app | Web/CLI interface first; mobile can come later |
| Third-party integrations | Control is system of record; no Salesforce/QuickBooks sync |
| Replacing Control UI | This augments Control, doesn't replace it; complex config stays in Control |
| Automated writes without confirmation | Safety rule — Claude always confirms before any write operation |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| CORE-01 | Phase 1: Core Skill Verification | Complete |
| CORE-02 | Phase 1: Core Skill Verification | Complete |
| CORE-03 | Phase 2: Schema Infrastructure | Complete |
| CORE-04 | Phase 2: Schema Infrastructure | Complete |
| CORE-05 | Phase 2: Schema Infrastructure | Complete |
| CORE-06 | Phase 2: Schema Infrastructure | Complete |
| SALES-01 | Phase 4: Sales Skill Verification | Complete |
| SALES-02 | Phase 4: Sales Skill Verification | Complete |
| SALES-03 | Phase 4: Sales Skill Verification | Complete |
| SALES-04 | Phase 4: Sales Skill Verification | Complete |
| SALES-05 | Phase 4: Sales Skill Verification | Complete |
| SALES-06 | Phase 4: Sales Skill Verification | Complete |
| FIN-01 | Phase 5: Financial Foundation | Pending |
| FIN-02 | Phase 5: Financial Foundation | Pending |
| FIN-03 | Phase 5: Financial Foundation | Pending |
| WIKI-01 | Phase 3: Wiki Knowledge Formalization | Complete |
| WIKI-02 | Phase 3: Wiki Knowledge Formalization | Complete |
| WIKI-03 | Phase 3: Wiki Knowledge Formalization | Complete |
| WIKI-04 | Phase 3: Wiki Knowledge Formalization | Complete |
| WIKI-05 | Phase 3: Wiki Knowledge Formalization | Complete |
| GLOSS-01 | Phase 6: Glossary Skill Creation | Pending |
| GLOSS-02 | Phase 6: Glossary Skill Creation | Pending |
| GLOSS-03 | Phase 6: Glossary Skill Creation | Pending |
| TEST-01 | Phase 7: Test Suite Execution | Pending |
| TEST-02 | Phase 7: Test Suite Execution | Pending |
| TEST-03 | Phase 7: Test Suite Execution | Pending |
| TEST-04 | Phase 7: Test Suite Execution | Pending |
| TEST-05 | Phase 7: Test Suite Execution | Pending |
| TEST-06 | Phase 7: Test Suite Execution | Pending |
| DOC-01 | Phase 8: Documentation & Milestone Close | Pending |
| DOC-02 | Phase 8: Documentation & Milestone Close | Pending |
| DOC-03 | Phase 8: Documentation & Milestone Close | Pending |
| DOC-04 | Phase 8: Documentation & Milestone Close | Pending |
| DOC-05 | Phase 8: Documentation & Milestone Close | Pending |
| DOC-06 | Phase 8: Documentation & Milestone Close | Pending |

**Coverage:**
- v1 (Milestone 1) requirements: 35 total (7 categories)
- Mapped to phases: 35
- Unmapped: 0

**By category:**
- Core Foundation (6): Phase 1 (2), Phase 2 (4)
- Sales Intelligence (6): Phase 4 (6)
- Financial Foundation (3): Phase 5 (3)
- Wiki Knowledge Base (5): Phase 3 (5)
- Glossary (3): Phase 6 (3)
- Test Suite & Validation (6): Phase 7 (6)
- Documentation & Maintainability (6): Phase 8 (6)

---
*Requirements defined: 2026-02-08*
*Last updated: 2026-02-08 after roadmap v2 (8-phase structure with parallelism)*
