---
name: control-erp-production
description: Track artwork approvals, monitor station workload, identify production bottlenecks, and analyze station dwell time. Use when user asks about artwork, proofing, approvals, stations, production, workload, WIP, bottlenecks, dwell time, "what's in production", "pending approval", "which stations are backed up", or department queue queries. Depends on control-erp-core for business rules.
---

# Control ERP Production Workflow -- Artwork Pipeline, Station Workload & Dwell Analysis

**Depends on:** `control-erp-core` (always read core first for business rules)

---

## Overview

This skill enables complete production visibility for FLS Banners across three pillars:

### Three Production Pillars

**1. Artwork Pipeline (PROD-01):** Track artwork through the approval lifecycle -- from In Design through Pending Approval to Approved/Produced. Monitor stuck items (>7 days in status), revision rates (22% average), and turnaround times (54-hour average approval cycle). Artwork statuses are user-configurable via `_ArtworkStatus` lookup table; FLS actively uses 7 of 11 available statuses.

**2. Station Workload (PROD-02):** Monitor WIP counts and dollar values at each production station. FLS uses a **dual order-level and line-item-level station model** with numbered stages (0-Design through 6-Shipping). Orders have a high-level StationID (order milestone), AND each line item has its own StationID (granular production routing). Queries must be context-aware to avoid double-counting.

**3. Station Dwell Time (PROD-03):** Analyze how long orders and line items sit at each station to identify bottlenecks. Based on Journal station transition timestamps (JournalActivityType=45), not TimeCard data -- FLS does not track per-job time via TimeCard. Average dwell time per station reveals which stages need efficiency improvement to meet the 1-business-day turnaround goal.

### Key Statistics

- **67 active stations** (98 total including inactive and time-tracking stations)
- **~67 WIP orders** currently in production (StatusID IN 1,2)
- **7 active artwork statuses** used at FLS (out of 11 available): In Design, Pending Approval, Approved, Produced, Unknown, Voided, Rejected
- **54-hour average** artwork approval turnaround (creation to approval)
- **22% revision rate** (artwork sent back from Pending Approval to In Design)
- **1.06M station change records** in Journal table (JournalActivityType=45)

---

## ⚠️ CRITICAL WARNINGS

> **WARNING: Data Model and Query Patterns**
>
> 1. **Geography columns in Station table:** NEVER use `SELECT *` on Station. Always specify columns explicitly -- geography/spatial columns crash queries. Safe columns: ID, StationName, SortIndex, DepartmentID, ShowForWIP, ShowForLineItem, ShowForLineWIP, ShowForLineSale, ShowOnTimeClock, MarkLIComplete.
>
> 2. **Dual station model:** Orders have a StationID (order-level) AND each line item has its own StationID (line-item-level). These can differ. Query them separately to avoid double-counting. Example: Order at "1-Approved" may have line items at "Waiting For Shipment" or "0-Design-Proof". Use TransHeader.StationID for order-level, TransDetail.StationID for line-item-level.
>
> 3. **Journal table volume:** 5.18M total rows, 1.06M station changes (JournalActivityType=45). ALWAYS filter by date range for dwell time queries. Default: 3-month rolling window. Never query Journal without date constraints.
>
> 4. **Artwork StatusID=8 ("Unknown") is NOT an error:** 25,274 groups have this status -- orders that completed without formal artwork workflow. This is intentional (reorders, stock items). EXCLUDE from actionable pipeline: filter `StatusID NOT IN (7,8,9)` to show only items needing action (7=Produced, 8=Unknown, 9=Voided are terminal/non-actionable).
>
> 5. **Artwork ApprovalDT can be stale:** On cloned/converted orders, ArtworkApprovalDT carries from the source order. Always filter `ArtworkApprovalDT > GroupCreatedDT` when calculating turnaround. Without this filter, approval times show negative durations. Also exclude outliers: `DATEDIFF(day, GroupCreatedDT, ArtworkApprovalDT) < 90`.
>
> 6. **Do NOT use TimeCard for production time:** FLS does not track per-job time via TimeCard. Station dwell time (Journal transitions) is the production time metric. TimeCard ClassTypeID 20050/20051 are for employee clock-in/clock-out only, not per-job tracking.
>
> 7. **Do NOT hardcode artwork statuses or station names:** Use `_ArtworkStatus` lookup table and `Station` table with ShowFor* flags. Both are user-configurable. Never assume station IDs or status IDs -- always JOIN to lookup tables for text labels.

---

## TABLE ARCHITECTURE

### Core Production Tables

