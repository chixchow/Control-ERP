# Control ERP Database - Domain Classification

## Overview

The StoreData database contains **187 tables** organized into **16 functional domains**. This document groups every table by its business function.

---

## 1. Accounts & Contacts (10 tables)

Core customer/vendor management and contact tracking.

| Table | Row Count | Description |
|-------|-----------|-------------|
| Account | 54,719 | Master customer/vendor records |
| AccountContact | 64,851 | Contact people linked to accounts |
| AccountContactUserField | 64,785 | Custom fields for account contacts |
| AccountUserField | 54,752 | Custom fields for accounts |
| Address | 125,614 | Physical/mailing addresses |
| AddressLink | 304,987 | Links addresses to accounts/contacts/transactions |
| PhoneNumber | 314,805 | Phone/fax numbers for any entity |
| ContactActivity | 163,716 | Activity log for contacts (calls, notes, etc.) |
| CCToken | 21,292 | Stored credit card tokens |
| CCTransaction | 21,551 | Credit card transaction records |

**Key relationships:** Account -> AddressLink -> Address; Account -> AccountContact -> PhoneNumber; Account -> CCToken -> CCTransaction

---

## 2. Orders & Transactions (17 tables)

Transaction lifecycle from estimates through invoices, including line items, modifications, and history.

| Table | Row Count | Description |
|-------|-----------|-------------|
| TransHeader | 232,243 | Transaction headers (estimates, orders, invoices, POs, service tickets) |
| TransDetail | 538,252 | Transaction line items |
| TransHeaderHistory | 1 | Historical snapshots of transaction headers |
| TransDetailHistory | 1 | Historical snapshots of transaction details |
| TransHeaderUserField | 192,099 | Custom fields for transactions |
| TransDetailParam | 773,195 | Parameters/attributes for line items |
| TransDetailParamHistory | 1 | Historical snapshots of detail params |
| TransMod | 214,783 | Transaction modifications/options |
| TransModHistory | 1 | Historical snapshots of modifications |
| TransPart | 2,256,954 | Parts/materials used on transactions |
| TransPartHistory | 1 | Historical snapshots of transaction parts |
| TransTax | 1 | Tax calculations per line item |
| TransTaxHistory | 1 | Historical snapshots of tax records |
| TransVariation | 233,043 | Order variations (multiple versions of same item) |
| TransVariationHistory | 1 | Historical snapshots of variations |
| VendorTransDetail | 129,647 | Vendor-side transaction details (PO line items) |
| VendorTransDetailHistory | 1 | Historical snapshots of vendor details |

**Key relationships:** TransHeader -> TransDetail -> TransDetailParam; TransHeader -> TransVariation -> TransDetail; TransHeader -> TransPart -> Part; TransHeader.DefaultOrderID -> TransHeader (self-ref for estimate->order->invoice chain)

**TransactionType values:** 1=Estimate, 3=Order, 4=Invoice, 6=Service Ticket, 8=Purchase Order

---

## 3. Products & Pricing (12 tables)

Product catalog, pricing plans, and pricing configuration.

| Table | Row Count | Description |
|-------|-----------|-------------|
| Product | 277 | Product/goods item definitions |
| DerivedProduct | 2 | Products derived from base products |
| PricingPlan | 317 | Pricing plan definitions |
| PricingTable | 165 | Pricing tables with quantity breaks |
| PricingElement | 1,754 | Individual pricing elements |
| PricingLevel | 6 | Customer pricing tiers (A, B, C, etc.) |
| PricingLink | 9,657 | Links pricing plans to products |
| PricingGraphic | 253 | Graphic/image pricing items |
| CatalogItem | 581 | Online catalog items |
| QuickProduct | 11 | Quick-add product shortcuts |
| ProdModLink | 901 | Product-modifier associations |
| Element | 223 | Base elements (used in pricing/products) |

**Key relationships:** Product -> PricingLink -> PricingPlan -> PricingTable -> PricingElement; Product.StationID -> Station; Product.AccountCodeID -> GLAccount

---

## 4. Inventory & Parts (13 tables)

Physical inventory, parts/materials tracking, and warehouse management.

