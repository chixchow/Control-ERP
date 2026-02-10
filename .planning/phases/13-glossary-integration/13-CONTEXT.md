# Phase 13: Glossary Integration + Reports Catalog - Context

**Gathered:** 2026-02-09
**Status:** Ready for planning

<domain>
## Phase Boundary

Users asking any business question get routed to the correct skill, and users asking about reports get guidance on which Crystal Report to run vs. when to use a custom query. All 36 cataloged Crystal Reports are documented with coverage status. A gap log tracks uncovered domains for future skill development.

</domain>

<decisions>
## Implementation Decisions

### Domain routing rules
- When a query is ambiguous (e.g., "top customers" could route to sales, financial, or customers), ask the user to clarify rather than guessing
- Cross-domain queries should combine results from multiple skills into a unified answer
- Routing approach (auto-execute vs recommend) and domain boundary strictness are Claude's discretion, optimized for token efficiency

### Reports catalog depth
- Crystal Reports are the **source of proven query patterns** — skills are their replacement, not companions
- Each report entry includes: name, category, purpose, extracted SQL, and coverage status (Replaced by [skill] / Partially covered / Not yet covered)
- Include the actual extracted SQL from reports — these are 15+ years of battle-tested queries that skills should learn from
- Coverage tracking creates a migration roadmap showing which reports are fully replaced by skills
- When a report isn't covered by a skill yet, acknowledge the gap honestly — do not attempt to synthesize unvalidated data
- Report placement (centralized vs distributed across domain skills) is Claude's discretion, optimized for token efficiency
- Whether to re-analyze .rpt files or use existing extraction (report_summary.md, output/reports/*.md) is Claude's discretion

### Routing coverage & gaps
- **CRITICAL: Never guess.** Employees take LLM output as fact. If a query falls outside covered domains, say it's not covered — do not attempt unvalidated answers
- Maintain a separate **gap log file** tracking every uncovered domain with example queries that triggered it — easiest to check, informs future skill development
- Build 20+ test queries for routing validation (success criteria: 90%+ accuracy)
- Test queries should reflect **mixed team** usage: sales, production, accounting, management — wide range of vocabulary and question styles
- Include realistic language: typos, casual phrasing, abbreviations — how people actually type

### Report discovery UX
- **Natural language first** — users ask questions, glossary finds matching reports
- When ambiguous, show a **ranked list by user intent** — best match first with brief descriptions
- **LLM answers first** using skills. Only recommend running the Crystal Report in Control when it genuinely makes more sense (complex formatting, capability gaps)
- Report entries show name and purpose only — no parameter details (users already know how to run their reports)

### Claude's Discretion
- Routing mechanism: auto-execute skill vs recommend skill to user
- Domain boundary design: strict vs overlapping, optimized for routing accuracy
- Report catalog placement: centralized in glossary vs distributed to domain skills vs hybrid — decide based on token efficiency
- Whether existing report analysis is sufficient or .rpt files need re-extraction

</decisions>

<specifics>
## Specific Ideas

- Crystal Reports have been running for 15+ years — their SQL is proven correct. The catalog should preserve these query patterns so skills can build on what works rather than reinventing
- The glossary is fundamentally a **routing layer** — it maps questions to the right skill, not a skill itself
- Gap logging creates a natural backlog: every "I don't know" becomes a future skill requirement
- Test queries from a mixed team audience: casual sales rep asking "whos our biggest customer" should route just as well as a manager asking "show me quarterly revenue by product line"

</specifics>

<deferred>
## Deferred Ideas

### Query access control (future milestone)
- Role-based restrictions on sensitive queries (e.g., employees shouldn't see balance sheet, gross/net profit)
- Guard rails on who can ask what type of questions
- Will be required at some point but not for initial v1.1 release

</deferred>

---

*Phase: 13-glossary-integration*
*Context gathered: 2026-02-09*
