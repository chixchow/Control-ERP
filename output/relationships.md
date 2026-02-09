# Control ERP Database Relationships

<!-- Internal consistency verified 2026-02-09: 89 explicit FKs documented, all FK references point to valid tables in schema set -->

## Explicit Foreign Key Relationships

These are formally declared FK constraints in the database.

### Account Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_Account_DivisionData | Account | DivisionID | DivisionData | ID |
| FK_Account_PricingLevel | Account | PricingLevelID | PricingLevel | ID |
| FK_Account_Promotion | Account | PromotionID | Promotion | ID |
| FK_Account_VendorPaymentTerms | Account | VendorPaymentTermsID | PaymentTerms | ID |
| FK_CCToken_Account | CCToken | AccountID | Account | ID |
| FK_CCToken_AccountContact | CCToken | ContactID | AccountContact | ID |
| FK_CCTransaction_Account | CCTransaction | AccountID | Account | ID |
| FK_CCTransaction_CCToken | CCTransaction | TokenID | CCToken | ID |
| FK_CCTransaction_TransHeader | CCTransaction | TransHeaderID | TransHeader | ID |

### Artwork Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_ArtworkComment_ArtworkItem | ArtworkComment | ArtworkItemID | ArtworkItem | ID |
| FK_ArtworkComment__ArtworkCommentType | ArtworkComment | CommentTypeID | _ArtworkCommentType | ID |
| FK_ArtworkComment_ArtworkPlayer | ArtworkComment | PlayerID | ArtworkPlayer | ID |
| FK_ArtworkComment_ArtworkProofFile | ArtworkComment | ProofFileID | ArtworkProofFile | ID |
| FK_ArtworkDefaultRoleLink_ArtworkDefaultPlayer | ArtworkDefaultRoleLink | DefaultPlayerID | ArtworkDefaultPlayer | ID |
| FK_ArtworkDefaultRoleLink__ArtworkRole | ArtworkDefaultRoleLink | RoleID | _ArtworkRole | ID |
| FK_ArtworkDigestEntry_ArtworkGroup | ArtworkDigestEntry | GroupID | ArtworkGroup | ID |
| FK_ArtworkDigestEntry_ArtworkPlayer | ArtworkDigestEntry | PlayerID | ArtworkPlayer | ID |
| FK_ArtworkDigestEntry_TransHeader | ArtworkDigestEntry | TransHeaderID | TransHeader | ID |
| FK_ArtworkGroup_Account | ArtworkGroup | AccountID | Account | ID |
| FK_ArtworkGroup__ArtworkBand | ArtworkGroup | BandID | _ArtworkBand | ID |
| FK_ArtworkGroup__ArtworkCollectionType | ArtworkGroup | CollectionTypeID | _ArtworkCollectionType | ID |
| FK_ArtworkGroup__ArtworkPriority | ArtworkGroup | PriorityID | _ArtworkPriority | ID |
| FK_ArtworkGroup__ArtworkStatus | ArtworkGroup | StatusID | _ArtworkStatus | ID |
| FK_ArtworkGroup_TransHeader | ArtworkGroup | TransHeaderID | TransHeader | ID |
| FK_ArtworkGroupStatusHistory_ArtworkGroup | ArtworkGroupStatusHistory | ArtworkGroupID | ArtworkGroup | ID |
| FK_ArtworkGroupStatusHistory__ArtworkStatus | ArtworkGroupStatusHistory | FromStatusID | _ArtworkStatus | ID |
| FK_ArtworkGroupStatusHistory_ArtworkPlayer | ArtworkGroupStatusHistory | PlayerID | ArtworkPlayer | ID |
| FK_ArtworkGroupStatusHistory__ArtworkStatus1 | ArtworkGroupStatusHistory | ToStatusID | _ArtworkStatus | ID |
| FK_ArtworkGroupTransDetailLink_ArtworkGroup | ArtworkGroupTransDetailLink | ArtworkGroupID | ArtworkGroup | ID |
| FK_ArtworkGroupTransDetailLink_TransDetail | ArtworkGroupTransDetailLink | TransDetailID | TransDetail | ID |
| FK_ArtworkItem_ArtworkGroup | ArtworkItem | GroupID | ArtworkGroup | ID |
| FK_ArtworkLog_Account | ArtworkLog | AccountID | Account | ID |
| FK_ArtworkLog_ArtworkGroup | ArtworkLog | ArtworkGroupID | ArtworkGroup | ID |
| FK_ArtworkLog_ArtworkItem | ArtworkLog | ArtworkItemID | ArtworkItem | ID |
| FK_ArtworkLog__ArtworkLogType | ArtworkLog | LogTypeID | _ArtworkLogType | ID |
| FK_ArtworkLog_ArtworkPlayer | ArtworkLog | PlayerID | ArtworkPlayer | ID |
| FK_ArtworkLog_ArtworkProofFile | ArtworkLog | ProofFileID | ArtworkProofFile | ID |
| FK_ArtworkLog_TransHeader | ArtworkLog | TransHeaderID | TransHeader | ID |
| FK_ArtworkNotificationHistory_ArtworkGroup | ArtworkNotificationHistory | GroupID | ArtworkGroup | ID |
| FK_ArtworkNotificationHistory_ArtworkPlayer | ArtworkNotificationHistory | PlayerID | ArtworkPlayer | ID |
| FK_ArtworkPlayer_AccountContact | ArtworkPlayer | ContactID | AccountContact | ID |
| FK_ArtworkPlayer_Employee | ArtworkPlayer | EmployeeID | Employee | ID |
| FK_ArtworkPlayer_TransHeader | ArtworkPlayer | TransHeaderID | TransHeader | ID |
| FK_ArtworkPlayerRoleLink_ArtworkPlayer | ArtworkPlayerRoleLink | PlayerID | ArtworkPlayer | ID |
| FK_ArtworkPlayerRoleLink__ArtworkRole | ArtworkPlayerRoleLink | RoleID | _ArtworkRole | ID |
| FK_ArtworkProofFile_ArtworkItem | ArtworkProofFile | ArtworkItemID | ArtworkItem | ID |

