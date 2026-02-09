# ClassTypeID Reference

Complete reference for all ClassTypeID values in the Control ERP database.

ClassTypeID is a polymorphic type identifier used throughout Control to map object types to their database tables. It enables the `ParentID + ParentClassTypeID` polymorphic foreign key pattern that allows one table to reference many different entity types.

**Source:** Cyrious Control wiki documentation (350+ entries verified)

---

## Quick Reference: Most Used ClassTypeIDs

These are the ClassTypeIDs you'll encounter most often when querying the Control database.

| ClassTypeID | What It Is | Database Table | Common Usage |
|-------------|------------|----------------|--------------|
| 2000 | Company/Account | Account | Customer or vendor account records |
| 3000 | Contact | AccountContact | Contact person at a company |
| 3500 | Employee | Employee | Employee records |
| 4001 | Address | Address | Physical/mailing address |
| 4002 | Address Link | AddressLink | Links addresses to entities |
| 4100 | Phone Number | PhoneNumber | Phone/fax numbers for any entity |
| 10000 | Order/Estimate | TransHeader | Transaction header (order, estimate, PO, etc.) |
| 10100 | Line Item | TransDetail | Transaction detail (line item on order) |
| 10200 | Modifier | TransMod | Line item modifier/option |
| 10300 | Part on Line Item | TransPart | Part/material used on a line item |
| 10400 | Variation | TransVariation | Order variation (multiple versions) |
| 10500 | Tax | TransTax | Tax calculation on line item |
| 11000 | Purchase Order | TransHeader | Vendor purchase order |
| 11100 | PO Detail | VendorTransDetail | PO line item |
| 12000 | Product | CustomerGoodsItem (Product) | Product/goods item |
| 12014 | Part | Part | Part/material/supply |
| 12200 | Inventory | Inventory | Inventory record |
| 14030 | Transaction Detail Parameter | TransDetailParam | Line item parameter/attribute |
| 20000 | Master Payment | Journal, Payment | Payment record header |
| 20050 | Master Time Card | Journal, TimeCard | Clock in/out parent record |
| 20051 | Detail Time Card | Journal, TimeCard | Station time detail record |
| 20060 | Email Activity | Journal, EmailActivity | Email send/receive log |
| 20065 | Note Activity | Journal | Note/comment activity |
| 20500 | Transaction Activity | Journal | Transaction-related activity |
| 20510 | Company Activity | Journal | Company-related activity |
| 23000 | Macro | RuleMacro | Automation macro definition |
| 26100 | Station | Station | Production station/workcenter |

---

## FLS Banners Specific Notes

### GoodsItemClassTypeID on TransDetail
FLS Banners uses **12000** for Product on TransDetail.GoodsItemClassTypeID. The wiki also documents ClassTypeID 49 for "Product" — this appears to be an older or alternative mapping. For FLS queries, use **12000**.

Similarly, FLS uses **12014** for Part (ClassTypeID 30 may be an alternative).

### TimeCard ClassTypeIDs
- **20050** = Clock in/out parent record (the overall shift)
- **20051** = Station time detail (time spent at a specific production station within the shift)

```sql
-- Get shift records (clock in/out)
SELECT EmployeeID, StartTime, EndTime
FROM dbo.TimeCard
WHERE ClassTypeID = 20050 AND StartTime >= @Date

-- Get station time detail
SELECT EmployeeID, StationID, StartTime, EndTime
FROM dbo.TimeCard
WHERE ClassTypeID = 20051 AND StartTime >= @Date
```

---

## Complete ClassTypeID Table

All 350+ ClassTypeID values organized by functional category.

