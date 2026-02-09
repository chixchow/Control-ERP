# Requirements: Project Cornerstone v1.1

**Defined:** 2026-02-09
**Core Value:** Any FLS team member can ask a business question in plain English and get an accurate, formatted answer from their ERP data -- without touching Control's UI or knowing SQL.

## v1.1 Requirements

Requirements for read-layer domain expansion. Each maps to roadmap phases.

### Financial Depth

- [ ] **FIN-04**: AR aging report matching Control's built-in report (current, 30, 60, 90+ day buckets with customer breakdown)
- [ ] **FIN-05**: AP tracking and vendor payment queries (outstanding payables, vendor balances, payment history)
- [ ] **FIN-06**: P&L reporting with product-line breakdown (revenue, COGS, gross margin by product category)
- [ ] **FIN-07**: Cash flow and bank balance queries (current bank balances, cash in/out summary, payment forecast)

### Customer Intelligence

- [ ] **CUST-01**: Customer lookup, contact info, account status queries (search by name, view contacts, addresses, phone numbers, account flags)
- [ ] **CUST-02**: Customer segmentation and CLV analysis (revenue ranking, order frequency, average order value, lifetime value calculation)
- [ ] **CUST-03**: Churn detection (customers inactive > N days, last order date, revenue at risk)

### Inventory Management

- [ ] **INV-01**: Parts and stock level queries (current quantities, parts by category, low stock identification)
- [ ] **INV-02**: Reorder point monitoring and alerts (parts below reorder threshold, suggested reorder quantities)
- [ ] **INV-03**: Warehouse and material planning queries (warehouse inventory, material consumption rates, usage history)

### Production Workflow

- [ ] **PROD-01**: Artwork status and approval pipeline queries (artwork by status, pending approvals, proof history, turnaround time)
- [ ] **PROD-02**: Station workload and scheduling queries (jobs per station, station utilization, bottleneck identification)
- [ ] **PROD-03**: Time tracking and labor analysis from TimeCard tables (hours by employee, station time, labor cost per order)

### Reports Catalog

- [ ] **RPT-01**: Crystal Report catalog -- what exists, parameters, when to use vs. custom query (36 reports mapped to user intents, integrated into glossary routing)

## Validation Standard

Every domain skill must validate against Control's built-in reports before shipping. Same standard as v1.0 (99.98% accuracy baseline). Where no matching Control report exists, queries must be internally consistent with validated core skill results (e.g., customer revenue totals must match core revenue baseline).

## Future Requirements

Deferred to later milestones. Tracked but not in current roadmap.

### HR / Payroll

- **HR-01**: Employee, payroll, PTO queries (read-only)
- **HR-02**: Timecard reporting and labor cost analysis

### Analytics

- **ANLYT-01**: Sales forecasting based on historical patterns
- **ANLYT-02**: Trend analysis and seasonal pattern detection
- **DASH-01**: Interactive executive dashboard (React artifacts)

### Division Reporting

- **DIV-01**: Banner Division vs Apparel Division breakdowns for all query types

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Write operations (any domain) | Milestone 3 scope -- all writes through CHAPI |
| Division-level reporting | Future nice-to-have, not core to read-layer completeness |
| HR/Payroll queries | Lower priority, not actively queried by team |
| Sales forecasting / dashboards | Analytics layer deferred until query foundation complete |
| Bank reconciliation | Write operation (matching transactions), not read-only |
| GL journal entry creation | Write operation |
| Inventory adjustments | Write operation -- CHAPI only |
| Real-time production monitoring | Point-in-time queries only, no streaming |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| FIN-04 | Phase 9 | Pending |
| FIN-05 | Phase 9 | Pending |
| FIN-06 | Phase 9 | Pending |
| FIN-07 | Phase 9 | Pending |
| CUST-01 | Phase 10 | Pending |
| CUST-02 | Phase 10 | Pending |
| CUST-03 | Phase 10 | Pending |
| INV-01 | Phase 11 | Pending |
| INV-02 | Phase 11 | Pending |
| INV-03 | Phase 11 | Pending |
| PROD-01 | Phase 12 | Pending |
| PROD-02 | Phase 12 | Pending |
| PROD-03 | Phase 12 | Pending |
| RPT-01 | Phase 13 | Pending |

**Coverage:**
- v1.1 requirements: 14 total
- Mapped to phases: 14
- Unmapped: 0

---
*Requirements defined: 2026-02-09*
*Last updated: 2026-02-09 after roadmap creation*
