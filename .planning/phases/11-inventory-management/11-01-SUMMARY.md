---
phase: 11-inventory-management
plan: 01
subsystem: inventory
tags: [inventory, stock-levels, reorder-monitoring, parts, warehouse, purchasing]
requires: [control-erp-core, phase-09-financial]
provides:
  - Stock level queries with Available/OnHand distinction
  - Reorder monitoring with suggested quantities
  - Part search and category browsing
  - Two-tier confidence model (Primary: PO data, Secondary: inventory data)
affects: [11-02-purchasing-intelligence]
tech-stack:
  added: []
  patterns: [two-tier-confidence-model, summary-first-drill-down, progressive-filtering]
key-files:
  created:
    - skills/control-erp-inventory/control-erp-inventory-SKILL.md
  modified: []
key-decisions:
  - decision: Two-tier confidence model separating reliable PO data from caveated inventory quantities
    rationale: FLS inventory data is incomplete during migration from Google Sheets; PO data is reliable
    impact: Users get accurate cost data even when quantity data is questionable
  - decision: Default to Warehouse 10 for all queries
    rationale: Warehouse 10 has 273 parts with stock vs 3 in Apparel WH; 98% of inventory is in default warehouse
    impact: Simplified queries, users rarely need warehouse breakdown
  - decision: Progressive filtering (IsActive -> TrackInventory -> context-dependent qty filter)
    rationale: 4,487 non-tracked parts create noise; stock checks exclude zeros, reorder checks include zeros
    impact: Clean results focused on relevant inventory
duration: 2min23s
completed: 2026-02-09
---

# Phase 11 Plan 01: Inventory Skill Foundation Summary

**One-liner:** Created control-erp-inventory skill with two-tier confidence model, stock level queries (INV-01), and reorder monitoring (INV-02) — ready for Plan 02 purchasing intelligence extension.

---

## Performance

**Duration:** 2 minutes 23 seconds
**Commits:** 1 task commit
**Files created:** 1 (313 lines)
**Verification:** All success criteria met

---

## Accomplishments

### What Was Built

Created the `control-erp-inventory` skill with foundational architecture for inventory management at FLS Banners:

**Two-Tier Confidence Model:**
- **Primary Tier (Reliable):** Purchasing intelligence from Type 7 PO data
- **Secondary Tier (Caveated):** Inventory quantities with data quality warnings

**INV-01: Stock Level Queries (4 variants)**
1. Part search by name (LIKE matching)
2. Broad overview (top 25 by QuantityOnHand)
3. Parts by category summary
4. Part detail card (full drill-down)

**INV-02: Reorder Monitoring**
1. Parts below reorder point (sorted by urgency)
2. Reorder summary by category
3. Coverage note (only 74 of 392 parts have reorder points configured)

**Critical Infrastructure:**
- 7 warnings in prominent callout box (data quality, 4-row-per-part, use Inventory not Part)
- Complete table architecture documenting 10 tables with row counts
- FLS warehouse configuration (Warehouse 10 = default, has 98% of stock)
- Canonical Part-to-Inventory join pattern with all required filters
- Inventory formula and 3 tracking modes documented
- Part category reference with top 10 categories by count

### Requirements Met

- **INV-01 COMPLETE:** Users can search for parts and get current stock levels with Available/OnHand distinction, unit context, and data quality warnings
- **INV-02 COMPLETE:** Users can see parts below reorder threshold with suggested quantities and estimated cost

---

## Task Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 + 2 | 080437c | Created inventory skill with table architecture, two-tier model, INV-01 stock queries, INV-02 reorder monitoring (comprehensive single commit) |

**Note:** Tasks 1 and 2 were completed in a single efficient implementation since both were part of creating the skill file.

---

## Files Created

**skills/control-erp-inventory/control-erp-inventory-SKILL.md** (313 lines)
- YAML frontmatter with comprehensive NL trigger terms
- Overview with two-tier confidence model explanation
- 7 critical warnings in blockquote callout
- Table architecture (10 tables documented)
- FLS warehouse configuration
- Inventory formula and tracking modes
- Part category reference
- Canonical join pattern
- INV-01: 4 stock level query variants
- INV-02: Reorder monitoring queries
- Plan 02 placeholder for purchasing intelligence

---

## Decisions Made

### 1. Two-Tier Confidence Model

