# Phase 12: Production Workflow - Research

**Researched:** 2026-02-09
**Domain:** Control ERP Production Workflow (Artwork Pipeline, Station Workload, Station Dwell Time)
**Confidence:** HIGH -- all findings based on live database queries against FLS StoreData

## Summary

This research investigated the three pillars of Phase 12: (1) artwork approval pipeline, (2) station workload monitoring, and (3) station dwell time analysis. All queries were executed against the live FLS StoreData database.

Key discoveries:
- The artwork pipeline uses 11 status values, but FLS actively uses only 7 (never In Review, In PrePress, or In Production). StatusID=8 ("Unknown") accounts for 38% of all active groups and represents artwork on orders that advanced to Sale/Closed without going through the formal artwork workflow.
- FLS has 98 stations (67 active) organized into a clear numbered-stage hierarchy (0-Design through 6-Shipping), with a confirmed dual order-level and line-item-level model.
- Station dwell time is calculated from consecutive Journal entries (JournalActivityType=45), which record point-in-time "Station Changed" events with 1.06M total records. EndDateTime is always NULL -- dwell is the delta between consecutive StartDateTime values for the same TransactionID+LinkID.
- The average artwork approval turnaround (creation to approval) is ~54 hours (~2.25 business days) for 2,978 groups since 2025.

**Primary recommendation:** Build the production skill around three query families: (a) ArtworkGroup status aggregation with joins to TransHeader/TransDetail, (b) TransHeader.StationID + TransDetail.StationID for WIP workload, and (c) Journal LAG/LEAD window functions for dwell time calculation.

---

## Research Area 1: Artwork Tables Structure

### Confidence: HIGH (live database queries)

### _ArtworkStatus Lookup Table (11 rows)

| ID | Text | FLS Active Groups (IsActive=1) | Notes |
|----|------|-------------------------------|-------|
| 0 | Undefined | 0 | Never used |
| 1 | In Design | 4,965 | Initial status; designer working |
| 2 | In Review | 0 | **Never used at FLS** |
| 3 | Pending Approval | 902 | Sent to customer |
| 4 | Approved | 1,362 | Customer approved |
| 5 | In PrePress | 0 | **Never used at FLS** |
| 6 | In Production | 0 | **Never used at FLS** |
| 7 | Produced | 36,683 | Final terminal status |
| 8 | Unknown | 25,274 | See gotcha below |
| 9 | Voided | 1,677 | Cancelled artwork |
| 10 | Rejected | 110 | Customer rejected |

**Total active ArtworkGroups: 70,973** (out of 84,770 total including inactive)

**GOTCHA - StatusID=8 ("Unknown")**: This is NOT an error state. 25,274 active groups have this status. Investigation reveals:
- 96% (5,567 of 5,790 recent) are on orders with StatusID=4 (Closed)
- 4% (223) are on orders with StatusID=3 (Sale)
- These are artwork groups on orders that were advanced to Sale/Closed **without going through the formal artwork approval workflow**. The artwork status was never explicitly set.
- For pipeline queries (pending/stuck items), **filter out StatusID IN (7, 8, 9)** to show only actionable artwork.

### Artwork Status Transitions (2025+ data, top 10)

| From | To | Count | Notes |
|------|----|-------|-------|
| In Design | Unknown | 6,213 | Order sold/closed without artwork workflow |
| In Design | Pending Approval | 4,103 | Normal flow: designer sends for approval |
| Approved | Produced | 3,053 | Normal flow: approved artwork produced |
| Pending Approval | Approved | 3,018 | Normal flow: customer approves |
| Pending Approval | In Design | 896 | **Revision request** (change requested) |
| In Design | Approved | 155 | Skipped Pending Approval step |
| In Design | Voided | 116 | Artwork cancelled during design |
| Approved | In Design | 114 | Sent back after approval |
| Pending Approval | Unknown | 104 | Order advanced while pending |
| Pending Approval | Voided | 46 | Cancelled while pending |

**Key insight:** The revision rate is significant -- 896 groups sent back from Pending Approval to In Design (22% of approval submissions result in revision).

### Artwork Approval Turnaround Time

- **Average: 54 hours** (~2.25 business days) for groups where ArtworkApprovalDT > GroupCreatedDT and < 90 days
- **Sample size:** 2,978 approved groups since 2025-01-01
- **Gotcha:** ArtworkApprovalDT can carry forward from cloned/converted orders, resulting in ArtworkApprovalDT < GroupCreatedDT. Always filter `ArtworkApprovalDT > GroupCreatedDT` when calculating turnaround.
- **Revision count:** 618 groups had at least 1 revision (Pending Approval -> In Design) since 2025

### Other Artwork Lookup Tables

