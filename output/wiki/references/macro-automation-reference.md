# Macro Automation System Quick Reference

Fast-lookup reference for Control ERP's macro automation engine.

---

## Overview

Macros are Control's primary automation tool for workflows, notifications, status updates, and scheduled tasks. The system consists of:

- **RuleMacro** - Macro definition (ClassTypeID 23000)
- **RuleAction** - Actions executed when macro fires (ClassTypeIDs 231XX range)
- **RuleActivity** - Scheduled macro instances on Activity Manager

**Access:** Sales & Marketing > Macro Setup

---

## Macro Categories (14 Categories)

| Category | Entity Type | Primary Table | Notes |
|----------|-------------|---------------|-------|
| Order Macros | Orders (Type 1) | TransHeader | Status/station changes, payments |
| Estimate Macros | Estimates (Type 2) | TransHeader | Follow-up, conversion tracking |
| Company Macros | Accounts | Account | Stage changes, credit holds |
| Contact Macros | Contacts | AccountContact | Contact notifications |
| Employee Macros | Employees | Employee | HR notifications |
| Bill Macros | Bills (Type 8) | TransHeader | AP workflow |
| Purchase Order Macros | POs (Type 7) | TransHeader | PO receipts |
| Service Ticket Macros | Service Tickets (Type 6) | TransHeader | Service workflow |
| Line Item Macros | Line Items | TransDetail | Item-level station changes |
| Part Macros | Parts | Part | Inventory alerts |
| Receivables Macros | A/R | TransHeader | Past-due notifications |
| Receiving Document Macros | Receiving Docs | TransHeader | Material receipt |
| Recurring Order Macros | Recurring Orders | TransHeader | Recurring billing |
| Work Assignment Macros | Work Assignments | Journal | Task completion |

---

## RuleAction Types (10 Types)

| ClassTypeID | Action Type | Description |
|-------------|------------|-------------|
| **23110** | Contact Activity Action | Create contact activity (call, meeting, task) |
| **23120** | Email Action | Send email with merge fields |
| **23130** | Report Action | Run/print/email Crystal or SQL report |
| **23141** | UDF Action | Set/update User Defined Field value |
| **23144** | Popup Action | Display popup message to user |
| **23146** | Transaction Action | Change order status, station, properties |
| **23148** | Another Macro Action | Chain/execute another macro |
| **23150** | Company Activity Action | Create company-level activity |
| **23160** | Service Contract Action | Create/modify service contracts |
| **23170** | Work Assignment Action | Create work assignment |
| **23180** | Transaction Update Action | Update transaction fields |

---

## Macro Message Types (95 Types, IDs 0-94)

### General Events (0-4)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 0 | Undefined | No | |
| 1 | TierObjNew | No | |
| 2 | TierObjEdit | No | |
| 3 | TierObjDelete | No | |
| 4 | TierObjVoid | No | |

### Order Events (5-13)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 5 | OrderNew | No | |
| 6 | OrderEdit | No | |
| 7 | OrderStatusChange | No | |
| 8 | OrderStationChange | No | |
| 9 | OrderPayment | No | |
| 10 | OrderVoid | No | |
| 11 | OrderBuilt | No | |
| 12 | OrderSale | No | |
| 13 | OrderClosed | No | |

### Estimate Events (14-20)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 14 | EstimateNew | No | |
| 15 | EstimateEdit | No | |
| 16 | EstimateStatusChange | No | |
| 17 | EstimateStationChange | Yes | Line Item ID |
| 18 | EstimateVoid | No | |
| 19 | EstimateConverted | No | |
| 20 | EstimateLost | No | |

### Company Events (21-30)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 21 | CompanyNew | No | |
| 22 | CompanyEdit | No | |
| 23 | CompanyDelete | No | |
| 24 | CompanyPayment | No | |
| 25 | CompanyCreditChange | No | |
| 26 | CompanyFirstOrder | No | |
| 27 | CompanyNewOrder | Yes | New TransHeaderID |
| 28 | CompanyNewEstimate | Yes | New TransHeaderID |
| 29 | ContactNew | No | |
| 30 | ContactDelete | No | |

### Payment, User, SMS Events (31-35)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 31 | PaymentNew | No | |
| 32 | PaymentVoid | No | |
| 33 | UserNew | No | |
| 34 | UserEdit | No | |
| 35 | SMSCommand | No | |

### Employee Events (36-38)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 36 | EmployeeNew | No | |
| 37 | EmployeeEdit | No | |
| 38 | EmployeeDelete | No | |