### Employee Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_Employee_EmployeeGroup | Employee | GroupID | EmployeeGroup | ID |
| FK_EmployeeGroup_DivisionData | EmployeeGroup | DivisionID | DivisionData | ID |
| FK_EmployeeGroup_EmployeeGroup | EmployeeGroup | ParentID | EmployeeGroup | ID |

### Inventory & Parts Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_GoodsItemPartLink_CustomerGoodsItem | GoodsItemPartLink | GoodsItemID | Product | ID |
| FK_GoodsItemPartLink_Part | GoodsItemPartLink | PartID | Part | ID |
| FK_Inventory_DivisionData | Inventory | DivisionID | DivisionData | ID |
| FK_Part_InventoryAccount | Part | AssetAccountID | GLAccount | ID |
| FK_Part_ExpenseAccount | Part | ExpenseAccountID | GLAccount | ID |
| FK_Part_GLDepartment | Part | GLDepartmentID | GLAccount | ID |
| FK_Part_Station | Part | StationID | Station | ID |

### Finance & GL Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_Journal_DivisionData | Journal | DivisionID | DivisionData | ID |
| FK_Ledger_DivisionData | Ledger | DivisionID | DivisionData | ID |
| FK_Ledger_Employee | Ledger | EmployeeID | Employee | ID |

### Product Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_CustomerGoodsItem_IncomeAccount | Product | AccountCodeID | GLAccount | ID |
| FK_CustomerGoodsItem_Station | Product | StationID | Station | ID |

### Production & Workflow Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_CCON.PrCA.MyBoard_CCON.PrCA.Board | PrCA.Board.MyBoard | BoardID | PrCA.Board.Data | ID |
| FK_CCON.PrCA.MyBoard_Employee | PrCA.Board.MyBoard | EmployeeID | Employee | ID |
| FK_CCON.PrCA.StationBoardLink_CCON.PrCA.Board | PrCA.Board.StationLink | BoardID | PrCA.Board.Data | ID |
| FK_CCON.PrCA.StationBoardLink_Station | PrCA.Board.StationLink | StationID | Station | ID |
| FK_Job.Data_Station | PrCA.Job.Data | StationID | Station | ID |
| FK_Job.TransactionJobLink_Job.Data | PrCA.Job.TransactionLink | JobID | PrCA.Job.Data | ID |
| FK_Job.TransactionJobLink_TransHeader | PrCA.Job.TransactionLink | TransHeaderID | TransHeader | ID |

