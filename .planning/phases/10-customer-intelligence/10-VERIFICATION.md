---
phase: 10-customer-intelligence
verified: 2026-02-09T13:15:00Z
status: gaps_found
score: 6/7 must-haves verified
gaps:
  - truth: "User can ask 'show customer segmentation' and get RFM-scored segments with correct named categories"
    status: partial
    reason: "RFM query has inverted Recency scoring: NTILE(5) OVER (ORDER BY Recency ASC) assigns quintile 1 to most-recent customers, but segment criteria treat high scores (>=4) as Champions. This means Champions are actually the LEAST recent customers and Lost are the MOST recent -- completely inverted."
    artifacts:
      - path: "skills/control-erp-customers/control-erp-customers-SKILL.md"
        issue: "Line 650: ORDER BY Recency ASC gives score 1 to most-recent. Lines 666-673 treat R_Score>=4 as Champions (should be most-recent, but gets least-recent). Fix: change to ORDER BY Recency DESC."
    missing:
      - "Fix NTILE(5) OVER (ORDER BY Recency ASC) to ORDER BY Recency DESC on line 650, so that lowest recency (most recent) gets highest score (5)"
      - "Same fix needed in Segment Summary CTE which says 'Same CTE as above' (line 698-699)"
  - truth: "Cross-reference to references/customer-analysis.md is a dead link"
    status: failed
    reason: "Line 1067 references references/customer-analysis.md which does not exist. Decision CUST-NO-EXTRACT explicitly chose NOT to extract. Cross-reference section not updated."
    artifacts:
      - path: "skills/control-erp-customers/control-erp-customers-SKILL.md"
        issue: "Lines 1065-1067 and 1075: reference non-existent file and contain [TBD] marker"
    missing:
      - "Remove or update the 'To Reference Files' section to reflect that no extraction was done"
      - "Remove [TBD] from line 1075"
      - "Remove [TBD via discovery query] from line 445 (Journal ClassTypeID for ContactActivity)"
---

# Phase 10: Customer Intelligence Verification Report

**Phase Goal:** Users can look up any customer, understand their value and behavior, and identify at-risk accounts -- through a new control-erp-customers skill
**Verified:** 2026-02-09T13:15:00Z
**Status:** gaps_found
**Re-verification:** No -- initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | User can search for a customer by name, number, or contact and get a complete profile | VERIFIED | 3-step smart search (lines 170-216), executive profile with all required fields (lines 228-328), 4 drill-down queries (lines 332-452) |
| 2 | User can ask "who are our top customers" and get revenue ranking with YoY | VERIFIED | Customer Ranking section (lines 455-598), TOP 10 with YoY comparison, alternative rankings (order count, AOV, lifetime, by year), Pareto analysis |
| 3 | User can ask "show customer segmentation" and get RFM-scored segments | PARTIAL | RFM section exists (lines 611-734) with 7 named segments, NTILE(5) scoring, segment summary query. However, Recency scoring is INVERTED -- ORDER BY Recency ASC gives low scores to recent customers, but criteria treat high scores as Champions |
| 4 | User can ask "which customers are at risk" and get dormant customer detection | VERIFIED | Personalized dormancy (lines 737-919), LAG() for order gaps, AVG+1.5*STDEV threshold, 4-order minimum with 90-day fallback, combined risk score, simple dormancy fallback, spend trend detection |
| 5 | Customer revenue totals are internally consistent with core baseline ($3,052,952.52) | VERIFIED | Revenue validation section (lines 109-164) with explicit cross-check query and result, top 5 regression anchors |
| 6 | All queries use correct field names and patterns (CompanyName, SubTotalPrice, SaleDate, Walk-In exclusion) | VERIFIED | CompanyName: 33 occurrences (AccountName only in warnings). SubTotalPrice: 19 uses. SaleDate: 54 uses. AccountNumber > 0: 9 uses. TransactionType = 1: 17 uses. IsClient = 1: 9 uses |
| 7 | Skill file is a complete, well-structured Claude skill with YAML frontmatter and NL routing | VERIFIED | YAML frontmatter with name and description (lines 1-4). 43 NL routing entries (lines 926-970). Cross-domain routing (lines 972-975). Workflow examples (lines 979-1054) |

**Score:** 6/7 truths verified (1 partial due to RFM scoring inversion)

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `skills/control-erp-customers/control-erp-customers-SKILL.md` | Complete customer intelligence skill | EXISTS, SUBSTANTIVE (1,075 lines) | 10 major sections, real SQL queries, no placeholders except 2 TBD markers |
| `skills/control-erp-customers/references/customer-analysis.md` | Referenced in cross-references | MISSING (by design) | Decision CUST-NO-EXTRACT chose not to extract, but cross-reference not cleaned up |

