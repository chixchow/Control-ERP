# Known Gotchas and Pitfalls (DOC-05)

These gotchas were discovered and validated during Milestone 1 development. Each represents a real error that produced incorrect results. The wrong/correct patterns and impact magnitudes are validated against actual database queries.

---

1. **TransDetailParam IsActive Filter**
   - WRONG: `WHERE tdp.IsActive = 1`
   - CORRECT: No IsActive filter on TransDetailParam
   - Impact: $1.33M revenue undercount (DyeSub shows $464K instead of $1,793K)
   - Skill reference: control-erp-core, "Standard Query Filters" section (lines 119-125); control-erp-sales, Template 3 warning (line 147)

2. **SubTotalPrice vs TotalPrice**
   - WRONG: `SUM(TotalPrice)` for revenue
   - CORRECT: `SUM(SubTotalPrice)` for revenue
   - Impact: $11.6K overcount per year (includes sales tax in revenue figure)
   - Skill reference: control-erp-core, "CRITICAL: Revenue Query Formula" section (lines 14-29)

3. **SaleDate vs OrderCreatedDate**
   - WRONG: `WHERE OrderCreatedDate BETWEEN @Start AND @End`
   - CORRECT: `WHERE SaleDate BETWEEN @Start AND @End`
   - Impact: Wrong time period assignment (orders attributed to entry date, not sale date)
   - Skill reference: control-erp-core, "Date Filtering Patterns" section (lines 129-136)

4. **EstimateCreatedDate for Type 2**
   - WRONG: `WHERE SaleDate BETWEEN @Start AND @End` on Type 2 records
   - CORRECT: `WHERE EstimateCreatedDate BETWEEN @Start AND @End`
   - Impact: Zero results returned (SaleDate and OrderCreatedDate are always NULL on Type 2)
   - Skill reference: control-erp-core, "Date Filtering Patterns" section (lines 134-136, 160-162)

5. **SaleDate IS NOT NULL filter**
   - WRONG: `WHERE TransactionType = 1` without SaleDate filter
   - CORRECT: `WHERE TransactionType = 1 AND SaleDate IS NOT NULL`
   - Impact: Includes WIP and Built orders that are not yet sales in revenue totals
   - Skill reference: control-erp-core, "Revenue-Specific Filters" section (lines 113-117)

6. **CompanyName not AccountName**
   - WRONG: `SELECT AccountName FROM Account`
   - CORRECT: `SELECT CompanyName FROM Account`
   - Impact: Query failure (AccountName column does not exist on Account table)
   - Skill reference: MEMORY.md, "Key Schema Gotchas" section

7. **DueDate means different things by type**
   - WRONG: Treating DueDate uniformly across transaction types for aging
   - CORRECT: Type 1 DueDate = production/shipping deadline; Type 8 DueDate = vendor payment deadline
   - Impact: Incorrect AR aging buckets (aging from production date instead of invoice date)
   - Skill reference: control-erp-financial, "IMPORTANT CAVEATS" section, Caveat 2 (line 842)

8. **GL view vs Ledger table**
   - WRONG: Using GL view for comprehensive ledger analysis
   - CORRECT: Use Ledger table directly when off-balance-sheet entries are needed
   - Impact: Missing ~307K entries (11% of Ledger) -- cost accounting entries for parts expensed at purchase
   - Skill reference: control-erp-financial, "IMPORTANT CAVEATS" section, Caveat 1 (line 840); control-erp-core, "GL / Ledger Architecture" section (lines 308-318)

9. **Web order deduplication**
   - WRONG: Counting WebOrderID (Import_Order_Number) without deduplication
   - CORRECT: Use `ROW_NUMBER() OVER (PARTITION BY uf.Import_Order_Number ORDER BY th.OrderNumber ASC)` and keep only rn=1
   - Impact: 8.4% overcount of web orders (cloned orders share Import_Order_Number)
   - Skill reference: control-erp-core, "Web Import Identification" section (lines 59-75)

10. **Header vs Detail revenue gap**
    - WRONG: Expecting `TransHeader.SubTotalPrice = SUM(TransDetail.SubTotalPrice)` for the same orders
    - CORRECT: Accept ~$7K gap due to header-level adjustments; use header-level for revenue totals
    - Impact: ~$7K discrepancy causes false alarm if not expected
    - Skill reference: control-erp-sales, "IMPORTANT CAVEATS" section, Caveat 1 (line 356)

11. **Type 2 in revenue totals**
    - WRONG: Including TransactionType 2 (estimates) in revenue calculations
    - CORRECT: Filter to `TransactionType = 1` only for revenue
    - Impact: Double-counting (both estimate SubTotalPrice and converted order SubTotalPrice counted)
    - Skill reference: control-erp-core, "TransactionType Reference" section (lines 40, 50-53)

12. **TC_ variables not in TransDetailParam**
    - WRONG: Looking for TC_FabricCategory or TC_ProductName in TransDetailParam
    - CORRECT: Use `td.Description LIKE '%Table Cover%'` pattern matching for table cover identification
    - Impact: Zero results for table cover product queries using variable lookup
    - Skill reference: control-erp-sales, "Container 2: DyeLux-Full Print Table Cover" section (lines 62-63)

13. **pre-2017 SQL Server (no STRING_AGG)**
    - WRONG: Using `STRING_AGG(column, ',')` in queries
    - CORRECT: Use `STUFF(...FOR XML PATH(''))` for string concatenation
    - Impact: Query syntax error -- STRING_AGG requires SQL Server 2017+
    - Skill reference: CLAUDE.md, "Rules" section; output/skill/SKILL.md (line 21)

---

All gotchas are actively prevented by the skill files referenced above. Loading the appropriate skill before querying ensures these errors are avoided.