| Table | Rows | Purpose | Key Columns |
|-------|------|---------|-------------|
| **TransHeader** | 232,243 | Orders (WIP = StatusID IN 1,2) | TransactionType=1, StatusID, StationID (order-level), SubTotalPrice, OrderNumber, DueDate |
| **TransDetail** | ~500K | Line items with independent StationID | StationID (LI-level), TransHeaderID, MeAndSonsSubTotalPrice, ParentID, Description, Quantity |
| **Station** | 98 (67 active) | Production stations (numbered 0-6) | StationName, SortIndex, ShowForWIP, ShowForLineItem, ShowForLineWIP, ShowOnTimeClock, DepartmentID, MarkLIComplete |
| **Journal** | 5.18M (1.06M station changes) | Station transitions (JournalActivityType=45) | TransactionID, LinkID (NULL=order-level), StationID (new station), StartDateTime, EndDateTime (always NULL), EmployeeID |

### Artwork Pipeline Tables

| Table | Rows | Purpose | Key Columns |
|-------|------|---------|-------------|
| **ArtworkGroup** | 84,770 (70,973 active) | Artwork groups per order | TransHeaderID, StatusID, AccountID, GroupCreatedDT, StatusDT, ArtworkApprovalDT, ArtworkDueDT, Description, GroupName |
| **_ArtworkStatus** | 11 | Artwork status lookup | ID, Text |
| **ArtworkGroupStatusHistory** | 164,226 | Audit trail of status changes | ArtworkGroupID, FromStatusID, ToStatusID, StatusDT, PlayerID |
| **ArtworkGroupTransDetailLink** | varies | Links artwork to line items | ArtworkGroupID, TransDetailID |
| **ArtworkPlayer** | 176,445 | Named roles on orders | TransHeaderID, EmployeeID, ContactID |
| **ArtworkPlayerRoleLink** | 174,373 | Player-to-role mapping | PlayerID, RoleID |
| **_ArtworkRole** | 8 | Role lookup (Designer=1, Approver=9) | ID, Text |
| **_ArtworkPriority** | 6 | Priority lookup (Normal=2, Rush=4, Urgent=5) | ID, Text |
| **ArtworkItem** | 41,797 | Individual artwork files | GroupID, CurrentVersion, OriginalFileName |
| **ArtworkProofFile** | 41,741 | Proof file versions | ArtworkItemID, Version, IsLatestVersion |

### Supporting Tables

| Table | Rows | Purpose | Key Columns |
|-------|------|---------|-------------|
| **Employee** | varies | Employee lookup for designers | ID, FirstName, LastName, IsActive |
| **Account** | varies | Customer lookup for artwork | ID, CompanyName |

---

## Artwork Status Reference

From `_ArtworkStatus` table, FLS usage data:

| StatusID | Text | FLS Active Count | Pipeline? | Notes |
|----------|------|-----------------|-----------|-------|
| 0 | Undefined | 0 | No | Never used |
| 1 | In Design | 4,965 | **Yes** | Designer working on artwork |
| 2 | In Review | 0 | No | Never used at FLS |
| 3 | Pending Approval | 902 | **Yes** | Sent to customer for approval |
| 4 | Approved | 1,362 | **Yes** | Customer approved, ready for production |
| 5 | In PrePress | 0 | No | Never used at FLS |
| 6 | In Production | 0 | No | Never used at FLS |
| 7 | Produced | 36,683 | No | Terminal -- artwork used in production |
| 8 | Unknown | 25,274 | No | Order completed without artwork workflow (reorders, stock items) |
| 9 | Voided | 1,677 | No | Cancelled artwork |
| 10 | Rejected | 110 | **Yes** | Customer rejected, needs revision |

**Actionable Pipeline Filter:** `StatusID NOT IN (7, 8, 9)` -- excludes Produced, Unknown, and Voided (terminal/non-actionable states).

**Revision Transition:** Pending Approval (3) → In Design (1) indicates customer requested changes. Count these for revision rate analysis.

---

## Artwork Relationship Map

