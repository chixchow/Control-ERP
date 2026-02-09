---
phase: 05-financial-foundation
verified: 2026-02-09T12:00:00Z
status: passed
score: 3/3 must-haves verified
gaps: []
---

# Phase 5: Financial Foundation Verification Report

**Phase Goal:** GL/Ledger architecture, system account NodeIDs, and payment GL offset logic are documented in the financial skill with enough detail that financial queries in Milestone 2 have a solid foundation
**Verified:** 2026-02-09
**Status:** PASSED
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | `control-erp-financial-SKILL.md` documents GL view vs Ledger table structure, sign conventions, and GLClassificationType reference -- a user asking "How does GL work in Control?" gets a clear answer | VERIFIED | Lines 14-25: GL view defined as `SELECT * FROM Ledger WHERE OffBalanceSheet = 0`, distinction between Ledger (all entries) and GL (financial only) explained. Lines 68-71: Sign conventions for revenue, expense, asset, liability. Lines 113-135: Complete GLClassificationType reference with 16 codes covering balance sheet and income statement categories. |
| 2 | GL system account NodeIDs documented (11=WIP, 12=Built, 14=AR, 21=Orders Due, 24=Customer Deposits, 91=Undeposited Funds) with their purpose in the order lifecycle | VERIFIED | Lines 137-154: Critical System Account NodeIDs table with 12 entries (11=WIP, 12=Built, 14=AR, 21=Orders Due, 24=Order Prepayments, 90=Cash-Associated Bank-Checking, 92=Undeposited MCVisa, etc.). Lines 156-172: Supporting System Accounts with 11 additional accounts (15, 25, 34, 52, 60, 61, 93, 543, 10528, 10530, 10531). Lines 174-192: Complete Undeposited Fund Accounts (13 accounts with Active/Legacy status). NodeID 91 correctly labeled "Undeposited Cash" (exact DB name). |
| 3 | Payment GL offset logic documented -- how WIP/Built deposits vs Sale payments flow through GL accounts, with example journal entries | VERIFIED | Lines 280-348: Full order lifecycle GL entries (Stage 1: WIP/Orders Due, Stage 2: Deposit to Undeposited/Prepayments, Stage 2.5: Built cost flow, Stage 3a: Sale prepaid, Stage 3b: Sale unpaid/credit). Lines 350-363: Bill lifecycle GL entries. Lines 366-498: Payment posting patterns (Path A: Prepaid via NodeID 24, Path B: Credit/AR via NodeID 14), deposit workflow (two-step Undeposited -> Cash-Checking), TenderType reference, and Payment ClassTypeID reference. |

**Score:** 3/3 truths verified

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `skills/control-erp-financial/control-erp-financial-SKILL.md` | Financial skill with GL architecture, system accounts, payment logic | VERIFIED (856 lines, YAML frontmatter, no stubs) | Exists, substantive (856 lines), valid YAML frontmatter with name/description, zero TODO/FIXME/placeholder patterns found |

### Key Link Verification

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| Critical System Account NodeIDs table | GL Transaction Flows section | NodeIDs 11, 12, 14, 21, 24, 90, 91, 92 referenced in journal entries | WIRED | Stage 1 uses NodeID 11/21, Stage 2 uses 91/92/93/24, Stage 2.5 uses 11/12/34/60/61/10414, Stage 3a/3b uses 21/11/24/14 |
| Deposit Workflow section | Payment Posting Patterns section | Two-step process references Paths A/B | WIRED | Step 1 references "documented in Paths A/B above" (line 404), Step 2 shows Cash-Checking (90) clearing Undeposited accounts |
| Built Cost Flow section | GL Transaction Flows section | Stage 2.5 between Stage 2 and Stage 3 | WIRED | Lines 292-327 document Stage 2.5 with NodeIDs 11, 12, 34, 60, 61, 10414; references that "At sale, NodeID 34 clears as costs move to COGS" connecting to Stage 3 |
| Ledger Field Reference | Off-Balance Sheet section | OffBalanceSheet field explained in both | WIRED | Field defined in table (line 42), full explanation in dedicated section (lines 74-96), referenced in Built cost flow (lines 307-315) |
| Complete Undeposited Fund Accounts | TenderType Reference | BankAccountID maps to Undeposited accounts | WIRED | TenderType table (lines 445-454) maps payment methods to Undeposited account NodeIDs that are all listed in the Complete Undeposited Fund Accounts table (lines 176-192) |

