# CHAPI Architecture Reference

Developer reference for Control HTTP API (CHAPI) and SQL Bridge architecture.

---

## 1. Overview

**CHAPI** (Cyrious Host Application Programming Interface) is Control's server-side service responsible for:
- ID generation for database records
- Record locking and unlocking
- Licensing and validation
- HTTP endpoint for triggering stored procedures

**SQL Bridge** is a collection of SQL Server stored procedures and functions that safely insert or update data into the Control database and notify CHAPI of changes. CHAPI then notifies SSLIP and all copies of Control about the updates.

**When to use CHAPI:**
- All write operations (INSERT, UPDATE, DELETE) to the Control database
- Any external integration that modifies Control data
- Website imports, external program imports, third-party integrations

**Critical Rule:** Never perform direct SQL writes without using SQL Bridge. Direct writes will not notify Control, causing data inconsistency and potential overwrites.

---

## 2. HTTP Endpoint

CHAPI provides a TCP endpoint that external programs can call to trigger stored procedures in the StoreData database.

### URL Format

```
http://{server}:12556/chapi/sqlmacro?name={ProcName}&{param}={value}
```

**Components:**
- `{server}` - DNS name or IP address of the Control server
- `12556` - Standard CHAPI port (may be customized during installation)
- `/chapi/sqlmacro` - Endpoint path for stored procedure execution
- `name={ProcName}` - Name of the stored procedure to invoke
- `{param}={value}` - URL parameters passed to the stored procedure

**Optional Parameters:**
- `stripquotes=true` - Strips quotes from both ends of parameter values
- Multiple parameters can be chained with `&`

### Example

```
http://localhost:12556/chapi/sqlmacro?name=TestProcedure&IntValue=3&StringValue=Hello
```

This executes:
```sql
EXEC TestProcedure @IntValue=3, @StringValue='Hello'
```

### HTTP Methods

- **GET** - For simple parameter passing
- **POST** - For complex data or large payloads

### Response Format

Returns JSON with HTTP 200 OK:
```json
[{"IntValue":"3","StringValue":"Hello"}]
```

---

## 3. SQL Bridge Core Functions

All SQL Bridge functions are in the `SQLBridge.Chapi` schema. These are the foundational functions for safe database operations.