### System & Configuration (ClassTypeID < 2000)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| -1 | No Class Type | | |
| 100 | Action | Yes | |
| 200 | Navigation Item | Yes | |
| 300 | Unit | Yes | |
| 350 | Title | | Element |
| 360 | System Data | | |
| 400 | Printer Device Type | | |
| 500 | System Image | Yes | |
| 610 | Explorer Query | | |
| 700 | Store Constant | | StoreConstant |
| 701 | Store Constant Category | | Element |
| 800 | Virtual Query | Yes | SystemData |
| 801 | SQL Function | | |
| 850 | Generic Template | | |
| 900 | Web Browser | | |
| 1000 | Store | | Store |
| 1100 | Option | | UserOption |
| 1101 | System Option | | UserOption |
| 1301 | Commission Rate | | CommissionRate |
| 1500 | Pricing Graphic | | PricingGraphic |
| 1501 | System Graphic | | |
| 1502 | Graphic | | Graphic |
| 1503 | Payment Account Graphic | | |
| 1600 | Selection List Category | | PricingElement |
| 1601 | Selection List | | SelectionList |
| 1602 | Selection List Item | | SelectionListItem |
| 1750 | Activity Pick List | | Element |
| 1751 | Activity Pick List Item | | Element |
| 1760 | Calendar Link | | CalendarLink |
| 1770 | Calendar View | | |
| 1780 | Calendar Status | | CalendarStatus |
| 1800 | Activity Time Span | | ActivityTimeSpan |
| 1810 | Signature Usage Type | | Element |
| 1811 | Signature Activity | | |

---

### Entities (ClassTypeID 2000-9999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 2000 | Company | | Account |
| 2001 | Company UDF Value | | |
| 2100 | Vendor | | |
| 2500 | Service Contract Type | | ServiceContractType |
| 2600 | Service Ticket Type | | Element |
| 2700 | Service Ticket Priority | | ServiceTicketPriority |
| 3000 | Company Contact | | AccountContact |
| 3400 | Employee Group | | EmployeeGroup |
| 3500 | Employee | | Employee |
| 3501 | Employee Contact | | EmployeeContact |
| 4000 | Address Type | | Element |
| 4001 | Address | | Address |
| 4002 | Address Link | | AddressLink |
| 4100 | Phone Number | | PhoneNumber |
| 4101 | Phone Number Type | | Element |
| 5000 | Tax Class | | TaxClass |
| 5002 | Tax Link | | TaxLink |
| 5003 | Postal Code Tax Class | | PostalCodeTaxClass |
| 5004 | Postal Code Tax Class List | | Element |
| 5005 | Product Taxability Code | | ProductTaxabilityCode |
| 5006 | State Tax Exempt Link | | |
| 5500 | Marking List | | Element |
| 5501 | Marking List Item | | MarketingListItem |
| 5511 | Prospect (Company) Stage | | Element |
| 5512 | Client (Company) Stage | | Element |
| 5600 | Disposition List | | Element |
| 5601 | Disposition | | Element |
| 6000 | User | | UserName |
| 6002 | Security Right | | |
| 6003 | Security Link | | SecurityLink |
| 6004 | Security Right Action | | |
| 6005 | Lock Time Out | | |
| 6006 | WebPro Security Right Action | | |
| 6010 | Security Right Template | | Element |
| 6020 | Security Right Change | | SecurityRightChange |
| 6030 | Partial Security Right Template | | |
| 6040 | Partial Security Right | | |
| 6050 | WebPro Security Right Template | | Element |
| 6060 | WebPro Security Right | | |
| 6070 | WebPro Security Right Change | | |
| 6130 | Shipping Methods | | Element |
| 7000 | Speed Note | | SpeedNote |
| 7100 | PO Speed Note | | SpeedNote |
| 7500 | Standard Pricing Formula | | SpeedNote |
| 8000 | GL Account Group | | GLAccount |
| 8001 | GL Account Item | | GLAccount |
| 8002 | Payment Method | | PaymentAccount |
| 8050 | Computed Account Group | | GLAccount |
| 8051 | Computed Cost Account | | GLAccount |
| 8055 | Computed Cost Account | | GLAccount |
| 8100 | Bill Payment Method | | PaymentAccount |
| 8900 | GL Entry | | GL |
| 8901 | Inventory Log | | (Deprecated) |
| 8911 | Daily Closeout | | Journal, Closeout |
| 8912 | Monthly Closeout | | Journal, Closeout |
| 8913 | Yearly Closeout | | Journal, Closeout |
| 8914 | Export Closeout | | Journal, Closeout |
| 8915 | Royalty Closeout | | Journal, Closeout |
| 8916 | CC Settlement | | Journal, Closeout |
| 8917 | Weekly Closeout | | Journal, Closeout |
| 8918 | Quarterly Closeout | | Journal, Closeout |
| 8919 | Session Closeout | | Journal, Closeout |
| 8951 | Element | | |
| 9000 | Contact Manager | | |
| 9201 | Recurring Activity | | RecurringActivity |
| 9202 | Recurring Macro Activity | | RecurringActivity |
| 9750 | Account Reconciliation | | Journal |