**_ArtworkBand (5 values):**
| ID | Text |
|----|------|
| 0 | Undefined |
| 1 | On Hold |
| 2 | In Queue |
| 3 | In Process |
| 4 | W4 Release |

**_ArtworkPriority (6 values):**
| ID | Text |
|----|------|
| 0 | Undefined |
| 1 | Low |
| 2 | Normal |
| 3 | High |
| 4 | Rush |
| 5 | Urgent |

**_ArtworkCollectionType (3 values):**
| ID | Text |
|----|------|
| 0 | Undefined |
| 1 | Collection |
| 2 | ChoiceGroup |

**_ArtworkCommentType (5 values):**
| ID | Text |
|----|------|
| 0 | Undefined |
| 1 | Text |
| 2 | Pin |
| 3 | Box |
| 4 | Line |

**_ArtworkRole (8 values):**
| ID | Text |
|----|------|
| 0 | Undefined |
| 1 | Designer |
| 2 | Reviewer |
| 3 | PrePress Designer |
| 4 | Subscriber |
| 5 | Adhoc |
| 9 | Approver |
| 10 | Commenter |

**_ArtworkLogType (16 values):**
| ID | Text |
|----|------|
| 0 | Unknown |
| 1 | GroupCreated |
| 2 | GroupInactive |
| 3 | ArtworkFileAdded |
| 4 | ArtworkFileDeleted |
| 5 | CommentChange |
| 6 | Monitoring |
| 7 | StatusChange |
| 8 | OptionChange |
| 9 | Action |
| 10 | CustomerActivity |
| 11 | Debug |
| 12-15 | Info/Warn/Error/Fatal |

### Artwork Table Relationship Map

```
TransHeader (order)
  |
  +--< ArtworkGroup (1:many, via TransHeaderID)
  |     |  Fields: GroupName, StatusID, GroupCreatedDT, StatusDT, ArtworkApprovalDT, ArtworkDueDT
  |     |  FK: AccountID -> Account, StatusID -> _ArtworkStatus, BandID -> _ArtworkBand
  |     |  FK: PriorityID -> _ArtworkPriority, CollectionTypeID -> _ArtworkCollectionType
  |     |
  |     +--< ArtworkGroupTransDetailLink (1:many, via ArtworkGroupID)
  |     |     Links artwork group to specific line items (TransDetail)
  |     |
  |     +--< ArtworkGroupStatusHistory (1:many, via ArtworkGroupID)
  |     |     Audit trail: FromStatusID, ToStatusID, StatusDT, PlayerID
  |     |     164,226 rows total
  |     |
  |     +--< ArtworkItem (1:many, via GroupID)
  |     |     Individual artwork files within a group
  |     |     Fields: ItemIndex, CurrentVersion, OriginalFileName
  |     |     41,797 rows
  |     |     |
  |     |     +--< ArtworkProofFile (1:many, via ArtworkItemID)
  |     |     |     Version tracking for proof files
  |     |     |     Fields: Version, IsLatestVersion, VersionFileName, FileAddedDT
  |     |     |     41,741 rows
  |     |     |
  |     |     +--< ArtworkComment (1:many, via ArtworkItemID)
  |     |           Comments from players
  |     |           Fields: CommentTypeID, CommentDT, CommentText, PlayerID
  |     |           4,109 rows
  |     |
  |     +--< ArtworkLog (1:many, via ArtworkGroupID)
  |     |     Activity log: LogTypeID -> _ArtworkLogType
  |     |     208,887 rows
  |     |
  |     +--< ArtworkNotificationHistory (1:many, via GroupID)
  |           Email notification records
  |           156,832 rows
  |
  +--< ArtworkPlayer (1:many, via TransHeaderID)
        Named roles on an order
        Fields: EmployeeID, ContactID, PersonID, PersonClassTypeID, URLKey
        176,445 rows
        |
        +--< ArtworkPlayerRoleLink (1:many, via PlayerID)
              Links player to role: RoleID -> _ArtworkRole
              174,373 rows
```

### Verified Join Paths (artwork to order/line-item)

```sql
-- Artwork group to order
ArtworkGroup.TransHeaderID -> TransHeader.ID

-- Artwork group to line items (many-to-many via link table)
ArtworkGroup.ID -> ArtworkGroupTransDetailLink.ArtworkGroupID
ArtworkGroupTransDetailLink.TransDetailID -> TransDetail.ID

-- Artwork player (designer/approver) identification
ArtworkPlayer.TransHeaderID -> TransHeader.ID
ArtworkPlayer.EmployeeID -> Employee.ID
ArtworkPlayer.ContactID -> AccountContact.ID
ArtworkPlayerRoleLink.PlayerID -> ArtworkPlayer.ID
ArtworkPlayerRoleLink.RoleID -> _ArtworkRole.ID
```

