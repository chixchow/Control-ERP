# Project Cornerstone: AI-Powered Control ERP Interface

## What This Is

A natural language interface to the Control ERP system at FLS Banners ($3M → $15M growth trajectory). Claude skills translate plain English questions into accurate SQL queries, formatted reports, and eventually record creation — all without the team needing to learn Control's UI or write SQL. The system reads directly from the StoreData SQL Server database for queries and uses CHAPI (Control's HTTP API) for all write operations.

## Core Value

Any FLS team member can ask a business question in plain English and get an accurate, formatted answer from their ERP data — without touching Control's UI or knowing SQL.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

- ✓ **CORE-01**: Validated business rules mapping TransactionType, StatusID, and pricing logic against known 2025 financials ($3,053,541.85 matched to 99.98%) — existing (`control-erp-core`)
- ✓ **CORE-02**: TransactionType reference validated — Type 1=Order/Sale, Type 2=Estimate, Type 7/8/9=Vendor chain, Types 3/4/5 unused at FLS — existing (`control-erp-core`)
- ✓ **CORE-03**: Revenue query formula validated — `SUM(SubTotalPrice)` on TransHeader with `TransactionType=1`, `SaleDate` filtering, header-level totals — existing (`control-erp-core`)
- ✓ **CORE-04**: Natural language → TransactionType mapping (user says "sales" → Type 1, "estimates" → Type 2, etc.) — existing (`control-erp-core`)
- ✓ **SALES-01**: Product variable architecture documented — DyeSub Print container with FP_ProductDescription (categories) and FP_ProductID (SKUs) — existing (`control-erp-sales`)
- ✓ **SALES-02**: Table Cover product identification via Description pattern matching (TC_ variables not stored in TransDetailParam) — existing (`control-erp-sales`)
- ✓ **SALES-03**: Product category revenue breakdown validated against Control "Sales by Product" report (DyeSub Print = $1,793K, 58.7% of revenue) — existing (`control-erp-sales`)
- ✓ **SALES-04**: TransDetailParam query rules documented — NEVER filter IsActive or ParentClassTypeID on this table — existing (`control-erp-sales`)
- ✓ **SCHEMA-01**: All 187 database tables schema-documented with columns, data types, nullable flags, row counts — existing (`output/schemas/`)
- ✓ **SCHEMA-02**: All FK relationships mapped (explicit + inferred) — existing (`output/relationships.md`)
- ✓ **SCHEMA-03**: Tables classified into functional domains — existing (`output/domains.md`)
- ✓ **WIKI-01**: Cyrious wiki crawled (1,769 pages across control/reports/support subdomains) with 6 knowledge extracts — existing (`output/wiki/`)
- ✓ **WIKI-02**: Complete ClassTypeID mapping (350+ entries) extracted from wiki — existing (`output/wiki/extracts/`)
- ✓ **WIKI-03**: CHAPI architecture documented — HTTP endpoint port 12556, SQLBridge functions, 16+ import stored procedures — existing (wiki extracts)
- ✓ **WIKI-04**: GL account system mapped — fixed system account IDs (11=WIP, 12=Built, 14=AR, 21=Orders Due, 24=Customer Deposits, 91=Undeposited Funds) — existing (wiki extracts)
- ✓ **FIN-01**: GL/Ledger architecture documented — GL view vs Ledger table, sign conventions, GLClassificationType reference — existing (`control-erp-financial`, started)

### Active

<!-- Current scope. Building toward these. -->

**Read-Layer Skills (Milestone 2):**
- [ ] **FIN-02**: AR aging report matching Control's built-in report
- [ ] **FIN-03**: AP tracking and vendor payment queries
- [ ] **FIN-04**: P&L reporting with product-line breakdown
- [ ] **FIN-05**: Cash flow and bank balance queries
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
- [ ] **GLOSS-01**: FLS-specific terminology mapping (business language → database entities)

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
- Wiki knowledge: 1,769 pages crawled and extracted into 6 knowledge documents
- Schema: 187 tables fully documented with relationships and domain classification

**Team:**
- Cain (CMO/President) — project owner, primary user
- Taylor, Gretel — business partners, stakeholders for validation gates
- Gretel — accounting team, validates financial queries

**Prior Work:**
- `control-erp-core` skill: Complete, validated to 99.98% against known 2025 financials
- `control-erp-sales` skill: Complete with product variable architecture
- `control-erp-financial` skill: Started — GL/Ledger architecture documented, needs AR/AP/P&L
- 187 table schemas in `output/schemas/`
- Relationships mapped in `output/relationships.md`
- Domains classified in `output/domains.md`
- Crystal Report analysis in `output/reports/` and `output/report_join_patterns.md`
- Wiki knowledge base in `output/wiki/` with extracts
- Knowledge log in `documentation/control-erp-knowledge-log.md`
- Validation results in `validation/`

## Constraints

- **Read path**: Direct SQL via MCP tools. Fast, flexible, no middleware.
- **Write path**: MUST go through CHAPI stored procedures. No direct SQL writes. Data integrity depends on this.
- **Validation gates**: Each phase must be validated against known Control report outputs before proceeding. No skill ships without proving accuracy.
- **Wiki dependency**: Much domain knowledge comes from the Cyrious wiki crawl. If wiki information conflicts with actual database behavior, database wins.
- **Timeline**: Wiki data compressed original 26-week estimate to ~6 weeks for read-layer completion. Write operations add additional time pending CHAPI accessibility testing.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Split monolithic skill into domain-focused skills | Single skill was too large for context window, mixed concerns, hard to maintain | ✓ Good — core/sales/financial separation is clean |
| Use SubTotalPrice not TotalPrice for revenue | SubTotalPrice matches FLS's definition of "income" (pre-tax) | ✓ Good — validated to 99.98% |
| Use SaleDate not OrderCreatedDate for date filtering | SaleDate reflects when sale was finalized, not when order was created | ✓ Good — matches Control reports |
| Never filter IsActive on TransDetailParam | Control sets IsActive=false after product config; data is still valid | ✓ Good — fixed $1.3M discrepancy |
| All writes through CHAPI, never direct SQL | Data integrity, business rules, validation, audit trails managed by Control | — Pending (not yet implemented) |
| Three-milestone structure | M1=Foundation+Sales (done), M2=Read-layer build-out, M3=Write operations | — Pending |
| Wiki crawl before Phase 3+ development | Discovered CHAPI docs, GL mapping, ClassTypeIDs — eliminated research phases | ✓ Good — compressed timeline by ~6 weeks |

---
*Last updated: 2026-02-08 after GSD initialization*
