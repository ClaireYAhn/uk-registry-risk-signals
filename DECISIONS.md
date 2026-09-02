# Decisions & Observations
A running log of design decisions, the reasoning behind them, and things found while exploring the data. Newest entries at the bottom.

---

## 2026-09-02 — Initial data exploration

Manual walkthrough of a live company record on the Companies House web interface, before writing any collection code.

Sample: company 06852450. Appears to be an ordinary trading company;
useful precisely because the patterns below are routine rather than suspicious.

### 1. Officers with identical appointment and resignation dates
Some officer records show `appointed_on` equal to `resigned_on`.

Hypothesis: an artefact of company formation agents, who appoint nominee officers at incorporation and replace them immediately.

**Schema implication:** resigned officers must be retained in full.
Filtering them out would erase this signal entirely.

### 2. Typo variants in address strings
Observed "Moorhurst Patrners" and "Moorhurst Partners" as the registered address of the same company, corrected via a filing.

Address normalisation is therefore as necessary as name normalisation.
Exact string matching would treat these as distinct addresses. 

### 3. Corporate PSCs (relevant legal entities)
PSCs are not always individuals. Observed two companies registered as PSCs of the sample company.

The ownership graph is therefore multi-level. Reaching a natural person may require recursive traversal through intermediate entities.

### 4. Clustering of change filings
Seven separate changes (address, two directors, secretary, two PSCs) were filed between 2025-08-01 and 2025-08-04.

Candidate feature: density of change events within a rolling window. 

### 5. Formation agent pattern, observed directly
Incorporated 2009-03-19. Nine days later, both initial officers - an individual and a corporate secretary - were terminated and replaced by the actual principals.

Candidate feature: interval between incorporation and departure of initial officers.

Candidate graph signal: frequency with which the same nominee names appear across unrelated companies.

### 6. Corporate directors in historical records
A limited company was appointed as a director in 2012.

ECCTA moves to prohibit corporate directors, so this structure will survive only in historical data. Relevant to how far back the slice should reach.

### 7. Inconsistent name formatting across eras
Older filings are entirely lowercase ("director appointed nickolas garth rimes"); recent ones use title case with honorifics.

The same individual appears as both "Nickolas Rimes" and "Nickolas Garth Rimes" in filings weeks apart.
Concrete evidence of the entity resolution difficulty, visible within a single company's history.

### 8. Undocumented legacy filing formats
Pre-2013 capital records appear as unparseable strings, e.g. `Ad 19/03/09\gbp si 299@1=299\gbp ic 1/300\`
Post-2013 equivalents are structured ("Statement of capital ... GBP 303").

Supports restricting the slice to recently incorporated companies, on data quality grounds rather than convenience.

### 9. Filings carry two distinct dates — most important finding
Each filing has a submission date and an event date, and they diverge.

Example: PSC notifications for events dated 2018-03-03 were not filed until 2019-03-28, a delay of roughly twelve months. The same twelve-month lag recurs for a later PSC notification.

**Schema implication:** both dates must be stored as separate columns. Collapsing them would destroy the ability to measure reporting lag.

Candidate feature: filing delay in days, and its distribution by event type.

### 10. Accounting reference date changes
Extended in 2014 (March to September), shortened back in 2017.

ARD changes have legitimate commercial uses but also defer filing deadlines.

Candidate feature: number of ARD changes, direction, and proximity to an approaching deadline.

### 11. Duplicate confirmation statements
Two confirmation statements filed ten days apart (2019-03-13 and 2019-03-22), both marked "no updates", where one per year is expected.

Possibly an administrative error. Treated as a candidate indicator of filing hygiene rather than wrongdoing.

### 12. Care-of addresses
The registered office moved from a private rural address to two successive "C/O" service addresses in London.

This is entirely routine. The signal is not the address itself but concentration: how many companies share it, and what became of them.

Requires distinguishing legitimate professional service providers from those flagged by the registry.

### 13. Movement between accounting size categories
Filings moved from small company accounts to micro-entity accounts and later to full accounts.

Micro-entity filings disclose the least. Periods of reduced disclosure are worth flagging.

### 14. Company profile returns current state only
No historical addresses or status. The `links` object provides paths to
related endpoints — follow these rather than constructing URLs.

Directly usable flags: `has_charges`, `has_insolvency_history`,
`registered_office_is_in_dispute`, and `overdue` on both accounts and
confirmation statement. `etag` may allow cheap change detection.

### 15. Officers endpoint supplies pre-linked identities — major finding
Each officer carries a `person_number` and a stable officer ID under `links.officer.appointments`.

The same individual appearing twice in this company shares both, meaning Companies House already performs some entity resolution, and the officer appointments endpoint exposes every company that person is linked to.

This is a substantial head start on the graph. It also raises a research question: how reliable is the official matching, and where does it fail?
Do not assume completeness.

### 16. Observation 5 corrected
Filing history suggested initial officers departed nine days after incorporation. The officers endpoint shows `appointed_on` and `resigned_on` both equal to the incorporation date.

The nine-day gap was filing lag, not tenure. Observations 1 and 5 describe the same phenomenon. Reinforces observation 9: event dates and submission dates must never be conflated.

### 17. ECCTA verification data is already exposed
Directors carry `identity_verification_details` with `appointment_verification_start_on`. The company secretary does not.

This is the field to capture either side of the 2026-11-17 deadline.

### 18. Address typo present in live data, not just history
Two directors at the same registered office are recorded as "Moorhurst Partners Llp" and "Moorhurst Patrners Llp" concurrently.

Normalisation is required before any address-based aggregation.

### 19. Corporate officers link to their own company record
The corporate director carries an `identification` block with `registration_number`, allowing traversal into that company's record.

### 20. Pagination
`items_per_page` is 35. Companies with more officers require paging.

### 21. Snapshot depends on the slice, not the schema
The proposal originally named schema definition as the blocking prerequisite for the November snapshot. That was wrong: raw API responses can be stored without any schema at all.

The actual prerequisite is the slice definition (§12.5) — there has to be a target set before anything can be captured.

Revised ordering: disqualification counts → slice definition → snapshot collector → schema → full pipeline.

Proposal §12 and §13 updated accordingly.

### 22. ECCTA changed the registry's role, not just its rules
Read: Companies House, "Then and now: The impact of ECCTA"
https://companieshouse.blog.gov.uk/2025/10/08/then-and-now-the-impact-of-the-economic-crime-and-corporate-transparency-act-on-companies-house

Before the Act, filings were accepted largely on trust with limited verification. The registry now rejects names at incorporation, removes false entries proactively, and can strike off companies formed on a false basis.

Reported enforcement: 100,400 companies had false or misleading information queried or removed between March 2024 and March 2025, and over 10,200 suspicious applications were rejected.

One-line framing for the project background: ECCTA turned Companies House from a passive record keeper into an active gatekeeper.

### 23. Verification scope has a documented gap
Read: GOV.UK, "Companies House confirms identity verification rollout
from 18 November 2025"
https://www.gov.uk/government/news/companies-house-confirms-identity-verification-rollout-from-18-november-2025

Verification for corporate directors, corporate members of LLPs, and officers of corporate PSCs commences later, with no timeline stated.

Layered corporate structures therefore remain outside verification even after the transition period ends on 17 November 2026. Directly relevant to observations 3 and 6.

Supports the argument in proposal §7.6 with a primary government source rather than inference.

---

## Open questions

- Slice definition: which SIC codes, which incorporation window
- History-preserving schema: confirm before any collection begins
- Snapshot strategy around the ECCTA verification deadline (2026-11-17)