### Supporting Tables (low priority for Phase 12)

| Table | Rows | Purpose | Phase 12 Relevance |
|-------|------|---------|-------------------|
| ArtworkSchema | 19 | Schema versioning | None (internal) |
| ArtworkDefaultPlayer | 11 | Default designer/approver per account | Low (config only) |
| ArtworkDefaultRoleLink | 11 | Default role assignments | Low (config only) |
| ArtworkDigestEntry | 0 | Email digest queue | None (empty) |
| ArtworkNotificationHistory | 156,832 | Email delivery tracking | Low (notification audit) |
| ArtworkLog | 208,887 | Detailed activity log | Medium (deeper audit trail) |

---

## Research Area 2: Station Hierarchy

### Confidence: HIGH (live database queries, complete enumeration)

### Station Counts
- **Total stations:** 98
- **Active stations:** 67
- **Inactive stations:** 31

### FLS Station Hierarchy (Numbered-Stage System)

FLS uses a numbered prefix system to indicate production stages. The key production workflow is:

#### Stage 0: Design (Department ID: 1016)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 1016 | 0-Design | Yes | -- | -- | -- | *Department parent*
| 10004 | 0-Design-Prefilght | Yes | WIP, Est | -- | -- |
| 10002 | 0-Design-Need Art | Yes | WIP, Est | -- | -- |
| 10003 | 0-Design-Request Art | Yes | WIP, Est | -- | -- |
| 1020 | 0-Design-Proof | Yes | WIP, Est | LI (all) | Yes |
| 1012 | 0-Design-Wait for Art | Yes | WIP, Est | LI (all) | -- |
| 1069 | 0-Design-Complete | Yes | WIP | LI (all) | -- |

#### Stage 1: Approved (Department ID: 1024)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 1024 | 1-Approved | Yes | WIP, Est | LI (all) | -- | *Both order and LI*

#### Stage 1.1: Pre-Production (Department ID: 10008)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 10008 | 1.1 Pre-Production | Yes | -- | LI (all) | Yes | *LI only*

#### Stage 2: Production (Department ID: 1025)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 1025 | 2-Production | Yes | WIP | -- | -- | *Department parent, order-level*
| 1027 | 2-Production-Digital | Yes | -- | LI (all) | Yes |
| 1073 | 2-Production-West Side | Yes | -- | LI (all) | Yes |
| 1028 | 2-Production-SP | Yes | -- | LI (all) | Yes |
| 10001 | 2-Production-DTG | Yes | -- | LI (all) | Yes |
| 1030 | 2-Production-Garment | Yes | -- | LI (all) | Yes |
| 1031 | 2-Production-Embroidery | Yes | -- | LI (all) | Yes |
| 1029 | 2-Production-Contract | Yes | WIP | LI (all) | -- |
| 1071 | 2-Production-Press Ons | Yes | -- | LI (all) | Yes |

#### Stage 2.1: Print In Progress (Department ID: 10007)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 10007 | 2.1-Print In Progess | Yes | -- | LI (all) | Yes |

#### Stage 3: Transfer (Department ID: 1068)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 1068 | 3-Transfer | Yes | -- | LI (all) | Yes |

#### Stage 4: Cutting (Department ID: 1043)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 1043 | 4-Cutting | Yes | -- | LI (all) | Yes |

#### Stage 5: Finishing (Department ID: 1032)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 1032 | 5-Finishing | Yes | -- | LI (all) | Yes |

#### Stage 6: Shipping (Department ID: 1033)
| ID | StationName | Active | OrderLevel | LineItemLevel | TimeClock |
|----|-------------|--------|------------|--------------|-----------|
| 1033 | 6-Shipping | Yes | WIP | LI (all) | Yes | *Both order and LI*

#### Non-Numbered Active Stations (Workflow/Administrative)

**Hold/Status stations (order-level):**
| ID | StationName | Shows For |
|----|-------------|-----------|
| 1014 | On-Hold | WIP, Estimates |
| 1034 | On-Hold- Artwork | WIP, Estimates |
| 1035 | On-Hold- Payment | WIP, Estimates |
| 1023 | Out For Approval | WIP, Estimates |

**Estimate tracking stations:**
| ID | StationName |
|----|-------------|
| 1010 | Estimate Sent |
| 1061 | Estimate Sent-Pending |
| 10005 | Estimate Sent-Call |
| 1011 | Verbal - Phone |
| 1015 | Converted |

**Follow-up stations:**
| ID | StationName |
|----|-------------|
| 1013 | Follow-up (department) |
| 1057 | Follow-up-24hr |
| 1058 | Follow-up- 72 |
| 1060 | Follow-up-Reminder |
| 1067 | Follow-up-Out for Approval |

