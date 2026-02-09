# Phase 5: Financial Foundation - Research

**Researched:** 2026-02-08
**Domain:** Control ERP GL/Ledger architecture, financial skill gap analysis
**Confidence:** HIGH

## Summary

The existing financial skill at `skills/control-erp-financial/control-erp-financial-SKILL.md` (576 lines) already covers the majority of FIN-01, FIN-02, and FIN-03 requirements. It documents the GL view vs Ledger table distinction, sign conventions, GLClassificationType reference, system account NodeIDs, payment GL offset logic, and includes detailed journal entry examples for the full order lifecycle.

However, database validation reveals **several specific gaps and inaccuracies** that need correction before the success criteria can be met. These are surgical fixes, not wholesale rewrites. The phase is approximately 85% complete based on existing work -- the remaining 15% is verification, corrections, and targeted additions.

**Primary recommendation:** This phase is a validation-and-fix pass, not a build-from-scratch effort. Create two focused plans: (1) verify and correct the existing skill content against live database evidence, and (2) add the missing documentation for undocumented fields and the deposit workflow.

## Standard Stack

Not applicable -- this phase produces documentation (Markdown skill files), not code. The "stack" is:

### Core
| Tool | Purpose | Why Standard |
|------|---------|--------------|
| MSSQL MCP tools | Query live database to verify GL entries | Source of truth for all claims |
| Markdown | Skill file format | Matches existing skill structure |

### Supporting
| Tool | Purpose | When to Use |
|------|---------|-------------|
| Wiki extracts | Cross-reference GL lifecycle documentation | Validate against vendor documentation |
| Crystal Reports analysis | Verify join patterns used in financial reports | Already completed in Phase 2 |

## Architecture Patterns

### Existing Skill File Structure (preserve this)
```
skills/control-erp-financial/
  control-erp-financial-SKILL.md   # Single file, all financial knowledge
```

### Pattern 1: Gap-Fix Approach
**What:** Rather than rewriting, identify specific sections that need correction/addition and make targeted edits.
**When to use:** When existing documentation is 85%+ complete with known gaps.
**Rationale:** Preserves validated content (AR/AP aging queries, P&L queries, natural language interpretation table) while fixing specific inaccuracies.

### Anti-Patterns to Avoid
- **Full rewrite:** The existing skill has been validated against FLS AR report ($80,899), AP report ($125,319), GL registers, and Trial Balance. Rewriting risks losing this validated context.
- **Adding redundant content:** The core skill already documents GL/Ledger basics. The financial skill should reference core, not duplicate.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| GL lifecycle documentation | New GL lifecycle from scratch | Wiki extract `orders_accounting_knowledge.md` section 5 + live DB verification | Wiki has authoritative 5-stage lifecycle with journal entries |
| NodeID discovery | Manual account browsing | `SELECT ID, AccountName FROM GLAccount WHERE ID IN (...)` | Direct DB query is source of truth |

## Common Pitfalls

### Pitfall 1: Incorrect NodeID Names
**What goes wrong:** The current skill has inaccurate names for some NodeIDs.
**Evidence:**
- Skill says NodeID 91 = "Undeposited Funds" -- actual name is "Undeposited Cash"
- Skill says NodeID 24 = "Order Prepayments" -- wiki calls it "Customer Deposits" (both names appear; DB uses "Order Prepayments", wiki uses "Customer Deposits")
- Skill lists NodeID 90 as "Cash-Checking" -- actual name is "Cash-Associated Bank-Checking"
**How to avoid:** Use exact `GLAccount.AccountName` from database, note wiki aliases.
**Severity:** LOW -- names are close enough for query use but should be corrected.

### Pitfall 2: Missing Undeposited Fund Accounts
**What goes wrong:** Skill only documents 4 undeposited accounts (91, 92, 10137, and references "Cash (91)" inaccurately). There are actually 13 active undeposited fund accounts (GLClassificationType=1007).
**Evidence:** Live query shows:

