---
phase: 05-financial-foundation
plan: 02
subsystem: financial
tags: [payment, deposits, cost-flow, built-status, off-balance-sheet, documentation]
requires: [05-01-system-accounts-ledger-fields]
provides: [deposit-workflow, payment-tendertype-reference, payment-classtypeid-reference, built-cost-flow, off-balance-sheet-explanation]
affects: [06-glossary-skill, future-payment-queries]
tech-stack:
  added: []
  patterns: [two-step-deposit-process, accrual-vs-cost-accounting]
key-files:
  created: []
  modified: [skills/control-erp-financial/control-erp-financial-SKILL.md]
decisions:
  - id: deposit-two-step-pattern
    what: Documented deposit workflow as two-step process (Payment -> Undeposited, then Deposit -> Cash-Checking)
    why: Users need to understand why payments don't immediately appear in cash accounts
    impact: Clarifies Payment.Undeposited flag and DepositJournalID usage
  - id: tendertype-reference
    what: Created complete TenderType mapping (0-8, NULL) to payment methods and BankAccountIDs
    why: Essential reference for payment method analysis and queries
    impact: Users can now query by payment method (cash, check, CC, ACH, wire)
  - id: payment-classtypeid-complete
    what: Documented all 14 Payment ClassTypeID values (20000-20038)
    why: Clarifies payment subtypes (order payment vs bill payment vs refund vs failed, etc.)
    impact: Users can filter to ClassTypeID IN (20001, 20009) for standard payment queries
  - id: built-stage-25
    what: Added Built status as "Stage 2.5" between WIP and Sale
    why: Orders marked Built have distinct cost flow through NodeID 34 (FGI)
    impact: Completes the GL transaction flow lifecycle documentation
  - id: off-balance-sheet-explained
    what: Documented off-balance sheet entries with accrual vs cost accounting distinction
    why: ~307K off-balance sheet entries exist but are excluded from GL view - users need to know when to include them
    impact: Clarifies why GL view vs Ledger table matters for production cost analysis
metrics:
  duration: 181s
  completed: 2026-02-09
---

# Phase 5 Plan 2: Payment & Cost Flow Documentation Summary

Added payment deposit workflow, TenderType/ClassTypeID references, Built status cost flow, and off-balance sheet explanation to the financial skill.

## One-Liner
Two-step deposit process (Undeposited -> Cash-Checking via DepositJournalID), complete Payment TenderType/ClassTypeID references, Built status FGI cost flow (NodeID 34), and off-balance sheet accrual vs cost accounting explanation (~307K OBS entries).

## What Was Built

### Task 1: Deposit Workflow and Payment Reference Tables
Added three new subsections to PAYMENT POSTING PATTERNS section:

1. **Deposit Workflow (Undeposited -> Bank)**
   - Documented two-step deposit process: Payment -> Undeposited account, then Deposit -> Cash-Checking (90)
   - Explained `Payment.Undeposited` flag (1 = not deposited, 0 = deposited)
   - Added queries for undeposited payments and deposit batch entries
   - Documented `Ledger.DepositJournalID` grouping for batch deposits

2. **Payment.TenderType Reference**
   - Complete mapping of TenderType values (0-8, NULL) to payment methods
   - Default BankAccountID for each payment method
   - Key finding: ACH (5) and Wire (7) post directly to Cash-Checking, bypassing undeposited step
   - Note: TenderType 3 and 6 not observed at FLS

3. **Payment ClassTypeID Reference**
   - All 14 Payment ClassTypeID values documented (20000-20038)
   - Clarified subtypes: order payment (20001), bill payment (20009), refunds (20004/20005), failed (20007), master groupings (20000/20037), etc.
   - Query guidance: filter to `ClassTypeID IN (20001, 20009)` for standard payment analysis

### Task 2: Built Cost Flow and Off-Balance Sheet Documentation
Added cost flow and accounting mode documentation to two sections:

1. **Off-Balance Sheet Entries** (in GL/LEDGER ARCHITECTURE section)
   - Explained why OffBalanceSheet=1 entries exist and are excluded from GL view
   - Documented two part cost tracking modes:
     - **Accrual (financial)**: Parts as inventory assets (NodeID 10414), appear in GL view
     - **Cost Accounting (non-financial)**: Parts expensed at purchase (NodeIDs 60/61), off-balance sheet to prevent double-counting
   - Count context: ~307K off-balance sheet vs ~2.44M on-balance sheet (~11% of Ledger entries)
   - Query example showing how to include OBS entries for total production cost analysis