```
TransHeader (order)
  |
  +--< ArtworkGroup (1:many, via TransHeaderID)
  |     |  StatusID -> _ArtworkStatus
  |     |  AccountID -> Account
  |     |  PriorityID -> _ArtworkPriority
  |     |
  |     +--< ArtworkGroupTransDetailLink (links to TransDetail line items)
  |     |     ArtworkGroupID -> ArtworkGroup.ID
  |     |     TransDetailID -> TransDetail.ID
  |     |
  |     +--< ArtworkGroupStatusHistory (audit trail)
  |     |     ArtworkGroupID -> ArtworkGroup.ID
  |     |     FromStatusID, ToStatusID -> _ArtworkStatus.ID
  |     |     StatusDT (timestamp), PlayerID
  |     |
  |     +--< ArtworkItem (artwork files)
  |           GroupID -> ArtworkGroup.ID
  |           +--< ArtworkProofFile (proof versions)
  |           |     ArtworkItemID -> ArtworkItem.ID
  |           |     Version, IsLatestVersion
  |           |
  |           +--< ArtworkComment (annotations)
  |                 ArtworkItemID -> ArtworkItem.ID
  |                 CommentTypeID -> _ArtworkCommentType.ID
  |
  +--< ArtworkPlayer (1:many, via TransHeaderID)
        EmployeeID -> Employee
        +--< ArtworkPlayerRoleLink (RoleID -> _ArtworkRole)
              RoleID=1 (Designer), RoleID=9 (Approver)
```

---

## ARTWORK PIPELINE QUERIES (PROD-01)

### 4a. Artwork Pipeline Summary (Actionable Items Only)

**When to use:** User asks "what artwork is pending", "show me artwork status", "artwork pipeline"

```sql
SELECT
    s.Text AS ArtworkStatus,
    COUNT(*) AS GroupCount,
    COUNT(DISTINCT ag.TransHeaderID) AS OrderCount,
    AVG(DATEDIFF(day, ag.StatusDT, GETDATE())) AS AvgDaysInStatus
FROM ArtworkGroup ag
JOIN _ArtworkStatus s ON ag.StatusID = s.ID
JOIN TransHeader th ON ag.TransHeaderID = th.ID
WHERE ag.IsActive = 1
  AND th.TransactionType = 1
  AND th.StatusID IN (0, 1, 2)  -- New, WIP, Built orders only
  AND ag.StatusID NOT IN (7, 8, 9)  -- Exclude Produced, Unknown, Voided
GROUP BY ag.StatusID, s.Text
ORDER BY ag.StatusID
```

**Formatting guidance:**
- Present as summary table: "**[Status]**: [GroupCount] artwork groups across [OrderCount] orders (avg [AvgDaysInStatus] days in status)"
- Highlight Pending Approval (StatusID=3) and Rejected (StatusID=10) as "needs attention"
- Always note total: "**Active pipeline**: [total] artwork groups on [total orders] open orders"

---

### 4b. Artwork Drill-Down by Status

**When to use:** User wants details of a specific status -- "show me pending approvals", "which artwork is rejected"

```sql
SELECT
    th.OrderNumber,
    a.CompanyName AS Customer,
    ag.GroupName,
    ag.GroupCreatedDT AS Created,
    ag.StatusDT AS StatusSince,
    ag.ArtworkDueDT AS DueDate,
    DATEDIFF(day, ag.StatusDT, GETDATE()) AS DaysInStatus,
    ag.Description,
    (SELECT COUNT(*) FROM ArtworkGroupStatusHistory agsh
     WHERE agsh.ArtworkGroupID = ag.ID
     AND agsh.FromStatusID = 3 AND agsh.ToStatusID = 1) AS RevisionCount,
    pri.Text AS Priority
FROM ArtworkGroup ag
JOIN TransHeader th ON ag.TransHeaderID = th.ID
JOIN Account a ON ag.AccountID = a.ID
LEFT JOIN _ArtworkPriority pri ON ag.PriorityID = pri.ID
WHERE ag.IsActive = 1
  AND ag.StatusID = @StatusID  -- 1=In Design, 3=Pending Approval, 4=Approved, 10=Rejected
  AND th.TransactionType = 1
  AND th.StatusID IN (0, 1, 2)
ORDER BY ag.StatusDT ASC
```

**Formatting guidance:**
- Show oldest items first (longest wait at top)
- If RevisionCount > 0, flag: "(revised [N] times)"
- If DaysInStatus > 7, flag: "**OVERDUE** -- [N] days in status"
- Include DueDate when available

**Common @StatusID values:**
- 1 = In Design
- 3 = Pending Approval
- 4 = Approved
- 10 = Rejected

---

### 4c. Stuck Artwork Detection (7-Day Threshold)

**When to use:** User asks "what artwork is stuck", "which approvals are overdue", "artwork bottlenecks"

Based on the 54-hour average turnaround, 3x = ~7 days as "stuck" threshold:

```sql
SELECT
    th.OrderNumber,
    a.CompanyName AS Customer,
    s.Text AS CurrentStatus,
    ag.GroupName,
    ag.StatusDT AS StatusSince,
    DATEDIFF(day, ag.StatusDT, GETDATE()) AS DaysStuck,
    ag.ArtworkDueDT AS DueDate,
    (SELECT COUNT(*) FROM ArtworkGroupStatusHistory agsh
     WHERE agsh.ArtworkGroupID = ag.ID
     AND agsh.FromStatusID = 3 AND agsh.ToStatusID = 1) AS RevisionCount,
    pri.Text AS Priority
FROM ArtworkGroup ag
JOIN TransHeader th ON ag.TransHeaderID = th.ID
JOIN Account a ON ag.AccountID = a.ID
JOIN _ArtworkStatus s ON ag.StatusID = s.ID
LEFT JOIN _ArtworkPriority pri ON ag.PriorityID = pri.ID
WHERE ag.IsActive = 1
  AND ag.StatusID IN (1, 3, 10)  -- In Design, Pending Approval, Rejected
  AND DATEDIFF(day, ag.StatusDT, GETDATE()) > 7
  AND th.TransactionType = 1
  AND th.StatusID IN (0, 1, 2)
ORDER BY DATEDIFF(day, ag.StatusDT, GETDATE()) DESC
```

**Note:** The 7-day threshold is 3x the 54-hour average turnaround. Claude may adjust the threshold if user requests (e.g., "stuck more than 3 days"). StatusID=4 (Approved) is excluded from stuck detection -- approved items are progressing normally toward production.

---

### 4d. Artwork Approval Turnaround Time

**When to use:** User asks "how long does artwork approval take", "artwork turnaround", "approval time"

Average time from artwork creation to approval:

```sql
SELECT
    COUNT(*) AS ApprovedCount,
    AVG(DATEDIFF(hour, ag.GroupCreatedDT, ag.ArtworkApprovalDT)) AS AvgTurnaroundHours,
    AVG(DATEDIFF(hour, ag.GroupCreatedDT, ag.ArtworkApprovalDT)) / 24.0 AS AvgTurnaroundDays,
    MIN(DATEDIFF(hour, ag.GroupCreatedDT, ag.ArtworkApprovalDT)) AS MinHours,
    MAX(DATEDIFF(hour, ag.GroupCreatedDT, ag.ArtworkApprovalDT)) AS MaxHours
FROM ArtworkGroup ag
JOIN TransHeader th ON ag.TransHeaderID = th.ID
WHERE ag.IsActive = 1
  AND ag.StatusID IN (4, 7)  -- Approved or Produced (completed the approval cycle)
  AND ag.ArtworkApprovalDT IS NOT NULL
  AND ag.ArtworkApprovalDT > ag.GroupCreatedDT  -- CRITICAL: exclude cloned-order stale dates
  AND DATEDIFF(day, ag.GroupCreatedDT, ag.ArtworkApprovalDT) < 90  -- exclude outliers
  AND th.TransactionType = 1
  AND ag.GroupCreatedDT >= DATEADD(MONTH, -6, GETDATE())  -- last 6 months
```

**Note:** FLS average is ~54 hours (~2.25 business days). The `ArtworkApprovalDT > GroupCreatedDT` filter is CRITICAL -- cloned orders carry stale approval dates. The 90-day outlier filter prevents abandoned-then-approved items from skewing averages.

---

### 4e. Turnaround Trend by Month

**When to use:** User asks "is approval time improving", "turnaround trend", "approval performance over time"

```sql
SELECT
    FORMAT(ag.ArtworkApprovalDT, 'yyyy-MM') AS ApprovalMonth,
    COUNT(*) AS Approvals,
    AVG(DATEDIFF(hour, ag.GroupCreatedDT, ag.ArtworkApprovalDT)) AS AvgTurnaroundHours,
    SUM(CASE WHEN DATEDIFF(day, ag.GroupCreatedDT, ag.ArtworkApprovalDT) > 1 THEN 1 ELSE 0 END) AS ExceededOneDayGoal
FROM ArtworkGroup ag
JOIN TransHeader th ON ag.TransHeaderID = th.ID
WHERE ag.IsActive = 1
  AND ag.StatusID IN (4, 7)
  AND ag.ArtworkApprovalDT IS NOT NULL
  AND ag.ArtworkApprovalDT > ag.GroupCreatedDT
  AND DATEDIFF(day, ag.GroupCreatedDT, ag.ArtworkApprovalDT) < 90
  AND th.TransactionType = 1
  AND ag.ArtworkApprovalDT >= DATEADD(MONTH, -12, GETDATE())
GROUP BY FORMAT(ag.ArtworkApprovalDT, 'yyyy-MM')
ORDER BY FORMAT(ag.ArtworkApprovalDT, 'yyyy-MM')
```

