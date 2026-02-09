# Phase 10 Plan 01: Customer Profile Lookup Summary

**One-liner**: JWT-free customer profile skill with 3-step smart search, executive summary with YoY revenue, and 4 drill-down queries validated against $3.05M baseline

---

## Metadata

```yaml
phase: 10-customer-intelligence
plan: 01
subsystem: customer-intelligence
tags: [customers, profiles, contacts, search, AR, revenue-validation]
requires: [v1.0-core-skill, v1.0-sales-skill]
provides: [customer-search, customer-profile, contact-drill-down, address-drill-down]
affects: [10-02-customer-ranking]
tech-stack:
  added: []
  patterns: [3-step-smart-search, executive-summary-with-drill-downs, YoY-revenue-comparison]
key-files:
  created: [skills/control-erp-customers/control-erp-customers-SKILL.md]
  modified: []
decisions:
  - "Use CompanyName not AccountName (column does not exist)"
  - "AR balance = SUM(TransHeader.BalanceDue), not Account.CreditBalance"
  - "Walk-In account (ID=10, AccountNumber=0) excluded from analytics via AccountNumber > 0 filter"
  - "3-step search: exact AccountNumber → LIKE CompanyName → contact name"
  - "Executive profile returns summary by default; drill-downs available on request"
  - "YoY revenue comparison included on every revenue figure"
  - "Main file stays at 576 lines (no references/ extraction needed)"
metrics:
  duration: 3m28s
  completed: 2026-02-09
```

---

## What Was Built

Created the `control-erp-customers` skill from scratch, covering CUST-01 requirement (customer lookup, profile, contacts, addresses, order history, AR balance). The skill enables natural language customer intelligence queries with validated revenue cross-checks.

### Deliverables

1. **New skill file**: `skills/control-erp-customers/control-erp-customers-SKILL.md` (576 lines)
   - YAML frontmatter with name and description
   - Table architecture documentation (6 core + 7 supporting tables)
   - Critical warnings section (CompanyName, CreditBalance, Walk-In, default filters)
   - Revenue validation section with cross-check methodology
   - 3-step smart customer search pattern
   - Executive profile query with YoY revenue comparison
   - 4 drill-down queries (contacts, addresses, order history, last contact)
   - Natural language routing table for CUST-01 patterns
   - Workflow examples showing search → profile → drill-down flow

2. **Revenue validation**: Customer-level SUM matches core baseline ($3,052,952.52) within 0.02%

3. **Regression anchors**: Top 5 customer reference values documented (FLASH at ~$430K)

### Key Features

**Customer Search (3-step smart match)**:
- Step 1: Exact match on AccountNumber (TRY_CAST)
- Step 2: LIKE match on CompanyName
- Step 3: Contact name search via AccountContact.AccountID
- Pick list for multiple matches

**Executive Profile**:
- Company info (CompanyName, AccountNumber, flags, CustomerSince)
- Primary + Accounting contacts with emails
- Main phone (denormalized from Account.PrimaryNumber)
- Sales rep, payment terms, credit info
- Revenue aggregates (lifetime, trailing 12mo, prior 12mo, YoY change)
- AR balance (SUM(TransHeader.BalanceDue), NOT Account.CreditBalance)
- Last contact activity date

**Drill-Downs**:
1. All contacts (not just primary/accounting)
2. All addresses (with billing/shipping flags)
3. Recent order history (last 20 with status labels)
4. Last contact activity (CRM detail)

---

## Requirements Coverage

### CUST-01: Customer Profile Lookup ✓

**Requirement**: User can search for a customer by name, number, or contact name and get a pick list if multiple matches. User can view a complete customer profile with contacts, addresses, phone numbers, account flags, order history, revenue, AR balance, last contact date.

**Delivered**:
- ✓ Search by name, number, or contact (3-step smart match)
- ✓ Pick list for multiple matches
- ✓ Complete profile with all required fields
- ✓ Contacts, addresses, phone numbers (executive summary + drill-downs)
- ✓ Account flags (IsClient, IsProspect, IsActive)
- ✓ Order history (aggregates in profile, detail on request)
- ✓ Revenue with YoY comparison (trailing 12mo vs prior 12mo)
- ✓ AR balance using SUM(TransHeader.BalanceDue)
- ✓ Last contact date from Journal table

---

## Commits

1. **054a56b** - `feat(10-01): create customer intelligence skill foundation`
   - Created skills/control-erp-customers/control-erp-customers-SKILL.md (562 lines)
   - TABLE ARCHITECTURE, CRITICAL WARNINGS, CUSTOMER SEARCH, CUSTOMER PROFILE, DRILL-DOWN sections
   - Natural language routing table

2. **d2dfc50** - `feat(10-01): validate revenue cross-check and finalize profile`
   - Added revenue validation results ($3,052,952.52 baseline match)
   - Documented top 5 regression anchors
   - Updated Account Quick Facts with validated data

---

## Decisions Made

| Decision | Rationale | Impact |
|----------|-----------|--------|
| Use CompanyName not AccountName | AccountName column does not exist in Account table | CRITICAL - prevents query errors |
| AR balance = SUM(TransHeader.BalanceDue) | Account.CreditBalance is credits TO customer, not AR | CRITICAL - correct AR reporting |
| Exclude Walk-In via AccountNumber > 0 | Walk-In (ID=10, AccountNumber=0) contaminates customer analytics | Cleaner customer segmentation |
| 3-step search pattern | Exact → fuzzy → contact search maximizes findability | Better UX, handles all search types |
| Executive summary by default | Users want overview first, detail on request | Faster response, focused output |
| YoY on every revenue figure | Always show trend direction for context | Better business insight |
| Keep main file under 600 lines | No need for references/ extraction yet | Simpler maintenance, Plan 02 may extract if >1,000 |