### Requirements Coverage

| Requirement | Status | Evidence |
|-------------|--------|----------|
| FIN-01: GL/Ledger architecture, sign conventions, GLClassificationType reference | SATISFIED | GL view definition (lines 14-25), sign conventions (lines 68-71), GLClassificationType reference (lines 113-135), complete Ledger field reference (lines 34-66) |
| FIN-02: GL system account NodeIDs documented (11=WIP, 12=Built, 14=AR, 21=Orders Due, 24=Customer Deposits, 91=Undeposited Funds) | SATISFIED | Critical System Account NodeIDs (lines 137-154), Supporting System Accounts (lines 156-172), Complete Undeposited Fund Accounts (lines 174-192). Names use exact database AccountName values. |
| FIN-03: Payment GL offset logic documented (WIP/Built deposits vs Sale payments) | SATISFIED | GL Transaction Flows stages 1-3b (lines 276-348), Built cost flow stage 2.5 (lines 292-327), Payment posting paths A/B (lines 366-398), Deposit workflow (lines 400-438), TenderType reference (lines 441-466), Payment ClassTypeID reference (lines 468-498) |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| (none) | - | - | - | No TODO, FIXME, placeholder, or stub patterns found in the 856-line file |

### Human Verification Required

### 1. GL Entry Accuracy Against Live Database

**Test:** Query GL entries for a known order (e.g., a recent prepaid order) and compare the actual Ledger entries against the Stage 1 -> Stage 2 -> Stage 3a flow documented in the skill.
**Expected:** Debit/credit accounts and amounts match the documented pattern (WIP/Orders Due at creation, Undeposited/Prepayments at payment, Revenue/COGS at sale).
**Why human:** Requires live database access and a specific order to trace. Structural verification confirmed the patterns are documented but cannot confirm they match real data.

### 2. NodeID Account Names Match Live Database

**Test:** Run `SELECT ID, AccountName FROM GLAccount WHERE ID IN (11, 12, 14, 21, 24, 90, 91, 92, 93)` and compare to the Critical System Account NodeIDs table.
**Expected:** Every AccountName in the skill matches the database exactly.
**Why human:** Requires live database access. The SUMMARY claims names were verified against the database but verification cannot confirm this without running the query.

### 3. Off-Balance Sheet Entry Count

**Test:** Run `SELECT OffBalanceSheet, COUNT(*) FROM Ledger GROUP BY OffBalanceSheet` and compare to the documented ~307K OBS / ~2.44M on-balance-sheet counts.
**Expected:** Counts are in the documented range (allowing for growth since research date).
**Why human:** Requires live database access to confirm the statistical claims.

### Gaps Summary

No gaps found. All three phase success criteria are fully met in the artifact:

1. **GL architecture** is documented with the GL view definition, Ledger vs GL distinction, sign conventions, GLClassificationType reference (16 codes), and a complete Ledger field reference with EntryType mappings.

2. **System account NodeIDs** are documented in three tiers: 12 critical system accounts, 11 supporting system accounts, and 13 complete undeposited fund accounts. Names use exact database AccountName values (e.g., "Undeposited Cash" not "Undeposited Funds" for NodeID 91, "Cash-Associated Bank-Checking" not "Cash-Checking" for NodeID 90).

3. **Payment GL offset logic** is documented through the full order lifecycle (Stages 1 through 3b including Stage 2.5 Built), two payment posting paths (prepaid via NodeID 24, credit/AR via NodeID 14), deposit workflow (two-step Undeposited -> Cash-Checking), TenderType reference (8 values with BankAccountID mappings), Payment ClassTypeID reference (14 values), and a Closeout/period locking section. The Built cost flow (NodeID 34/FGI) and off-balance sheet entries (NodeIDs 60/61) add production cost context.

The skill file is 856 lines with proper YAML frontmatter, zero stub patterns, and comprehensive cross-referencing between sections.

---

_Verified: 2026-02-09_
_Verifier: Claude (gsd-verifier)_