---

### 4f. Revision Rate Analysis

**When to use:** User asks "how often do we revise artwork", "revision rate", "how many rejections"

Percentage of artwork going back for revision:

```sql
SELECT
    COUNT(*) AS TotalSubmissions,
    SUM(CASE WHEN agsh.FromStatusID = 3 AND agsh.ToStatusID = 1 THEN 1 ELSE 0 END) AS Revisions,
    CAST(SUM(CASE WHEN agsh.FromStatusID = 3 AND agsh.ToStatusID = 1 THEN 1 ELSE 0 END) AS FLOAT)
        / NULLIF(COUNT(*), 0) * 100 AS RevisionRatePct
FROM ArtworkGroupStatusHistory agsh
JOIN ArtworkGroup ag ON agsh.ArtworkGroupID = ag.ID
JOIN TransHeader th ON ag.TransHeaderID = th.ID
WHERE ag.IsActive = 1
  AND th.TransactionType = 1
  AND agsh.StatusDT >= DATEADD(MONTH, -6, GETDATE())
  AND agsh.FromStatusID = 3  -- transitions FROM Pending Approval
```

**Note:** FLS revision rate is ~22% (896 revisions out of ~4,100 approval submissions since 2025). A revision is defined as Pending Approval (3) → In Design (1).

---

### 4g. Artwork by Designer

**When to use:** User asks "what is [designer] working on", "show artwork by designer"

What is a specific designer working on:

```sql
SELECT
    th.OrderNumber,
    a.CompanyName AS Customer,
    ag.GroupName,
    s.Text AS ArtworkStatus,
    ag.StatusDT AS StatusSince,
    DATEDIFF(day, ag.StatusDT, GETDATE()) AS DaysInStatus,
    pri.Text AS Priority
FROM ArtworkGroup ag
JOIN TransHeader th ON ag.TransHeaderID = th.ID
JOIN Account a ON ag.AccountID = a.ID
JOIN _ArtworkStatus s ON ag.StatusID = s.ID
LEFT JOIN _ArtworkPriority pri ON ag.PriorityID = pri.ID
JOIN ArtworkPlayer ap ON ap.TransHeaderID = th.ID
JOIN ArtworkPlayerRoleLink aprl ON aprl.PlayerID = ap.ID AND aprl.RoleID = 1  -- Designer role
JOIN Employee e ON ap.EmployeeID = e.ID
WHERE ag.IsActive = 1
  AND ag.StatusID NOT IN (7, 8, 9)
  AND th.TransactionType = 1
  AND th.StatusID IN (0, 1, 2)
  AND e.ID = @EmployeeID  -- or use: (e.FirstName + ' ' + e.LastName) LIKE '%name%'
ORDER BY ag.StatusDT ASC
```

**Finding the EmployeeID:** To search by name:
```sql
SELECT ID, FirstName, LastName
FROM Employee
WHERE (FirstName + ' ' + LastName) LIKE '%name%'
  AND IsActive = 1
```

---

### 4h. Artwork Pipeline with Line Item Detail

**When to use:** User wants to see which specific line items an artwork group covers

```sql
SELECT
    th.OrderNumber,
    ag.GroupName,
    s.Text AS ArtworkStatus,
    td.Description AS LineItem,
    td.Quantity,
    td.MeAndSonsSubTotalPrice AS LineValue
FROM ArtworkGroup ag
JOIN TransHeader th ON ag.TransHeaderID = th.ID
JOIN _ArtworkStatus s ON ag.StatusID = s.ID
JOIN ArtworkGroupTransDetailLink agtdl ON agtdl.ArtworkGroupID = ag.ID
JOIN TransDetail td ON agtdl.TransDetailID = td.ID
WHERE ag.IsActive = 1
  AND ag.StatusID NOT IN (7, 8, 9)
  AND th.TransactionType = 1
  AND th.StatusID IN (0, 1, 2)
  AND th.ID = @TransHeaderID  -- specific order
ORDER BY ag.StatusID, td.SortOrder
```

---

## CRITICAL REMINDERS FOR THE EXECUTOR

When executing artwork pipeline queries:

1. **Always exclude terminal statuses:** Filter `StatusID NOT IN (7, 8, 9)` for actionable pipeline views
2. **Always filter WIP orders:** Join TransHeader and filter `TransactionType = 1` AND `StatusID IN (0, 1, 2)`
3. **Always use stale date filter:** For turnaround calculations, filter `ArtworkApprovalDT > GroupCreatedDT`
4. **Always filter date ranges:** For Journal-related queries (upcoming Plan 02), use 3-month rolling window
5. **Never SELECT * from Station:** Specify columns explicitly to avoid geography column errors
6. **Revision definition:** `ArtworkGroupStatusHistory WHERE FromStatusID=3 AND ToStatusID=1`
7. **Stuck threshold:** 7 days = 3x average turnaround; adjustable per user request

