# Schema Verification Report
Generated: $(date -u +"%Y-%m-%dT%H:%M:%SZ")

## Environment Constraint

**Database Access:** NOT AVAILABLE from current Mac environment
- MCP server configured for Windows (cmd /c npx)
- Database server: 192.168.75.11:1433
- Current platform: Darwin (Mac)
- Verification approach: Internal consistency checks of existing documentation

## Files Verified

### Total Count
- Schema files: 187 (matches expectation)
- Explicit FK relationships: 89

### Spot-Check Sample (10 tables)

2026-02-09T04:19:31Z

#### 1. Account
- **Structure:** ✓ Complete (Overview, Columns, Foreign Keys, Sample Data)
- **Columns:** 103 columns documented
- **Row Count:** 54,719 (reasonable for customer/vendor table)
- **Foreign Keys:** 4 documented (DivisionData, PricingLevel, Promotion, PaymentTerms)
- **Sample Data:** ✓ Includes all documented columns
- **Issues:** None

#### 2. TransHeader
- **Structure:** ✓ Complete
- **Columns:** 216 columns documented  
- **Row Count:** 232,243 (reasonable for transaction header table)
- **Foreign Keys:** 10 documented (Employee IDs, DivisionData, Station, Account, self-reference)
- **Sample Data:** ✓ Present (limited sample shown)
- **Issues:** None

#### 3. TransDetail
- **Structure:** ✓ Complete
- **Columns:** 202 columns documented
- **Row Count:** 538,252 (reasonable for detail table, ~2.3 per header avg)
- **Foreign Keys:** 3 documented (DivisionData, Station, TransVariation)
- **Sample Data:** ✓ Present
- **Issues:** None

#### 4. Product
- **Structure:** ✓ Complete
- **Columns:** 106 columns documented
- **Row Count:** 277 (reasonable for product catalog)
- **Foreign Keys:** 2 documented (GLAccount, Station)
- **Sample Data:** ✓ Complete with all columns
- **Issues:** None

#### 5. Employee
- **Structure:** ✓ Complete
- **Columns:** 116 columns documented
- **Row Count:** 103 (reasonable for employee roster)
- **Foreign Keys:** 1 documented (EmployeeGroup - marked as implied)
- **Sample Data:** ✓ Present
- **Issues:** None

#### 6. TimeCard
- **Structure:** ✓ Complete + Notes section explaining ClassTypeID 20050/20051
- **Columns:** 41 columns documented
- **Row Count:** 159,448 (reasonable for time tracking)
- **Foreign Keys:** None documented (uses polymorphic references)
- **Sample Data:** ✓ Present
- **Issues:** None
- **Special:** Includes warning about geography columns (LatLonStart, LatLonEnd)

#### 7. Payment
- **Structure:** ✓ Complete
- **Columns:** 56 columns documented
- **Row Count:** 286,084 (reasonable, ~1.2 per transaction header)
- **Foreign Keys:** None documented (uses polymorphic references)
- **Sample Data:** ✓ Present in table format
- **Issues:** None

#### 8. Part
- **Structure:** ✓ Complete
- **Columns:** 86 columns documented
- **Row Count:** 7,684 (reasonable for parts inventory)
- **Foreign Keys:** 4 documented (GLAccount x2, GLAccount, Station)
- **Sample Data:** ✓ Present in table format
- **Issues:** None

#### 9. Shipments
- **Structure:** ✓ Complete
- **Columns:** 54 columns documented
- **Row Count:** 70,584 (reasonable for shipment tracking)
- **Foreign Keys:** 1 documented (DivisionData)
- **Sample Data:** ✓ Complete with XML example in ShipLineItemsXML
- **Issues:** None

#### 10. Station
- **Structure:** ✓ Complete
- **Columns:** 48 columns documented
- **Row Count:** 98 (reasonable for production stations)
- **Foreign Keys:** None documented
- **Sample Data:** ✓ Present
- **Issues:** None

## Internal Consistency Checks

### Column Count Validation
All 10 tables have documented column counts ranging from 41 (TimeCard) to 216 (TransHeader).
Sample data sections reference columns that appear in the Columns table.

