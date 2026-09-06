---
name: eac-voting-system-certification-lookup
description: >-
  Look up the EAC certification status of a voting system, and resolve it to the manufacturer,
  the accredited test laboratory that tested it, the VVSG standard it was tested against, and
  the certification and test-report documents — using the EAC's live, anonymous JSON:API.
api: Election Assistance Commission Content JSON:API
base_url: https://www.eac.gov/jsonapi
auth: none
operations:
  - getResourceIndex
  - listNodeVotingSystem
  - getNodeVotingSystem
  - listNodeManufacturer
  - listNodeVotingSystemTestLab
  - listNodeTestingStandard
  - listTaxonomyTermCertificationStatuses
  - listMediaDocument
  - listFileFile
generated: '2026-09-06'
method: generated
source: openapi/election-assistance-commission-jsonapi-openapi.yml
---

# Look up a voting system's EAC certification

The EAC certifies voting systems against the Voluntary Voting System Guidelines. There are 117
system records. This skill answers "is this system certified, by whom was it tested, against
which standard, and where is the paperwork".

Everything is anonymous. Send no credential. Send `Accept: application/vnd.api+json`.

## 1. Know the vocabulary before you filter

`listTaxonomyTermCertificationStatuses` — `GET /taxonomy_term/certification_statuses` — returns
exactly four terms, and they are the whole state machine:

- Certified System
- Systems Under Test
- Terminated Systems
- Withdrawn or Decertified Systems

There is no provisional or conditional state. If a system is not in one of these four, it is
not in the programme.

## 2. Find the system

`listNodeVotingSystem` — `GET /node/voting_system`

Filter by name fragment:

```
GET /node/voting_system
  ?filter[t][path]=title
  &filter[t][operator]=CONTAINS
  &filter[t][value]=EVS
  &page[limit]=50
```

An unknown field in a filter returns HTTP 400 with
`Invalid nested filtering. The field ... does not exist.` Read the field names off a resource
object first; they are Drupal machine names, almost all prefixed `field_`.

## 3. Resolve the relationships in one call

Do not make four calls. Side-load:

```
GET /node/voting_system
  ?filter[t][path]=title&filter[t][operator]=CONTAINS&filter[t][value]=EVS
  &include=field_manufacturer,field_testing_lab,field_certification_status,field_testing_standard
```

The related objects come back in the top-level `included` array. Verified 2026-09-06:
`field_manufacturer` resolves to `node--manufacturer`, `field_testing_lab` to
`node--voting_system_test_lab`, `field_certification_status` to
`taxonomy_term--certification_statuses`.

Read these attributes off the system record:

- `title`, `field_version`
- `field_testing_status`
- `field_testing_application_date`, `field_application_accepted_date`
- `field_certification_date`
- `field_withdrawn_decertified_date`, `field_terminated_date`

A system with a `field_certification_date` and no withdrawal or termination date, whose
`field_certification_status` is "Certified System", is currently certified.

## 4. Get the documents

Document relationships are DOUBLED. `field_certification` and `field_certification_media`,
`field_test_report` and `field_test_report_media`, and so on for advisory notices,
correspondence, testing documents, termination and withdrawal documents.

**Read the `_media` variants.** They resolve to `media--document` (`listMediaDocument`), which
carries `field_media_document` pointing at `file--file` (`listFileFile`), whose `uri.url` is the
downloadable path. Two hops:

```
GET /node/voting_system/{id}?include=field_certification_media.field_media_document
```

## 5. Get one system by id

`getNodeVotingSystem` — `GET /node/voting_system/{id}` where `{id}` is the JSON:API UUID from a
collection response, NOT `drupal_internal__nid`. An unknown UUID returns HTTP 404 with a
JSON:API error document.

## Rules

- **This API is read-only.** Every write verb returns HTTP 405
  (`JSON:API is configured to accept only read operations`). Nothing you do here can change
  anything, so no confirmation step is needed before any call in this skill.
- **Paginate.** `page[limit]` is capped at 50. Follow `links.next`. No total count is returned;
  if you need one, walk to exhaustion.
- **Check `meta.omitted`.** A 200 with empty `data` plus a `meta.omitted` block means "not
  permitted", not "no records".
- **Retry 503 up to three times, then move on.** Some collections on this host return 503
  persistently rather than transiently.
- **Cache with ETag.** Responses carry `cache-control: max-age=31536000`. Use
  `If-None-Match` rather than trusting a year-old copy — certification status does change.
- **Manufacturer and test-lab records carry named individuals and phone numbers.** They are
  public records; still treat them as personal data and do not republish them.
- **Cite the record.** The EAC does not document this API, so any answer built from it should
  name the collection and the date it was read.