| Table | Row Count | Description |
|-------|-----------|-------------|
| Inventory | 27,268 | Inventory records per part/warehouse |
| InventoryLog | 906,714 | Inventory transaction log (in/out/adjust) |
| Part | 7,684 | Part/material/supply definitions |
| PartUsageCard | 286,891 | Part usage tracking per transaction |
| PartConsumptionJournal | 0 | Part consumption records |
| PartInfoTemplate | 1 | Templates for part information |
| PartInventoryConversion | 2,290 | Unit conversion rules for parts |
| QuickPart | 1 | Quick-add part shortcuts |
| Warehouse | 5 | Warehouse/location definitions |
| Reservation | 1 | Inventory reservations |
| AssemblyLink | 19 | Bill of materials / assembly links |
| GoodsItemPartLink | 1,388 | Links products to their required parts |
| Old_PartUserField | 2 | Legacy custom fields for parts |

**Key relationships:** Part -> Inventory -> InventoryLog; Part -> GoodsItemPartLink -> Product; Part.StationID -> Station; Part -> GLAccount (expense, asset, department)

---

## 5. Finance & GL (11 tables)

General ledger, journal entries, payments, and financial configuration.

| Table | Row Count | Description |
|-------|-----------|-------------|
| GLAccount | 452 | Chart of accounts |
| Ledger | 2,748,661 | General ledger entries |
| Journal | 5,184,393 | Journal entries (double-entry) |
| Payment | 286,084 | Payment records |
| PaymentAccount | 45 | Payment method configurations |
| PaymentPlan | 1 | Payment plan definitions |
| PaymentTerms | 28 | Net-30, Net-60, etc. terms |
| ScheduledPayment | 2 | Scheduled/recurring payments |
| Closeout | 10,510 | End-of-day closeout records |
| DiscountTable | 10 | Discount schedule definitions |
| DiscountTableItem | 39 | Discount table breakpoints |

**Key relationships:** Journal -> Ledger; Journal/Ledger.DivisionID -> DivisionData; Ledger.EmployeeID -> Employee; Part/Product.AccountCodeID -> GLAccount

---

## 6. Payroll & Time (18 tables)

Employee management, time tracking, payroll, and commissions.

| Table | Row Count | Description |
|-------|-----------|-------------|
| Employee | 103 | Employee records |
| EmployeeContact | 26 | Employee emergency/alternate contacts |
| EmployeeGroup | 10 | Employee departments/groups |
| EmployeeUserField | 49 | Custom fields for employees |
| TimeCard | 159,448 | Time clock entries (ClassTypeID 20050=parent, 20051=station detail) |
| TimeClockStatus | 8 | Time clock status lookup |
| Payroll | 1 | Payroll batch headers |
| PayrollPaycheck | 1 | Individual paychecks |
| PayrollPaycheckPayItem | 1 | Pay items on paychecks |
| PayrollPayItem | 36 | Pay item definitions (hourly, salary, etc.) |
| PayrollPayItemLink | 1 | Links pay items to employees |
| PayrollPTO | 1 | PTO/vacation tracking |
| PayrollTaxTable | 3 | Tax table definitions |
| PayrollTaxTableRow | 29 | Tax bracket rows |
| CommissionPlan | 0 | Commission plan definitions |
| CommissionPlanException | 0 | Commission plan exceptions |
| CommissionRate | 5 | Commission rate tiers |
| SalesGoal | 248 | Sales goal targets by employee/period |

**Key relationships:** Employee -> EmployeeGroup -> DivisionData; Employee.GroupID -> EmployeeGroup; TimeCard uses ClassTypeID to distinguish parent (20050) from station detail (20051) records

---

## 7. Artwork & Proofing (17 tables)

Artwork/design workflow, proof management, and approval tracking.

| Table | Row Count | Description |
|-------|-----------|-------------|
| ArtworkItem | 41,797 | Individual artwork items/files |
| ArtworkGroup | 84,770 | Groups of artwork items (per order/job) |
| ArtworkGroupStatusHistory | 164,226 | Status change audit trail |
| ArtworkGroupTransDetailLink | 78,998 | Links artwork groups to transaction line items |
| ArtworkComment | 4,109 | Comments/feedback on artwork |
| ArtworkDefaultPlayer | 11 | Default participant assignments |
| ArtworkDefaultRoleLink | 11 | Default role assignments |
| ArtworkDigestEntry | 0 | Digest notification entries |
| ArtworkLog | 208,887 | Activity log for artwork |
| ArtworkNotificationHistory | 156,832 | Notification history |
| ArtworkPlayer | 176,445 | Participants in artwork workflow |
| ArtworkPlayerRoleLink | 174,373 | Role assignments for participants |
| ArtworkProofFile | 41,741 | Proof file records |
| ArtworkSchema | 19 | Schema versioning for artwork module |
| Graphic | 954 | Graphic/image records |
| TransDetailGraphic | 1 | Graphics linked to transaction details |
| TransDetailGraphicHistory | 1 | Historical graphic links |
| TransDetailGraphicTmp | 1 | Temporary graphic links |