**Post-production stations:**
| ID | StationName | Shows For |
|----|-------------|-----------|
| 1049 | Waiting For Pick-up | Sale, LI |
| 1050 | Waiting For Shipment | Sale, LI |
| 1036 | Picked-Up | Sale, LI (MarkLIComplete=true) |
| 1037 | Shipped | Sale |
| 1038 | Delivered | Sale |
| 1042 | Built | LI |

**Time/Payroll stations (not production-related):**
| ID | StationName |
|----|-------------|
| 200 | Time (department) |
| 201 | Time Off (department) |
| 202, 1002, 1004 | Time-Other, Time-Installation, Time-Maintenance |
| 1001, 1005-1008 | Time Off-Doctor/Personal/Sick/Break/Vacation |

**PO stations (purchase order lifecycle):**
| ID | StationName |
|----|-------------|
| 100 | Requested |
| 102 | Ordered |
| 103 | Bill Received |
| 104 | Partial Billed |
| 105 | Closed |
| 106 | Cancelled |
| 107 | Rejected |
| 108 | Received |
| 109 | Partial Received |
| 110 | Backordered |

### Station Classification Logic

Determining order-level vs line-item-level for each station:
- **Order-level**: `ShowForWIP = 1` or `ShowForBuilt = 1` or `ShowForSale = 1` or `ShowForPendingEstimates = 1`
- **Line-item-level**: `ShowForLineItem = 1` or `ShowForLineWIP = 1` or `ShowForLineBuilt = 1` or `ShowForLineSale = 1`
- **Dual-level (both)**: Station has BOTH order-level and line-item-level flags set (e.g., 1-Approved, 0-Design-Proof, 6-Shipping)
- **TimeClock stations**: `ShowOnTimeClock = 1` (employee clock stations, not production stations per se)
- **MarkLIComplete**: When set, moving LI to this station marks it complete (only "Picked-Up" has this)

---

## Research Area 3: Station Transition Tracking (Dwell Time)

### Confidence: HIGH (live database queries)

### How Station Transitions Are Recorded

Station changes are recorded as **Journal entries** with:
- `JournalActivityType = 45` (Station Changed)
- `Description = 'Station Changed'`
- **1,063,074 total records** with this activity type
- `Journal.StationID` = the **new** station (the station the item moved TO)
- `Journal.TransactionID` = TransHeader.ID (the order)
- `Journal.LinkID` = TransDetail.ID (the line item, NULL for order-level changes)
- `Journal.StartDateTime` = timestamp of the station change
- `Journal.EndDateTime` = **always NULL** (these are point-in-time events, not spans)
- `Journal.EmployeeID` = who made the change

### Dwell Time Calculation Method

Since EndDateTime is always NULL, dwell time at a station is calculated as:

```
Dwell Time = (next station change StartDateTime) - (current station change StartDateTime)
```

For the same TransactionID + LinkID combination:

```sql
-- Dwell time calculation using CROSS APPLY to find previous station
SELECT
    curr.TransactionID,
    th.OrderNumber,
    curr.LinkID,
    prev_s.StationName AS Station,
    prev.StartDateTime AS ArrivedAt,
    curr.StartDateTime AS LeftAt,
    DATEDIFF(minute, prev.StartDateTime, curr.StartDateTime) AS DwellMinutes
FROM Journal curr
CROSS APPLY (
    SELECT TOP 1 j2.StartDateTime, j2.StationID
    FROM Journal j2
    WHERE j2.JournalActivityType = 45
      AND j2.TransactionID = curr.TransactionID
      AND ISNULL(j2.LinkID, 0) = ISNULL(curr.LinkID, 0)
      AND j2.StartDateTime < curr.StartDateTime
    ORDER BY j2.StartDateTime DESC
) prev
LEFT JOIN Station prev_s ON prev.StationID = prev_s.ID
LEFT JOIN TransHeader th ON curr.TransactionID = th.ID
WHERE curr.JournalActivityType = 45
```

**Alternative approach using LAG() window function** (may be more efficient for aggregation):

```sql
SELECT
    TransactionID,
    LinkID,
    StationID,
    StartDateTime AS ArrivedAt,
    LEAD(StartDateTime) OVER (
        PARTITION BY TransactionID, ISNULL(LinkID, 0)
        ORDER BY StartDateTime
    ) AS LeftAt,
    DATEDIFF(minute, StartDateTime,
        LEAD(StartDateTime) OVER (
            PARTITION BY TransactionID, ISNULL(LinkID, 0)
            ORDER BY StartDateTime
        )
    ) AS DwellMinutes
FROM Journal
WHERE JournalActivityType = 45
```

### Sample Dwell Time Data (order-level, Feb 2026)