---

## SECTION 5: STATION HIERARCHY REFERENCE

### Station Classification Model

FLS uses a dual-level station model where stations can track:
- **Order-level**: High-level milestones (flags: `ShowForWIP = 1` or `ShowForBuilt = 1` or `ShowForSale = 1` or `ShowForPendingEstimates = 1`)
- **Line-item-level**: Granular production routing (flags: `ShowForLineItem = 1` or `ShowForLineWIP = 1` or `ShowForLineBuilt = 1` or `ShowForLineSale = 1`)
- **Dual-level (both)**: Station tracks BOTH orders and line items independently (e.g., 1-Approved, 0-Design-Proof, 6-Shipping)

**Classification Logic:**
```
Order-level    = ShowForWIP OR ShowForBuilt OR ShowForSale OR ShowForPendingEstimates
Line-item-level = ShowForLineItem OR ShowForLineWIP OR ShowForLineBuilt OR ShowForLineSale
Dual-level     = (Order-level flags) AND (Line-item-level flags)
```

---

### FLS Production Stage Hierarchy (Numbered-Stage System)

FLS organizes production into numbered stages 0-6 with substations:

| Stage | Station Name | ID | Order-Level | LI-Level | TimeClock | Notes |
|-------|-------------|-----|-------------|----------|-----------|-------|
| **0** | **0-Design** (dept) | 1016 | -- | -- | -- | Department parent |
| 0 | 0-Design-Preflight | 10004 | WIP, Est | -- | -- | |
| 0 | 0-Design-Need Art | 10002 | WIP, Est | -- | -- | |
| 0 | 0-Design-Request Art | 10003 | WIP, Est | -- | -- | |
| 0 | 0-Design-Proof | 1020 | WIP, Est | LI (all) | Yes | **Dual-level** |
| 0 | 0-Design-Wait for Art | 1012 | WIP, Est | LI (all) | -- | **Dual-level** |
| 0 | 0-Design-Complete | 1069 | WIP | LI (all) | -- | **Dual-level** |
| **1** | **1-Approved** | 1024 | WIP, Est | LI (all) | -- | **Dual-level** |
| **1.1** | **1.1 Pre-Production** | 10008 | -- | LI (all) | Yes | LI only |
| **2** | **2-Production** (dept) | 1025 | WIP | -- | -- | Order-level only |
| 2 | 2-Production-Digital | 1027 | -- | LI (all) | Yes | LI only |
| 2 | 2-Production-West Side | 1073 | -- | LI (all) | Yes | LI only |
| 2 | 2-Production-SP | 1028 | -- | LI (all) | Yes | LI only |
| 2 | 2-Production-DTG | 10001 | -- | LI (all) | Yes | LI only |
| 2 | 2-Production-Garment | 1030 | -- | LI (all) | Yes | LI only |
| 2 | 2-Production-Embroidery | 1031 | -- | LI (all) | Yes | LI only |
| 2 | 2-Production-Contract | 1029 | WIP | LI (all) | -- | **Dual-level** |
| 2 | 2-Production-Press Ons | 1071 | -- | LI (all) | Yes | LI only |
| **2.1** | **2.1-Print In Progress** | 10007 | -- | LI (all) | Yes | LI only |
| **3** | **3-Transfer** | 1068 | -- | LI (all) | Yes | LI only |
| **4** | **4-Cutting** | 1043 | -- | LI (all) | Yes | LI only |
| **5** | **5-Finishing** | 1032 | -- | LI (all) | Yes | LI only |
| **6** | **6-Shipping** | 1033 | WIP | LI (all) | Yes | **Dual-level** |

**TimeClock column:** Indicates `ShowOnTimeClock = 1` (employee clock stations for time tracking)

---

### Administrative Stations (Non-Production)

**Hold Stations (order-level):**
- On-Hold (1014)
- On-Hold-Artwork (1034)
- On-Hold-Payment (1035)
- Out For Approval (1023)

**Post-Production Stations:**
- Waiting For Pick-up (1049) -- Sale, LI
- Waiting For Shipment (1050) -- Sale, LI
- Picked-Up (1036) -- Sale, LI, MarkLIComplete=true
- Shipped (1037) -- Sale
- Delivered (1038) -- Sale
- Built (1042) -- LI

