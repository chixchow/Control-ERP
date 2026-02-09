# Crystal Reports Catalog (DOC-04)

Reference catalog for Control ERP Crystal Reports at FLS Banners. Use this to determine whether a built-in report already exists before writing a custom query.

**Data source:** ReportMenuItem table metadata (899 rows), `output/report_summary.md` (36 .rpt files), and 13 wiki-documented standard reports. MCP database query was unavailable during creation (Mac environment); catalog was compiled from existing schema documentation and report analysis files.

---

## Overview

- **Report menu items:** 899 entries in ReportMenuItem table
- **Physical .rpt files:** 36 Crystal Report files on disk (`V:\Control Reports\`)
- **Wiki-documented reports:** 13 standard reports with authoritative join patterns
- **Report types:** Custom .rpt files (FileName field) and system report templates (UseSystemReport=true)
- **Most-referenced report:** DropShipInvoice.rpt (referenced by ~450 menu items)

---

## Report Catalog

### Financial Reports

| # | Report Name | Filename | Menu Category | Wiki Documented | Analysis File |
|---|-------------|----------|---------------|-----------------|---------------|
| 1 | FLS Income Statement | FLS Income Statement.rpt | Financial | No | `output/reports/FLS_Income_Statement.md` |
| 2 | FLS Balance Sheet | FLS Balance Sheet.rpt | Financial | No | `output/reports/FLS_Balance_Sheet.md` |
| 3 | A/R Detail | A_R Detail.rpt | Financial | Yes | `output/reports/A_R_Detail.md` |
| 4 | AR Report - Summary | AR Report - Summary.rpt | Financial | Yes | `output/reports/AR_Report_-_Summary.md` |
| 5 | A/P Aging Detail | A_P Aging Detail.rpt | Financial | No | `output/reports/A_P_Aging_Detail.md` |

### WIP / Production Reports

| # | Report Name | Filename | Menu Category | Wiki Documented | Analysis File |
|---|-------------|----------|---------------|-----------------|---------------|
| 6 | WIP Summary - Order Level | WIP Summary - Order Level.rpt | Production | Yes | `output/reports/WIP_Summary___Order_Level.md` |
| 7 | WIP By Line Item Station | WIP By Line Item Station.rpt | Production | Yes | `output/reports/WIP_By_Line_Item_Station.md` |
| 8 | WIP By Machine | WIP By Machine.rpt | Production | No | `output/reports/WIP_By_Machine.md` |
| 9 | WIP By Calendar Month Due | WIP By Calendar Month Due.rpt | Production | No | `output/reports/WIP_By_Calendar_Month_Due.md` |

### Sales Reports

| # | Report Name | Filename | Menu Category | Wiki Documented | Analysis File |
|---|-------------|----------|---------------|-----------------|---------------|
| 10 | Orders Placed Between | Orders Placed Between.rpt | Sales | Yes | `output/reports/Orders_Placed_Between.md` |
| 11 | Sales - By Product | Sales - By Product.rpt | Sales | Yes | `output/reports/Sales___By_Product.md` |
| 12 | Sales - By Product Category | Sales - By Product Category.rpt | Sales | Yes | `output/reports/Sales___By_Product_Category.md` |
| 13 | Sales - By Customer (Volume) | Sales - By Customer (Volume).rpt | Sales | No | `output/reports/Sales___By_Customer_(Volume).md` |
| 14 | Sales - By Customer (Frequency) | Sales - By Customer (Frequency).rpt | Sales | No | `output/reports/Sales___By_Customer_(Frequency).md` |
| 15 | Sales - By Sales Account | Sales - By Sales Account.rpt | Sales | No | `output/reports/Sales___By_Sales_Account.md` |
| 16 | Sales - By All Salespeople | Sales - By All Salespeople.rpt | Sales | No | `output/reports/Sales___By_All_Salespeople.md` |
| 17 | Sales - By Primary Salesperson | Sales - By Primary Saleperson.rpt | Sales | No | `output/reports/Sales___By_Primary_Saleperson.md` |
| 18 | Based On Sales by Price | Based On Sales by Price.rpt | Sales | No | `output/reports/Based_On_Sales_by_Price.md` |
| 19 | Based on Orders Placed by Price | Based on Orders Placed by Price.rpt | Sales | No | `output/reports/Based_on_Orders_Placed_by_Price.md` |
| 20 | Sales Graph | Sales Graph.rpt | Sales | No | `output/reports/Sales_Graph.md` |
| 21 | Sales Report | Sales Report.rpt | Sales | Yes | `output/reports/Sales_Report.md` |
| 22 | Sales - Yearly by Calendar Month | Sales - Yearly Sales By Calendar Month.rpt | Sales | No | `output/reports/Sales___Yearly_Sales_By_Calendar_Month.md` |
| 23 | Sales - Yearly by Salesperson | Sales - Yearly Sales By Salesperson.rpt | Sales | No | `output/reports/Sales___Yearly_Sales_By_Salesperson.md` |

### Estimate Reports

| # | Report Name | Filename | Menu Category | Wiki Documented | Analysis File |
|---|-------------|----------|---------------|-----------------|---------------|
| 24 | Converted Estimates | Converted Estimates.rpt | Estimates | No | `output/reports/Converted_Estimates.md` |
| 25 | Pending Estimates | Pending Estimates.rpt | Estimates | No | `output/reports/Pending_Estimates.md` |
| 26 | Last Order Aging | Last Order Aging.rpt | Estimates | No | `output/reports/Last_Order_Aging.md` |
| 27 | Est. vs Act. Cost Summary By Order | Est. vs Act. Cost Summary By Order.rpt | Estimates | No | `output/reports/Est._vs_Act._Cost_Summary_By_Order.md` |

### Inventory and Product Reports

| # | Report Name | Filename | Menu Category | Wiki Documented | Analysis File |
|---|-------------|----------|---------------|-----------------|---------------|
| 28 | All Parts Listing | All Parts Listing.rpt | Inventory | No | `output/reports/All_Parts_Listing.md` |
| 29 | Catalog Listing | Catalog Listing.rpt | Inventory | No | `output/reports/Catalog_Listing.md` |
| 30 | Inventory Listing | Inventory Listing.rpt | Inventory | No | `output/reports/Inventory_Listing.md` |
| 31 | Inventory History | Inventory History.rpt | Inventory | No | `output/reports/Inventory_History.md` |
| 32 | Product Detail | Product Detail.rpt | Products | No | `output/reports/Product_Detail.md` |
| 33 | Product Parameters | Product Parameters.rpt | Products | No | `output/reports/Product_Parameters.md` |

### Other Reports

| # | Report Name | Filename | Menu Category | Wiki Documented | Analysis File |
|---|-------------|----------|---------------|-----------------|---------------|
| 34 | Last Contact Aging | Last Contact Aging.rpt | CRM | No | `output/reports/Last_Contact_Aging.md` |
| 35 | Shipped By Date | Shipped By Date.rpt | Shipping | No | `output/reports/Shipped_By_Date.md` |
| 36 | Work Order with Proof | Work Order with Proof.rpt | Production | Yes | `output/reports/Work_Order_with_Proof.md` |

---

## DropShipInvoice and High-Frequency Reports

The ReportMenuItem table contains 899 entries, but many reference the same .rpt file across different menu locations and configurations.

**DropShipInvoice.rpt** is referenced by approximately 450 menu items -- more than any other report. This is the primary invoice/packing slip document used across multiple contexts (order types, divisions, print configurations). It functions as Control's default document output template for shipping and invoicing.

Other high-frequency reports include the Invoice Report and Estimate Report (system report templates used across divisions and order types).

---

## System Reports

Reports with `UseSystemReport = true` in ReportMenuItem use Control's built-in system report engine rather than external .rpt files. These are referenced by `SystemReportID` and rendered by the application.

The CrystalSystemReports table has only 1 record (a version marker). System report templates are stored in the ReportTemplate table (168 entries) and rendered via the Report table (211 entries).

**System report architecture:**
- `ReportTemplate` (168 rows) -- defines report layouts and criteria options
- `Report` (211 rows) -- named report instances linked to templates
- `ReportElement` (85 rows) -- report layout elements
- `ReportMenuItem` -- menu entries can reference either external .rpt files (via FileName) or system templates (via UseSystemReport + SystemReportID)

---

## Wiki-Documented Reports (Authoritative Join Patterns)

These 13 reports have wiki-documented table relationships and join patterns. Their SQL patterns are authoritative (verified from Cyrious wiki documentation) and are compiled in `output/wiki/references/crystal-reports-sql-patterns.md`.

| Report | Primary Source | Key Joins |
|--------|---------------|-----------|
| AR Detail Report | TransHeader | Account, PaymentTerms, Employee, EmployeeGroup |
| AR Summary Report | TransHeader | Account, AccountContact, PaymentTerms, Employee |
| Deposits Made Report | Journal | Payment, PaymentAccount, GLAccount, TransHeader |
| Estimate Report | TransHeader | TransVariation, TransDetail, Account, Employee, Address |
| GL Listing Report | Ledger | GLAccount, TransHeader, Account, Journal |
| Historical AR Report | GL (view) | TransHeader, Account, PaymentTerms, Employee |
| Invoice Report | TransHeader | TransDetail, Account, Address, Employee, Journal, Payment |
| Orders Placed Between | Ledger | TransHeader, Account, Employee, EmployeeGroup |
| Sales By X Report | GL (view) | TransHeader, TransDetail, Employee, GLAccount, Account |
| Sales Report | GL (view) | TransHeader, Account, Employee, EmployeeGroup |
| WIP and Built Report | TransHeader | Account, Station, Employee, EmployeeGroup |
| WIP By Line Item Report | TransDetail | TransHeader, Account, Station, Employee |
| Work Order Report | TransHeader | TransDetail, TransPart, Part, Shipment, Account |

**Key finding from wiki analysis:** The Sales Report and Sales By X Report use the GL view as their primary data source (not TransHeader directly). This is a significant architectural pattern -- Control's built-in sales reports pull revenue from general ledger entries, not order records.

---

## When to Use Reports vs Custom Queries

**Use a built-in Crystal Report when:**
- Exact formatted output is needed (invoices, packing slips, work orders)
- The document is customer-facing or regulatory
- The report already exists and covers the analysis needed
- Standard parameters (date range, division, salesperson) are sufficient

**Use a custom SQL query when:**
- Ad-hoc analysis combining data from multiple domains
- Product-level drill-down using TransDetailParam variables (FP_ProductDescription, FP_ProductID)
- Data aggregation or trending not covered by existing reports
- Cross-domain analysis (e.g., correlating sales with GL entries)
- Custom filtering logic (e.g., web order deduplication, table cover identification by Description)

**Use the skill query templates** (control-erp-sales Templates 1-9, control-erp-financial AR/AP/P&L templates) for common ad-hoc queries. These have been validated against actual database results.

---

## Inferred-Only Reports (No Wiki Documentation)

The following 23 report types appear in `output/wiki/references/crystal-reports-sql-patterns.md` as inferred patterns with no wiki documentation. Their join patterns were inferred from field names and database schema and may not match actual Crystal Report implementations:

Account Activity, Account List, Aging, Artwork Status, Commission Detail, Commission Summary, Customer List, Daily Sales, Employee List, Employee Time, GL Account Balance, Inventory, Item Price List, Order Status, Part Usage, Payment Detail, Payroll, Production Schedule, Profit Margin, Purchase Order, Quote, Shipping Manifest, Station Activity.

---

## Milestone 2 Deferred Items

The following Crystal Reports work is explicitly deferred to Milestone 2:

1. **.rpt file SQL extraction** -- Extracting embedded SQL from the binary .rpt files requires Visual Studio with Crystal Reports SDK on a Windows machine. The Mac development environment cannot parse these files. Once extracted, the SQL will be cross-referenced with skill query patterns to identify discrepancies.

2. **Crystal Report routing skill (RPT-01)** -- A skill that can route user questions to the appropriate built-in report (rather than writing custom SQL) targets Milestone 2. This requires the extracted SQL to understand what each report actually queries.

3. **ReportMenuItem live query** -- This catalog was compiled from existing documentation. A future session with MCP access should run the ReportMenuItem query to capture the full menu hierarchy, active/inactive status, and system report associations directly from the database.

4. **Report parameter documentation** -- The CriteriaOptions, SortingOptions, and OtherOptions fields in ReportMenuItem contain XML-formatted configuration that documents available parameters for each report. Parsing these would provide a complete parameter reference.

---

*Catalog compiled: 2026-02-09. Source: output/report_summary.md, output/reports/*.md, output/wiki/references/crystal-reports-sql-patterns.md, output/schemas/ReportMenuItem.md.*