### Transaction Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_TransDetail_DivisionData | TransDetail | ProductionDivisionID | DivisionData | ID |
| FK_TransDetail_Station | TransDetail | StationID | Station | ID |
| FK_TransDetail_TransVariation | TransDetail | VariationID | TransVariation | ID |
| FK_TransHeader_ApprovedBy | TransHeader | ApprovedByID | Employee | ID |
| FK_TransHeader_DefaultOrderID | TransHeader | DefaultOrderID | TransHeader | ID |
| FK_TransHeader_DefaultOrderItem | TransHeader | DefaultOrderItemID | TransDetail | ID |
| FK_TransHeader_DivisionData | TransHeader | DivisionID | DivisionData | ID |
| FK_TransHeader_OrderedBy | TransHeader | OrderedByID | Employee | ID |
| FK_TransHeader_DivisionData1 | TransHeader | ProductionDivisionID | DivisionData | ID |
| FK_TransHeader_ReceivedBy | TransHeader | ReceivedByID | Employee | ID |
| FK_TransHeader_RequestedBy | TransHeader | RequestedByID | Employee | ID |
| FK_TransHeader_SalesStation | TransHeader | SalesStationID | Station | ID |
| FK_TransHeader_ShipFromCustomer | TransHeader | ShipFromCustomerID | Account | ID |
| FK_TransPart_Part | TransPart | PartID | Part | ID |
| FK_TransPart_Station | TransPart | StationID | Station | ID |
| FK_TransPart_TransHeader | TransPart | TransHeaderID | TransHeader | ID |
| FK_TransTax_TransDetail | TransTax | TransDetailID | TransDetail | ID |
| FK_TransTax_TransHeader | TransTax | TransHeaderID | TransHeader | ID |
| FK_TransVariation_TransHeader | TransVariation | ParentID | TransHeader | ID |

### Shipping Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_Shipments_DivisionData | Shipments | ShipToDivisionID | DivisionData | ID |

### Warehouse Domain
| FK Name | Parent Table | Parent Column | Referenced Table | Referenced Column |
|---------|-------------|---------------|-----------------|-------------------|
| FK_Warehouse_DivisionData | Warehouse | DivisionID | DivisionData | ID |

---

## Implicit Relationships (By Column Naming Conventions)

These columns follow naming patterns that imply relationships, even without formal FK constraints.

### Universal Columns (Present in Nearly All Tables)
| Column | Appears In | Likely References | Notes |
|--------|-----------|------------------|-------|
| ClassTypeID | 179 tables | Enum/lookup value | Identifies the record type (e.g., Account=2000, Product=49) |
| SeqID | 174 tables | Internal sequence | Used for ordering/sequencing within parent |
| StoreID | 156 tables | Store.ID | Multi-store support; links records to their store |

### High-Frequency Relationship Columns
| Column | Appears In | Likely References | Notes |
|--------|-----------|------------------|-------|
| ParentID | 40 tables | Same table or parent record | Self-referencing hierarchy or parent record |
| ParentClassTypeID | 32 tables | ClassType lookup | Type of parent record |
| StationID | 31 tables | Station.ID | Production station assignment |
| AccountID | 31 tables | Account.ID | Customer/vendor account link |
| EmployeeID | 26 tables | Employee.ID | Employee assignment |
| TransHeaderID | 25 tables | TransHeader.ID | Transaction header link |
| PartID | 22 tables | Part.ID | Part/material reference |
| ContactID | 21 tables | AccountContact.ID | Contact person reference |
| DivisionID | 19 tables | DivisionData.ID | Business division |
| TransactionID | 18 tables | TransHeader.ID | Alternative transaction reference |
| TransDetailID | 18 tables | TransDetail.ID | Transaction line item |
| ProductID | 14 tables | Product.ID | Product/goods item |
| StatusID | 13 tables | Status lookup values | Record status |
| ImageID | 13 tables | Graphic.ID or external | Image/graphic reference |
| TaxClassID | 13 tables | TaxClass.ID | Tax classification |
| TransPartID | 13 tables | TransPart.ID | Transaction part link |
| UnitID | 13 tables | _Unit.UnitID | Unit of measure |
| WarehouseID | 12 tables | Warehouse.ID | Warehouse location |
| CompletedByID | 12 tables | Employee.ID | Completion tracking |
| JobID | 11 tables | PrCA.Job.Data.ID | Production job |
| CategoryID | 11 tables | SelectionList/Category | Category classification |
| AddressID | 10 tables | Address.ID | Physical address |
| GoodsItemID | 10 tables | Product.ID | Product reference (legacy name) |
| CreatedByID | 8 tables | Employee.ID | Audit: who created |
| GLDepartmentID | 7 tables | GLAccount.ID | GL department |
| InventoryID | 7 tables | Inventory.ID | Inventory record |
| ShippingAddressID | 7 tables | Address.ID | Ship-to address |
| SalespersonID | 4 tables | Employee.ID | Sales rep assignment |
| PricingLevelID | 4 tables | PricingLevel.ID | Pricing tier |
| SelectionListID | 4 tables | SelectionList.ID | Dropdown/list reference |
| VariationID | 4 tables | TransVariation.ID | Order variation |
| OrderID | 4 tables | TransHeader.ID | Order reference |
| PricingPlanID | 2 tables | PricingPlan.ID | Pricing plan |
| VendorID | 3 tables | Account.ID | Vendor (account with IsVendor) |