| Function | Purpose | When to Call | Parameter Signature |
|----------|---------|--------------|---------------------|
| **NextID** | Get next available ID(s) for new records | Before INSERT operations | `(ClassTypeID INT, Count INT)` → Returns INT |
| **NextNumber** | Get next sequential number (order#, estimate#, etc.) | Before INSERT for numbered entities | `(SpecialTypeName VARCHAR(32), Count INT)` → Returns INT |
| **Lock** | Lock a record for editing | Before UPDATE operations | `(ID INT, ClassTypeID INT)` → Returns BIT (1=success, 0=fail) |
| **Unlock** | Release lock on a record | After UPDATE operations complete | `(ID INT, ClassTypeID INT)` → Returns BIT |
| **Refresh** | Notify Control to reload a record | After INSERT/UPDATE/DELETE operations | `(ID INT, ClassTypeID INT, eTag INT)` → Returns BIT |
| **RefreshEx** | Refresh with macro trigger notifications | After operations that should trigger macros | `(ID INT, ClassTypeID INT, eTag INT, MacroMessageTypes, SessionID GUID)` → Returns BIT |
| **Recompute** | Recalculate derived fields (pricing, totals) | After changes that affect calculations | `(ID INT, ClassTypeID INT, Notes VARCHAR(256))` → Returns BIT |

### NextID Details

Allocates ID(s) from Control's ID tracking system. **Never** use SQL IDENTITY or manually assigned IDs.

**Example:**
```sql
DECLARE @OrderID INT = dbo.[SQLBridge.Chapi.NextID](10000, 1);  -- 1 TransHeader
DECLARE @LineItemID INT = dbo.[SQLBridge.Chapi.NextID](10100, 2);  -- 2 TransDetail records
```

ClassTypeIDs must match the target table. See wiki extract `database_integration_knowledge.md` for complete ClassTypeID mapping (350+ entries).

### NextNumber Details

Used for human-readable sequential numbers, not database IDs.

**Special Type Names:**
- `OrderNumber`
- `EstimateNumber`
- `InvoiceNumber`
- `PurchaseOrderNumber`
- `BillNumber`
- `ReceivingDocNumber`
- `CompanyNumber`
- `UserCloseOut`, `DailyCloseOut`, `MonthlyCloseOut`, `YearlyCloseOut`
- `ExportCloseOut`, `RoyaltyCloseOut`, `PCChargeSettlement`
- `WeeklyCloseout`, `QuarterlyCloseout`, `SessionCloseout`

**Example:**
```sql
DECLARE @OrderNumber INT = dbo.[SQLBridge.Chapi.NextNumber]('OrderNumber', 1);
```

### Lock/Unlock Details

**Lock** must be called before modifying any record. If lock fails (returns 0), another user is editing the record.

**Example:**
```sql
DECLARE @LockSuccess BIT = dbo.[SQLBridge.Chapi.Lock](3345, 10000);
IF @LockSuccess = 0
BEGIN
    RAISERROR('Record is locked by another user', 16, 1);
    RETURN;
END
-- Perform updates
SELECT dbo.[SQLBridge.Chapi.Unlock](3345, 10000);
```

### Refresh/RefreshEx Details

**Refresh** must be called after any INSERT/UPDATE/DELETE to notify Control clients to reload the record.

**eTag/SeqID:** Use -1 to force refresh regardless of version, or increment SeqID during UPDATE operations.

**Example:**
```sql
SELECT dbo.[SQLBridge.Chapi.Refresh](3345, 10000, -1);
```

**RefreshEx** includes macro message types for triggering automation rules.

**Macro Message Types:** 95 types (IDs 0-94) documented in `macro-automation-reference.md`. Common examples:
- 5: OrderNew
- 6: OrderEdit
- 12: OrderSale
- 21: CompanyNew

**Example:**
```sql
DECLARE @MacroMessages MacroMessageType;
INSERT INTO @MacroMessages (MessageType, ID, ClassTypeID)
VALUES (66, 1037, 26100);  -- OrderStationToChange

SELECT dbo.[SQLBridge.Chapi.RefreshEx](3345, 10000, -1, @MacroMessages, NULL);
```

---

## 4. Import Stored Procedures Catalog

Pre-built stored procedures for common import operations. All use SQL Bridge internally.

| Procedure | Purpose | Key Parameters | Returns |
|-----------|---------|----------------|---------|
| **import_company** | Create new customer with contact(s) | @CompanyName, @URL, @WorkPhoneAC, @WorkPhoneNumber, @BStreetAddress1, @BCity, @BState, @BPostalCode, @AddContact, @FirstName, @LastName, @EmailAddress | Account.ID, AccountContact.ID |
| **import_contact** | Add contact to existing customer | @AccountID or @CompanyName, @FirstName, @LastName, @Position, @EmailAddress | AccountContact.ID |
| **import_order** | Create new order | @AccountID, @OrderDescription, @RecomputeOnSave, @ContactID, @StatusID, @OrderDate, @PONumber | TransHeader.ID, OrderNumber |
| **import_estimate** | Create new estimate | @AccountID, @EstimateDescription, @RecomputeOnSave, @ContactID, @EstimateCreatedDate | TransHeader.ID, EstimateNumber |
| **import_line_item** | Add line item to order/estimate | @OrderNumber or @EstimateNumber or @THID, @ProductName or @ProductID, @Quantity, @RecomputeOnSave, @BasePrice | TransDetail.ID |
| **import_payment** | Apply payment to order | @THID or @OrderNumber, @PaymentMethod or @PaymentMethodID, @PaymentAmount, @PaymentDT, @EmployeeID | Payment.ID |
| **import_address** | Create address and address links | @StreetAddress1, @City, @State, @PostalCode, @AccountID (for linking) | Address.ID, AddressLink.ID |
| **import_shipping_information** | Add shipping info to order (single shipment) | @THID or @OrderNumber, @ShipToContactID, @ShipToStreetAddress1, @ShipToCity, @ShipToState, @ShipToPostalCode | Shipment.ID |
| **import_shipping_information_detail** | Add shipping info with multiple shipments | @THID (header), @TDID (per line item) | Shipment.ID |
| **import_part** | Create new part | @PartName, @PartTypeID, @CategoryID or @CategoryName, @Description, @SKU, @CopyFromPartID (optional) | Part.ID |
| **import_part_category** | Create part category | @CategoryName, @CategoryParentName or @CategoryParentID, @CopyDefaultsFromParentCategory | PartCategory.ID |
| **insert_part_on_a_line_item** | Add part to existing line item | @TDID, @PartID, @Quantity, @RefreshOnSave, @RecomputeOnSave | TransPart.ID |
| **convert_estimate_to_order** | Convert estimate to order | @EstimateTHID or @EstimateNumber, @NewDueDate, @IsServiceTicket | OrderNumber, TransHeader.ID |
| **add_note** | Add note to company/order/estimate | @OrderNumber or @EstimateNumber or @AccountID or @CompanyName, @Description, @Notes, @ContactID, @EmployeeID | Activity.ID |
| **customer_list_import** | Bulk import customers from CSV | @TableName (pre-imported CSV data) | Count of imported/updated records |
| **quickbooks_import** | Import from QuickBooks IIF file | @ImportPath, @ImportFile, @OrderStatusID, @IsPriceLocked | Table of created records |
| **import_text_file_to_cols** | Utility to parse delimited files | @FileName, @Delimiter | Table with columns Col01-Col30 |

**Note:** Full T-SQL source code for these procedures is available in `output/wiki/reference/stored_procedures/`. This catalog provides parameter signatures only.

---

## 5. Operation Patterns

### INSERT Pattern

```sql
-- 1. Request new ID(s)
DECLARE @AccountID INT = dbo.[SQLBridge.Chapi.NextID](2000, 1);  -- Account ClassTypeID

-- 2. Begin transaction
BEGIN TRANSACTION;

-- 3. Insert record with new ID
INSERT INTO Account (ID, CompanyName, IsActive, SeqID)
VALUES (@AccountID, 'New Company', 1, 1);

-- 4. Commit transaction
COMMIT TRANSACTION;

-- 5. Notify Control to refresh
SELECT dbo.[SQLBridge.Chapi.Refresh](@AccountID, 2000, -1);
```

### UPDATE Pattern

```sql
-- 1. Lock the record
DECLARE @LockSuccess BIT = dbo.[SQLBridge.Chapi.Lock](@AccountID, 2000);
IF @LockSuccess = 0
BEGIN
    RAISERROR('Record locked by another user', 16, 1);
    RETURN;
END

-- 2. Begin transaction
BEGIN TRANSACTION;

-- 3. Update record (increment SeqID)
UPDATE Account
SET CompanyName = 'Updated Company',
    SeqID = SeqID + 1
WHERE ID = @AccountID;

-- 4. Commit transaction
COMMIT TRANSACTION;

-- 5. Unlock the record
SELECT dbo.[SQLBridge.Chapi.Unlock](@AccountID, 2000);

-- 6. Notify Control to refresh
SELECT dbo.[SQLBridge.Chapi.Refresh](@AccountID, 2000, -1);
```

### DELETE Pattern

```sql
-- 1. Begin transaction
BEGIN TRANSACTION;

-- 2. Delete record
DELETE FROM Account WHERE ID = @AccountID;

-- 3. Commit transaction
COMMIT TRANSACTION;

-- 4. Notify Control to refresh (ID may no longer exist, but Control handles this)
SELECT dbo.[SQLBridge.Chapi.Refresh](@AccountID, 2000, -1);
```

### Complex Example: Create Order with Line Items

```sql
-- 1. Request all needed IDs
DECLARE @OrderID INT = dbo.[SQLBridge.Chapi.NextID](10000, 1);  -- TransHeader
DECLARE @OrderNumber INT = dbo.[SQLBridge.Chapi.NextNumber]('OrderNumber', 1);
DECLARE @LineItemID INT = dbo.[SQLBridge.Chapi.NextID](10100, 2);  -- 2 line items

-- 2. Begin transaction
BEGIN TRANSACTION;

-- 3. Insert order header
INSERT INTO TransHeader (ID, OrderNumber, AccountID, TransactionType, StatusID, IsActive, SeqID)
VALUES (@OrderID, @OrderNumber, @AccountID, 1, 1, 1, 1);

-- 4. Insert line items
INSERT INTO TransDetail (ID, ParentID, ProductID, Quantity, IsActive, SeqID)
VALUES (@LineItemID, @OrderID, @ProductID1, 100, 1, 1);

INSERT INTO TransDetail (ID, ParentID, ProductID, Quantity, IsActive, SeqID)
VALUES (@LineItemID + 1, @OrderID, @ProductID2, 50, 1, 1);

-- 5. Commit transaction
COMMIT TRANSACTION;

-- 6. Refresh and recompute
SELECT dbo.[SQLBridge.Chapi.Refresh](@OrderID, 10000, -1);
SELECT dbo.[SQLBridge.Chapi.Recompute](@OrderID, 10000, 'Order created via import');
```

---

## 6. Safety Rules

**Critical rules for all write operations:**

1. **Always use CHAPI/SQL Bridge** - Never perform direct SQL writes without SQL Bridge functions
2. **Always use NextID()** - Never use IDENTITY columns or manually assigned IDs
3. **Always Lock before modify** - Call Lock() before UPDATE, check return value
4. **Always Unlock after modify** - Call Unlock() even if UPDATE fails
5. **Always Refresh after modify** - Call Refresh() after INSERT/UPDATE/DELETE
6. **Always increment SeqID** - Increment SeqID on every UPDATE
7. **Always use transactions** - Wrap multi-record operations in BEGIN/COMMIT TRANSACTION
8. **Always handle lock failures** - Check Lock() return value, gracefully handle 0 (locked)
9. **Always use -1 for UserID** - Use -1 for system operations, real Employee.ID when possible
10. **Always Recompute after pricing changes** - Call Recompute() when line items/parts change

**Failure modes:**
- **No NextID():** Control may assign same ID, causing key violations and data loss
- **No Lock():** Concurrent edits may overwrite each other
- **No Unlock():** Record remains locked indefinitely, blocking all users
- **No Refresh():** Control displays stale data, may overwrite changes on next edit
- **No SeqID increment:** Control thinks record hasn't changed, ignores refresh
- **No transaction:** Partial writes on error, database inconsistency
- **No Recompute():** Totals, taxes, and derived fields incorrect

---

## 7. Source Files

This reference was compiled from the following wiki and extract files:

**Primary sources:**
- `output/wiki/reference/chapi/chapi_url_endpoint_listener.md` - HTTP endpoint documentation
- `output/wiki/reference/chapi/chapi.md` - CHAPI overview
- `output/wiki/reference/sql_bridge/sql_bridge.md` - SQL Bridge core functions
- `output/wiki/extracts/database_integration_knowledge.md` - Architecture synthesis

**Secondary sources (stored procedures):**
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_company.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_contact.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_order.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_estimate.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_line_item.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_payment.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_address.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_shipping_information.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_shipping_information_detail.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_part.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_part_category.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_insert_part_on_a_line_item.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_convert_estimate_to_order.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_add_note_to_company_order_or_estimate.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_customer_list_import.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_quickbooks_import.md`
- `output/wiki/reference/stored_procedures/sql_stored_procedure_-_import_text_file_to_cols.md`

---

*Reference created: 2026-02-08*
*For Control ERP Database Mapping Project - Phase 3: Wiki Knowledge Formalization*
