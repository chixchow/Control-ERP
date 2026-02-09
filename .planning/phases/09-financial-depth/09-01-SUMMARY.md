---
phase: 09-financial-depth
plan: 01
subsystem: financial
tags: [AR, AP, aging, customer-breakdown, vendor-breakdown, payment-history]
depends_on:
  requires: [phase-05-financial]
  provides: [AR-detail-customer-breakdown, AP-detail-vendor-breakdown, vendor-payment-history, NL-routing-AR-AP-detail]
  affects: [09-02-plan]
tech_stack:
  added: []
  patterns: [customer-breakdown-aging, vendor-breakdown-aging, DueDate-vs-SaleDate-aging]
key_files:
  created: []
  modified:
    - skills/control-erp-financial/control-erp-financial-SKILL.md
decisions:
  - AR aging uses SaleDate exclusively; WIP/Built orders in separate bucket
  - AP aging uses DueDate exclusively; Current bucket = not yet due
  - Vendor payment history via Payment.ClassTypeID 20009 JOIN Journal JOIN TransHeader
metrics:
  duration: 2m22s
  completed: 2026-02-09
---

# Phase 9 Plan 01: AR/AP Detail with Customer/Vendor Breakdown Summary

AR and AP detail sections with customer/vendor breakdown queries, single-entity drill-down, and vendor payment history using validated SaleDate (AR) and DueDate (AP) aging patterns.

## What Was Done

### Task 1: AR Detail with Customer Breakdown (FIN-04)
Added three new AR query templates after the existing AR by Payment Terms section:

1. **Full AR Detail by Customer** -- TransHeader JOIN Account JOIN PaymentTerms with aging bucket columns (Current_0_30, Days_31_60, Days_61_90, Days_Over_90). Uses SaleDate for aging. WIP/Built orders (SaleDate IS NULL) shown in separate WIPFlag column. HasCreditAccount marked with asterisk.

2. **Single Customer AR Drill-Down** -- Invoice-level detail filtered by CompanyName LIKE. Shows OrderNumber, InvoiceNumber, SaleDate, Description, InvoiceTotal, PaymentTotal, BalanceDue, DaysOutstanding, StatusText, FinanceChargeAmount.

3. **AR Summary by Customer** -- Grouped version with one row per customer showing total AR and aging bucket totals. Ordered by TotalAR DESC for quick identification of largest debtors.

Added 3 NL routing entries: "AR detail" / "AR by customer" / "who owes us", "AR for [customer]" / "what does [customer] owe", "AR summary by customer".

### Task 2: AP Detail with Vendor Breakdown (FIN-05)
Added three new AP query templates after the existing AP Context section:

1. **Full AP Detail by Vendor** -- TransHeader (Type 8) JOIN Account JOIN PaymentTerms (VendorPaymentTermsID). Uses DueDate for aging with Current bucket (not yet due, DATEDIFF <= 0). Shows BillNumber, DueDate, Description, BillAmount, BalanceDue.

2. **AP Summary by Vendor** -- Grouped version with one row per vendor showing total AP and aging bucket totals. Ordered by TotalAP DESC.

3. **Vendor Payment History** -- Payment JOIN Journal JOIN TransHeader JOIN Account filtered by ClassTypeID 20009 (Bill Payment). Shows PaymentDate, BillNumber, Description, PaymentAmount, PaymentMethod (decoded TenderType), Reference (DisplayNumber).

Added 3 NL routing entries: "AP detail" / "AP by vendor" / "what do we owe", "AP for [vendor]" / "bills for [vendor]", "vendor payment history" / "payments to [vendor]".

## Decisions Made

| Decision | Rationale |
|----------|-----------|
| AR ages from SaleDate, AP ages from DueDate | Matches Control's built-in reports; DueDate has different semantics per transaction type |
| WIP/Built orders in separate bucket, not aging columns | Cannot age orders with no SaleDate; Control's own report handles them this way |
| Vendor payment history via ClassTypeID 20009 | 20009 = Bill Payment; filters to individual vendor bill payments only |
| VendorCreditBalance documented as note, not query | Simple field lookup, not a complex query pattern |

## Deviations from Plan

None -- plan executed exactly as written.

## Metrics

- **File growth:** 856 lines -> 1038 lines (+182 lines, 21% increase)
- **Under extraction threshold:** 1038 < 1200 lines (safe to keep in single file)
- **New NL routing entries:** 6 total (3 AR + 3 AP)
- **New query templates:** 6 total (3 AR + 3 AP)

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | f49f142 | feat(09-01): add AR detail with customer breakdown (FIN-04) |
| 2 | 7140b4b | feat(09-01): add AP detail with vendor breakdown and payment history (FIN-05) |

## Next Phase Readiness

Plan 09-02 (P&L with product-line analysis and cash flow) can proceed immediately. The skill file is at 1038 lines -- adding the estimated ~170 additional lines from plan 09-02 will bring it to ~1,208 lines, which is near the 1,200-line extraction threshold. Plan 09-02 should monitor the line count and extract to references/ if needed.