| NodeID | AccountName | Active? |
|--------|-------------|---------|
| 91 | Undeposited Cash | Active |
| 92 | Undeposited MCVisa | Active |
| 93 | Undeposited Checks | Active |
| 543 | Undeposited Discover | Active |
| 10137 | Undeposited Amex | Active |
| 10147 | Undeposited Discover old | Active |
| 10528 | Undeposited Authorize.net | Active |
| 10530 | Undeposited PayPal | Active |
| 10531 | Undeposited Shopify | Active |
| 10522-10525 | ZT Undeposited * | Legacy (ZoomTex) |

**How to avoid:** Document the full set, marking legacy accounts.

### Pitfall 3: Undocumented Ledger Fields
**What goes wrong:** The skill documents key GL/Ledger fields but omits several important ones that users may encounter or need for advanced queries.
**Evidence:** Ledger has 53 columns. Key undocumented fields:

| Field | Purpose | Values Found |
|-------|---------|-------------|
| `EntryType` | Categorizes the GL entry type | NULL, 1=Manual/Adjustments, 2=Order lifecycle (2.3M entries, dominant), 3=Payment-related (397K), 4=Credit adjustments (30), 5=Bill lifecycle (743) |
| `Classification` | Fine-grained entry classification | 49 distinct values -- maps to internal Control event types. 0=general (1.67M), 100=order sale events (901K), 10001/10006/10007=payment subtypes |
| `OffBalanceSheet` | Whether entry is on/off balance sheet | 0=on-balance (2.44M), 1=off-balance (307K) |
| `DepositJournalID` | Links to the deposit journal when payment is deposited | Groups payments into deposit batches |
| `Reconciled` / `ReconciliationDateTime` | Bank reconciliation tracking | Boolean + timestamp |
| `DivisionID` | Which division the entry belongs to | Links to Division table |
| `StationID` | Production station (for cost entries) | Links to Station table |
| `PartID` | Material/part for inventory entries | Links to Part table |
| `PayrollID` | Payroll entry link | Links to Payroll table |
| `WarehouseID` | Warehouse for inventory entries | Links to Warehouse table |

**How to avoid:** Add a "Complete Ledger Field Reference" subsection.

### Pitfall 4: Missing Payment ClassTypeID Reference
**What goes wrong:** Skill mentions only 20001 (Order Payment) and 20009 (Bill Payment). There are actually 15 distinct ClassTypeIDs in the Payment table.
**Evidence:**

| ClassTypeID | Count | Meaning (from Description samples) |
|-------------|-------|-------------------------------------|
| 20000 | 102,752 | Master Payment (parent grouping for multi-order payments) |
| 20001 | 119,108 | Order Payment (payment applied to specific order) |
| 20002 | 840 | Credit Payment (payment from customer credit balance) |
| 20003 | 3 | Change Returned |
| 20004 | 197 | Order Refund |
| 20005 | 500 | Credit Refund |
| 20006 | 28 | Credit Memo Payment |
| 20007 | 6,237 | Failed Payment |
| 20009 | 33,251 | Bill Payment |
| 20011 | 4 | Overpayment |
| 20012 | 463 | Payment Transferred to Credit |
| 20032 | 3,440 | Master Payment Authorization (CC auth grouping) |
| 20037 | 18,989 | Master Bill Payment (parent for multi-bill payments) |
| 20038 | 271 | Receipt (receiving document payment) |

**How to avoid:** Add complete Payment ClassTypeID reference table.

### Pitfall 5: Missing TenderType Reference
**What goes wrong:** The skill describes payment types by description text ("Check", "Cash", "Capture") but never documents the `Payment.TenderType` field.
**Evidence:**

| TenderType | Count | Meaning (from BankAccountID correlation) |
|------------|-------|------------------------------------------|
| 0 | 7,451 | Cash (BankAccountID=91 Undeposited Cash) |
| 1 | 70,942 | Check (BankAccountID=93 Undeposited Checks) |
| 2 | 167,463 | Credit Card (BankAccountID=92 Undeposited MCVisa) |
| 4 | 17,009 | Online/Authorize.net (BankAccountID=10528) |
| 5 | 5,911 | ACH/Wire (BankAccountID=90 Cash-Checking direct) |
| 7 | 711 | Wire Transfer (BankAccountID=90 for bill payments) |
| 8 | 4 | Other/Legacy |
| NULL | 16,593 | System entries (master payments, failed, etc.) |