---

### Transactions (ClassTypeID 10000-10999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 10000 | Order/Estimate/Template/Service Tickets | | TransHeader |
| 10001 | Order/Estimate/Template/Service Tickets History | | TransHeaderHistory |
| 10100 | Transaction Detail | | TransDetail |
| 10101 | Transaction Detail History | | TransDetailHistory |
| 10200 | Transaction Modifier | | TransMod |
| 10201 | Transaction Modifier History | | TransModHistory |
| 10300 | Transaction Part | | TransPart |
| 10301 | Transaction Part History | | TransPartHistory |
| 10400 | Transaction Variation | | TransVariation |
| 10401 | Transaction Variation History | | TransVariationHistory |
| 10500 | Transaction Tax | | TransTax |
| 10501 | Transaction Tax History | | TransTaxHistory |
| 10600 | Transaction Detail Graphic | | TransDetailGraphic |
| 10601 | Transaction Detail Graphic History | | TransDetailGraphicHistory |

---

### Vendor Transactions (ClassTypeID 11000-11999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 11000 | Purchase Order/Bill/Receiving Document | | TransHeader |
| 11001 | Purchase Order/Bill/Receiving Document History | | TransHeaderHistory |
| 11100 | PO Detail | | VendorTransDetail |
| 11101 | PO Detail History | | VendorTransDetailHistory |
| 11200 | PO Part | | |
| 11201 | PO Part History | | |
| 11300 | PO Variation | | TransVariation |
| 11301 | PO Variation History | | TransVariationHistory |

---

### Products & Pricing (ClassTypeID 12000-12999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 12000 | Product | | CustomerGoodsItem |
| 12001 | Product UDF | | ProductUserField |
| 12010 | Modifier | | CustomerGoodsItem |
| 12014 | Part | | Part |
| 12016 | Part Conversion Item | | PartInventoryConversion |
| 12020 | Product Category | | PricingElement |
| 12030 | Modifier Category | | PricingElement |
| 12035 | Part Category | | PricingElement |
| 12040 | Product Pricing Plan | | PricingPlan |
| 12041 | Modifier Pricing Plan | | PricingPlan |
| 12050 | Pricing Plan Type | | PricingElement |
| 12051 | Pricing Level | | PricingLevel |
| 12052 | Promotion | | Promotion |
| 12053 | Sales Goal Month | | SalesGoal |
| 12054 | Sales Goal Year | | SalesGoal |
| 12060 | Product Tax Exempt Link | | |
| 12070 | Product/Modifier Link | | ProdModLink |
| 12071 | Product/Part Link | | GoodsItemPartLink |
| 12072 | Modifier/Part Link | | GoodsItemPartLink |
| 12074 | Part/Employee Link | | |
| 12075 | Part Instance | | PricingElement |
| 12076 | Catalog Item | | CatalogItem |
| 12077 | Assembly Link | | AssemblyLink |
| 12080 | Pricing Method | | |
| 12090 | Pricing Method Parameter | | |
| 12095 | Pricing Method Table | | |
| 12100 | Pricing Family | | PricingElement |

---

### Inventory & Shipping (ClassTypeID 12100-13999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 12200 | Inventory | | Inventory |
| 12500 | Quick Product | | QuickProduct |
| 12501 | Quick Product List | | PricingLink |
| 13000 | Commission Plan | | Element, CommissionPlan |

