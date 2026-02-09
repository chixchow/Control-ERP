# Phase 6: Glossary Skill Creation - Research

**Researched:** 2026-02-08
**Domain:** FLS Banners business terminology and Control ERP technical terminology mapped to database entities
**Confidence:** HIGH

## Summary

Phase 6 creates a `control-erp-glossary` skill that serves as a translation layer between human-spoken FLS business language and the database queries/entities that answer those questions. The research reveals that all source material already exists across three completed skills (`control-erp-core`, `control-erp-sales`, `control-erp-financial`) and 12 wiki knowledge extracts. The glossary is a consolidation and indexing effort, not a discovery effort.

The glossary has two distinct halves: (1) FLS product line terminology -- mapping casual business names like "blankets," "SEG," "Fab Frame," "step and repeat" to the specific database queries that find those products, and (2) Control-specific technical terms -- defining CHAPI, SSLIP, CFL, SQLBridge, ClassTypeID, GoodsItemClassTypeID, and other system concepts with their database relationships.

All three requirements (GLOSS-01, GLOSS-02, GLOSS-03) can be satisfied from existing knowledge. No database queries, external research, or new discovery is needed. The work is pure synthesis and formatting.

**Primary recommendation:** Build the glossary as a single SKILL.md file with YAML frontmatter following the established pattern. Organize it as three lookup sections: (1) FLS product terminology to database queries, (2) Control technical terminology with database mappings, and (3) a natural language routing table that maps user questions to the right skill and query pattern.

## Standard Stack

Not applicable -- this phase produces a Markdown skill file, not code.

### Core
| Artifact | Location | Purpose | Status |
|----------|----------|---------|--------|
| control-erp-core-SKILL.md | `skills/control-erp-core/` | Business rules, TransactionType, StatusID | VERIFIED (Phase 1) |
| control-erp-sales-SKILL.md | `skills/control-erp-sales/` | Product architecture, FP_ variables, categories | VERIFIED (Phase 4) |
| control-erp-financial-SKILL.md | `skills/control-erp-financial/` | GL/Ledger, AR/AP, payment terms | EXISTS (Phase 5 pending) |
| Wiki knowledge extracts | `output/wiki/extracts/` | 12 extract files with all Control terminology | COMPLETE (Phase 3) |

### Supporting
| Source | Location | What It Provides |
|--------|----------|-----------------|
| database_integration_knowledge.md | `output/wiki/extracts/` | CHAPI, SSLIP, SQLBridge, ClassTypeID (350+ entries) |
| products_pricing_knowledge.md | `output/wiki/extracts/` | Product/Part/Variable/Modifier architecture |
| cfl_formula_language_knowledge.md | `output/wiki/extracts/` | CFL syntax, functions, property references |
| orders_accounting_knowledge.md | `output/wiki/extracts/` | Order lifecycle, status transitions, GL flows |
| macros_automation_knowledge.md | `output/wiki/extracts/` | Macro system, message types, event triggers |
| output/skill/SKILL.md | `output/skill/` | Schema reference with ClassTypeID quick reference |

## Architecture Patterns

### Skill File Structure (follow established pattern)
```
skills/control-erp-glossary/
  control-erp-glossary-SKILL.md   # Single file with YAML frontmatter
```

All three existing skills use this identical pattern: a single `*-SKILL.md` file in a dedicated subdirectory under `skills/`. No `references/` subdirectory is used in the skill directories (that pattern exists only in `output/skill/`).

### YAML Frontmatter Pattern (must follow exactly)
```yaml
---
name: control-erp-glossary
description: Maps FLS Banners business language and Control ERP technical terminology to database entities, queries, and the right skill to consult. Use when users reference FLS product names (blankets, SEG, Fab Frame, DyeLux, table throw, step and repeat), Control-specific terms (CHAPI, SSLIP, CFL, ClassTypeID), or when you need to route a question to the correct skill.
---
```