**How to avoid:** Add TenderType reference with BankAccountID mapping.

### Pitfall 6: Missing System Account NodeIDs
**What goes wrong:** Skill documents the most important system accounts but misses several that appear in actual GL entries.
**Evidence:** Missing from current documentation:

| NodeID | AccountName | GLClassificationType | Purpose |
|--------|-------------|---------------------|---------|
| 15 | Direct Costs from Bills | 1002 (Current Asset) | Intermediary for bill costs assigned to orders |
| 25 | Vendor Credit | (Liability) | Vendor credit balances |
| 34 | Cost Of Built - FGI | 1002 (Current Asset) | Holds costs during Built status |
| 52 | Credit Given | 5002 (Expense) | Customer credit adjustments |
| 60 | Unclassified Inventory | 1003 (Inventory) | Off-balance sheet cost tracking for expensed parts |
| 61 | Unclassified Expense | 5002 (Expense) | Off-balance sheet expense tracking |
| 93 | Undeposited Checks | 1007 (Undeposited) | Check deposits pending |
| 10528 | Undeposited Authorize.net | 1007 (Undeposited) | Online CC deposits pending |
| 10530 | Undeposited PayPal | 1007 (Undeposited) | PayPal deposits pending |
| 10531 | Undeposited Shopify | 1007 (Undeposited) | Shopify deposits pending |

**How to avoid:** Add a "Supporting System Accounts" section below the critical ones.

### Pitfall 7: Deposit Workflow Undocumented
**What goes wrong:** The skill documents payment posting to undeposited accounts but never explains the subsequent deposit workflow (Undeposited -> Cash).
**Evidence:** Live GL entries show a clear two-step process:
1. Payment received: Debit Undeposited MCVisa (92), Credit Order Prepayments (24) or AR (14)
2. Deposit made: Debit Cash-Checking (90), Credit Undeposited MCVisa (92)

The deposit creates a Journal entry, and `Ledger.DepositJournalID` links payment GL entries to their deposit batch. `Payment.Undeposited` flag tracks whether a payment has been deposited.

**How to avoid:** Add "Deposit Workflow" section documenting the Undeposited -> Bank Account flow.

### Pitfall 8: Closeout Table Undocumented
**What goes wrong:** The Closeout table is referenced in wiki knowledge but not in the financial skill.
**Evidence:** Closeout table has 10,510 records with 5 active CloseoutTypes:

| CloseoutType | Text | Count | Purpose |
|-------------|------|-------|---------|
| 1 | Daily | 5,395 | Daily business close |
| 2 | Monthly | 263 | Monthly accounting close |
| 3 | Yearly | 21 | Yearly fiscal close |
| 5 | Export | 1,890 | GL export for external accounting |
| 6 | Credit Card Settlement | 2,940 | CC batch settlement |

ClassTypeID = 8911 for all entries. GL period locking uses last closeout date.

**How to avoid:** Add Closeout reference section.

## Code Examples

### Verified: Full GL Entry Set for a Prepaid Order Sale (Order #133688)

**Step 1 - Deposit received (WIP status):**
```
Debit  Undeposited MCVisa (92)    $22.00
Credit Order Prepayments (24)     $22.00
```

**Step 2 - Order marked Sale:**
```
Debit  Orders Due (21)            $22.00   -- reverse presale balancing
Credit WIP (11)                   $22.00   -- clear WIP tracking
Debit  Order Prepayments (24)     $22.00   -- clear deposit liability
Credit Screenprint Income (10100) $22.00   -- revenue recognition
```

**COGS entries (off-balance sheet parts):**
```
Credit Unclassified Inventory (60) $1.10   -- off-balance sheet
Debit  Unclassified Expense (61)   $1.10   -- off-balance sheet
```