### Service Ticket Events (39-48)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 39 | ServiceTicketNew | No | |
| 40 | ServiceTicketEdit | No | |
| 41 | ServiceTicketStatusChange | No | |
| 42 | ServiceTicketStationChange | No | |
| 43 | ServiceTicketPayment | No | |
| 44 | ServiceTicketVoid | No | |
| 45 | ServiceTicketBuilt | No | |
| 46 | ServiceTicketSale | No | |
| 47 | ServiceTicketClosed | No | |
| 48 | CompanyNewServiceTicket | Yes | New TransHeaderID |

### Line Item, Inventory, Events (49-52)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 49 | LineItemStationChange | No | |
| 50 | InventoryYellow | No | |
| 51 | InventoryRed | No | |
| 52 | LineItemNew | No | |

### Training & Work Assignment Events (53-58)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 53 | CourseEventNew | No | |
| 54 | CourseEventComplete | No | |
| 55 | CourseEventNewLink | No | |
| 56 | WorkAssignmentNew | No | |
| 57 | WorkAssignmentComplete | No | |
| 58 | OrderDropped | No | |

### GL Account Event (59)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 59 | GLAccountChange | No | |

### Status Change To/From Events (60-73)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 60 | OrderStatusToChange | Yes | New Status ID |
| 61 | OrderStatusFromChange | Yes | Old Status ID |
| 62 | EstimateStatusToChange | Yes | New Status ID |
| 63 | EstimateStatusFromChange | Yes | Old Status ID |
| 64 | ServiceTicketStatusToChange | Yes | New Status ID |
| 65 | ServiceTicketStatusFromChange | Yes | Old Status ID |
| 66 | OrderStationToChange | Yes | New Station ID |
| 67 | OrderStationFromChange | Yes | Old Station ID |
| 68 | EstimateStationToChange | Yes | New Station ID |
| 69 | EstimateStationFromChange | Yes | Old Station ID |
| 70 | ServiceTicketStationToChange | Yes | New Station ID |
| 71 | ServiceTicketStationFromChange | Yes | Old Station ID |
| 72 | LineItemStationToChange | Yes | New Station ID |
| 73 | LineItemStationFromChange | Yes | Old Station ID |

### Purchase Order Events (74-78)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 74 | PurchaseOrderNew | No | |
| 75 | PurchaseOrderEdit | No | |
| 76 | PurchaseOrderStatusChange | No | |
| 77 | PurchaseOrderStatusToChange | Yes | New Status ID |
| 78 | PurchaseOrderStatusFromChange | Yes | Old Status ID |

### Bill Events (79-83)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 79 | BillNew | No | |
| 80 | BillEdit | No | |
| 81 | BillStatusChange | No | |
| 82 | BillStatusToChange | Yes | New Status ID |
| 83 | BillStatusFromChange | Yes | Old Status ID |

### Receiving Document Events (84-88)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 84 | ReceivingDocNew | No | |
| 85 | ReceivingDocEdit | No | |
| 86 | ReceivingDocStatusChange | No | |
| 87 | ReceivingDocStatusToChange | No | |
| 88 | ReceivingDocStatusFromChange | No | |

### Account Stage Events (89-91)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 89 | AccountStageChange | No | |
| 90 | AccountStageToChange | Yes | New Stage ID |
| 91 | AccountStageFromChange | Yes | Old Stage ID |

### Shipment Events (92-94)

| ID | Name | NeedsAltID | AltID Description |
|----|------|------------|-------------------|
| 92 | ShipmentShipped | Yes | Shipment ID |
| 93 | ShipmentCreated | Yes | Shipment ID |
| 94 | ShipmentEdited | Yes | Shipment ID |

---

## Trigger Event Categories

### Order Triggers (13 event types)

- OrderNew, OrderEdit, OrderStatusChange, OrderStationChange
- OrderStatusToChange (To: WIP, Built, Sale, Closed, Voided)
- OrderStatusFromChange (From: WIP, Built, Sale, Closed, Voided)
- OrderStationToChange, OrderStationFromChange
- OrderPayment, OrderVoid, OrderBuilt, OrderSale, OrderClosed, OrderDropped

### Estimate Triggers (9 event types)

- EstimateNew, EstimateEdit, EstimateStatusChange, EstimateStationChange
- EstimateStatusToChange (To: Pending, Converted, Lost, Voided)
- EstimateStatusFromChange (From: Pending, Converted, Lost, Voided)
- EstimateStationToChange, EstimateStationFromChange
- EstimateVoid, EstimateConverted, EstimateLost

### Company Triggers (11 event types)