### Pattern 1: Term-to-Query Mapping
**What:** Each FLS product term maps directly to the SQL query pattern or Description LIKE pattern that finds it in the database.
**When to use:** For GLOSS-02 (FLS product terminology).
**Example:**
```markdown
### Blanket
**What it is:** A fleece or knit fabric product, typically used for promotional giveaways.
**Database query:** `WHERE td.Description LIKE '%Blanket%'`
**Product container:** Non-DyeSub (standalone product)
**See also:** control-erp-sales for revenue queries
```

### Pattern 2: Technical Term Definition with Database Link
**What:** Each Control technical term gets a plain-English definition plus its database relationship.
**When to use:** For GLOSS-03 (Control-specific terminology).
**Example:**
```markdown
### ClassTypeID
**What it is:** A polymorphic type identifier integer used throughout the Control database to identify what type of entity a record represents. Every major table has a ClassTypeID column.
**Why it matters:** Many tables (Journal, PricingElement, CustomerGoodsItem) store multiple entity types distinguished only by ClassTypeID. Without filtering by ClassTypeID, queries return mixed results.
**Database column:** `ClassTypeID` (int) on nearly every table
**Key values:** 10000=TransHeader, 10100=TransDetail, 2000=Account, 12000=Product, 12014=Part
**Full reference:** See database_integration_knowledge.md for all 350+ ClassTypeID mappings
```

### Pattern 3: Natural Language Routing Table
**What:** A lookup table that maps common user phrases to the skill that handles them and the query approach.
**When to use:** When Claude needs to decide which skill to consult.
**Example:**
```markdown
| User Says | Route To | Key |
|-----------|----------|-----|
| "Find blanket orders" | control-erp-sales | Description LIKE '%Blanket%' |
| "What is a SSLIP?" | control-erp-glossary (this skill) | See SSLIP definition |
| "AR aging" | control-erp-financial | AR Aging Buckets query |
```

### Anti-Patterns to Avoid
- **Duplicating query templates:** The glossary should REFERENCE the sales/financial/core skills, not copy their queries. Include only the minimal identification pattern (e.g., the WHERE clause or LIKE pattern), not full query templates.
- **Duplicating business rules:** The core skill owns TransactionType mappings, StatusID values, price field rules. The glossary points to them, never restates them.
- **Creating a "knowledge dump":** The glossary is a lookup tool, not an encyclopedia. Each entry should be concise (3-6 lines) with pointers to the skill that has the full detail.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| FLS product catalog | Querying the database for product names | Validated product categories from sales skill | The sales skill has validated 2025 revenue for every category against the Control "Sales by Product" report |
| ClassTypeID reference | Building from database metadata | Wiki extract `database_integration_knowledge.md` | 350+ mappings already compiled from the authoritative Cyrious wiki |
| Control technical definitions | Writing from training data memory | Wiki extracts (12 files) | Wiki documentation is the authoritative source for CHAPI, SSLIP, CFL, SQLBridge definitions |
| TransactionType/StatusID mappings | Redefining in glossary | Reference to control-erp-core | Core skill owns these, validated to 99.98% |

**Key insight:** The glossary skill is a cross-referencing and routing tool. Every fact it contains should trace back to one of the three existing skills or the wiki extracts. If the glossary contains a claim not backed by those sources, it is wrong.

## Common Pitfalls

### Pitfall 1: Terminology Overlap Between Skills
**What goes wrong:** The glossary defines "DyeSub Print" with revenue figures, but the sales skill also defines it. Now they can diverge.
**Why it happens:** Natural temptation to make the glossary self-contained.
**How to avoid:** The glossary entry for "DyeSub Print" says WHAT it is and WHERE to find it. It does NOT include revenue figures, query templates, or category breakdowns. Those live in the sales skill.
**Warning signs:** If a glossary entry exceeds 10 lines, it is probably duplicating another skill.

### Pitfall 2: Missing Product Terms
**What goes wrong:** A user says "step and repeat" and Claude does not know this refers to large-format backdrop banners.
**Why it happens:** The term inventory was built from database product names, not from how salespeople and customers actually talk.
**How to avoid:** Include BOTH the formal product category name (from FP_ProductDescription: "Backdrop") AND the common business names that FLS staff use ("step and repeat," "backdrop banner," "photo backdrop"). Source these from the sales skill's validated categories and the product descriptions in TransDetail.
**Warning signs:** If every glossary term matches a database column name exactly, colloquial terms are missing.