This confirms the financial skill's Stage 3a is directionally correct but:
- Revenue credit goes to specific product income account (not generic "[Product Revenue Account]")
- COGS entries may be off-balance sheet (OffBalanceSheet=1) for parts tracked as "Cost Accounting" rather than "Accrual"
- Multiple detail lines may create multiple revenue account credits

### Verified: GL Entry Set for an Unpaid/AR Order Sale (Order #133XXX)

```
Debit  Orders Due (21)            $24.48
Credit WIP (11)                   $24.48
Debit  Accounts Receivable (14)   $24.48   -- balance goes to AR
Credit Screenprint Income (10100) $24.48   -- revenue recognition
```

No Order Prepayments (24) involved. Confirms Path B in financial skill.

### Verified: Built Status GL Entries (Order #181413, multi-line)

```
Credit WIP (11)                   $4,883.00  -- remove from WIP (multiple lines)
Debit  Built (12)                 $4,883.00  -- transfer to Built (multiple lines)
Credit Inventory (10414)          $68.25     -- decrease inventory for accrual-tracked parts
Credit Unclassified Inventory (60) $40.54    -- decrease for non-accrual parts (off-balance)
Debit  Cost Of Built - FGI (34)   $108.79    -- hold costs until sale
```

### Verified: Deposit Workflow

```
Step 1: Payment received
  Debit  Undeposited Checks (93)  $282.96
  Credit Accounts Receivable (14) $282.96

Step 2: Deposit processed (JournalID 5289492 groups the batch)
  Debit  Cash-Checking (90)       $282.96
  Credit Undeposited Checks (93)  $282.96
```

### Verified: Journal ActivityType Reference (for financial queries)

| ActivityType | Text | Count | Financial Relevance |
|-------------|------|-------|---------------------|
| 0 | (none) | 2.7M+ | Payment entries, GL postings |
| 1 | Created | 229K | Order/record creation |
| 2 | Edited | 408K | Price/order edits (trigger GL corrections) |
| 7 | Order Status Marked Sale | 118K | **Revenue recognition event** |
| 8 | Order Status Marked Closed | 26K | Order fully paid |
| 9 | Order/Bill Voided | 3.9K | GL reversals |
| 38 | Purchase Order Converted | 19K | PO -> Bill conversion |

## State of the Art

| Current Skill State | Gap | Impact |
|---------------------|-----|--------|
| GL view documented correctly | None | View definition verified: `SELECT * FROM Ledger WHERE OffBalanceSheet = 0` |
| Sign conventions correct | None | Revenue negative (credit), Expenses positive (debit) confirmed |
| GLClassificationType complete | None | All 15 codes documented |
| System NodeIDs mostly correct | Missing 10 accounts, 2 name inaccuracies | Financial queries may miss supporting accounts |
| Payment posting patterns correct | Missing TenderType, full ClassTypeID list, deposit workflow | Users can't query by payment method programmatically |
| GL Transaction Flows mostly correct | Missing Built->Sale cost flow detail, off-balance sheet nuance | Advanced cost analysis queries won't account for OBS entries |
| AR/AP queries validated | None | Match FLS reports |
| P&L queries validated | None | Match Trial Balance export |
| Closeout not documented | Missing entirely | Users can't query period boundaries |

## Detailed Gap Analysis vs. Requirements

### FIN-01: GL/Ledger Architecture, Sign Conventions, GLClassificationType
**Status: 90% complete**

Already documented:
- GL view vs Ledger table distinction (verified correct)
- Key GL/Ledger fields (GLAccountID, Amount, EntryDateTime, TransHeaderID, ID)
- Sign conventions (verified correct)
- GLClassificationType reference (all codes present)
- GLAccount chart of accounts structure