**Lookup tables:** _ArtworkBand (5), _ArtworkCollectionType (3), _ArtworkCommentType (5), _ArtworkLogType (16), _ArtworkPriority (6), _ArtworkRole (8), _ArtworkStatus (11)

**Key relationships:** ArtworkGroup -> ArtworkItem -> ArtworkProofFile -> ArtworkComment; ArtworkGroup -> ArtworkPlayer -> ArtworkPlayerRoleLink -> _ArtworkRole; ArtworkGroup.TransHeaderID -> TransHeader

---

## 8. Production & Workflow (10 tables)

Production scheduling, station management, and job tracking.

| Table | Row Count | Description |
|-------|-----------|-------------|
| Station | 98 | Production stations/workcenters |
| ActivityTimeSpan | 1 | Activity duration tracking |
| PrCA.Board.Data | 12 | Production board definitions |
| PrCA.Board.Enum.SortType | 0 | Board sort type enum |
| PrCA.Board.MyBoard | 43 | User's board preferences |
| PrCA.Board.StationLink | 22 | Links stations to boards |
| PrCA.Job.Data | 383 | Production job records |
| PrCA.Job.TransactionLink | 383 | Links jobs to transactions |
| PrCA.Schema | 5 | Schema versioning for production module |
| JDFTask | 0 | JDF (Job Definition Format) task records |

**Key relationships:** Station -> PrCA.Board.StationLink -> PrCA.Board.Data; PrCA.Job.Data -> Station; PrCA.Job.TransactionLink -> TransHeader

---

## 9. Shipping (8 tables)

Shipment tracking and carrier integration.

| Table | Row Count | Description |
|-------|-----------|-------------|
| Shipments | 70,584 | Shipment records |
| ShippingMethod | 11 | Shipping method definitions |
| FedExShippingLog | 2,758 | FedEx API transaction log |
| FedExShippingLogForShipments | 0 | FedEx shipment tracking |
| FedExShippingLogForShipments_FLS | 7,172 | FedEx shipment detail log |
| UPSShippingLog | 68,382 | UPS API transaction log |
| UPSShippingLogForShipments | 0 | UPS shipment tracking |
| UPSShippingLogForShipments_FLS | 28,463 | UPS shipment detail log |

**Key relationships:** Shipments.ShipToDivisionID -> DivisionData

---

## 10. Marketing & CRM (5 tables)

Marketing campaigns, email tracking, and recurring activities.

| Table | Row Count | Description |
|-------|-----------|-------------|
| Promotion | 4 | Promotion/discount campaign definitions |
| MarketingListItem | 96 | Marketing list members |
| RecurringActivity | 24 | Recurring scheduled activities |
| EmailActivity | 92,854 | Email send/receive log |
| Link | 0 | Hyperlink records |

**Key relationships:** Account.PromotionID -> Promotion

---

## 11. System & Config (30 tables)

System settings, security, user preferences, reports, and administration.

| Table | Row Count | Description |
|-------|-----------|-------------|
| Store | 2 | Store/location definitions |
| StoreConstant | 81 | System-wide constants |
| StoreLogging | 58,041 | System event log |
| DBSettings | 1 | Database configuration |
| SecurityTemplate | 11 | Security permission templates |
| SecurityRightChange | 4,755 | Security permission overrides |
| UserName | 99 | User login records |
| UserOption | 2,313 | User preference settings |
| UserColumns | 1 | User grid column preferences |
| UserFieldDef | 562 | Custom field definitions |
| UserFieldLayout | 24 | Custom field layout configuration |
| SelectionList | 2,980 | Dropdown list definitions |
| SelectionListItem | 21,034 | Dropdown list items |
| Variable | 5,592 | Variable/parameter definitions |
| Warden | 0 | Record locking |
| Sequence | 14 | Auto-number sequences |
| OptionSet | 32 | Configuration option sets |
| IOTemplate | 344 | Import/export templates |
| Report | 211 | Report definitions |
| ReportElement | 85 | Report sub-elements |
| ReportMenuItem | 899 | Report menu structure |
| ReportTemplate | 168 | Report layout templates |
| CrystalSystemReports | 1 | Crystal Reports integration |
| CommandLog | 0 | Command execution log |
| RefreshMonitor | 0 | Data refresh tracking |
| RecomputeMonitor | 0 | Recomputation tracking |
| Dashboard | 86 | Dashboard widget configurations |
| AdvQuery | 209 | Advanced query/filter definitions |
| AdvQueryNode | 85 | Advanced query filter nodes |
| SpeedNote | 34 | Quick note templates |
| CustomRange | 6 | Custom date/value range definitions |