| Order# | From Station | To Station | Dwell (minutes) | Dwell (hours) |
|--------|-------------|------------|-----------------|---------------|
| 133526 | 2-Production | Waiting For Pick-up | 44,805 | 747h (~31 days) |
| 133769 | 2-Production | Waiting For Pick-up | 16,170 | 270h (~11 days) |
| 133816 | 2-Production | Waiting For Pick-up | 8,970 | 150h (~6 days) |
| 133855 | 6-Shipping | Shipped | 110 | 1.8h |
| 133794 | 6-Shipping | Shipped | 266 | 4.4h |
| 133837 | 6-Shipping | Shipped | 39 | 0.65h |

### Sample Dwell Time Data (line-item-level)

For order 133880 (TransactionID=233404), line item 567801:
1. 09:11 -> Station: 1-Approved
2. 16:04 -> Station: 1.1 Pre-Production
3. Dwell at "1-Approved" = 413 minutes (6.9 hours)

### Key Observations for Dwell Time Queries

1. **Order-level vs line-item-level transitions are tracked separately.** LinkID=NULL means order-level station change; LinkID=TransDetail.ID means line-item-level.
2. **The "current" dwell (items still at a station) has no departure time.** For current WIP, calculate dwell as `DATEDIFF(minute, last_station_change, GETDATE())`.
3. **Multiple line items change station simultaneously** when an order-level action triggers batch updates (e.g., order moved to "1-Approved" triggers all line items to "1-Approved" at the same timestamp).
4. **Journal volume is high** (1.06M records). Always filter by date range and/or TransactionID for performance.
5. **The first station assignment** when an order is created also generates a Journal entry. This is the "arrival" at the first station.

---

## Research Area 4: WIP Identification (Dual-Level Model)

### Confidence: HIGH (live database queries)

### Current WIP State (as of 2026-02-09)

**Order-level WIP (TransHeader.StatusID IN (1,2), TransactionType=1):** 67 orders

| Station | Order Count |
|---------|------------|
| 2-Production | 40 |
| 0-Design-Proof | 14 |
| Out For Approval | 6 |
| 1-Approved | 2 |
| On-Hold | 2 |
| 0-Design-Wait for Art | 1 |
| 6-Shipping | 1 |
| On-Hold- Payment | 1 |

**Line-item-level WIP (on WIP orders):** 116 line items

| Station | Line Item Count |
|---------|----------------|
| Waiting For Shipment | 42 |
| 0-Design-Proof | 37 |
| 2-Production-Embroidery | 12 |
| 2-Production-Garment | 11 |
| 2-Production-Press Ons | 5 |
| 2-Production-West Side | 3 |
| 1-Approved | 3 |
| 4-Cutting | 1 |
| 2-Production-Contract | 1 |
| 5-Finishing | 1 |

### Dual Station Model Verified

The order/line-item station duality is confirmed:

```
Order# 133892: Order Station = "1-Approved"
  Line Item 567845: LI Station = "Waiting For Shipment"  (different from order!)
  Line Item 567846: LI Station = "0-Design-Proof"       (different from order!)

Order# 133891: Order Station = "2-Production"
  Line Item 567841: LI Station = "Waiting For Shipment"  (line item ahead of order)

Order# 133890: Order Station = "0-Design-Proof"
  Line Item 567835: LI Station = "0-Design-Proof"        (same as order)
  Line Item 567836: LI Station = "0-Design-Proof"        (same as order)
```

**Key insight:** All WIP orders have a StationID set (no NULLs found). Line items can be at different stations than their parent order.

### WIP Query Patterns

**Order-level WIP by station:**
```sql
SELECT s.StationName, COUNT(*) AS OrderCount, SUM(th.SubTotalPrice) AS TotalValue
FROM TransHeader th
JOIN Station s ON th.StationID = s.ID
WHERE th.TransactionType = 1 AND th.StatusID IN (1, 2) AND th.IsActive = 1
GROUP BY s.StationName
ORDER BY COUNT(*) DESC
```

**Line-item-level WIP by station:**
```sql
SELECT s.StationName, COUNT(*) AS LineItemCount, SUM(td.MeAndSonsSubTotalPrice) AS TotalValue
FROM TransDetail td
JOIN Station s ON td.StationID = s.ID
JOIN TransHeader th ON td.TransHeaderID = th.ID
WHERE th.TransactionType = 1 AND th.StatusID IN (1, 2) AND th.IsActive = 1 AND td.IsActive = 1
GROUP BY s.StationName
ORDER BY COUNT(*) DESC
```

---

## Research Area 5: PrCA (Production Calendar) Tables

### Confidence: HIGH (live database queries)

The PrCA.* tables are a separate production scheduling feature:
- **PrCA.Board.Data**: 12 rows (production boards)
- **PrCA.Job.Data**: 383 rows (scheduled jobs)
- **PrCA.Job.TransactionLink**: 383 rows (links jobs to orders/line items)
- **PrCA.Board.StationLink**: Links boards to stations