---

### Parameters & Variables (ClassTypeID 14000-14999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 14020 | Goods Item Parameter Link | | PricingLink |
| 14030 | Transaction Detail Parameter | | TransDetailParam |
| 14031 | Transaction Detail Parameter History | | TransDetailParamHistory |
| 14040 | Pricing Table | | PricingTable |
| 14041 | Pricing Table Element | | (Deprecated) |
| 14045 | Pricing Table Category | | PricingElement |
| 14050 | Discount Table | | DiscountTable |
| 14051 | Discount Table Item | | DiscountTableItem |
| 14500 | Variable Category | | PricingElement |
| 14505 | Variable | | Variable |
| 14510 | Goods Item Variable Link | | PricingLink |
| 14515 | Pricing Plan Variable | | Variable |

---

### Templates & Reports (ClassTypeID 15000-17999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 15000 | Input Template/Pricing Setup | | IOTemplate |
| 15001 | Input Template Category | | PricingElement |
| 15010 | Output Template | | IOTemplate |
| 15011 | Output Template Category | | PricingElement |
| 15020 | Canvas | | |
| 15021 | Canvas Category | | |
| 16000 | Payment Terms | | PaymentTerms |
| 16100 | Payment Plan | | PaymentPlan |
| 16110 | Scheduled Payment | | ScheduledPayment |
| 17000 | Crystal System Report | | |
| 17001 | Report Template Category | | |
| 17002 | Crystal Report Template | | |
| 17003 | Report Category | | |
| 17004 | Report Item | | |
| 17006 | Report Group | | |
| 17010 | Action Report Template | | |
| 17011 | Word Report Template | | |
| 17012 | SMSQR System Report | | |
| 17100 | Word System Report | | |
| 17105 | SMS System Report | | |
| 17200 | Report Menu Change | | ReportMenuItem |
| 17210 | Report Menu Item | | |
| 17220 | Report Menu Template | | |
| 17230 | Report Menu Group | | ReportMenuItem |
| 17240 | Action Report Menu Item | | ReportMenuItem |
| 17250 | Word Report Menu Item | | ReportMenuItem |
| 17260 | SMS Report Menu Item | | ReportMenuItem |
| 17270 | Crystal Report Menu Item | | ReportMenuItem |
| 17280 | Report Menu BMP | | |
| 17290 | SQL Report Menu Item | | ReportMenuItem |
| 17310 | System Report Menu Item | | |
| 17330 | System Report Menu Group | | |
| 17340 | System Action Report Menu Item | | |
| 17350 | System Word Report Menu Item | | |
| 17360 | System SMS Report Menu Item | | |
| 17370 | System Crystal Report Menu Item | | |
| 17380 | System SQL Report Menu Item | | |
| 17500 | Cyrious Printer Type | | |
| 17600 | Custom Report Range | | |
| 18001 | Time Clock Status | | |
| 19000 | Database Group | | |
| 19001 | Database Item | | |

---

### Payments & GL (ClassTypeID 20000-20999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 20000 | Master Payment | | Journal, Payment |
| 20001 | Order Payment | | Journal, Payment |
| 20002 | Over Payment | | Journal, Payment |
| 20003 | Change Return | | Journal, Payment |
| 20004 | Order Refund | | Journal, Payment |
| 20005 | Credit Refund | | Journal, Payment |
| 20006 | Order Credit Memo | | Journal, Payment |
| 20007 | Failed Payment | | Journal, Payment |
| 20008 | Tip Recorded | | Journal, Payment |
| 20009 | Bill Payment | | Journal, Payment |
| 20010 | Credit Adjustment | | Journal |
| 20011 | Change Over Payment | | |
| 20015 | Write Off | | Journal |
| 20020 | Finance Charge | | Journal |
| 20025 | System Finance Charge | | Journal |
| 20030 | CC Activity | | Journal |
| 20035 | Journal Entry | | Journal |
| 20036 | Deposit | | Journal |
| 20037 | Master Bill Payment | | Journal, Payment |
| 20038 | Miscellaneous Receipt | | Journal |
| 20039 | Vendor Credit Adjustment | | |
| 20040 | Master Tip Payout | | Journal |
| 20041 | Detail Tip Payout | | Journal |