2. **Built Status — Cost Flow (Stage 2.5)** (in GL TRANSACTION FLOWS section)
   - Added intermediate stage between Stage 2 (Payment) and Stage 3 (Sale)
   - GL entries when order marked Built:
     - Transfer WIP (11) -> Built (12)
     - Accrual costs: Inventory (10414) -> Cost Of Built - FGI (34)
     - Non-accrual costs: Unclassified Inventory (60) -> Unclassified Expense (61) [off-balance sheet]
   - Clarified NodeID 34 (Cost Of Built - FGI = Finished Goods Inventory) holds costs until sale
   - At sale, NodeID 34 clears as costs move to COGS
   - Query to analyze Built cost entries using Journal.ActivityType = 6

3. **Natural Language Interpretation Table Updates**
   - Added 5 new user query patterns:
     - "undeposited payments" / "pending deposits"
     - "deposit history" / "bank deposits"
     - "payment method breakdown"
     - "closeout status" / "period locks"
     - "production costs for order" / "total cost including materials"

## Verification Results

All verification checks passed:
- ✅ TenderType appears in Payment reference section
- ✅ DepositJournalID documented in both Ledger field reference and deposit workflow section
- ✅ Cost Of Built / FGI appears in Built status cost flow section
- ✅ OffBalanceSheet explained in both GL architecture and Built cost flow sections
- ✅ File structure intact and logical: GL Architecture (with OBS) -> GLAccount -> GL Transaction Flows (with Built Stage 2.5) -> Payment Posting (with deposit/TenderType/ClassTypeID) -> AR -> AP -> P&L -> Closeout -> NL Interpretation
- ✅ Existing validated content unchanged (AR $80,899, AP $125,319 context notes still present)

## Deviations from Plan

None - plan executed exactly as written.

## Technical Details

**Files Modified:**
- `skills/control-erp-financial/control-erp-financial-SKILL.md` (691 -> 856 lines, +165 lines)

**Content Added:**
- 3 new subsections in PAYMENT POSTING PATTERNS section
- 1 new subsection in GL/LEDGER ARCHITECTURE section
- 1 new subsection in GL TRANSACTION FLOWS section (Stage 2.5)
- 5 new rows in Natural Language Interpretation table
- 5 new SQL queries (undeposited payments, deposit batches, payment method analysis, payment type analysis, Built cost entries)

**Key Patterns Documented:**
- Two-step deposit workflow: Payment -> Undeposited, Deposit -> Bank
- Payment method routing via TenderType -> BankAccountID
- Payment subtype filtering via ClassTypeID
- Built status intermediary cost hold via NodeID 34 (FGI)
- Off-balance sheet tracking for non-accrual parts via NodeIDs 60/61

## Impact

**Immediate Value:**
- Users can now query undeposited payments and understand why cash balances don't match payment totals
- Payment method analysis is straightforward with TenderType reference
- Payment queries can filter correctly using ClassTypeID guidance
- Built status GL entries are no longer mysterious - clear FGI cost flow documented
- Off-balance sheet entries explained - users know when to include them (production cost analysis) vs exclude them (standard financial reporting)

**Skills Enhanced:**
- control-erp-financial skill now has complete payment lifecycle documentation (posting -> deposit)
- Complete cost flow lifecycle (WIP -> Built/FGI -> Sale/COGS)
- Accrual vs cost accounting modes explained

**Future Benefits:**
- Supports Phase 6 glossary skill (payment terms, deposit terms, cost accounting terms)
- Foundation for payment analytics and cash flow analysis
- Enables accurate production cost queries including non-accrual materials

## Next Phase Readiness

**Ready to proceed:** Phase 5 complete (2/2 plans done). Financial skill now has comprehensive coverage:
- System accounts (05-01)
- Ledger field reference (05-01)
- Closeout/period locking (05-01)
- Deposit workflow (05-02)
- Payment references (05-02)
- Built cost flow (05-02)
- Off-balance sheet explanation (05-02)

**No blockers for:**
- Phase 6 (Glossary Skill) - financial terms ready
- Phase 7 (Test Suite) - financial queries ready for validation
- Phase 8 (Documentation & Release) - financial skill ready for release

## Commits

| Hash | Message |
|------|---------|
| b624e32 | feat(05-02): add deposit workflow and Payment reference tables |
| b567dda | feat(05-02): add Built cost flow and off-balance sheet documentation |

## Lessons Learned

**What Worked Well:**
- Task structure worked perfectly - two logical groupings (payment references, cost flow)
- Verification queries in plan spec ensured accuracy without live DB access
- Progressive disclosure: general GL architecture explanation -> specific cost flow details
- Natural language table updates make new content discoverable

**Context for Future:**
- Deposit workflow is two-step by design - payments don't instantly hit bank accounts
- ~11% of Ledger entries are off-balance sheet - significant volume but intentionally excluded from GL view
- Built status (StatusID 2) is optional - not all orders go through it (some go WIP -> Sale directly)
- NodeID 34 (Cost Of Built - FGI) is the key intermediary account for Built status cost flow