**Assessment:** Low usage at FLS (383 jobs since 2018). These are NOT the primary source for production tracking. The Journal-based station tracking is the authoritative source. PrCA tables can be documented but are not essential for Phase 12 goals.

---

## Architecture Patterns

### Recommended Skill File Structure

```
skills/control-erp-production/
  control-erp-production-SKILL.md    # Main skill file
  references/
    artwork-pipeline.md              # PROD-01: artwork queries and status reference
    station-workload.md              # PROD-02: WIP workload queries
    station-dwell-time.md            # PROD-03: dwell time analysis queries
```

### Pattern: Status-Based Artwork Pipeline

```sql
-- Artwork pipeline summary (actionable items only)
SELECT
    s.Text AS ArtworkStatus,
    COUNT(*) AS GroupCount,
    COUNT(DISTINCT ag.TransHeaderID) AS OrderCount
FROM ArtworkGroup ag
JOIN _ArtworkStatus s ON ag.StatusID = s.ID
JOIN TransHeader th ON ag.TransHeaderID = th.ID
WHERE ag.IsActive = 1
  AND ag.StatusID NOT IN (7, 8, 9)  -- Exclude Produced, Unknown, Voided
  AND th.TransactionType = 1
  AND th.StatusID IN (0, 1, 2)      -- New, WIP, Built orders only
GROUP BY ag.StatusID, s.Text
ORDER BY ag.StatusID
```

### Pattern: Stuck Artwork Detection

Based on the 54-hour average approval turnaround, a reasonable "stuck" threshold is 3x average = ~7 days:

```sql
-- Stuck artwork: pending approval > 7 days
SELECT ag.ID, th.OrderNumber, a.CompanyName, ag.GroupCreatedDT, ag.StatusDT,
       DATEDIFF(day, ag.StatusDT, GETDATE()) AS DaysInCurrentStatus
FROM ArtworkGroup ag
JOIN TransHeader th ON ag.TransHeaderID = th.ID
JOIN Account a ON ag.AccountID = a.ID
WHERE ag.IsActive = 1
  AND ag.StatusID IN (1, 3)  -- In Design or Pending Approval
  AND DATEDIFF(day, ag.StatusDT, GETDATE()) > 7
ORDER BY ag.StatusDT ASC
```

### Pattern: Dwell Time Aggregation with LEAD()

```sql
-- Average dwell time per station for order-level changes
WITH StationTransitions AS (
    SELECT
        j.TransactionID,
        j.StationID,
        s.StationName,
        j.StartDateTime AS ArrivedAt,
        LEAD(j.StartDateTime) OVER (
            PARTITION BY j.TransactionID
            ORDER BY j.StartDateTime
        ) AS LeftAt
    FROM Journal j
    JOIN Station s ON j.StationID = s.ID
    WHERE j.JournalActivityType = 45
      AND j.LinkID IS NULL  -- order-level only
      AND j.StartDateTime > DATEADD(month, -3, GETDATE())
)
SELECT
    StationName,
    COUNT(*) AS Transitions,
    AVG(DATEDIFF(minute, ArrivedAt, LeftAt)) AS AvgDwellMinutes,
    AVG(DATEDIFF(minute, ArrivedAt, LeftAt)) / 60.0 AS AvgDwellHours
FROM StationTransitions
WHERE LeftAt IS NOT NULL  -- exclude items still at the station
GROUP BY StationName
HAVING COUNT(*) > 10
ORDER BY AVG(DATEDIFF(minute, ArrivedAt, LeftAt)) DESC
```

### Anti-Patterns to Avoid

- **Do NOT use TimeCard for per-job time tracking**: FLS does not track per-job time via TimeCard. TimeCard ClassTypeID 20050/20051 are for employee clock-in/clock-out, not production time.
- **Do NOT use SELECT * on Station table**: Contains geography columns that crash queries. Always specify columns explicitly.
- **Do NOT hardcode artwork status values**: Use the _ArtworkStatus lookup table.
- **Do NOT hardcode station names/IDs**: Use the Station table and classify by ShowFor* flags.
- **Do NOT treat StatusID=8 ("Unknown") as an error**: It's the natural state for artwork on completed orders.
- **Do NOT calculate artwork turnaround without filtering ArtworkApprovalDT > GroupCreatedDT**: Cloned orders carry stale approval dates.
- **Do NOT query Journal without date range filters**: 1.06M records in JournalActivityType=45 alone; always constrain.

---

## Common Pitfalls

### Pitfall 1: Geography Columns in Station Table
**What goes wrong:** `SELECT *` on Station table crashes with geography/spatial column errors.
**How to avoid:** Always specify columns explicitly when querying Station.