### Pitfall 3: Stale Revenue or Count Numbers in the Glossary
**What goes wrong:** The glossary says "Swing Flags: $615K in 2025" and this number becomes stale in 2026.
**Why it happens:** Copying specific numbers from the sales skill into the glossary.
**How to avoid:** The glossary should say "Swing Flags: the largest DyeSub Print category" without specific dollar amounts. Revenue figures belong in the sales skill where they can be updated.
**Warning signs:** Dollar amounts or order counts appearing in glossary entries.

### Pitfall 4: Incomplete Control Technical Terms
**What goes wrong:** The glossary defines CHAPI and SSLIP but misses UDF, SeqID, PricingPlan, Macro, Variation, and other terms that users commonly ask about.
**Why it happens:** Only the explicitly required terms (CHAPI, SSLIP, CFL, SQLBridge, ClassTypeID, GoodsItemClassTypeID) are documented.
**How to avoid:** Include the full set of Control-specific concepts that a user might encounter. The wiki extracts identify all of these. A good test: if a user reads a Control error message or wiki page and sees a term, can the glossary explain it?

### Pitfall 5: Wrong Skill Routing
**What goes wrong:** The glossary routes "cost of goods" to the sales skill instead of the financial skill.
**Why it happens:** Ambiguous terms that could be handled by multiple skills.
**How to avoid:** For each routing entry, specify WHICH aspect each skill handles. "Revenue by product" goes to sales; "COGS from GL" goes to financial; "SubTotalPrice vs TotalPrice" goes to core.

## Code Examples

No code is produced in this phase. The deliverable is a Markdown skill file. Here are the key structural patterns:

### FLS Product Term Entry Format
```markdown
### Blanket
**Business meaning:** Promotional fleece or knit blankets, typically custom-printed with logos
**Aliases:** throw blanket, promotional blanket, fleece blanket
**Product container:** Non-DyeSub standalone product
**Database identification:** `TransDetail.Description LIKE '%Blanket%'`
**Revenue skill:** control-erp-sales (Template 8: Full Revenue Reconciliation)
```

### Control Technical Term Entry Format
```markdown
### CHAPI (Cyrious Host Application Programming Interface)
**What it is:** The server-side service that coordinates database writes between SQL Bridge, the SSLIP, and all connected Control clients. Runs as an HTTP endpoint on port 12556.
**Database functions:** `csf_chapi_nextid()`, `csf_chapi_nextnumber()`, `csf_chapi_lock()`, `csf_chapi_unlock()`, `csf_chapi_refresh()`, `csf_chapi_recompute()`
**Key relationship:** SQL Bridge stored procedures call CHAPI functions; CHAPI notifies SSLIP and Control clients of changes
**Detailed reference:** output/wiki/extracts/database_integration_knowledge.md, Section 5
```

### Natural Language Routing Entry Format
```markdown
| "find blanket orders" | control-erp-sales | Description LIKE '%Blanket%' pattern |
| "how much did we sell in feather flags" | control-erp-sales | Template 3 filtered by FP_ProductDescription |
| "what is a ClassTypeID" | control-erp-glossary | See ClassTypeID entry |
| "show me AR aging" | control-erp-financial | AR Aging Buckets query |
| "what TransactionType is an estimate" | control-erp-core | TransactionType Reference table |
```

## Inventory of FLS Product Terms (from existing sources)

These are the validated FLS product line terms that MUST appear in the glossary, sourced from the sales skill's validated 2025 categories:

### DyeSub Print Categories (via FP_ProductDescription, VariableID 11053)
| Term | Aliases to Include | DB Pattern |
|------|-------------------|------------|
| Swing Flags | swing flag, swooper | FP_ProductDescription = 'Swing Flags' |
| Feather Flags | feather flag, feather banner | FP_ProductDescription = 'Feather Flags' |
| Banners | banner, vinyl banner, fabric banner | FP_ProductDescription = 'Banners' |
| SEG | silicone edge graphics, SEG frame, seg graphic | FP_ProductDescription = 'SEG' |
| Tear Drop Flags | teardrop, tear drop | FP_ProductDescription = 'Tear Drop Flags' |
| Banner Stands | banner stand, retractable banner, roll-up banner | FP_ProductDescription = 'Banner Stands' |
| Fab Frames | fab frame, fabric frame, tension frame | FP_ProductDescription = 'Fab Frames' |
| Custom Flag | custom flags | FP_ProductDescription = 'Custom Flag' |
| Golf Flag 14x20 | golf flag, course flag | FP_ProductDescription = 'Golf Flag 14x20' |
| Table Runner | table runner | FP_ProductDescription = 'Table Runner' |
| Pillow Case Frames | pillowcase, pillow case | FP_ProductDescription = 'Pillow Case Frames' |
| Pop Up Banners | popup banner, pop-up | FP_ProductDescription = 'Pop Up Banners' |
| Backdrop | step and repeat, photo backdrop, backdrop banner | FP_ProductDescription = 'Backdrop' |
| Golf Flag 5x8 | putting green flag | FP_ProductDescription = 'Golf Flag 5x8' |
| By The Yard | fabric by the yard | FP_ProductDescription = 'By The Yard' |
| Golf Cart Flag | cart flag | FP_ProductDescription = 'Golf Cart Flag' |
| Bell Covers | bell cover, instrument cover | FP_ProductDescription = 'Bell Covers' |
| Swoopper Flags | swooper, swoopper | FP_ProductDescription = 'Swoopper Flags' |
| Trombone cover | trombone | FP_ProductDescription = 'Trombone cover' |
| Printed Dyesub Paper | dyesub paper, transfer paper | FP_ProductDescription = 'Printed Dyesub Paper' |

### Non-DyeSub Product Groups (via Description LIKE patterns)
| Term | Aliases | DB Pattern |
|------|---------|------------|
| DyeLux Table Cover | dyelux, full print table cover, fitted table cover | Description LIKE '%Dyelux%Table Cover%' OR '%FULL%Table Cover%' |
| Table Throw | table throw, table drape | Description LIKE '%Table Throw%' |
| Table Cover (generic) | table cover | Description LIKE '%Table Cover%' |
| Garments/Apparel | garment, embroidered, t-shirt, apparel, clothing | Description LIKE '%Garment%' OR '%Embroidered%' |
| Tent/Pop-Up | tent, pop-up tent, canopy | Description LIKE '%Tent%' OR '%Pop%Up%' |
| Design/Artwork | design fee, artwork, art charge | Description LIKE '%Design%' OR '%Artwork%' |

### Container Product Concepts
| Term | Definition |
|------|-----------|
| Container product | A product in Control that acts as a configurable shell housing many actual product lines through variables (e.g., DyeSub Print) |
| DyeSub Print | The primary container product at FLS, housing 20+ product categories via FP_ProductDescription variable. NOT a product itself -- never report "DyeSub Print" as a product name. |
| FP_ProductDescription | The TransDetailParam variable (VariableID 11053) that identifies the actual product category within DyeSub Print |
| FP_ProductID | The TransDetailParam variable (VariableID 11052) that identifies the specific SKU within DyeSub Print |

## Inventory of Control Technical Terms (from wiki extracts)

These are the Control-specific terms that MUST appear in the glossary:

### Required by GLOSS-03
| Term | Source | Database Relationship |
|------|--------|----------------------|
| CHAPI | database_integration_knowledge.md sec 5 | csf_chapi_* functions, HTTP endpoint port 12556 |
| SSLIP | database_integration_knowledge.md sec 6 | Server process: recompute, macros, Production Terminal, WebView |
| CFL | cfl_formula_language_knowledge.md | PricingPlan formula fields, Variable formulas, IOTemplate |
| SQLBridge | database_integration_knowledge.md sec 1 | csf_chapi_* stored procedures, csp_Import* procedures |
| ClassTypeID | database_integration_knowledge.md sec 2 | Every table's ClassTypeID column; 350+ type mappings |
| GoodsItemClassTypeID | control-erp-core TransDetail section | TransDetail.GoodsItemClassTypeID: 12000=Product(catalog), 49=Product, 30=Part |

