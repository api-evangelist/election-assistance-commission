---
name: eac-state-voter-information
description: >-
  Retrieve authoritative per-state voter registration and voting information from the EAC —
  upcoming election dates, registration deadlines, online-registration availability, the
  state's own registration and polling-place lookup URLs, and the NVRA mail registration form
  by language.
api: Election Assistance Commission Content JSON:API
base_url: https://www.eac.gov/jsonapi
auth: none
operations:
  - listNodeRegisterVoteAndContactState
  - listNodeStateElectionInformation
  - listNodeNvra
  - listTaxonomyTermStates
  - listTaxonomyTermOvrStatus
  - listTaxonomyTermNvraLanguages
generated: '2026-09-06'
method: generated
source: openapi/election-assistance-commission-jsonapi-openapi.yml
---

# Answer a voter's "how and when do I register" question, per state

Three collections carry this, and they are the most directly useful data on the EAC's whole
machine surface. All anonymous, all read-only, `Accept: application/vnd.api+json`.

## The registration and deadline record

`listNodeRegisterVoteAndContactState` — `GET /node/register_vote_and_contact_state` — 56 records:
50 states, DC and five territories. One record per jurisdiction, `title` is the state name.

```
GET /node/register_vote_and_contact_state
  ?filter[s][path]=title&filter[s][operator]=CONTAINS&filter[s][value]=Alabama
```

Each record carries up to five elections modelled as repeated field groups, not child records:

- `field_election_1` … `field_election_5` — the election name
- `field_election_1_election_date` … `field_election_5_election_date`
- `field_election_1_registration_de` … `field_election_5_registration_de` — the registration deadline

Plus the links a voter actually needs:

- `field_link_for_register_to_vote`
- `field_registration_lookup`
- `field_polling_place_lookup`
- `field_state_election_office_webs`
- `field_phone_1_800_number`, `field_phone_number`

**Iterate elections 1 through 5 and skip the empty ones.** Not every state populates all five,
and the empty slots are `null`, not absent.

## Online registration availability

`listNodeStateElectionInformation` — `GET /node/state_election_information` — 113 records.
`field_ovr_status` resolves to `taxonomy_term--ovr_status`, which has exactly three terms:

- Online Voter Registration
- No Online Voter Registration
- Voter Registration Not Required

Side-load it rather than making a second call:

```
GET /node/state_election_information?include=field_state,field_ovr_status&page[limit]=50
```

## The NVRA mail form, by language

`listNodeNvra` — `GET /node/nvra` — 155 records. There are more records than states because the
National Mail Voter Registration Form is published PER STATE PER LANGUAGE.
`field_supported_language` resolves to `taxonomy_term--nvra_languages`, which carries three
terms, named in the language itself: `English`, `En Español`, `中文`.

If asked for a form in a language the EAC does not publish, say so — do not fall back to
English silently. Note that the EAC's separate glossary vocabulary covers seven languages while
the NVRA form covers three; they are not the same list.

## Rules

- **Answer with the state's own link, not a summary.** These records exist to route a voter to
  their state election office. Return `field_link_for_register_to_vote` and
  `field_registration_lookup` verbatim.
- **Deadlines are time-critical and the API has no freshness guarantee.** There is no changelog,
  no versioning policy and a `cache-control: max-age=31536000` header. Send `If-None-Match`,
  report the `Last-Modified` value with the answer, and tell the user to confirm against the
  state election office before relying on a date.
- **Read-only.** Every write verb returns HTTP 405. No call in this skill can change anything.
- **Paginate.** `page[limit]` caps at 50; 113 state-election-information records need three pages.
  Follow `links.next`.
- **Check `meta.omitted`** on every collection response before concluding a state has no record.