**Other Stations:**
- Estimate tracking stations (Estimate Sent, Converted, etc.)
- Follow-up stations (Follow-up-24hr, Follow-up-72, etc.)
- Time/Payroll stations (Time, Time Off, etc.) -- NOT production-related
- PO stations (Requested, Ordered, Received, etc.) -- Purchase order lifecycle

---

### Station Discovery Query

```sql
-- Active production-relevant stations (excludes Time/PO stations)
-- CRITICAL: Never use SELECT * on Station (geography columns crash)
SELECT
    s.ID,
    s.StationName,
    s.SortIndex,
    s.ShowForWIP,
    s.ShowForLineItem,
    s.ShowForLineWIP,
    s.ShowOnTimeClock,
    s.DepartmentID,
    s.MarkLIComplete,
    CASE
        WHEN (s.ShowForWIP = 1 OR s.ShowForBuilt = 1 OR s.ShowForSale = 1)
             AND (s.ShowForLineItem = 1 OR s.ShowForLineWIP = 1)
        THEN 'Dual (order + line item)'
        WHEN s.ShowForWIP = 1 OR s.ShowForBuilt = 1 OR s.ShowForSale = 1
        THEN 'Order-level'
        WHEN s.ShowForLineItem = 1 OR s.ShowForLineWIP = 1
        THEN 'Line-item-level'
        ELSE 'Other'
    END AS StationLevel
FROM Station s
WHERE s.IsActive = 1
    AND s.ID NOT IN (200, 201, 202, 1001, 1002, 1004, 1005, 1006, 1007, 1008)  -- Exclude Time/PTO stations
    AND s.ID NOT BETWEEN 100 AND 110  -- Exclude PO lifecycle stations
ORDER BY s.SortIndex
```

**CRITICAL:** NEVER use `SELECT *` on Station. Geography columns cause query failures. Always explicitly list columns as shown above.

---

## SECTION 6: STATION WORKLOAD QUERIES (PROD-02)

### 6a. Order-Level WIP by Station (Stage Summary)

**When to use:** User asks "what's in production", "WIP by station", "show me the pipeline"

```sql
SELECT
    s.StationName,
    s.SortIndex,
    COUNT(*) AS OrderCount,
    SUM(th.SubTotalPrice) AS TotalRevenue,
    MIN(th.DueDate) AS EarliestDueDate,
    MAX(DATEDIFF(day, th.OrderCreatedDate, GETDATE())) AS OldestOrderDays
FROM TransHeader th
JOIN Station s ON th.StationID = s.ID
WHERE th.TransactionType = 1
  AND th.StatusID IN (1, 2)
  AND th.IsActive = 1
GROUP BY s.StationName, s.SortIndex
ORDER BY s.SortIndex
```

**Formatting guidance:**
- Present as a pipeline view sorted by stage (SortIndex)
- For each station: "**[StationName]**: [OrderCount] orders ($[TotalRevenue]) -- earliest due: [EarliestDueDate]"
- Summary line: "**Total WIP**: [sum] orders across [station count] stations"
- Reference baseline: FLS current WIP is ~67 orders (2-Production: 40, 0-Design-Proof: 14, Out For Approval: 6)

---

### 6b. Line-Item-Level WIP by Station

**When to use:** User asks "line items in production", "line item WIP", "what's at each station" (detailed view)

```sql
SELECT
    s.StationName,
    s.SortIndex,
    COUNT(*) AS LineItemCount,
    SUM(td.MeAndSonsSubTotalPrice) AS TotalValue,
    COUNT(DISTINCT td.TransHeaderID) AS OrdersRepresented
FROM TransDetail td
JOIN Station s ON td.StationID = s.ID
JOIN TransHeader th ON td.TransHeaderID = th.ID
WHERE th.TransactionType = 1
  AND th.StatusID IN (1, 2)
  AND th.IsActive = 1
  AND td.IsActive = 1
  AND td.ParentID = th.ID  -- Top-level line items only (avoids sub-items)
GROUP BY s.StationName, s.SortIndex
ORDER BY s.SortIndex
```

**Note:** `ParentID = th.ID` filters to top-level line items only (children of the header, not sub-line-items). This prevents double-counting from assembly/sub-item structures.

**Reference baseline:** FLS current LI WIP is ~116 items (Waiting For Shipment: 42, 0-Design-Proof: 37, 2-Production-Embroidery: 12)

---

### 6c. Specific Station Detail (Order List for a Station)

**When to use:** User asks "what's at [station]", "show me [station] queue", "[station] detail"