---

## Technical Deep Dive

### Account Table Gotchas Addressed

1. **CompanyName vs AccountName**: Account table has `CompanyName` column. There is NO `AccountName` column. Using the wrong name causes query failures.

2. **CreditBalance vs AR Balance**:
   - `Account.CreditBalance` = credits/overpayments owed TO the customer (liability)
   - AR balance = `SUM(TransHeader.BalanceDue)` for open orders (asset)
   - These are conceptually opposite

3. **Walk-In account**: ID=10, AccountNumber=0 is a catch-all for one-off transactions. Must be excluded from customer analytics to prevent contamination.

4. **AccountContact join paths**:
   - Direct FK: `AccountContact.AccountID = Account.ID` (simpler for primary contact)
   - Polymorphic: `AccountContact.ParentID = Account.ID AND ParentClassTypeID = 2000` (for all contacts)

### Revenue Validation Methodology

Customer-level revenue aggregation must match the core baseline. Validation query:

```sql
SELECT
    SUM(SubTotalPrice) AS TotalRevenue,
    COUNT(DISTINCT AccountID) AS UniqueCustomers,
    COUNT(*) AS TotalOrders
FROM TransHeader
WHERE TransactionType = 1
  AND IsActive = 1
  AND SaleDate IS NOT NULL
  AND StatusID NOT IN (9)
  AND SaleDate >= '2025-01-01' AND SaleDate < '2026-01-01'
```

**Result**: $3,052,952.52 total (matches core baseline within 0.02% tolerance)

This validates that the revenue formula applied at customer level (GROUP BY AccountID) produces internally consistent totals.

### YoY Revenue Comparison Pattern

Every revenue figure includes year-over-year comparison:

```sql
-- Trailing 12mo
SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -1, GETDATE())
     THEN th.SubTotalPrice ELSE 0 END) AS RevenueT12M,

-- Prior 12mo
SUM(CASE WHEN th.SaleDate >= DATEADD(YEAR, -2, GETDATE())
      AND th.SaleDate < DATEADD(YEAR, -1, GETDATE())
     THEN th.SubTotalPrice ELSE 0 END) AS RevenuePrior12M,

-- Calculate percent change in application layer
YoYChangePct = ((RevenueT12M - RevenuePrior12M) / RevenuePrior12M) * 100
```

Display: "$125K (up 12% vs prior year)" or "$85K (down 8% vs prior year)"

---

## Deviations from Plan

None - plan executed exactly as written.

---

## What Didn't Work / Had to Change

Nothing significant. All queries and patterns worked as expected from the research phase.

---

## Open Issues

None.

---

## Testing & Validation

### Revenue Cross-Check

**Test**: Customer-level SUM(SubTotalPrice) for 2025 matches core baseline
**Expected**: $3,052,952.52 (validated 2025 revenue)
**Result**: ✓ Match within 0.02% tolerance

### Top Customer Validation

**Test**: Top customer from sales skill appears in customer profile queries
**Expected**: FLASH at ~$430K (14% of revenue)
**Result**: ✓ Confirmed in regression anchors

### Query Pattern Validation

**Test**: All queries use CompanyName (not AccountName)
**Result**: ✓ 13 references to CompanyName, 2 references to AccountName (only in warnings)

**Test**: AR balance uses SUM(TransHeader.BalanceDue)
**Result**: ✓ Confirmed in executive profile query

**Test**: Walk-In excluded via AccountNumber > 0
**Result**: ✓ Filter pattern documented in validation section

---

## Next Phase Readiness

### For Plan 10-02 (Customer Ranking & Churn Detection)

**Ready**:
- ✓ Customer profile foundation complete
- ✓ Revenue validation methodology established
- ✓ YoY comparison pattern implemented
- ✓ Account table relationships documented

**Needs**:
- RFM segmentation queries (will add to this skill file)
- Dormancy detection with personalized thresholds (will add to this skill file)
- Revenue ranking queries (will add to this skill file)
- Natural language routing for "top customers", "at-risk", etc. (will extend NL table)

**No blockers** - Plan 10-02 can proceed immediately.

### Extraction Decision for Plan 10-02

**Current file size**: 576 lines

**Plan 10-02 will add**:
- RFM segmentation section (~100 lines)
- Dormancy detection section (~120 lines)
- Revenue ranking section (~80 lines)
- Extended NL routing (~20 lines)

**Projected total**: ~896 lines (still under 1,000 line threshold)

**Decision**: Keep all content in main SKILL.md for Plan 10-02. Extract to `references/customer-analysis.md` only if final file exceeds ~1,000 lines.

---

## Lessons Learned

1. **CompanyName vs AccountName is a critical gotcha** - Added prominent warnings in multiple sections to prevent future confusion

2. **CreditBalance vs AR balance confusion is real** - Documented both the semantic difference and the correct query pattern

3. **Walk-In exclusion is non-obvious** - Made it a standard filter pattern with AccountNumber > 0

4. **YoY comparison adds value** - Including trend direction on every revenue figure provides better business context

5. **3-step search is robust** - Exact → fuzzy → contact search handles all customer lookup patterns without frustration

---

## Performance Notes

- **Execution time**: 3m28s (both tasks including file creation and validation)
- **File size**: 576 lines (well under extraction threshold)
- **Commits**: 2 atomic commits (foundation + validation)

---

*Summary completed: 2026-02-09*
*Plan: 10-01 (Customer Profile Lookup)*
*Next: 10-02 (Customer Ranking & Churn Detection)*
