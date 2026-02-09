---
phase: 05-financial-foundation
plan: 01
subsystem: financial
tags: [gl, ledger, accounts, closeout, documentation]
requires: [01-core-skill, 02-schema-infrastructure]
provides: [corrected-system-accounts, ledger-field-reference, closeout-reference]
affects: [05-02-payment-offset-logic]
tech-stack:
  added: []
  patterns: []
key-files:
  created: []
  modified:
    - skills/control-erp-financial/control-erp-financial-SKILL.md
decisions:
  - key: system-account-names
    choice: Use exact GLAccount.AccountName from database, not wiki aliases
    rationale: Source of truth for account names; ensures consistency with live data
  - key: undeposited-funds-terminology
    choice: GLClassificationType 1007 = "Undeposited Funds" (category), individual accounts use specific names
    rationale: Distinguishes between category classification and account names
  - key: entry-type-confidence
    choice: EntryType values marked MEDIUM confidence
    rationale: Inferred from Description pattern correlation, not official documentation
metrics:
  duration: 2min36s
  completed: 2026-02-09
---

# Phase 5 Plan 1: GL Account Verification & Field Reference Summary

**One-liner:** Corrected system account NodeIDs to exact database names and added comprehensive Ledger field reference with EntryType mappings and Closeout documentation.

## What Was Built

Validated and corrected the existing financial skill's GL account documentation, fixing name inaccuracies and adding missing reference sections for advanced financial queries.

### Task 1: System Account Corrections
- Fixed NodeID 90 from "Cash-Checking" to "Cash-Associated Bank-Checking" (exact database name)
- Fixed NodeID 91 from "Undeposited Funds" to "Undeposited Cash" in Stage 2 GL entry example
- Added Supporting System Accounts section documenting 11 previously missing accounts:
  - 15 = Direct Costs from Bills (intermediary for bill costs)
  - 25 = Vendor Credit (vendor credit balances)
  - 34 = Cost Of Built - FGI (holds costs during Built status)
  - 52 = Credit Given (customer credit adjustments)
  - 60 = Unclassified Inventory (off-balance sheet cost tracking)
  - 61 = Unclassified Expense (off-balance sheet expense tracking)
  - 93 = Undeposited Checks (check deposits pending)
  - 543 = Undeposited Discover (Discover CC deposits pending)
  - 10528 = Undeposited Authorize.net (online CC deposits pending)
  - 10530 = Undeposited PayPal (PayPal deposits pending)
  - 10531 = Undeposited Shopify (Shopify deposits pending)
- Added Complete Undeposited Fund Accounts section listing all GLClassificationType=1007 accounts with Active/Legacy status
- Marked ZoomTex accounts (10522-10525) as Legacy

### Task 2: Ledger Field Reference & Closeout
- Added Complete Ledger Field Reference documenting 12 query-relevant fields beyond the 5 key fields:
  - EntryType (with value mappings)
  - Classification
  - OffBalanceSheet
  - DepositJournalID
  - Reconciled / ReconciliationDateTime
  - DivisionID, StationID, PartID, PayrollID, WarehouseID
  - ClassTypeID (always 8900)
- Added EntryType Reference table with 5 value mappings (NULL, 1=Manual, 2=Order lifecycle, 3=Payment, 4=Credit, 5=Bill) marked MEDIUM confidence
- Added CLOSEOUT — PERIOD LOCKING section documenting:
  - Purpose: locks GL periods to prevent backdated entries
  - Key fields: CloseoutType, StartDate, EndDate, ClosedDate, ClosedByID
  - ClassTypeID = 8911 for all Closeout records
  - CloseoutType reference: 1=Daily, 2=Monthly, 3=Yearly, 5=Export, 6=CC Settlement
  - Query example for last closeout dates by type

## Decisions Made

1. **System Account Names**: Use exact `GLAccount.AccountName` from database as authoritative source, not wiki aliases. Wiki may call NodeID 24 "Customer Deposits" but database uses "Order Prepayments" - skill documents the database name.

2. **Undeposited Funds Terminology**: GLClassificationType 1007 is correctly named "Undeposited Funds" as a category classification. Individual accounts within that category use specific names (Undeposited Cash, Undeposited MCVisa, etc.). Both usages are correct in their respective contexts.

3. **EntryType Confidence**: Ledger.EntryType value mappings marked MEDIUM confidence because they were inferred from Description field pattern correlation, not from official Cyrious documentation. Accurate for FLS data but may vary in other Control installations.

4. **ZoomTex Legacy Accounts**: NodeIDs 10522-10525 marked as Legacy rather than deleted. Historical data exists but ZoomTex division is defunct. Exclude from active reporting but preserve for historical queries.

## Deviations from Plan

None - plan executed exactly as written.

## Validation Results

All verification checks passed:

1. **Name accuracy check**: Confirmed "Undeposited Funds" only appears in GLClassificationType table (correct context) and in Supporting System Accounts table showing classification (also correct). No standalone account name references found.

2. **Cash-Checking check**: All references now use "Cash-Associated Bank-Checking" except in query comments where both names acceptable.

3. **New sections check**: All 5 required sections added:
   - Supporting System Accounts
   - Complete Undeposited Fund Accounts
   - Complete Ledger Field Reference
   - EntryType Reference
   - CLOSEOUT

4. **Protected sections check**: AR Snapshot, AR Aging, AP Snapshot, P&L queries unchanged (validated content preserved).

## Test Coverage

Not applicable - documentation-only changes. Validation performed via:
- Database research file cross-reference for all NodeID names
- Section presence verification
- Content accuracy spot-checks against research findings

## Next Phase Readiness

**Ready for 05-02**: Payment GL Offset Logic documentation. Task 2 additions (DepositJournalID, EntryType) provide foundation for documenting deposit workflow and payment classification in next plan.

**Blockers**: None.

**Concerns**: EntryType mappings are MEDIUM confidence. If users rely heavily on EntryType for advanced queries, recommend Windows environment validation against live database to confirm inferred values match actual Control business logic.

## Key Files Modified

### skills/control-erp-financial/control-erp-financial-SKILL.md
- **Lines changed**: +90 insertions
- **Sections added**: 5 new subsections (Supporting System Accounts, Complete Undeposited Fund Accounts, Complete Ledger Field Reference, EntryType Reference, CLOSEOUT)
- **Sections modified**: Critical System Account NodeIDs (2 name corrections), Stage 2 GL entry example (name correction)
- **Status**: FIN-01 and FIN-02 requirements now complete

## Performance Notes

- Duration: 2min36s (surgical edits to existing file)
- 2 atomic commits: (1) system account corrections, (2) field reference additions
- No database queries needed - all data sourced from phase research file
- No merge conflicts expected - single file, non-overlapping sections

## Success Criteria Met

- [x] All system account NodeIDs use exact database AccountName values
- [x] 11 missing system accounts documented with purpose and classification
- [x] All undeposited fund accounts listed with Active/Legacy status
- [x] Ledger field reference covers EntryType (with value map), Classification, OffBalanceSheet, DepositJournalID, Reconciled
- [x] Closeout section documents CloseoutType values and period locking with query example
- [x] No changes made to validated AR/AP/P&L sections
- [x] Two atomic commits created

---

*Plan executed 2026-02-09. Duration: 2min36s. All requirements met.*
