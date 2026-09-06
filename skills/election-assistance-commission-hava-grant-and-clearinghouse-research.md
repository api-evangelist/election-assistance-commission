---
name: eac-hava-grant-and-clearinghouse-research
description: >-
  Research HAVA grant awards and financial reporting by state and fiscal year, and search the
  561 Clearinghouse (Clearie) Award entries recognising election office programmes, from the
  EAC's live JSON:API.
api: Election Assistance Commission Content JSON:API
base_url: https://www.eac.gov/jsonapi
auth: none
operations:
  - listNodeEsFunds
  - listNodeStateFunds
  - listNodeClearinghouseAwards
  - listNodeEngineeringChangeOrder
  - listMediaDocument
  - listFileFile
  - listTaxonomyTermStates
generated: '2026-09-06'
method: generated
source: openapi/election-assistance-commission-jsonapi-openapi.yml
---

# Research HAVA grants and Clearinghouse Awards

## HAVA grant financials

Two parallel collections, 56 records each (50 states, DC, five territories):

- `listNodeEsFunds` — `GET /node/es_funds` — Election Security grant reporting
- `listNodeStateFunds` — `GET /node/state_funds` — state grant financials

`node--state_funds` carries the money: `field_total_award`, `field_federal_share`,
`field_state_share_20_percent`, `field__total_payment` (note the double underscore).

**The schema is fiscal-year-shaped, not record-shaped.** Reporting is one field PER YEAR —
`field_financial_report_fy18`, `field_financial_report_fy20`, `field_financial_report_fy21_semi`,
`field_financial_report_fy23_q1`, `field_financial_report_fy24_q3`, plus
`field_request_letter_2018`/`_2020`/`_2022`/`_2023` and `field_state_narrative_*`. There is no
child entity for a reporting period.

Consequences you must handle:

1. **Enumerate fields dynamically.** Fetch one record, read the `attributes` keys, and select
   the ones matching `field_financial_report_*` / `field_request_letter_*` /
   `field_state_narrative_*`. Do not hard-code a year list — the EAC adds a field each year.
2. **A missing year is a missing FIELD, not a null value**, on older records.
3. **Do not sum across years** without checking whether a value is an annual, semi-annual or
   quarterly report; the suffixes (`_semi`, `_q1`, `_q3`) distinguish them and they overlap.

Documents hang off `field_document_media` → `media--document` → `file--file` → `uri.url`:

```
GET /node/state_funds?include=field_document_media&page[limit]=50
```

## Clearinghouse (Clearie) Awards

`listNodeClearinghouseAwards` — `GET /node/clearinghouse_awards` — 561 entries.

Attributes: `field_program_title`, `field_program_summary`, `field_year`, `field_link`,
`field_unique_hash`. Four taxonomy relationships:

- `field_award` → `taxonomy_term--ch_award` (two terms: Winner, Honorable Mention)
- `field_ch_category` → `taxonomy_term--ch_category` (27 terms, e.g. Accessibility,
  Cybersecurity & Technology, Exemplary Services to Assist Military and Overseas Voters)
- `field_ch_state` → `taxonomy_term--ch_state`
- `field_office` → `taxonomy_term--ch_office`

Filter by year and side-load the taxonomies in one call:

```
GET /node/clearinghouse_awards
  ?filter[field_year]=2024
  &include=field_award,field_ch_category,field_ch_state,field_office
  &page[limit]=50
```

**Category names are not stable across years.** "Distinguished Voter Education and
Communications Initiatives" also appears split into "- Large" and "- Small/Medium" variants, and
"Creative and Original 'I Voted' Stickers" appears with and without quote marks. Group by
normalised category name, or report the raw term and say it is the EAC's own label.

## Engineering Change Orders (bonus: the one latency measure available)

`listNodeEngineeringChangeOrder` — `GET /node/engineering_change_order` — 275 records, each with
three dates:

- `field_date_submitted_by_manufact`
- `field_date_approved_by_vstl`
- `field_date_approved_by_eac`

This is the only place in the EAC's machine estate where regulatory review latency can be
computed. Sort newest first with `?sort=-field_date_approved_by_eac`.

## Rules

- **Read-only.** Every write verb returns HTTP 405; no call here has a side effect.
- **Paginate.** 561 award records is 12 pages at the `page[limit]` cap of 50. Follow
  `links.next`; no total count is returned.
- **Check `meta.omitted`.** `node--state_payment_requirements` is advertised but withheld from
  anonymous callers — it returns 200 with empty `data` and an omitted block, not a 403. Do not
  report "no payment requirement records exist".
- **These are financial and award records with no published freshness guarantee.** Report the
  `Last-Modified` header alongside any figure, and link the source document rather than
  paraphrasing a dollar amount.
- **Cite the collection and the read date.** The EAC does not document this API and makes no
  commitment about it.