### Pitfall 2: Journal Table Volume (5.18M rows total, 1.06M station changes)
**What goes wrong:** Unfiltered Journal queries time out or return too much data.
**How to avoid:** Always filter by `JournalActivityType = 45` AND a date range. For dwell time, use a rolling 3-6 month window.

### Pitfall 3: Artwork ApprovalDT from Cloned Orders
**What goes wrong:** ArtworkApprovalDT can be earlier than GroupCreatedDT when the order was cloned from a previously approved order.
**How to avoid:** Always filter `ArtworkApprovalDT > GroupCreatedDT` when calculating turnaround times.

### Pitfall 4: "Unknown" Artwork Status Misinterpretation
**What goes wrong:** Treating StatusID=8 as a bug or including it in "pending" counts inflates pipeline numbers by 25K+ groups.
**How to avoid:** Exclude StatusID IN (7, 8, 9) for actionable pipeline views. Document that StatusID=8 means "order completed without formal artwork workflow."

### Pitfall 5: Mixed Order-Level and Line-Item-Level Station Data
**What goes wrong:** Combining order-level and line-item-level WIP counts leads to double-counting.
**How to avoid:** Query order-level (TransHeader.StationID) and line-item-level (TransDetail.StationID) separately. For dual-level stations, present them as distinct views. Use Journal.LinkID to distinguish: NULL = order-level, non-NULL = line-item-level.

### Pitfall 6: Dwell Time for Currently-Active Items
**What goes wrong:** The LEAD() window function returns NULL for items still at their current station (no next transition). This drops current WIP from dwell calculations.
**How to avoid:** For current WIP dwell, use `COALESCE(next_transition, GETDATE())` to calculate time since last station change.

---

## Code Examples

### PROD-01: Artwork Pipeline Summary

```sql
-- Full artwork pipeline with counts and order details
SELECT
    s.Text AS Status,
    COUNT(*) AS ArtworkGroups,
    COUNT(DISTINCT ag.TransHeaderID) AS UniqueOrders,
    AVG(DATEDIFF(day, ag.StatusDT, GETDATE())) AS AvgDaysInStatus
FROM ArtworkGroup ag
JOIN _ArtworkStatus s ON ag.StatusID = s.ID
JOIN TransHeader th ON ag.TransHeaderID = th.ID
WHERE ag.IsActive = 1
  AND th.TransactionType = 1
  AND th.StatusID IN (0, 1, 2)  -- New, WIP, Built
  AND ag.StatusID NOT IN (7, 8, 9)  -- Exclude terminal states
GROUP BY ag.StatusID, s.Text
ORDER BY ag.StatusID
```

### PROD-01: Artwork Drill-Down by Status

```sql
-- Detail for a specific artwork status (e.g., Pending Approval)
SELECT
    th.OrderNumber,
    a.CompanyName,
    ag.GroupName,
    ag.GroupCreatedDT,
    ag.StatusDT,
    ag.ArtworkDueDT,
    DATEDIFF(day, ag.StatusDT, GETDATE()) AS DaysInStatus,
    ag.Description,
    (SELECT COUNT(*) FROM ArtworkGroupStatusHistory agsh
     WHERE agsh.ArtworkGroupID = ag.ID
     AND agsh.FromStatusID = 3 AND agsh.ToStatusID = 1) AS RevisionCount
FROM ArtworkGroup ag
JOIN TransHeader th ON ag.TransHeaderID = th.ID
JOIN Account a ON ag.AccountID = a.ID
WHERE ag.IsActive = 1
  AND ag.StatusID = 3  -- Pending Approval
  AND th.TransactionType = 1
  AND th.StatusID IN (0, 1, 2)
ORDER BY ag.StatusDT ASC
```

### PROD-02: Station Workload (Order-Level)

```sql
-- WIP orders by station with dollar values
SELECT
    s.StationName,
    COUNT(*) AS OrderCount,
    SUM(th.SubTotalPrice) AS TotalRevenue,
    MIN(th.DueDate) AS EarliestDueDate
FROM TransHeader th
JOIN Station s ON th.StationID = s.ID
WHERE th.TransactionType = 1
  AND th.StatusID IN (1, 2)
  AND th.IsActive = 1
GROUP BY s.StationName, s.SortIndex
ORDER BY s.SortIndex
```

### PROD-02: Station Workload (Line-Item-Level)

```sql
-- WIP line items by station
SELECT
    s.StationName,
    COUNT(*) AS LineItemCount,
    SUM(td.MeAndSonsSubTotalPrice) AS TotalValue
FROM TransDetail td
JOIN Station s ON td.StationID = s.ID
JOIN TransHeader th ON td.TransHeaderID = th.ID
WHERE th.TransactionType = 1
  AND th.StatusID IN (1, 2)
  AND th.IsActive = 1
  AND td.IsActive = 1
  AND td.ParentID = th.ID  -- Top-level line items only
GROUP BY s.StationName, s.SortIndex
ORDER BY s.SortIndex
```