### Additional Important Terms (should include for completeness)
| Term | Source | Why Important |
|------|--------|---------------|
| SeqID | wiki / database pattern | Version tracking -- every update increments SeqID; history tables use (ID, SeqID) composite |
| TransactionType | control-erp-core | 1=Order, 2=Estimate, 6=Service Ticket, 7=PO, 8=Bill, 9=Receiving |
| StatusID | control-erp-core | Status lifecycle values per TransactionType |
| SubTotalPrice | control-erp-core | Pre-tax revenue field (NOT TotalPrice) |
| SaleDate | control-erp-core | Revenue date field (NOT OrderCreatedDate) |
| UDF (User-Defined Field) | udfs_custom_fields_knowledge.md | Custom fields on Account, Contact, TransHeader, Product, Part |
| TransHeaderUserField | core skill / wiki | UDF storage table for orders (e.g., Import_Order_Number) |
| Macro | macros_automation_knowledge.md | RuleMacro: automation triggers on events (status change, payment, etc.) |
| Variation | orders_accounting_knowledge.md | TransVariation: alternate pricing configs within an estimate |
| Pricing Plan | products_pricing_knowledge.md | Different pricing formulas per customer group |
| Pricing Level | products_pricing_knowledge.md | Percentage adjustment (invisible to customer) |
| Promotion | products_pricing_knowledge.md | Visible discount applied to orders |
| Station | production knowledge | Production workflow stage (Station table, ClassTypeID 26100) |
| Closeout | orders_accounting_knowledge.md | Daily/Monthly/Yearly closeout timestamps for accounting periods |
| GL View | control-erp-financial | GL = VIEW on Ledger WHERE OffBalanceSheet = 0 |
| Ledger | control-erp-financial | The actual GL table (includes off-balance-sheet entries) |
| Journal | control-erp-financial | Activity/event log table linked to payments, time cards, notes |
| Payment | control-erp-financial | Payment records (Payment.ID = Journal.ID split table) |
| Division | core skill | Multi-division context: Company (Banners/Signs) vs Apparel |
| Container Product | control-erp-sales | Configurable product shell (DyeSub Print, DyeLux Table Cover) |
| FP_ Variables | control-erp-sales | FP_ProductDescription (11053), FP_ProductID (11052) |
| Import_Order_Number | core skill | UDF identifying web-imported orders via CHAPI |
| IsActive | core skill / CLAUDE.md | Standard active record filter -- but NOT for TransDetailParam |
| History Tables | core skill | *History tables with (ID, SeqID) composite keys for audit trails |
| Polymorphic Pattern | schema patterns | ParentID + ParentClassTypeID pattern used throughout Control |
| CustomerGoodsItem | products_pricing_knowledge.md | The actual table name for Products (ClassTypeID 12000) and Modifiers (ClassTypeID 12010) |
| PricingElement | products_pricing_knowledge.md | General-purpose table storing categories, families, plan types |
| CRIX | SSLIP docs | Cyrious Report Integration eXchange |
| Production Terminal | SSLIP docs | Web-based production workflow interface served by SSLIP |
| WebView | SSLIP docs | Customer-facing web portal served by SSLIP |

## Order Lifecycle Terminology (for routing table)

These workflow terms should be in the glossary for routing purposes:

| Term | Meaning | Route To |
|------|---------|----------|
| Estimate / Quote | Type 2 transaction, pre-sale | core (TransactionType 2) |
| Order / Sale | Type 1 transaction, revenue record | core (TransactionType 1) |
| WIP | Work in Progress (StatusID 1) | core (StatusID reference) |
| Built | Production complete (StatusID 2) | core (StatusID reference) |
| Sale (status) | Invoiced, revenue recognized (StatusID 3) | core (StatusID reference) |
| Closed | Fully paid (StatusID 4) | core (StatusID reference) |
| Voided | Cancelled (StatusID 9) | core (StatusID reference) |
| Converted | Estimate became an order (StatusID 13) | core (estimate lifecycle) |
| Lost | Estimate not won (StatusID 12) | core (estimate lifecycle) |
| Pending | Active estimate (StatusID 11) | core (estimate lifecycle) |
| PO / Purchase Order | Vendor purchasing (Type 7) | core (TransactionType 7) |
| Bill | Vendor invoice (Type 8) | financial (AP section) |
| AR / Accounts Receivable | Unpaid invoices | financial (AR section) |
| AP / Accounts Payable | Unpaid vendor bills | financial (AP section) |
| P&L / Profit and Loss | Income statement | financial (P&L section) |
| COGS | Cost of Goods Sold | financial (GLClassificationType 5001) |