---

## ClassTypeID Values (Key Record Type Identifiers)

ClassTypeID is the most important implicit relationship in Control. It appears in 179 tables and identifies what type of record a row represents.

### Known ClassTypeID Values
| ClassTypeID | Entity Type | Primary Table |
|------------|-------------|---------------|
| 2000 | Account (Customer/Vendor) | Account |
| 49 | Product (Goods Item) | Product |
| 30 | Part (Material/Supply) | Part |
| 32 | Employee | Employee |
| 8001 | GL Account | GLAccount |
| 20050 | TimeCard (Clock In/Out Parent) | TimeCard |
| 20051 | TimeCard (Station Time Detail) | TimeCard |

### ClassTypeID + ParentID Pattern
Many tables use the combination of `ClassTypeID` + `ParentID` + `ParentClassTypeID` to create polymorphic relationships. For example:
- A `Variable` record with `ParentClassTypeID=49` and `ParentID=123` belongs to Product #123
- A `Variable` record with `ParentClassTypeID=2000` and `ParentID=456` belongs to Account #456
- This pattern allows a single table to store data for multiple entity types

### StoreID Pattern
Most tables include `StoreID` referencing `Store.ID`. In multi-store deployments, this scopes records to a specific store location. Related columns like `LinkStoreID`, `ParentStoreID`, `CategoryStoreID` follow the same pattern for cross-references.

---

## Self-Referencing Relationships

| Table | Column | Description |
|-------|--------|-------------|
| EmployeeGroup | ParentID | Hierarchical group structure |
| TransHeader | DefaultOrderID | Links invoice/order back to original order |

---

## Key Join Patterns

### Transaction Chain (Estimate -> Order -> Invoice)
```
TransHeader (TransactionType=1, Estimate)
  -> TransHeader (TransactionType=3, Order) via DefaultOrderID
    -> TransHeader (TransactionType=4, Invoice) via DefaultOrderID
```

### Transaction to Line Items
```
TransHeader.ID = TransDetail.ParentID (via ParentID + ClassTypeID pattern)
TransHeader.ID = TransVariation.ParentID (explicit FK)
TransDetail.VariationID = TransVariation.ID
```

### Transaction to Parts Used
```
TransHeader.ID = TransPart.TransHeaderID
TransPart.PartID = Part.ID
TransPart.StationID = Station.ID
```

### Account to Contacts to Addresses
```
Account.ID = AccountContact.ParentID (via ClassTypeID=2000)
Account/AccountContact -> AddressLink (via ClassTypeID pattern)
AddressLink -> Address.ID
Account/AccountContact -> PhoneNumber (via ClassTypeID pattern)
```

### Artwork Workflow
```
ArtworkGroup.TransHeaderID -> TransHeader.ID (links to order)
ArtworkGroup.AccountID -> Account.ID (links to customer)
ArtworkGroup -> ArtworkItem (GroupID)
ArtworkItem -> ArtworkProofFile (ArtworkItemID)
ArtworkItem -> ArtworkComment (ArtworkItemID)
ArtworkGroup -> ArtworkPlayer (TransHeaderID-based)
ArtworkPlayer -> ArtworkPlayerRoleLink -> _ArtworkRole
ArtworkGroup -> ArtworkGroupStatusHistory (status tracking)
ArtworkGroup -> ArtworkGroupTransDetailLink -> TransDetail (links to line items)
```

### Financial Posting
```
TransHeader/TransDetail -> Journal (via ClassTypeID pattern)
Journal -> Ledger (via ClassTypeID pattern)
Ledger.EmployeeID -> Employee.ID
Journal.DivisionID / Ledger.DivisionID -> DivisionData.ID
```

### Production Workflow
```
Station.ID <- PrCA.Board.StationLink.StationID
PrCA.Board.StationLink.BoardID -> PrCA.Board.Data.ID
PrCA.Job.Data.StationID -> Station.ID
PrCA.Job.TransactionLink.JobID -> PrCA.Job.Data.ID
PrCA.Job.TransactionLink.TransHeaderID -> TransHeader.ID
```

### Payments
```
Payment -> TransHeader (via ClassTypeID pattern)
Payment -> Account (via ClassTypeID pattern)
CCTransaction.AccountID -> Account.ID
CCTransaction.TransHeaderID -> TransHeader.ID
CCTransaction.TokenID -> CCToken.ID
CCToken.AccountID -> Account.ID
```
