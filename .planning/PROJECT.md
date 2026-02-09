# Project Cornerstone: AI-Powered Control ERP Interface

## What This Is

A natural language interface to the Control ERP system at FLS Banners ($3M revenue, scaling to $10-15M). Four validated Claude skills (core, sales, financial, glossary) translate plain English questions into accurate SQL queries against the StoreData SQL Server database. The system includes 187 fully-documented table schemas, a formalized wiki knowledge base (1,769 pages), and a 21-test validation suite proving 99.98% accuracy against known financials. Future milestones will expand the read-layer to all business domains and add write operations through CHAPI.

## Core Value

Any FLS team member can ask a business question in plain English and get an accurate, formatted answer from their ERP data — without touching Control's UI or knowing SQL.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

**v1.0 — Foundation + Sales (35 requirements):**
- ✓ **CORE-01**: Core skill with validated TransactionType mappings, StatusID reference, pricing logic — v1.0
- ✓ **CORE-02**: Knowledge log structured with date/query/finding/source format — v1.0
- ✓ **CORE-03**: 187 table schemas with columns, types, nullable flags, row counts — v1.0
- ✓ **CORE-04**: FK relationships mapped (89 explicit + inferred from naming patterns) — v1.0
- ✓ **CORE-05**: Domain classification (16 domains, all tables grouped) — v1.0
- ✓ **CORE-06**: ClassTypeID reference (350+ entries) in skill-consumable format — v1.0
- ✓ **SALES-01**: DyeSub Print container variable architecture documented — v1.0
- ✓ **SALES-02**: Table Cover identification patterns documented — v1.0
- ✓ **SALES-03**: Non-container product identification documented — v1.0
- ✓ **SALES-04**: Product revenue validated ($1,793K DyeSub, matching Control report within $3) — v1.0
- ✓ **SALES-05**: Customer revenue validated (FLASH #1 at $430K, top 10 = 49.3%) — v1.0
- ✓ **SALES-06**: Sales trends validated (monthly, quarterly, YoY) — v1.0
- ✓ **FIN-01**: GL/Ledger architecture, sign conventions, GLClassificationType reference — v1.0
- ✓ **FIN-02**: GL system account NodeIDs (11=WIP, 12=Built, 14=AR, 21=Orders Due, etc.) — v1.0
- ✓ **FIN-03**: Payment GL offset logic (deposit workflow, TenderType, Built cost flow) — v1.0
- ✓ **WIKI-01**: 1,769 wiki pages crawled across 3 subdomains — v1.0
- ✓ **WIKI-02**: 12 knowledge extracts produced (6 required, 12 delivered) — v1.0
- ✓ **WIKI-03**: CHAPI architecture documented (17 stored procs, HTTP endpoint) — v1.0
- ✓ **WIKI-04**: Crystal Reports SQL patterns extracted — v1.0
- ✓ **WIKI-05**: Macro automation documented (95 message types, 11 RuleAction types) — v1.0
- ✓ **GLOSS-01**: Glossary skill created with YAML frontmatter — v1.0
- ✓ **GLOSS-02**: Product line terminology mapped (20 DyeSub + 7 non-DyeSub categories) — v1.0
- ✓ **GLOSS-03**: Control technical terms defined (CHAPI, SSLIP, CFL, SQLBridge, etc.) — v1.0
- ✓ **TEST-01**: 21-test validation suite executed with formal scorecard — v1.0
- ✓ **TEST-02**: Revenue baseline $3,052,952.52 (0.02% variance from known $3,053,541.85) — v1.0
- ✓ **TEST-03**: DyeSub Print matches Control report ($1,793,445, within $3) — v1.0
- ✓ **TEST-04**: Top 10 customers match Control report — v1.0
- ✓ **TEST-05**: Monthly/quarterly/YoY queries internally consistent — v1.0
- ✓ **TEST-06**: Known-gotcha tests all pass ($1.3M+ error prevention) — v1.0
- ✓ **DOC-01**: Skill architecture with dependency graph — v1.0
- ✓ **DOC-02**: Query pattern reference (39 NL triggers across 8 categories) — v1.0
- ✓ **DOC-03**: Order lifecycle documentation (9 stages with table references) — v1.0
- ✓ **DOC-04**: Crystal Reports catalog (36 reports) — v1.0
- ✓ **DOC-05**: Known gotchas (13 entries, $1.3M+ errors prevented) — v1.0
- ✓ **DOC-06**: Validation methodology (6-step reproducible process) — v1.0

### Active

<!-- Current scope. Building toward these. -->

**Read-Layer Skills (Milestone 2):**
- [ ] **FIN-04**: AR aging report matching Control's built-in report
- [ ] **FIN-05**: AP tracking and vendor payment queries
- [ ] **FIN-06**: P&L reporting with product-line breakdown
- [ ] **FIN-07**: Cash flow and bank balance queries
- [ ] **CUST-01**: Customer lookup, contact info, account status queries
- [ ] **CUST-02**: Customer segmentation and CLV analysis
- [ ] **CUST-03**: Churn detection (customers inactive > N days)
- [ ] **INV-01**: Parts and stock level queries
- [ ] **INV-02**: Reorder point monitoring and alerts
- [ ] **INV-03**: Warehouse and material planning queries
- [ ] **PROD-01**: Artwork status and approval pipeline queries
- [ ] **PROD-02**: Station workload and scheduling queries
- [ ] **PROD-03**: Time tracking and labor analysis from TimeCard tables
- [ ] **HR-01**: Employee, payroll, PTO queries (read-only)
- [ ] **HR-02**: Timecard reporting and labor cost analysis
- [ ] **RPT-01**: Crystal Report catalog — what exists, parameters, when to use vs. custom query
- [ ] **ANLYT-01**: Sales forecasting based on historical patterns
- [ ] **ANLYT-02**: Trend analysis and seasonal pattern detection
- [ ] **DASH-01**: Interactive executive dashboard (React artifacts)

**Write-Layer Skills (Milestone 3):**
- [ ] **WRITE-01**: Create customer accounts/contacts through CHAPI import stored procedures
- [ ] **WRITE-02**: Create estimates through CHAPI with confirmation workflow
- [ ] **WRITE-03**: Create orders through CHAPI with full validation and confirmation
- [ ] **WRITE-04**: Create POs and receiving documents through CHAPI
- [ ] **WRITE-05**: Inventory adjustments through CHAPI
- [ ] **WRITE-06**: Intent verification system — Claude always confirms before any write
- [ ] **WRITE-07**: Audit trail for all write operations
- [ ] **ORCH-01**: Orchestrator skill routing natural language to correct domain skill(s)
- [ ] **ORCH-02**: Multi-step workflow handling spanning multiple skills

### Out of Scope

- **Real-time sync/streaming** — Queries are point-in-time reads, not live feeds. Control doesn't support change data capture.
- **Replacing Control UI entirely** — This augments Control, doesn't replace it. Complex product configuration still happens in Control.
- **Multi-company support** — FLS Banners single-company instance only. No multi-tenant abstraction.
- **Mobile app** — Web/CLI interface only. Mobile can come later.
- **Direct SQL writes** — ALL writes go through CHAPI. No exceptions. Ever. This is a hard safety rule.
- **Third-party integrations** — No Salesforce, QuickBooks, etc. Control is the system of record.

## Context

**Company:** FLS Banners — custom flag, banner, and signage manufacturer. $3M current revenue scaling to $10-15M.

**ERP System:** Cyrious Control (sign industry ERP). SQL Server backend (StoreData database), 192+ tables. Proprietary, industry-specific with limited documentation — which is why the wiki crawl was so valuable.

**Technical Environment:**
- Database: SQL Server accessed via MCP MSSQL tools
- Skills: Claude Code skill files (SKILL.md + reference docs)
- Write path: CHAPI HTTP API on port 12556 with SQLBridge stored procedures
- Wiki knowledge: 1,769 pages crawled and extracted into 12 knowledge documents
- Schema: 187 tables fully documented with relationships and domain classification

**Team:**
- Cain (CMO/President) — project owner, primary user
- Taylor, Gretel — business partners, stakeholders for validation gates
- Gretel — accounting team, validates financial queries

**Current State (after v1.0):**
- `control-erp-core` skill: Complete, validated to 99.98% against known 2025 financials
- `control-erp-sales` skill: Complete with product variable architecture, validated against Control reports
- `control-erp-financial` skill: Foundation complete — GL/Ledger architecture, system accounts, payment logic
- `control-erp-glossary` skill: Complete — FLS terminology, Control terms, NL routing
- 187 table schemas in `output/schemas/`
- Relationships mapped in `output/relationships.md`
- Domains classified in `output/domains.md`
- Crystal Report analysis in `output/reports/` and `output/report_join_patterns.md`
- Wiki knowledge base in `output/wiki/` with 12 extracts
- Knowledge log in `documentation/control-erp-knowledge-log.md`
- Validation scorecard in `validation/milestone1-scorecard.md`
- Documentation suite in `output/docs/` (architecture, query patterns, gotchas, methodology)

## Constraints

- **Read path**: Direct SQL via MCP tools. Fast, flexible, no middleware.
- **Write path**: MUST go through CHAPI stored procedures. No direct SQL writes. Data integrity depends on this.
- **Validation gates**: Each phase must be validated against known Control report outputs before proceeding. No skill ships without proving accuracy.
- **Wiki dependency**: Much domain knowledge comes from the Cyrious wiki crawl. If wiki information conflicts with actual database behavior, database wins.
- **Timeline**: Wiki data compressed original 26-week estimate to ~6 weeks for read-layer completion. Write operations add additional time pending CHAPI accessibility testing.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Split monolithic skill into domain-focused skills | Single skill was too large for context window, mixed concerns, hard to maintain | ✓ Good — core/sales/financial/glossary separation is clean |
| Use SubTotalPrice not TotalPrice for revenue | SubTotalPrice matches FLS's definition of "income" (pre-tax) | ✓ Good — validated to 99.98% |
| Use SaleDate not OrderCreatedDate for date filtering | SaleDate reflects when sale was finalized, not when order was created | ✓ Good — matches Control reports |
| Never filter IsActive on TransDetailParam | Control sets IsActive=false after product config; data is still valid | ✓ Good — fixed $1.3M discrepancy |
| All writes through CHAPI, never direct SQL | Data integrity, business rules, validation, audit trails managed by Control | — Pending (not yet implemented) |
| Three-milestone structure | M1=Foundation+Sales, M2=Read-layer build-out, M3=Write operations | ✓ Good — M1 shipped on schedule |
| Wiki crawl before Phase 3+ development | Discovered CHAPI docs, GL mapping, ClassTypeIDs — eliminated research phases | ✓ Good — compressed timeline by ~6 weeks |
| 8-phase structure with 4-wave parallelism | 3+2+2+1 wave structure maximized throughput with 5 concurrent agents | ✓ Good — 15 plans in 1.04 hours |
| Cross-reference validation when MCP unavailable | Validated against 2026-02-07 live MCP results from Mac environment | ✓ Good — historical 2025 data is stable |
| Glossary uses cross-reference pattern | Points to owning skills rather than duplicating query templates | ✓ Good — single source of truth maintained |

---
*Last updated: 2026-02-09 after v1.0 milestone*