**Decision:** Separate skill into Primary Tier (reliable PO data) and Secondary Tier (caveated inventory quantities)

**Context:** FLS is migrating inventory data from Google Sheets to Control. Research revealed 214 negative QuantityOnHand records and -$2.1M part-level valuation (vs $651K GL balance).

**Rationale:** PO history (Type 7 transactions) is well-maintained and reliable for cost lookups. Inventory quantities are incomplete and should always be caveated.

**Impact:** Users get accurate "what does X cost" answers even when "what's in stock" answers are questionable. Clear separation prevents users from over-relying on incomplete data.

### 2. Default to Warehouse 10

**Decision:** All queries default to WarehouseID=10 (Default Warehouse for Company)

**Context:** FLS has 2 real warehouses: Default (273 parts with stock) and Apparel WH (3 parts with stock).

**Rationale:** 98% of inventory is in the default warehouse. Apparel WH is rarely relevant.

**Impact:** Simplified queries, cleaner results. Warehouse breakdown only surfaced if user explicitly asks.

### 3. Progressive Filtering Pattern

**Decision:** Apply filters in stages: IsActive=1 -> TrackInventory=1 -> context-dependent qty filter

**Context:** 7,684 total parts, but only 392 have TrackInventory=1. Stock checks should exclude zero-qty; reorder checks should include zero-qty.

**Rationale:** Non-tracked parts (4,487) create noise. Zero-qty handling depends on query intent (stock vs reorder).

**Impact:** Clean, focused results. Stock queries show what's available; reorder queries show what's needed even if currently zero.

### 4. Always Show Unit Context

**Decision:** Include DisplayUnitText in all queries, note when inventory units differ from display units

**Context:** 50 parts have UseInvUnitsForDisplay=0 (display units differ from inventory tracking units). AverageCost is per inventory unit.

**Rationale:** Costs without unit context are meaningless. A roll at $1,035 is reasonable; displayed as "meter" it looks wrong.

**Impact:** Users always understand the unit context for quantities and costs.

### 5. Inventory Table with Full Filter Chain

**Decision:** All Inventory joins must include ClassTypeID=12200, IsGroup=0, IsDivisionSummary=0

**Context:** Each part has 4 Inventory rows (2 warehouses x 2 summary types). Without filters, quantities are 2x or 4x actual.

**Rationale:** This is the Part-to-Inventory equivalent of the $1.3M TransDetailParam bug from v1.0. Critical pattern that must be enforced.

**Impact:** Accurate quantities. Documented in warnings section to prevent future errors.

---

## Deviations from Plan

None -- plan executed exactly as written.

---

## Issues Encountered

None.

---

## Next Phase Readiness

**Plan 11-02 Prerequisites:**
- Skill file structure ready for extension ✓
- Two-tier model established for purchasing intelligence ✓
- Part-to-Inventory join patterns documented ✓
- Warehouse configuration discovered ✓

**Blocker Status:** None. Plan 02 can proceed immediately.

**Handoff Notes for Plan 02:**
- Add purchasing intelligence queries (PO history, last price paid, vendor costs, order frequency)
- Add material consumption analysis (PartUsageCard queries)
- Add inventory valuation section (GL NodeID 10414, not part-level calculation)
- Add NL routing table for query disambiguation
- Consider extraction to references/ if skill exceeds 1,200 lines (currently 313 lines)

---

## Validation Notes

All verification criteria met:
1. ✓ YAML frontmatter with name and description including NL triggers
2. ✓ Two-tier model documented in overview
3. ✓ All 7 critical warnings present in blockquote section
4. ✓ Table architecture lists 10 tables with row counts and key columns
5. ✓ FLS warehouse configuration identifies Warehouse 10 as default
6. ✓ Canonical Part-to-Inventory join includes ClassTypeID=12200, IsGroup=0, IsDivisionSummary=0
7. ✓ INV-01: 4 stock level queries present
8. ✓ INV-02: Reorder query sorts by urgency, includes zero-qty parts
9. ✓ Every Inventory query includes required filters
10. ✓ DisplayUnitText included in all result sets
11. ✓ Part.InventoryID explicitly warned against
12. ✓ Plan 02 placeholder comment present

---

*Summary generated: 2026-02-09*
*Phase: 11-inventory-management*
*Plan: 11-01*