### Row Count Reasonableness
| Table | Row Count | Assessment |
|-------|-----------|------------|
| Account | 54,719 | ✓ Reasonable for 20+ years of customer/vendor data |
| TransHeader | 232,243 | ✓ Reasonable transaction volume |
| TransDetail | 538,252 | ✓ ~2.3 items per transaction average |
| Product | 277 | ✓ Reasonable catalog size for print shop |
| Employee | 103 | ✓ Reasonable for mid-size company |
| TimeCard | 159,448 | ✓ High volume time tracking (20051 detail records) |
| Payment | 286,084 | ✓ Slightly more than transactions (deposits, refunds) |
| Part | 7,684 | ✓ Reasonable parts inventory |
| Shipments | 70,584 | ✓ ~30% of transactions ship |
| Station | 98 | ✓ Reasonable station count |

### Foreign Key Consistency
Verified that FK References columns point to valid tables mentioned in schema set:
- DivisionData: Referenced by Account, TransHeader, TransDetail, Shipments ✓
- Employee: Referenced by TransHeader (4 FKs) ✓
- Station: Referenced by TransHeader, TransDetail, Product, Part ✓
- GLAccount: Referenced by Product, Part ✓
- Account: Referenced by TransHeader ✓
- TransHeader/TransDetail: Self-references and cross-references ✓

### Special Column Handling
- **Geography columns:** TimeCard correctly warns about LatLonStart/LatLonEnd ✓
- **Polymorphic references:** TimeCard, Payment correctly note ClassTypeID-based links ✓
- **XML columns:** Shipments includes XML sample in ShipLineItemsXML ✓

## Relationships.md Verification

### Structure Check
✓ Organized by domain (Account, Artwork, Employee, Inventory, Finance, Product, Production, Transaction, Shipping, Warehouse)
✓ Includes "Implicit Relationships" section documenting naming conventions
✓ Includes "ClassTypeID Values" section with known type identifiers
✓ Includes "Key Join Patterns" section with SQL examples

### FK Count
- **Explicit FKs documented:** 89 relationships
- **Domains covered:** 10 (Account, Artwork, Employee, Inventory, Finance, Product, Production, Transaction, Shipping, Warehouse)

### Implicit Relationship Documentation
✓ Universal columns documented (ClassTypeID in 179 tables, StoreID in 156 tables)
✓ High-frequency columns documented (ParentID in 40 tables, StationID in 31 tables, etc.)
✓ ClassTypeID pattern explained (polymorphic type identifier)
✓ Common join patterns provided with SQL examples

## Verification Confidence

### HIGH Confidence (Verified)
- ✅ 187 schema files exist
- ✅ All 10 spot-checked files have complete structure
- ✅ Column counts documented for all 10 tables
- ✅ Row counts are reasonable and consistent with business context
- ✅ FK references point to valid tables
- ✅ Special column types (geography, XML) are documented
- ✅ relationships.md has comprehensive structure
- ✅ 89 explicit FKs documented
- ✅ Implicit relationship patterns documented

### MEDIUM Confidence (Cannot Verify Without DB Access)
- ⚠️ Column data types match INFORMATION_SCHEMA.COLUMNS
- ⚠️ Nullable flags match database definition
- ⚠️ FK count of 89 matches sys.foreign_keys actual count
- ⚠️ Row counts are current (within 10%)
- ⚠️ Sample data represents actual current records

### LOW Confidence (Not Attempted)
- ❌ Live database query verification
- ❌ Actual vs documented FK comparison
- ❌ Current row count verification

## Recommendations

1. **For full verification:** Run verification queries from Windows environment with MCP access
2. **For current phase:** Accept internal consistency checks as sufficient given:
   - Schema files are recent (Feb 7, 2026)
   - All structural requirements met
   - Internal consistency verified
   - FK references are valid
3. **Document limitation:** Note environmental constraint in SUMMARY.md

## Conclusion

**Internal consistency verification: PASSED**
- All 187 schema files exist
- 10 spot-checked tables have complete structure
- FK references are valid
- Row counts are reasonable
- Special columns documented

**Database verification: DEFERRED**
- Requires Windows environment with MCP access
- Cannot connect from current Mac environment
- Recommendation: Accept internal verification or re-run from Windows

