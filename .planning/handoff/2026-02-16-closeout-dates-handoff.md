# Session Handoff

**Created:** 2026-02-16T06:40:00-06:00
**Project:** /Users/cain/projects/control-db-map
**Branch:** master

---

## Summary
Discovered that FLS Banners' Income Statement report uses the Ledger table with monthly Closeout timestamps as date boundaries — NOT raw SaleDate on TransHeader. Validated a query that matches the January 2026 income statement to the penny ($147,836.16). The sales skill needs to be updated with this closeout-based pattern.

## Completed Work
- Investigated discrepancy between ad-hoc TransHeader query ($354K) and income statement ($147,836)
- Identified the Ledger table as the actual source for income statement data
- Discovered the Closeout table (CloseoutType=2 for monthly) stores period boundary timestamps
- Validated exact match: Ledger entries between Dec closeout (2025-12-31 22:34:34) and Jan closeout (2026-01-31 22:34:37) = $147,836.16
- Updated Claude memory file with project skills location and key rules

## Files Changed
| File | Action | Description |
|------|--------|-------------|
| `/Users/cain/.claude/projects/-Users-cain/memory/MEMORY.md` | created | Added Control ERP skills locations and key business rules |

## Key Decisions
- **Ledger + Closeout = Income Statement source**: The income statement report draws from GL Ledger entries, filtered by monthly Closeout timestamps — not TransHeader.SaleDate or TransHeader.SubTotalPrice
- **Sign convention**: Revenue in Ledger is negative (credit); multiply by -1 for display
- **GL Account filtering**: Income accounts found via `gl.PathName2 LIKE '%Product Sales%' OR gl.PathName2 LIKE '%Income/Sales%'`

## Next Steps
1. **Add closeout date rules to `control-erp-core-SKILL.md`** — Document the Closeout table, CloseoutType values, how to get period boundaries, and the relationship between Closeout and Ledger.EntryDateTime
2. **Add "Monthly Income Statement" section to `control-erp-sales-SKILL.md`** — Add a new query template that uses Closeout boundaries to match the income statement report exactly
3. Consider whether existing sales skill date filtering patterns should reference closeout dates as an alternative to SaleDate for period reporting

## Context to Preserve

### Closeout Table Structure
```
Closeout table:
- CloseoutType: 1=Daily, 2=Monthly, 3=Yearly, 5=Export, 6=CC Settlement
- ModifiedDate: when the closeout ran (the timestamp that defines the period boundary)
- CloseOutPeriod: incrementing period number
- Daily closeouts run ~8 PM Mon-Fri
- Monthly closeouts run ~10:30 PM on last day of month (actual time varies +-2 min)
```

### January 2026 Closeout Boundaries
```
Dec monthly closeout: 2025-12-31 22:34:34 (CloseOutPeriod 284)
Jan monthly closeout: 2026-01-31 22:34:37 (CloseOutPeriod 285/286)
```

### Validated Income Statement Query
```sql
-- Get monthly closeout boundaries
-- Prior month: SELECT ModifiedDate FROM Closeout WHERE CloseoutType = 2 AND CloseOutPeriod = @PriorPeriod
-- Target month: SELECT ModifiedDate FROM Closeout WHERE CloseoutType = 2 AND CloseOutPeriod = @TargetPeriod

-- Income statement matching query
SELECT gl.AccountName,
  SUM(l.Amount) * -1 AS Income
FROM Ledger l
JOIN GLAccount gl ON l.GLAccountID = gl.ID
WHERE l.EntryDateTime > '2025-12-31 22:34:34'   -- prior month closeout
  AND l.EntryDateTime <= '2026-01-31 22:34:37'   -- target month closeout
  AND l.IsActive = 1
  AND (gl.PathName2 LIKE '%Product Sales%'
       OR gl.PathName2 LIKE '%Income/Sales%')
GROUP BY gl.AccountName
ORDER BY Income DESC
-- Total: $147,836.16 (exact match to printed income statement)
```

### GL Account Hierarchy (PathName mapping)
- `Revenue/Product Sales/Product Sales - Digital` — DyeSub, Table Covers, Flags, SEG, etc.
- `Revenue/Product Sales/Garments` — Embroidery, Screenprint
- `Revenue/Product Sales` — Shipping, Services, Promotion
- `Income/Sales` — Artwork-Design
- `Income/Sales/Accessories and Displays` — Banner Stand, Carry Bags

### Reference Files
- `skills/control-erp-sales/control-erp-sales-SKILL.md` - Needs new Monthly Income Statement section with closeout-based query
- `skills/control-erp-core/control-erp-core-SKILL.md` - Needs Closeout table documentation and date boundary rules
- `skills/control-erp-financial/` - May also need closeout references for P&L reporting
- `CLAUDE.md` - Project instructions and phase documentation

## Resume Instructions

```bash
cd /Users/cain/projects/control-db-map
```

Then: `/gsd:resume-work`

**Immediate action:** Add the closeout-based monthly income statement query pattern to the sales and core skills as described in Next Steps.