### Key Link Verification

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| Customer Search | Account table | CompanyName LIKE, AccountNumber TRY_CAST | WIRED | 3-step pattern with correct field names |
| Executive Profile | TransHeader | AccountID FK, validated revenue formula | WIRED | SubTotalPrice, SaleDate, TransactionType=1, StatusID NOT IN (9) |
| Executive Profile | AR Balance | SUM(TransHeader.BalanceDue) | WIRED | Correctly uses BalanceDue, NOT CreditBalance (8 BalanceDue refs) |
| Drill-down Contacts | AccountContact | AccountID FK | WIRED | Correct join with IsActive filter |
| Drill-down Addresses | AddressLink + Address | ParentClassTypeID=2000 | WIRED | Polymorphic join pattern correct |
| RFM Scoring | TransHeader | NTILE(5) window functions | PARTIAL | Query structure correct but Recency scoring direction inverted |
| Dormancy Detection | TransHeader | LAG() window function | WIRED | 4-level CTE with statistical thresholds, combined risk score |
| Revenue Validation | Core baseline | $3,052,952.52 cross-check | WIRED | Explicit validation query and result documented |

### Requirements Coverage

| Requirement | Status | Blocking Issue |
|-------------|--------|----------------|
| CUST-01: Customer lookup, contact info, account status | SATISFIED | None |
| CUST-02: Customer segmentation and CLV analysis | PARTIALLY SATISFIED | RFM Recency scoring inverted -- Champions and Lost categories will be swapped |
| CUST-03: Churn detection (customers inactive > N days) | SATISFIED | None |

### Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| control-erp-customers-SKILL.md | 445 | `[TBD via discovery query]` for Journal ClassTypeID | Warning | Does not block functionality -- query works without ClassTypeID filter |
| control-erp-customers-SKILL.md | 1067 | Dead cross-reference to `references/customer-analysis.md` | Warning | Misleading -- file does not exist, decision was made not to extract |
| control-erp-customers-SKILL.md | 1075 | `~[TBD] (references)` | Warning | Leftover placeholder in footer |
| control-erp-customers-SKILL.md | 650 | `ORDER BY Recency ASC` in NTILE(5) | Blocker | Inverts Recency scoring -- Champions become Lost and vice versa |

### Human Verification Required

### 1. RFM Segment Assignment Accuracy
**Test:** Run the RFM query against live database and check if "Champions" segment contains customers who ordered recently and frequently
**Expected:** Champions should be recent, frequent, high-value customers. With current bug, Champions will be dormant high-value customers.
**Why human:** Need live query execution to confirm the scoring inversion produces wrong results

### 2. Personalized Dormancy Threshold Reasonableness
**Test:** Run the personalized dormancy query and check that flagged customers are actually overdue relative to their personal ordering cadence
**Expected:** Customers flagged should have DaysSinceLastOrder > PersonalThreshold, with reasonable threshold values (not 0, not 9999)
**Why human:** Statistical thresholds (AVG + 1.5 STDEV) may produce unreasonable values for edge cases

### 3. Customer Profile Completeness
**Test:** Look up a known customer (e.g., Flash Apparel) and verify all profile fields populate correctly
**Expected:** CompanyName, AccountNumber, contacts, phone, revenue (lifetime + YoY), AR balance, last contact date all populated
**Why human:** Cannot verify actual data population without live database execution

### Gaps Summary

**1 blocker found, 3 warnings:**

The primary gap is a **Recency scoring inversion in the RFM segmentation query**. The NTILE(5) window function uses `ORDER BY Recency ASC` which assigns quintile 1 to the most-recent customers (lowest day count). But the segment classification criteria treat high scores (>=4) as "Champions" and low scores (<=2) as "Lost/At Risk". This means:

- "Champions" (R>=4) will actually be the LEAST recent customers (most dormant)
- "Lost" (R<=2) will actually be the MOST recent customers (just ordered)
- "At Risk" and "Cant Lose Them" segments are similarly inverted

The fix is straightforward: change `ORDER BY Recency ASC` to `ORDER BY Recency DESC` so that the most recent customers get the highest quintile score (5).

Additionally, there are 3 warning-level issues:
- 2 TBD markers remaining in the file (lines 445, 1075)
- 1 dead cross-reference to a file that was intentionally not created (line 1067)

These warnings do not block goal achievement but should be cleaned up.

---

_Verified: 2026-02-09T13:15:00Z_
_Verifier: Claude (gsd-verifier)_
