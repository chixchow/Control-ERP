# Control ERP Glossary - Gap Log

**Purpose:** Track every query domain that falls outside current skill coverage. When a user query cannot be answered by any skill, log the domain here. This creates a natural backlog for future skill development.

**Update policy:** Add entries when:
- A user asks a question no skill can answer
- The domain has tables in the database (not a random curiosity)
- The question represents a recurring business need

## Uncovered Domains

| Domain | Example Queries | Tables Exist | Priority | Notes |
|--------|----------------|-------------|----------|-------|
| Payroll / TimeCard | "show me timecards", "who worked overtime", "payroll summary" | Yes (Payroll, PayrollPaycheck, TimeCard) | Medium | Tables fully mapped; FLS does not use per-job time tracking |
| Commissions | "commission rates", "sales commission report", "rep earnings" | Yes (CommissionPlan, CommissionRate) | Low | Tables exist but unknown if FLS uses commissions actively |
| Tax Compliance | "sales tax collected", "tax by state", "tax report" | Yes (TaxClass, TaxLink, PostalCodeTaxClass) | Medium | Tax tables mapped; may be useful for compliance reporting |
| Service Tickets | "open service tickets", "service history" | Yes (ServiceContractType, ServiceTicketPriority) | Low | Only 1 record found in 2025; minimal usage |
| Shipping Detail | "tracking numbers", "shipped today", "shipment history" | Yes (Shipments, FedExShippingLog, UPSShippingLog) | Medium | Tables mapped; shipped-by-date report is not yet covered |
| Product Configuration | "product parameters", "product detail", "how is [product] configured" | Yes (Product, Variable, PricingPlan) | Medium | Product Detail and Product Parameters reports not covered |
| Est vs Actual Cost | "estimated vs actual cost", "cost variance" | Yes (TransDetail cost fields) | Low | Est. vs Act. Cost Summary report not covered |

## Crystal Reports Not Yet Covered

These 5 Crystal Reports have no skill replacement. Run them in Control until a skill is developed:

1. **Est. vs Act. Cost Summary By Order.rpt** — Compares estimated costs to actual costs per order
2. **Product Detail.rpt** — Full product configuration and pricing details
3. **Product Parameters.rpt** — Product parameter/variable definitions
4. **Shipped By Date.rpt** — Shipment tracking by date range
5. **Work Order with Proof.rpt** — Formatted work order with embedded proof images (requires rendering, not query-replaceable)

---

*Last updated: 2026-02-10*
*Domains added: 7 | Reports uncovered: 5*