Gaps:
1. **Ledger field completeness** -- 53 columns, only ~5 documented. Key missing: EntryType, Classification, OffBalanceSheet detail, DepositJournalID, Reconciled, DivisionID, StationID, PartID, PayrollID, WarehouseID
2. **EntryType reference** -- 6 values (NULL, 1-5), completely undocumented
3. **Ledger ClassTypeID** -- 8900 for nearly all entries (2.75M), not documented
4. **Off-balance sheet explanation** -- mentioned in passing but not explained (307K entries for parts with non-accrual cost tracking, uses NodeID 60/61)

### FIN-02: GL System Account NodeIDs
**Status: 75% complete**

Already documented (correct):
- 11=WIP, 12=Built, 14=AR, 21=Orders Due, 22=AP, 23=Customer Credit Balance, 24=Order Prepayments
- 90=Cash-Checking, 92=Undeposited MCVisa, 10137=Undeposited Amex, 10412=Cash-MM, 10414=Inventory
- All Revenue NodeIDs (GLClassificationType 4000)
- All COGS NodeIDs (GLClassificationType 5001)

Gaps:
1. **Missing system accounts:** 15 (Direct Costs from Bills), 25 (Vendor Credit), 34 (Cost Of Built - FGI), 52 (Credit Given), 60 (Unclassified Inventory), 61 (Unclassified Expense), 93 (Undeposited Checks), 543 (Undeposited Discover), 10528 (Undeposited Authorize.net), 10530 (Undeposited PayPal), 10531 (Undeposited Shopify)
2. **Name inaccuracies:** NodeID 91 documented as "Undeposited Funds" but actual name is "Undeposited Cash". NodeID 90 documented as "Cash-Checking" but actual name is "Cash-Associated Bank-Checking"
3. **Bank accounts (GLClassificationType 1000)** not documented: 90=Cash-Checking, 10133=Cash-Savings, 10142=Trade and Barter, 10412=Cash-MM, 10526=PayPal
4. **Purpose/lifecycle role** of NodeIDs 15, 34, 60, 61 in the Built status and cost tracking flows not explained

### FIN-03: Payment GL Offset Logic
**Status: 80% complete**

Already documented (correct):
- Path A: Prepaid -> Order Prepayments (24) -- verified with order #133688
- Path B: Credit/AR -> Accounts Receivable (14) -- verified with live data
- Payment description format -- verified correct
- ~70%/30% split Path A vs Path B -- stated but not empirically verified in this research

Gaps:
1. **Deposit workflow** -- the second step (Undeposited -> Cash) is completely missing. Payments first hit Undeposited accounts, then a separate deposit action moves funds to Cash-Checking (90). This is a critical accounting flow.
2. **Payment.TenderType** -- numeric field mapping payment methods, completely undocumented
3. **Payment.SourceTypeID** -- tracks payment processing source (Manual, Elavon, FreedomPay, etc.), undocumented
4. **Payment ClassTypeID** -- only 2 of 15 values documented. Missing: Master Payment (20000), Credit Payment (20002), Refund (20004/20005), Failed Payment (20007), Payment to Credit (20012), Master Auth (20032), Master Bill Payment (20037), Receipt (20038)
5. **Built status cost flow** -- the intermediate "Cost Of Built - FGI" (NodeID 34) account and how costs transfer at Built and then clear at Sale is not documented
6. **Off-balance sheet cost entries** -- parts tracked as "Cost Accounting" (non-financial) create OffBalanceSheet=1 entries in Ledger between NodeID 60 (Unclassified Inventory) and 61 (Unclassified Expense), filtered out by the GL view. Not documented.
7. **Closeout table** -- used for GL period locking, not documented in financial skill
8. **Journal ActivityType reference** -- 55 distinct values, financial-relevant ones not documented

## Open Questions

1. **Payment.TenderType values 3 and 6** -- not observed in FLS data. May exist in other Control installations. Recommend noting as "not observed at FLS" rather than claiming they don't exist.
   - What we know: Values 0,1,2,4,5,7,8 are mapped. 3 and 6 are absent.
   - Recommendation: Document observed values only, note gaps.