### PROD-03: Average Dwell Time by Station

```sql
-- Average station dwell time (order-level, last 3 months)
WITH OrderTransitions AS (
    SELECT
        j.TransactionID,
        j.StationID,
        j.StartDateTime,
        LEAD(j.StartDateTime) OVER (
            PARTITION BY j.TransactionID
            ORDER BY j.StartDateTime
        ) AS NextTransitionTime
    FROM Journal j
    WHERE j.JournalActivityType = 45
      AND j.LinkID IS NULL
      AND j.StartDateTime > DATEADD(month, -3, GETDATE())
)
SELECT
    s.StationName,
    COUNT(*) AS TransitionCount,
    AVG(DATEDIFF(minute, ot.StartDateTime, ot.NextTransitionTime)) / 60.0 AS AvgDwellHours,
    AVG(DATEDIFF(minute, ot.StartDateTime, ot.NextTransitionTime)) / 1440.0 AS AvgDwellDays
FROM OrderTransitions ot
JOIN Station s ON ot.StationID = s.ID
WHERE ot.NextTransitionTime IS NOT NULL
  AND (s.ShowForWIP = 1 OR s.ShowForLineWIP = 1)
GROUP BY s.StationName, s.SortIndex
HAVING COUNT(*) > 10
ORDER BY s.SortIndex
```

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Artwork status labels | Hardcoded status text | `JOIN _ArtworkStatus` | Values are DB-driven |
| Station names/hierarchy | Hardcoded station list | `JOIN Station` with ShowFor* flags | Stations are user-configurable |
| Dwell time spans | Custom span tracking | Journal JournalActivityType=45 with LEAD() | Already captured at every station change |
| Revision counting | Parse status history manually | `COUNT(*) WHERE FromStatusID=3 AND ToStatusID=1` | Single aggregate query |
| Order-vs-LI classification | Parse station name prefixes | `ShowForWIP`/`ShowForLineItem` flags | Explicit flags exist |

---

## Open Questions

### 1. Dwell Time Performance at Scale
- **What we know:** 1.06M Journal records for station changes. LEAD/LAG window functions work on small samples. The CROSS APPLY approach was also verified.
- **What's unclear:** Whether a 3-month aggregate dwell query will perform well against 1M+ rows.
- **Recommendation:** Use date-range filters aggressively. Consider suggesting a 1-month default window with option to expand. Test both LEAD() and CROSS APPLY during implementation.

### 2. StatusID=8 ("Unknown") Root Cause
- **What we know:** 96% are on Closed/Sale orders. They represent artwork that was never formally tracked through the approval pipeline.
- **What's unclear:** Whether this is an intentional "skip artwork approval" workflow or a data quality issue. Some orders may not need artwork approval (e.g., reorders, stock items).
- **Recommendation:** Treat as intentional (exclude from actionable pipeline). Document for the user that "Unknown" = "no artwork workflow was used."

### 3. PrCA Board Usage
- **What we know:** 12 boards, 383 jobs (since 2018). Low but non-zero usage.
- **What's unclear:** Whether FLS still uses the Production Calendar or has moved entirely to station-based tracking.
- **Recommendation:** Mention PrCA tables in documentation but do NOT build queries against them for Phase 12. If users ask about the "production board", point them to PrCA.Board.Data.

---

## Sources

### Primary (HIGH confidence)
- Live FLS StoreData database queries (all results above)
- Existing schema documentation in `output/schemas/*.md`
- Wiki knowledge extract: `output/wiki/extracts/production_inventory_knowledge.md`
- Phase context: `.planning/phases/12-production-workflow/12-CONTEXT.md`

### Secondary (MEDIUM confidence)
- Cyrious Wiki documentation on artwork approval workflow, station configuration, and production terminal
- Wiki-documented SQL patterns for Journal JournalActivityType=45

---

## Metadata

**Confidence breakdown:**
- Artwork tables: HIGH -- all lookup values, status distributions, and join paths verified with live queries
- Station hierarchy: HIGH -- complete 98-station enumeration, department groupings, ShowFor* flags all queried
- Dwell time approach: HIGH -- verified Journal records contain station transitions; sample dwell calculations produced realistic results
- WIP model: HIGH -- dual order/LI station model confirmed with live data
- Pitfalls: HIGH -- all discovered from actual data anomalies (Unknown status, stale ApprovalDT, NULL EndDateTime)

**Research date:** 2026-02-09
**Valid until:** 2026-03-09 (station configuration may change; status distributions are dynamic)