## Sizing Estimate

Based on the existing skills as reference points:

| Skill | Lines | Sections |
|-------|-------|----------|
| control-erp-core | 348 | 12 sections |
| control-erp-sales | 368 | 6 sections |
| control-erp-financial | 576 | 10 sections |
| **control-erp-glossary (estimated)** | **250-350** | **5-6 sections** |

The glossary should be SHORTER than the other skills because it contains definitions and pointers, not query templates. Target sections:
1. YAML frontmatter + introduction
2. FLS Product Line Terminology (GLOSS-02)
3. Control Technical Terminology (GLOSS-03)
4. Order Lifecycle & Workflow Terminology
5. Natural Language Routing Table
6. Cross-references to other skills

## Open Questions

1. **"Step and repeat" mapping:** The sales skill lists "Backdrop" as a DyeSub Print category ($5,823). Industry standard term "step and repeat" likely maps here, but this should be confirmed by checking actual TransDetail descriptions for backdrop orders. LOW impact -- can be verified during execution.

2. **Apparel sub-terminology:** FLS has an Apparel division. Do users refer to specific garment types (polos, t-shirts, hoodies) that need glossary entries? The sales skill groups everything under "Garments/Apparel" without sub-categories. This is a discretion area -- include common garment terms pointing to the Apparel division.

3. **TC_ variables:** The sales skill documents that TC_ variables (TC_FabricCategory, TC_ProductName) are NOT saved to TransDetailParam. The glossary should note this as a "known limitation" under the DyeLux Table Cover entry. Users asking about table cover fabric types will hit this wall.

## Sources

### Primary (HIGH confidence)
- `skills/control-erp-core/control-erp-core-SKILL.md` -- TransactionType, StatusID, business rules, Natural Language mapping
- `skills/control-erp-sales/control-erp-sales-SKILL.md` -- Product architecture, FP_ variables, 20 DyeSub categories, revenue validation
- `skills/control-erp-financial/control-erp-financial-SKILL.md` -- GL/Ledger, AR/AP, payment terms, GLClassificationType
- `output/wiki/extracts/database_integration_knowledge.md` -- CHAPI, SSLIP, SQLBridge, ClassTypeID (350+), stored procedures
- `output/wiki/extracts/products_pricing_knowledge.md` -- Product/Part/Variable/Modifier architecture, CFL usage
- `output/wiki/extracts/cfl_formula_language_knowledge.md` -- Full CFL reference
- `output/wiki/extracts/orders_accounting_knowledge.md` -- Order lifecycle, GL transaction flows
- `output/wiki/extracts/macros_automation_knowledge.md` -- Macro system architecture

### Secondary (HIGH confidence -- same project, prior phases)
- `output/skill/SKILL.md` -- Schema overview, ClassTypeID quick reference
- `output/wiki/KNOWLEDGE_BASE.md` -- Wiki crawl metadata and extract index
- `.planning/phases/01-core-skill-verification/01-RESEARCH.md` -- Phase 1 approach (verification pattern)
- `.planning/phases/04-sales-skill-verification/04-RESEARCH.md` -- Phase 4 approach (validation pattern)

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH -- all source material exists and has been validated in prior phases
- Architecture: HIGH -- skill file pattern is well-established across 3 existing skills
- Content inventory (FLS terms): HIGH -- validated product categories from sales skill with 2025 revenue data
- Content inventory (Control terms): HIGH -- 350+ ClassTypeIDs and all technical terms from wiki extracts
- Pitfalls: HIGH -- identified from patterns observed in prior phase execution

**Research date:** 2026-02-08
**Valid until:** Indefinite (source material is internal project artifacts, not external libraries)