- CompanyNew, CompanyEdit, CompanyDelete
- CompanyPayment, CompanyCreditChange
- CompanyFirstOrder, CompanyNewOrder, CompanyNewEstimate, CompanyNewServiceTicket
- AccountStageChange, AccountStageToChange, AccountStageFromChange

### Contact Triggers (2 event types)

- ContactNew, ContactDelete

### Service Ticket Triggers (13 event types)

- Same structure as Order triggers (New, Edit, Status Change To/From, Station Change To/From, Payment, Void, Built, Sale, Closed)

### Line Item Triggers (4 event types)

- LineItemNew, LineItemStationChange
- LineItemStationToChange, LineItemStationFromChange

### Purchase Order Triggers (5 event types)

- PurchaseOrderNew, PurchaseOrderEdit, PurchaseOrderStatusChange
- PurchaseOrderStatusToChange, PurchaseOrderStatusFromChange

### Bill Triggers (5 event types)

- BillNew, BillEdit, BillStatusChange
- BillStatusToChange, BillStatusFromChange

### Receiving Document Triggers (5 event types)

- ReceivingDocNew, ReceivingDocEdit, ReceivingDocStatusChange
- ReceivingDocStatusToChange, ReceivingDocStatusFromChange

### Other Triggers

- EmployeeNew, EmployeeEdit, EmployeeDelete
- UserNew, UserEdit
- PaymentNew, PaymentVoid
- InventoryYellow, InventoryRed
- WorkAssignmentNew, WorkAssignmentComplete
- CourseEventNew, CourseEventComplete, CourseEventNewLink
- GLAccountChange
- SMSCommand
- ShipmentCreated, ShipmentEdited, ShipmentShipped

---

## Database Tables

### RuleMacro Table

**ClassTypeID:** 23000 (Macro)

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| ID | int | Primary key |
| ClassTypeID | int | Always 23000 |
| RuleName | nvarchar(35) | Macro name (display name) |
| Description | text | Long description |
| CategoryType | int | Macro category (Order, Estimate, Company, etc.) |
| IsActive | bit | Macro enabled/disabled |
| IsPrivate | bit | Only visible to creator |
| IsSystem | bit | System macro (cannot delete) |
| CanRunManually | bit | Can be run from menu |
| RuleExecuteType | int | Execution mode (auto vs. manual) |
| RuleDataSource | int | Data source type (current entity, SQL, saved query) |
| AutoTriggerType | int | Type of automatic trigger |
| AutoTriggerTargetClassTypeID | int | ClassTypeID of trigger target |
| AutoTriggerTargetID | int | ID of trigger target (e.g., StationID) |
| FilterType | int | Filter condition type |
| FilterCondition | text | Filter SQL/condition |

### RuleAction Table

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| ID | int | Primary key |
| ClassTypeID | int | Action type (23110-23180 range) |
| RuleMacroID | int | FK to RuleMacro.ID |
| SortIndex | int | Execution order (lower = earlier) |
| ActionName | nvarchar(128) | Action display name |
| IsActive | bit | Action enabled/disabled |

**Action-specific columns vary by ClassTypeID.**

### RuleActivity Table

**ClassTypeID:** 9201 (Recurring Activity) or 9202 (Recurring Macro Activity)

**Key Columns:**

| Column | Type | Description |
|--------|------|-------------|
| ID | int | Primary key |
| ClassTypeID | int | 9201 or 9202 |
| RuleMacroID | int | FK to RuleMacro.ID (for macro activities) |
| Description | nvarchar(256) | Activity description |
| ActivityDateTime | datetime | Scheduled date/time |
| RecurrencePattern | int | Recurrence type (Daily, Weekly, Monthly) |
| RecurrenceInterval | int | Interval (e.g., every 3 days) |
| RecurrenceDayOfWeek | int | Day of week for weekly recurrence |
| RecurrenceDayOfMonth | int | Day of month for monthly recurrence |
| RecurrenceEndDate | datetime | When recurrence stops |
| IsCompleted | bit | Activity completed |
| CompletedDateTime | datetime | Completion timestamp |

---

## Macro Execution Modes

### Automatic (Background)

**Configuration:** "Run this macro automatically" checked, "Run on the server in the background" checked

**Behavior:**
- Executes on SSLIP (Server-Side Logic and Integration Processor)
- No user interaction possible (no popups)
- Uses System Setup email settings (not user settings)
- Runs even when no users are logged in

**Use Cases:**
- Nightly/daily reports
- Scheduled status updates
- Automated notifications
- Recurring tasks

### Automatic (Workstation)

**Configuration:** "Run this macro automatically" checked, "Run on the server in the background" unchecked

**Behavior:**
- Executes on user's local workstation when trigger fires
- Can show popups
- Uses user's email settings
- Requires user to be logged in

