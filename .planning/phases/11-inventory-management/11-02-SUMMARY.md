---
phase: 11-inventory-management
plan: 02
subsystem: inventory
tags: [purchasing, PO-history, vendor-costs, material-consumption, inventory-valuation, NL-routing]
requires: [control-erp-core, control-erp-financial, 11-01]
provides:
  - Purchasing intelligence (PO history, last price paid, vendor analysis)
  - Material consumption analysis (usage rates, months of supply)
  - Inventory valuation (GL-based authoritative, part-level estimate)
  - Natural language routing for all inventory queries
affects: [13-glossary-integration]
tech-stack:
  added: []
  patterns: [GL-first-valuation, polymorphic-vendor-link, consumption-based-forecasting]
key-files:
  created: []
  modified:
    - skills/control-erp-inventory/control-erp-inventory-SKILL.md (313 -> 662 lines, +349 lines)
key-decisions:
  - decision: GL NodeID 10414 is authoritative for inventory valuation ($651K)
    rationale: Part-level calculation produces -$2.1M (meaningless); GL matches balance sheet
    impact: Users get correct total valuation from GL, part-level is estimate only
  - decision: VendorTransDetail polymorphic link documented with two-path guidance
    rationale: 160 parts use CatalogItem path (12076), 118 use direct Part path (12014)
    impact: Queries try CatalogItem first, fall back to direct Part for complete coverage
  - decision: Months of supply calculation marked as SECONDARY TIER
    rationale: Depends on both inventory quantities and consumption records being accurate
    impact: Users warned to caveat these results given current data quality
  - decision: NL routing table covers 18+ query patterns across 3 requirements
    rationale: Comprehensive routing prevents query disambiguation issues
    impact: Claude can route any inventory question to specific query section
duration: 1min53s
completed: 2026-02-09
---

# Phase 11 Plan 02: Purchasing Intelligence and NL Routing Summary

**One-liner:** Extended inventory skill with purchasing intelligence (6 queries), material consumption (4 queries), GL-first inventory valuation ($651K NodeID 10414), and comprehensive NL routing -- completing INV-03 and all Phase 11 requirements.

---

## Performance

**Duration:** 1 minute 53 seconds
**Commits:** 2 task commits
**Lines added:** 349 (313 -> 662)
**Verification:** All success criteria met

---

## Accomplishments

### What Was Built

Extended the `control-erp-inventory` skill with three major additions:

**1. Purchasing Intelligence (PRIMARY TIER -- Reliable)**

6 query templates covering complete purchasing intelligence:
- Last price paid via CatalogItem path (12076) and direct Part path (12014)
- PO history (last 10 orders for a part)
- Price trend analysis (20 historical orders)
- Top vendors by PO count and total spend
- Most frequently ordered parts (last 12 months) with stock context
- VendorTransDetail polymorphic link reference table
- PO Status reference (Type 7 StatusID values)

All queries use `th.ID DESC` for reliable ordering (not SaleDate).

**2. Material Consumption & Usage (INV-03)**

4 query templates for consumption analysis:
- Average monthly consumption (last 12 months)
- Top consumed parts with stock context
- Usage history (recent events)
- Months of supply remaining (consumption-based forecast)

Uses PartUsageCard (286K non-voided records). Months of supply properly caveated as SECONDARY TIER.

**3. Inventory Valuation (INV-03)**

3 query templates with GL-first approach:
- Total inventory value from GL NodeID 10414 ($651K authoritative)
- GL sub-account breakdown (all inventory-type accounts)
- Part-level valuation (QuantityBilled * AverageCost for AccrueCosts=1 parts only)

Guidance: Default to GL for "what's our inventory worth", use part-level only for operational estimates.

**4. Natural Language Routing**

Complete routing table:
- 18+ trigger patterns mapped to specific query sections
- Cross-skill routing documented (AP -> financial, revenue -> sales/customers)
- Covers all INV-01, INV-02, INV-03 patterns

**5. Data Quality Section**

Documents current migration status and post-migration expectations.

### Requirements Met

- **INV-03 COMPLETE:** Purchasing intelligence (reliable PO data), material consumption analysis, inventory valuation (GL-first)
- **Phase 11 COMPLETE:** All three requirements (INV-01, INV-02, INV-03) fully covered

---

## Task Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | fb03315 | Added purchasing intelligence queries (6 templates, VendorTransDetail polymorphic link reference, PO status reference) |
| 2 | 68ea3ca | Added material consumption (4 queries), inventory valuation (3 queries with GL-first), NL routing (18+ patterns), data quality section |

---

## Files Modified

**skills/control-erp-inventory/control-erp-inventory-SKILL.md** (+349 lines, 313 -> 662)
- Section 6: PURCHASING INTELLIGENCE (152 lines)
  - 6 query templates
  - VendorTransDetail polymorphic link reference
  - PO status reference
- Section 7: MATERIAL CONSUMPTION & USAGE (57 lines)
  - 4 query templates
  - PartUsageCard coverage note
  - Months of supply caveated as secondary tier
- Section 8: INVENTORY VALUATION (43 lines)
  - GL-based total (NodeID 10414)
  - Sub-account breakdown
  - Part-level estimate (with caveats)
  - Guidance on when to use each
- Section 9: NATURAL LANGUAGE INTERPRETATION (39 lines)
  - 18+ trigger patterns
  - Cross-skill routing
- Section 10: DATA QUALITY & MIGRATION STATUS (17 lines)
  - Current state documentation
  - Post-migration expectations

---

## Decisions Made

### 1. GL NodeID 10414 as Authoritative Inventory Valuation

**Decision:** When user asks "what's our inventory worth", default to GL NodeID 10414 balance ($651K), NOT part-level sum.