2. **Ledger.Classification values** -- 49 distinct values observed. Full mapping unknown. Some map to GLAccount NodeIDs (e.g., 10001, 10006, 10007 are payment subtypes). Others are order lifecycle codes (100=sale, 400=bill).
   - What we know: Classification appears to be a denormalized copy of some internal event type.
   - Recommendation: Document the most common values with counts but flag this as LOW confidence. This field is rarely needed for user queries.

3. **Ledger.EntryType values** -- 5 observed values plus NULL. Correlation:
   - 1 = Manual journal entries (bank fees, loan payments)
   - 2 = Order lifecycle entries (largest set by far)
   - 3 = Payment-related entries
   - 4 = Credit adjustments
   - 5 = Bill lifecycle entries
   - Recommendation: Document with MEDIUM confidence, note this is inferred from Description patterns.

4. **How does the financial skill interact with the sales skill (Phase 4)?** -- Phase 4 sales skill uses GL view as primary source for Sales Report (per decision 03-02). The financial skill documents GL queries. Need to ensure no contradictions between the two skills.
   - Recommendation: Cross-reference during implementation.

## Sources

### Primary (HIGH confidence)
- Live database queries against StoreData (all NodeIDs, field values, GL entries verified 2026-02-08)
- GL view definition verified from `sys.sql_modules`
- `skills/control-erp-financial/control-erp-financial-SKILL.md` (existing skill, 576 lines)
- `skills/control-erp-core/control-erp-core-SKILL.md` (core business rules)

### Secondary (MEDIUM confidence)
- `output/wiki/extracts/orders_accounting_knowledge.md` -- Wiki-sourced GL lifecycle, confirmed against live DB
- `output/wiki/extracts/sql_queries_reference.md` -- GL integrity check queries from wiki
- `output/wiki/extracts/database_integration_knowledge.md` -- ClassTypeID mappings

### Tertiary (LOW confidence)
- Ledger.EntryType and Ledger.Classification value meanings -- inferred from Description correlation, not from official documentation

## Metadata

**Confidence breakdown:**
- Existing content accuracy: HIGH -- verified against live database, all major claims confirmed
- Gap identification: HIGH -- systematic comparison of skill content vs database schema and data
- Gap severity assessment: HIGH -- based on actual user query scenarios and requirement specs
- EntryType/Classification mappings: MEDIUM -- inferred from data patterns, not official docs

**Research date:** 2026-02-08
**Valid until:** 2026-03-08 (stable -- GL structure rarely changes)

## Planning Implications

### Effort Estimate
This is a **validation-and-fix phase**, not a build phase. Estimated effort:

| Plan | Task Count | Effort |
|------|-----------|--------|
| 05-01: Verify and correct financial skill | 4-5 tasks | Small -- targeted edits to existing file |
| 05-02: Document payment GL offset logic | 3-4 tasks | Small -- add sections for deposit workflow, TenderType, ClassTypeIDs |

### Task Breakdown Guidance for Planner

**Plan 05-01 should include:**
1. Fix NodeID name inaccuracies (91, 90)
2. Add missing system account NodeIDs (15, 25, 34, 52, 60, 61, 93, 543, 10528, 10530, 10531)
3. Add complete Undeposited Funds account listing
4. Add Ledger field reference (EntryType, Classification, OffBalanceSheet detail, DepositJournalID)
5. Add Closeout table reference

**Plan 05-02 should include:**
1. Add deposit workflow documentation (Undeposited -> Bank Account flow)
2. Add Payment.TenderType reference table
3. Add Payment.SourceTypeID reference table
4. Add complete Payment ClassTypeID reference (all 15 values)
5. Add Built status cost flow detail (NodeIDs 34, 60, 61 roles)
6. Add off-balance sheet cost tracking explanation

### What NOT to Change
- AR/AP queries (validated)
- P&L queries (validated)
- Revenue NodeID tables (complete)
- COGS NodeID tables (complete)
- Natural Language Interpretation table (correct)
- Important Caveats section (correct, well-calibrated)
- GL Transaction Flows Stages 1-3b (directionally correct, only need nuance additions)