**Use Cases:**
- Validation popups on order save
- User notifications
- Interactive macros

### Manual

**Configuration:** "Run this macro manually or on a schedule" checked, "Allow this macro to appear in the menu" checked

**Behavior:**
- Appears in "Run Macros" dropdown on entity toolbars
- Appears in Advanced Explorer toolbar
- User explicitly chooses when to run

**Use Cases:**
- On-demand reports
- Bulk updates
- Ad-hoc tasks

### Scheduled

**Configuration:** Created via Activity Manager > New > Macro Activity

**Behavior:**
- Runs at specific date/time
- Can recur (daily, weekly, monthly)
- Can run on server (background) or local (if logged in)

**Use Cases:**
- Daily sales report at 8 AM
- Weekly inventory alerts on Mondays
- Monthly commission calculations

---

## Data Sources (4 Types)

1. **Use the current [entity]** - Single record that triggered the event (uses `<%TriggerID%>` placeholder in SQL)
2. **Use a SQL Query** - Custom SQL returning ID and ClassTypeID columns
3. **Use a Saved Query** - Pre-saved Advanced Explorer query
4. **An Indirect Change Event** - SQL code in "Indirect Change Event" formula box (most powerful)

**`<%TriggerID%>` Placeholder:**
Runtime replacement with ID of triggering record.

**Example:**
```sql
SELECT TH.ID, TH.ClassTypeID
FROM TransHeader TH
WHERE TH.ID = <%TriggerID%>
  AND TH.StatusID = 3  -- Sale only
```

---

## Common Action Configurations

### Email Action (23120)

**Key Settings:**
- **To:** Order Contact, Salesperson, Billing Contact, Custom Address
- **Subject:** Supports merge fields (e.g., `<%OrderNumber%>`, `<%CompanyName%>`)
- **Message:** HTML body with merge fields
- **CC / BCC:** Additional recipients

**Merge Fields:** Dynamic placeholders replaced at runtime (e.g., `<%OrderNumber%>`, `<%StatusText%>`, `<%DueDate%>`)

### Report Action (23130)

**Print Options:**
- Quick Print - Direct to printer
- Quick Email - Email as attachment
- Quick Save - Save to Report Export folder
- Quick Execute - Run SQL without output (SQL reports only)
- Preview - Open for review

**Report Types:**
- System Report - Built-in Crystal templates
- Custom Report - User-created .rpt files
- SQL - Inline T-SQL code

**Print Report As Group:** Run once for all records vs. once per record

### UDF Action (23141)

**Operations:**
- **Set Value** - Update field to specific value
- **Set Only If Blank** - Update only if currently empty
- **Clear User Field** - Clear the field value

### Transaction Action (23146)

**Operations:**
- **Replace Station** - Change order/estimate station
- **Replace Status** - Change status (to Sale, Closed, etc.)
- **Save Record** - Save transaction after changes

### Chain Macro Action (23148)

**Purpose:** Execute another macro as sub-step

**Use Cases:**
- Group multiple daily macros into master macro
- Separate complex logic into manageable pieces

---

## Recurring Activity Notes

- First population: 25 minutes after SSLIP starts
- Subsequent population: Every 6 hours (default Activity Manager setting)
- **CRITICAL:** Only one SSLIP should handle background processes (set `DisableBackgroundProcesses=1` on others)
- Scheduled report macros require report file in SSLIP folder
- High-frequency recurring activities can degrade performance

---

## Email Delivery

### Two Methods

1. **Outlook** - Uses Microsoft Outlook (2000+)
2. **Internal** - Built-in SMTP (does NOT support SSL)

**Configuration:** Setup > System Setup > Email Options

**Per-User Override:** Setup > User Options (per employee)

**SSLIP Email:** Always uses System Setup email settings (not user settings)

### SMS Text Messaging

**Format:** `phonenumber@carriersuffix.extension`

**Example:** `2225558888@txt.att.net` (AT&T)

**Limits:** 140 characters

**Works with:** Any email macro action

---

## Source Files

**Primary source:**
- `output/wiki/extracts/macros_automation_knowledge.md` - Comprehensive macro system documentation

**Related references:**
- `output/wiki/reference/sql_bridge/sql_bridge.md` - RefreshEx() MacroMessageType usage
- `output/wiki/extracts/database_integration_knowledge.md` - ClassTypeID mapping

---

*Quick reference created: 2026-02-08*
*For Control ERP Database Mapping Project - Phase 3: Wiki Knowledge Formalization*
*Based on Cyrious Control wiki documentation and community knowledge*