---

## 12. Tax & Compliance (7 tables)

Tax configuration, rates, and exemptions.

| Table | Row Count | Description |
|-------|-----------|-------------|
| TaxClass | 14 | Tax class definitions |
| TaxLink | 15 | Tax class linkages |
| PostalCodeTaxClass | 899 | Tax rates by postal code |
| StateTaxExemptionLink | 725 | State tax exemption records |
| ProductTaxabilityCode | 6 | Product taxability codes |
| ProductTaxExemptLink | 3 | Product tax exemption links |
| Country | 0 | Country definitions |

---

## 13. Service (3 tables)

Service contracts and ticket management.

| Table | Row Count | Description |
|-------|-----------|-------------|
| ServiceContractType | 1 | Service contract type definitions |
| ServiceTicketPriority | 1 | Service ticket priority levels |
| ContractPeriod | 1 | Contract billing period definitions |

---

## 14. Education/Training (3 tables)

Course and training management.

| Table | Row Count | Description |
|-------|-----------|-------------|
| CourseTerm | 1 | Course term/semester definitions |
| CourseContactLink | 1 | Links contacts to courses |
| CourseUserField | 2 | Custom fields for courses |

---

## 15. Rules & Automation (3 tables)

Business rules and automation engine.

| Table | Row Count | Description |
|-------|-----------|-------------|
| RuleAction | 170,273 | Rule action execution log |
| RuleActivity | 293,757 | Rule activity log |
| RuleMacro | 68 | Rule macro definitions |

---

## 16. Miscellaneous/Lookup (12 tables)

Artwork enums, utility tables, and miscellaneous.

| Table | Row Count | Description |
|-------|-----------|-------------|
| _ArtworkBand | 5 | Artwork band/tier lookup |
| _ArtworkCollectionType | 3 | Artwork collection type lookup |
| _ArtworkCommentType | 5 | Comment type lookup |
| _ArtworkLogType | 16 | Log entry type lookup |
| _ArtworkPriority | 6 | Priority level lookup |
| _ArtworkRole | 8 | Participant role lookup |
| _ArtworkStatus | 11 | Artwork status lookup |
| _Unit | 55 | Unit of measure lookup |
| DivisionData | 3 | Business division definitions |
| CalendarLink | 13,944 | Calendar event links |
| CalendarStatus | 1 | Calendar status lookup |
| PAPISchema | 9 | PAPI schema versioning |
| WI_Tab_Table | 898 | Web interface tab config |
| WI_Tab_Table1 | 833 | Web interface tab config (alt) |
| WebHookLog | 1,497 | Webhook execution log |
| Util.MacroMessageType | 95 | Macro message type lookup |
| Util.Numbers | 65,536 | Numbers table (1-65536, utility) |
| HistoricalC3SCustomer | 33 | Historical C3S customer mapping |

---

## Summary Statistics

| Domain | Tables | Total Rows |
|--------|--------|------------|
| Accounts & Contacts | 10 | 1,174,819 |
| Orders & Transactions | 17 | 4,341,133 |
| Products & Pricing | 12 | 13,770 |
| Inventory & Parts | 13 | 1,232,544 |
| Finance & GL | 11 | 8,229,773 |
| Payroll & Time | 18 | 160,001 |
| Artwork & Proofing | 17 | 932,067 |
| Production & Workflow | 10 | 532 |
| Shipping | 8 | 177,359 |
| Marketing & CRM | 5 | 92,978 |
| System & Config | 30 | 92,504 |
| Tax & Compliance | 7 | 1,662 |
| Service | 3 | 3 |
| Education/Training | 3 | 4 |
| Rules & Automation | 3 | 464,098 |
| Miscellaneous/Lookup | 12 | 82,399 |
| **TOTAL** | **187** | **~16.5M** |