**For ORDER-LEVEL detail:**
```sql
-- Order-level detail for a specific station
SELECT
    th.OrderNumber,
    a.CompanyName AS Customer,
    th.SubTotalPrice AS OrderValue,
    th.DueDate,
    DATEDIFF(day, th.OrderCreatedDate, GETDATE()) AS OrderAgeDays,
    th.StatusID
FROM TransHeader th
JOIN Station s ON th.StationID = s.ID
JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 1
  AND th.StatusID IN (1, 2)
  AND th.IsActive = 1
  AND s.StationName LIKE '%station_name%'
ORDER BY th.DueDate ASC
```

**For LINE-ITEM detail:**
```sql
-- Line-item detail for a specific station
SELECT
    th.OrderNumber,
    a.CompanyName AS Customer,
    td.Description AS LineItem,
    td.Quantity,
    td.MeAndSonsSubTotalPrice AS LineValue,
    s.StationName
FROM TransDetail td
JOIN Station s ON td.StationID = s.ID
JOIN TransHeader th ON td.TransHeaderID = th.ID
JOIN Account a ON th.AccountID = a.ID
WHERE th.TransactionType = 1
  AND th.StatusID IN (1, 2)
  AND th.IsActive = 1
  AND td.IsActive = 1
  AND s.StationName LIKE '%station_name%'
ORDER BY th.OrderNumber, td.SortOrder
```

**Guidance:** When the user names a station:
1. Search by LIKE pattern against `StationName`
2. For **dual-level stations** (e.g., 1-Approved, 6-Shipping): Run BOTH queries and present both views
3. For **line-item-only stations** (e.g., 2-Production-Embroidery): Only run the line-item query
4. For **order-level-only stations** (e.g., 2-Production dept): Only run the order query

---

### 6d. Stage-Level Summary (Combined Order + Line-Item View)

**When to use:** User asks "pipeline overview", "stage summary", "production stages", "high-level view"

```sql
-- Parse stage number from station name prefix for grouped summary
SELECT
    CASE
        WHEN s.StationName LIKE '0-%' THEN '0-Design'
        WHEN s.StationName LIKE '1-%' OR s.StationName = '1-Approved' THEN '1-Approved'
        WHEN s.StationName LIKE '1.1%' THEN '1.1-Pre-Production'
        WHEN s.StationName LIKE '2-%' THEN '2-Production'
        WHEN s.StationName LIKE '2.1%' THEN '2.1-Print In Progress'
        WHEN s.StationName LIKE '3-%' THEN '3-Transfer'
        WHEN s.StationName LIKE '4-%' THEN '4-Cutting'
        WHEN s.StationName LIKE '5-%' THEN '5-Finishing'
        WHEN s.StationName LIKE '6-%' THEN '6-Shipping'
        ELSE 'Other (' + s.StationName + ')'
    END AS Stage,
    COUNT(DISTINCT th.ID) AS WIPOrders,
    SUM(th.SubTotalPrice) AS TotalOrderValue
FROM TransHeader th
JOIN Station s ON th.StationID = s.ID
WHERE th.TransactionType = 1
  AND th.StatusID IN (1, 2)
  AND th.IsActive = 1
GROUP BY
    CASE
        WHEN s.StationName LIKE '0-%' THEN '0-Design'
        WHEN s.StationName LIKE '1-%' OR s.StationName = '1-Approved' THEN '1-Approved'
        WHEN s.StationName LIKE '1.1%' THEN '1.1-Pre-Production'
        WHEN s.StationName LIKE '2-%' THEN '2-Production'
        WHEN s.StationName LIKE '2.1%' THEN '2.1-Print In Progress'
        WHEN s.StationName LIKE '3-%' THEN '3-Transfer'
        WHEN s.StationName LIKE '4-%' THEN '4-Cutting'
        WHEN s.StationName LIKE '5-%' THEN '5-Finishing'
        WHEN s.StationName LIKE '6-%' THEN '6-Shipping'
        ELSE 'Other (' + s.StationName + ')'
    END
ORDER BY MIN(s.SortIndex)
```

**Note:** This gives a high-level pipeline view. The CASE statement groups substations into their parent stage. For detailed substation breakdown, use queries 6a/6b.

---

## CRITICAL REMINDERS FOR STATION WORKLOAD QUERIES

1. **Always filter `TransactionType = 1` AND `StatusID IN (1, 2)`** for WIP
2. **Always use `ParentID = th.ID`** for top-level line items (prevents double-counting)
3. **Never `SELECT *` from Station** (geography columns crash queries)
4. **For dual-level stations:** Present both order and line-item views
5. **For line-item-only stations:** Only present line-item view
6. **For order-level-only stations:** Only present order view