**Context:** Part-level calculation (QuantityOnHand * AverageCost) produces -$2.1M (meaningless negative value). Research confirmed 214 negative QuantityOnHand records and poor data quality.

**Rationale:** GL balance matches the balance sheet and is the accounting source of truth. Part-level data is unreliable during migration.

**Impact:** Users get correct total valuation. Part-level queries are available but clearly marked as estimates.

### 2. VendorTransDetail Polymorphic Link Two-Path Guidance

**Decision:** Document both CatalogItem (12076) and direct Part (12014) paths, recommend trying CatalogItem first.

**Context:** 160 tracked parts have PO history via CatalogItem; 118 via direct Part link. ItemClassTypeID determines which path is valid.

**Rationale:** CatalogItem path is the primary one for raw materials with vendor catalog linkage. Direct Part path covers garments and outsource items.

**Impact:** Complete PO history coverage. Queries try primary path first, fall back to secondary for comprehensive results.

### 3. Months of Supply Calculation Marked as SECONDARY TIER

**Decision:** Caveat months-of-supply query results as secondary tier with data quality warnings.

**Context:** Calculation requires both accurate inventory quantities AND accurate consumption records. Both are incomplete during migration.

**Rationale:** Honest about data quality limitations. Users shouldn't over-rely on forecasts based on incomplete data.

**Impact:** Users get consumption-based forecasting capability but understand it's directional, not authoritative.

### 4. Comprehensive NL Routing Table (18+ Patterns)

**Decision:** Create complete routing table covering all inventory query types across INV-01, INV-02, INV-03.

**Context:** Plan 02 completes all three requirements. Routing must cover stock levels, reorder monitoring, purchasing, consumption, and valuation.

**Rationale:** Prevents query disambiguation issues. Claude can route any inventory question to the specific query section.

**Impact:** Clean handoff to Phase 13 glossary integration. Cross-skill boundaries clearly defined.

### 5. th.ID DESC for PO Ordering (Not SaleDate)

**Decision:** All PO queries use `th.ID DESC` for ordering by recency, not SaleDate.

**Context:** Research found SaleDate filtering on Type 7 records was unreliable. TransHeader.ID is auto-increment and always present.

**Rationale:** th.ID is a reliable recency proxy. Auto-increment means higher ID = more recent PO.

**Impact:** PO history and price trend queries work correctly without date field issues.

---

## Deviations from Plan

None -- plan executed exactly as written.

---

## Issues Encountered

None.

---

## Next Phase Readiness

**Phase 11 Status:** COMPLETE (2/2 plans)

**Phase 12 Prerequisites:**
- Inventory skill complete ✓
- Part table queries documented ✓
- PartUsageCard patterns established ✓

**Phase 13 Prerequisites:**
- NL routing table ready for glossary integration ✓
- Cross-skill boundaries documented ✓
- All inventory query patterns cataloged ✓

**Blocker Status:** None. Phase 12 can proceed immediately (parallel with Phase 13 if desired).

---

## Validation Notes

All verification criteria met:

### Task 1 Verification
1. ✓ Section 6 (PURCHASING INTELLIGENCE) has "PRIMARY TIER" label
2. ✓ Last price paid queries exist for both paths (CatalogItem 12076 and direct Part 12014)
3. ✓ Guidance explains to try CatalogItem path first, then direct Part path
4. ✓ PO history query shows last 10 orders for a part
5. ✓ Price history/trend query available
6. ✓ Top vendors query present
7. ✓ Most frequently ordered parts query present with LEFT JOIN to Inventory for stock context
8. ✓ VendorTransDetail polymorphic link reference table documents all 3 ItemClassTypeID values
9. ✓ PO Status reference table present
10. ✓ All PO queries use th.ID DESC for ordering (not SaleDate)
11. ✓ All queries use TransactionType = 7

### Task 2 Verification
1. ✓ Section 7 (MATERIAL CONSUMPTION) has 4 queries: monthly average, top consumed, usage history, months of supply
2. ✓ Months of supply query caveated as SECONDARY TIER
3. ✓ PartUsageCard queries use IsVoided = 0 filter
4. ✓ Section 8 (INVENTORY VALUATION) has GL-based total ($651K reference), sub-account breakdown, and part-level estimate
5. ✓ GL valuation uses GLAccountID = 10414 (not part-level sum)
6. ✓ Part-level valuation uses QuantityBilled (not QuantityOnHand) and only AccrueCosts=1 parts
7. ✓ Section 9 (NL INTERPRETATION) has routing for ALL query sections (at least 18 trigger patterns)
8. ✓ Cross-skill routing documented (AP -> financial, revenue -> sales/customers)
9. ✓ Section 10 (DATA QUALITY) documents migration status and data quality warnings
10. ✓ No references to QuantityOnHand * AverageCost for total valuation
11. ✓ Plan 02 placeholder comment removed (replaced with actual content)
12. ✓ File line count tracked: 662 lines (well below 1,200-line extraction threshold)

### Overall Success Criteria
- ✓ INV-03 fully covered: purchasing intelligence, material consumption, inventory valuation
- ✓ Purchasing is PRIMARY TIER (reliable), inventory quantities are SECONDARY TIER (caveated)
- ✓ GL-based valuation is the default for "what's our inventory worth" ($651K NodeID 10414)
- ✓ NL routing covers all inventory-related natural language patterns with clear section mapping
- ✓ Cross-skill boundaries documented (AP -> financial, revenue -> sales)
- ✓ Data quality and migration status documented for user awareness
- ✓ Complete skill file ready for deployment covering INV-01, INV-02, INV-03

---

*Summary generated: 2026-02-09*
*Phase: 11-inventory-management*
*Plan: 11-02*