---

### Activities & Journal (ClassTypeID 20050-21999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 20050 | Master Time Card | | Journal, TimeCard |
| 20051 | Detail Time Card | | Journal, TimeCard |
| 20060 | Email Activity | | Journal, EmailActivity |
| 20061 | Email Activity Template | | Journal, EmailActivity |
| 20065 | Note Activity | | Journal |
| 20070 | Viewing CC Number Activity | | Journal |
| 20075 | JDF Activity | | |
| 20500 | Transaction Activity | | Journal |
| 20505 | Line Item Activity | | Journal |
| 20510 | Company Activity | | Journal |
| 20515 | Company Stage Change Activity | | Journal |
| 20520 | Transaction Part Usage Activity | | Journal |
| 20521 | Station Activity | | Journal |
| 20522 | Proof Approval Activity | | Journal |
| 20530 | Inventory Adjustment | | Journal |
| 20531 | Inventory TransPartDetail | | |
| 20535 | Global Inventory Adjustment Activity | | Journal |
| 20536 | Inventory Division Summary Adjustment Header | | |
| 20540 | Import Activity | | Journal |
| 20550 | Global Cost Adjustment Activity | | Journal |
| 20560 | Part Usage Processor | | |
| 20565 | Part Usage Card | | Journal, PartUsageCard |
| 20566 | Inventory Transfer Header | | |
| 20567 | Inventory Transfer Detail | | |
| 20570 | Backup Activity | | Journal |
| 20580 | Service Item | | Journal |
| 20590 | Report Activity | | Journal |
| 20600 | Login Activity | | Journal |
| 20610 | Logout Activity | | Journal |
| 20620 | Authentication Activity | | Journal |
| 20630 | EULA Terms Activity | | |
| 20700 | Reservation Activity | | Journal |
| 21000 | Activity Viewer | | |
| 21100 | Contact Activity | | Journal, ContactActivity |
| 21150 | Contact Activity Template | | Journal, ContactActivity |
| 21200 | Macro Activity | | Journal, RuleActivity |
| 21250 | Macro Activity Template | | Journal, RuleActivity |
| 21300 | Work Assignment | | Journal, ContactActivity |
| 21350 | Work Assignment Template | | Journal, ContactActivity |

---

### User Fields & Queries (ClassTypeID 22000-22999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 22000 | User Field | | UserFieldDef |
| 22001 | User Field Layout | | UserFieldLayout |
| 22300 | CCCS Configuration | | |
| 22500 | Advanced Query Node | | |
| 22501 | Advanced Query Criteria | | |
| 22502 | Advanced Query Boolean Node | | |
| 22503 | Advanced Query Table | | |
| 22504 | Advanced Query Criteria Category | | |
| 22505 | Criteria Builder Category | | |
| 22506 | Advanced Query Macro | | |
| 22507 | User Field Builder Category | | |
| 22508 | DB Criteria Builder Category | | |
| 22509 | Static Criteria Builder Category | | |
| 22510 | Advanced Query Visual Element | | |
| 22600 | Advanced Query Manager | | AdvQuery |
| 22610 | Advanced Query Manager Category | | |
| 22700 | User Query Criteria | | AdvQueryNode |
| 22701 | User Query Boolean Node | | AdvQueryNode |
| 22702 | User Query Table | | AdvQueryNode |
| 22703 | User Query Macro | | AdvQueryNode |

---

### Rules & Automation (ClassTypeID 23000-23999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 23000 | Macro | | RuleMacro |
| 23005 | Macro Category | | |
| 23110 | Contact Activity Action | | Journal, RuleAction |
| 23120 | Email Action | | Journal, RuleAction |
| 23130 | Report Action | | Journal, RuleAction |
| 23141 | UDF Action | | Journal, RuleAction |
| 23144 | Popup Action | | Journal, RuleAction |
| 23146 | Transaction Action | | Journal, RuleAction |
| 23148 | Another Macro Action | | Journal, RuleAction |
| 23150 | Company Activity Action | | Journal, RuleAction |
| 23160 | Service Contract Action | | Journal, RuleAction |
| 23170 | Work Assignment Action | | Journal, RuleAction |
| 23180 | Transaction Update Action | | Journal, RuleAction |
| 23181 | Transaction Station Action | | Journal, RuleAction |
| 23182 | Company Stage Action | | Journal, RuleAction |

---

### Replication & System (ClassTypeID 24000-25999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 24001 | Groom | | |
| 24002 | Matchmaker | | |
| 24003 | SSLIP's Client | | |
| 24010 | SSLIP's Log | | |
| 24011 | HoneyDoLog | | |
| 24020 | SSLIP's Region | | |
| 24021 | SSLIP's User | | |
| 24030 | HoneyDoSQLBatch | | |
| 24031 | HoneyDoAuthorize | | |
| 24040 | HoneyDoGroup | | |
| 25000 | Language | | |
| 25010 | Translation Item | | |

---

### Production & Scheduling (ClassTypeID 26000-29999)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 26000 | Line Item Stage | | Element |
| 26001 | Service Ticket Stage | | Element |
| 26002 | Vendor Stage | | Element |
| 26100 | Station | | Station |

---

### Education & Service (ClassTypeID 30000+)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 30000 | Course | | |
| 30030 | Course Event | | CourseEvent |
| 30050 | Course Contact Link | | CourseContactLink |
| 30100 | Course Section Type | | PricingElement |
| 30110 | Course Section | | |
| 30200 | Contract Period | | ContractPeriod |
| 31000 | Dashboard | | Dashboard |
| 31001 | System Dashboard | | |
| 31002 | Dashboard Instrument | | |

---

### Payroll (ClassTypeID 35000+)

| ClassTypeID | Object Type | System Data | Database Table |
|-------------|-------------|-------------|----------------|
| 35250 | Payroll | | Journal |
| 35300 | Payroll Payment Summary | | Journal, Payment |
| 35301 | Payroll Payment Detail | | Journal, Payment |

---

## Usage Examples

### Finding Records by Parent Entity

```sql
-- All phone numbers for an account
SELECT * FROM dbo.PhoneNumber
WHERE ParentID = @AccountID AND ParentClassTypeID = 2000

-- All addresses for an account
SELECT a.*
FROM dbo.AddressLink al
JOIN dbo.Address a ON al.AddressID = a.ID
WHERE al.ParentID = @AccountID AND al.ParentClassTypeID = 2000

-- All line items for an order
SELECT * FROM dbo.TransDetail
WHERE ParentID = @TransHeaderID AND ParentClassTypeID = 10000

-- All parameters for a line item
SELECT * FROM dbo.TransDetailParam
WHERE ParentID = @TransDetailID AND ParentClassTypeID = 10100
```

### Discovering ClassTypeID Values in Use

```sql
-- See all ClassTypeIDs in TimeCard table
SELECT DISTINCT ClassTypeID, COUNT(*) AS RecordCount
FROM dbo.TimeCard
GROUP BY ClassTypeID

-- See all ParentClassTypeIDs referencing PhoneNumber
SELECT DISTINCT ParentClassTypeID, COUNT(*) AS RecordCount
FROM dbo.PhoneNumber
GROUP BY ParentClassTypeID

-- Find all entities of a specific ClassTypeID in Journal
SELECT ClassTypeID, COUNT(*) AS ActivityCount
FROM dbo.Journal
WHERE ClassTypeID = 20065  -- Note Activity
GROUP BY ClassTypeID
```

---

## Related Documentation

- **[Field Values Reference](field_values.md)** - TransactionType, StatusID, and other field value mappings
- **[Wiki Database Integration Extract](../../../output/wiki/extracts/database_integration_knowledge.md)** - Original source documentation
