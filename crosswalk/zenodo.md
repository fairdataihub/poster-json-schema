# poster_schema.json to Zenodo Crosswalk

Mapping between [poster_schema.json v0.2](../poster_schema.json) (DataCite 4.7) and the [Zenodo REST API deposit metadata](https://developers.zenodo.org/#deposit-metadata).

Use this document as the reference when building or updating the Zenodo metadata payload in `posters-science/server/utils/zenodo.ts`.

Last verified against production record [20185965](https://zenodo.org/records/20185965) on 2026-05-14.

## Data flow

poster_schema.json fields are extracted by poster2json and stored in the `PosterMetadata` table (Prisma). The Zenodo publish path in `zenodo.ts` reads from this table, not from a poster_schema.json file directly. Some fields (like `title` and `description`) live on the `Poster` model itself rather than `PosterMetadata`.

```
poster PDF
  -> poster2json (extraction + normalization)
  -> posters-science API (stores to Poster + PosterMetadata tables)
  -> zenodo.ts (reads DB, builds Zenodo deposit payload)
  -> Zenodo REST API
```

### Two-phase upload

The code writes metadata to Zenodo in two passes:

1. **Legacy deposit API** (`PUT /deposit/depositions/{id}`) sends the core metadata. This API has limited support for structured fields (single affiliation string per creator, combined name field, no name type, no funder name, no description type).

2. **InvenioRDM API** (`PUT /records/{id}/draft`) via `convertLegacyDraftToRdmPayload()` then overwrites the draft with richer data. This second pass sends:
   - Separate `given_name` and `family_name` for each creator
   - `person_or_org.type` ("personal" or "organizational") for each creator
   - All affiliations per creator (with optional ROR IDs), not just the first
   - ORCID identifiers with checksum validation
   - Funder name and optional ROR-based funder ID
   - Award number and award title
   - `submissionAbstract` as `additional_descriptions` with type "abstract"
   - `resource_type: {id: "poster"}` (InvenioRDM format)
   - `resourceTypeGeneral` on related identifiers (mapped via `DATACITE_TO_INVENIORDM_TYPE`)

The field mapping tables below note which pass handles each field where relevant.

## Field Mapping

### Required by Zenodo

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `titles[0].title` | `Poster.title` | `title` | First title, plain string | Implemented |
| `creators[].name` | `PosterMetadata.creators` (JSONB) | `creators[].name` | Falls back to `"familyName, givenName"` if `name` is absent | Implemented |
| `creators[].givenName` | `PosterMetadata.creators` (JSONB) | `person_or_org.given_name` (RDM only) | Not sent in legacy pass. The RDM pass sends given name and family name as separate fields. | Implemented (RDM) |
| `creators[].familyName` | `PosterMetadata.creators` (JSONB) | `person_or_org.family_name` (RDM only) | Not sent in legacy pass. See above. | Implemented (RDM) |
| `creators[].nameType` | `PosterMetadata.creators` (JSONB) | `person_or_org.type` (RDM only) | Sent as "personal" (default) or "organizational". Not sent in legacy pass. | Implemented (RDM) |
| `creators[].affiliation[0].name` | `PosterMetadata.creators` (JSONB) | `creators[].affiliation` (legacy) / `affiliations[{name, id?}]` (RDM) | Legacy pass sends first affiliation name only as a string. RDM pass sends all affiliations with optional ROR IDs. | Implemented (legacy: first only; RDM: all) |
| `creators[].nameIdentifiers[]` where scheme=ORCID | `PosterMetadata.creators` (JSONB) | `creators[].orcid` (legacy) / `person_or_org.identifiers[]` (RDM) | `extractOrcid()` pulls the bare ID. RDM pass validates ORCID format and checksum. Only ORCID is extracted; other name identifier schemes are dropped. | Implemented |
| `descriptions[0].description` | `Poster.description` | `description` | Plain string from `Poster.description` column. See [Description / Abstract](#description--abstract) for details. | Implemented (partial — see below) |
| `rightsList[0].rightsIdentifier` | `PosterMetadata.license` | `license` | SPDX ID as-is (e.g. `cc-by-4.0`). Must match a Zenodo license ID from `/api/licenses`. | Implemented |
| (hardcoded) | — | `upload_type` | Always `"poster"` | Implemented |
| (hardcoded) | — | `publication_type` | Always `"poster"` | Implemented |
| `publicationYear` | `PosterMetadata.publicationYear` | `publication_date` | Sent as year string (e.g. `"2024"`). Zenodo accepts `YYYY` or `YYYY-MM-DD`. | Implemented |

### Description / Abstract

The poster schema supports an array of typed descriptions (abstract, methods, technical info, etc.). Currently only the first description string reaches Zenodo, via a simplified path:

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `descriptions[0].description` | `Poster.description` | `description` | Plain string. HTML subset is allowed by Zenodo but not currently used. | Implemented |
| `descriptions[].descriptionType` | (not stored separately) | `additional_descriptions[].type.id` (RDM only) | Not sent in legacy pass. The RDM pass sends `submissionAbstract` as an additional description with type "abstract". | Implemented (RDM, partial) |
| `content.sections[]` | `PosterMetadata.posterContent` (JSONB) | — | Full poster text sections. Not appended to `description`. | Not mapped (see TODO) |

**What's missing:** The Zenodo `description` field currently gets a single short sentence (the first `descriptions[]` entry). For posters, this is often a one-liner like _"An automated pipeline that combines..."_ rather than a true abstract. The `content.sections[]` data (Introduction, Methodology, Results, Discussion) is extracted and stored but never makes it into the Zenodo description — it only lives inside the deposited `poster.json` file.

Enriching the Zenodo `description` with `content.sections[]` would significantly improve discoverability and record quality on Zenodo. See [Action items for posters-science](#action-items-for-posters-science).

### Keywords and Subjects

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `subjects[].subject` | `PosterMetadata.subjects` (string array) | `keywords` | Collect all non-empty subject strings into a flat array | Implemented |
| `subjects[]` where `subjectScheme` and `valueUri` are present | (not stored) | `subjects[]` | Map to `{term, identifier, scheme}`. Only entries with a controlled-vocabulary URI. | Not mapped (low priority) |
| `researchField` | `PosterMetadata.domain` | — | OpenAlex domain. Not sent to Zenodo. | Not mapped (could append to `keywords`) |

### Language

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `language` | `PosterMetadata.language` | `language` (legacy) / `languages[{id}]` (RDM) | Legacy: Zenodo converts ISO 639-1 to 639-3 automatically. RDM: expects ISO 639-3 (e.g., "eng"). poster2json stores ISO 639-3 codes. | Implemented |

### Dates

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `dates[]` where `dateType = "Issued"` | Set at publish time | `publication_date` | The "Issued" date is recorded when the poster is published to Zenodo. Sent as the `publication_date` alongside `publicationYear`. | Implemented (PR #26) |
| `dates[]` where `dateType = "Submitted"` | (not sent to Zenodo) | — | Tracked internally. Represents when the user uploaded the poster. | N/A |
| `dates[]` where `dateType = "Presented"` | (not sent to Zenodo) | — | Derived from conference dates when available. Not a Zenodo deposit field. | N/A |

### Related Identifiers

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `relatedIdentifiers[].relatedIdentifier` | `PosterMetadata.relatedIdentifiers` (JSONB) | `related_identifiers[].identifier` | Direct | Implemented |
| `relatedIdentifiers[].relatedIdentifierType` | `PosterMetadata.relatedIdentifiers` (JSONB) | `related_identifiers[].scheme` | Lowercased | Implemented |
| `relatedIdentifiers[].relationType` | `PosterMetadata.relatedIdentifiers` (JSONB) | `related_identifiers[].relation` | Lowercase first char: `IsCitedBy` -> `isCitedBy` | Implemented |
| `relatedIdentifiers[].resourceTypeGeneral` | `PosterMetadata.relatedIdentifiers` (JSONB) | `related_identifiers[].resource_type` (RDM only) | Not sent in legacy pass. RDM pass maps via `DATACITE_TO_INVENIORDM_TYPE` lookup. | Implemented (RDM) |

### Identifiers

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| (Zenodo-minted) | `PosterMetadata.doi` | `prereserve_doi` | Pre-reserve on deposit creation. Zenodo DOI written back to `PosterMetadata.identifiers` after publish. | Implemented |
| `identifiers[]` where `identifierType = "DOI"` (pre-existing) | `PosterMetadata.identifiers` (JSONB) | `doi` | Use only if the poster already has a DOI from another source | Not mapped |
| `identifiers[]` (non-DOI: arXiv, PMID, etc.) | `PosterMetadata.identifiers` (JSONB) | `related_identifiers[]` | Should map as `relation: "isAlternateIdentifier"` with appropriate scheme | Not mapped |

**What's missing:** Pre-existing DOIs and non-DOI identifiers (arXiv, PMID) in `identifiers[]` are extracted by poster2json and stored in the DB, but `zenodo.ts` only reads `metaIdentifiers` to inject the Zenodo DOI back after publishing — it never sends the pre-existing identifiers to Zenodo as `related_identifiers`. For the example poster (CarD-T), this means 3 related DOIs and 1 arXiv ID are lost from the Zenodo record. See [Action items for posters-science](#action-items-for-posters-science).

### Conference

Zenodo stores conference fields internally but returns them under a `meeting` object (InvenioRDM migration renamed `conference_*` to `meeting.*`). Submit using the `conference_*` deposit API fields.

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `conference.conferenceName` | `PosterMetadata.conferenceName` | `conference_title` | Direct. Returns as `meeting.title`. | Implemented |
| `conference.conferenceAcronym` | `PosterMetadata.conferenceAcronym` | `conference_acronym` | Direct. Returns as `meeting.acronym`. | Implemented |
| `conference.conferenceLocation` | `PosterMetadata.conferenceLocation` | `conference_place` | Direct. Zenodo expects `"city, country"` format. Returns as `meeting.place`. | Implemented |
| `conference.conferenceUri` | `PosterMetadata.conferenceUri` | `conference_url` | Direct. Returns as `meeting.url`. | Implemented |
| `conference.conferenceStartDate` + `conference.conferenceEndDate` | `PosterMetadata.conferenceStartDate` + `conferenceEndDate` | `conference_dates` | Combined into string, e.g. `"2025-10-15 - 2025-10-17"`. Falls back to single date or `conferenceYear` as string. Returns as `meeting.dates`. | Implemented |
| `conference.conferenceIdentifier` | `PosterMetadata.conferenceIdentifier` | (none) | No direct Zenodo equivalent. | N/A |
| `conference.conferenceSeries` | `PosterMetadata.conferenceSeries` | (none) | No direct Zenodo equivalent. | N/A |

### Funding

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `fundingReferences[].awardNumber` | `PosterMetadata.fundingReferences` (JSONB) | `grants[].id` (legacy) / `funding[].award.number` (RDM) | Legacy: validated against Zenodo's `/awards?q={awardNumber}` API, uses canonical award ID on match. Grants with no match are skipped in legacy. RDM: sends award number directly. If Zenodo rejects the grants payload, retries without grants. | Implemented |
| `fundingReferences[].funderName` | `PosterMetadata.fundingReferences` (JSONB) | `funding[].funder.name` (RDM only) | Not sent in legacy pass (funders not in Zenodo's awards DB are silently skipped). RDM pass sends funder name with optional ROR-based funder ID. Falls back to name-only if ROR ID is rejected. | Implemented (RDM) |
| `fundingReferences[].awardTitle` | `PosterMetadata.fundingReferences` (JSONB) | `funding[].award.title` (RDM only) | Not sent in legacy pass. | Implemented (RDM) |

### Version

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `version` | `PosterMetadata.version` | `version` | Direct, sent if non-empty | Implemented (bug — see below) |

**Bug:** `zenodo.ts` sends `meta.version` in the metadata PUT (line 455), then AFTER the PUT defaults `meta.version` to `"1"` for new deposits (line 513). This means the default `"1"` is saved to the DB but never actually sent to Zenodo. If `PosterMetadata.version` is null at publish time, the Zenodo record gets no version. See [Action items for posters-science](#action-items-for-posters-science).

### Formats

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `formats[]` | `PosterMetadata.format` (single string) | — | Zenodo infers format from uploaded files. No metadata field equivalent. | N/A |

### Content (poster-specific, no Zenodo metadata equivalent)

These fields are poster-specific extensions with no direct Zenodo metadata mapping. They are preserved in the deposited `poster.json` file.

| poster_schema.json | DB source | Notes |
|---|---|---|
| `content.sections[]` | `PosterMetadata.posterContent` (JSONB) | Full poster text. Could enrich Zenodo `description` — see [Description / Abstract](#description--abstract). |
| `tableCaptions[]` | `PosterMetadata.tableCaptions` (JSONB) | Preserved in poster.json only. |
| `imageCaptions[]` | `PosterMetadata.imageCaptions` (JSONB) | Preserved in poster.json only. |
| `researchField` | `PosterMetadata.domain` | OpenAlex domain. Could append to `keywords`. |

## Implementation Notes

### Creator ORCID handling

Zenodo accepts both the full URI and the bare ORCID. It normalizes to the bare ID on storage. `extractOrcid()` in `zenodo.ts` strips the URI prefix.

```
nameIdentifier: "https://orcid.org/0000-0001-2345-6789"
  -> orcid: "0000-0001-2345-6789"
```

### Language code handling

The legacy deposit API accepts ISO 639-1 (2-letter) and converts to ISO 639-3 (3-letter) automatically. The RDM API expects ISO 639-3 directly (e.g., "eng"). poster2json stores ISO 639-3 codes via `langdetect`, so the RDM path gets the right format. Verified on sandbox: legacy input `"en"` stored as `"eng"`.

### Relation type case conversion

poster_schema.json uses PascalCase (DataCite convention). Zenodo uses camelCase. Lowercase the first character:

```
IsCitedBy    -> isCitedBy
Cites        -> cites
IsPartOf     -> isPartOf
References   -> references
```

### Grant validation

The implementation validates each award number against Zenodo's `/awards` API and uses Zenodo's canonical award ID if a match is found. This avoids format mismatches regardless of the funder identifier type stored in poster.json. If Zenodo rejects the grants payload, the code retries without grants so the rest of the metadata still publishes.

### Conference field naming

The deposit API uses `conference_*` field names (flat), but the records API returns them under a nested `meeting` object. This is a Zenodo/InvenioRDM naming inconsistency, not a bug in our mapping:

| Deposit API (submit) | Records API (read back) |
|---|---|
| `conference_title` | `meeting.title` |
| `conference_acronym` | `meeting.acronym` |
| `conference_dates` | `meeting.dates` |
| `conference_place` | `meeting.place` |
| `conference_url` | `meeting.url` |

## Action items for posters-science

Changes needed in `server/utils/zenodo.ts`. Listed by priority.

### 1. Send `identifiers[]` as `related_identifiers` (high)

Pre-existing DOIs and non-DOI identifiers (arXiv, PMID) are stored in `PosterMetadata.identifiers` but never sent to Zenodo. The `metaIdentifiers` array is already parsed (~line 342) but only used post-publish to inject the Zenodo DOI back. These should be included in the `related_identifiers` payload with `relation: "isAlternateIdentifier"`, excluding the Zenodo DOI itself. Merge them with the existing `zenodoRelated` array before building the metadata object.

### 2. Enrich Zenodo `description` with poster content sections (high)

The Zenodo `description` currently gets only `Poster.description` — typically a single sentence. The full poster text (Introduction, Methodology, Results, Discussion) is already stored in `PosterMetadata.posterContent` as structured sections. The metadata construction should build a richer description by appending these sections (Zenodo accepts an HTML subset including `<h3>` and `<p>`). Lead with the existing description, then append each section with its title.

### 3. Fix version default timing (low)

The code defaults `meta.version` to `"1"` for new deposits, but does so AFTER the metadata PUT (~line 513). The default gets saved to the DB but was never sent to Zenodo. Move the default assignment to before the PUT, or resend.

### 4. Append `researchField` to `keywords` (low)

`PosterMetadata.domain` (OpenAlex domain, e.g. `"Life Sciences"`) could be appended to the `keywords` array so it's searchable on Zenodo.

### 5. Persist `descriptionType` (future)

The poster schema supports typed descriptions (`Abstract`, `Methods`, `TechnicalInfo`, `Other`). The type tag is not currently persisted through the API to the DB. Low priority — item 2 is more impactful.

## Verified production record

[Zenodo 20185965](https://zenodo.org/records/20185965) (CarD-T poster, published 2026-05-14):

| Field | Landed on Zenodo? | Notes |
|---|---|---|
| title | Yes | |
| description | Yes | Single sentence only — sections not included |
| creators (3) | Yes | Names, first affiliations, ORCIDs for 2/3 |
| keywords (3) | Yes | From `subjects[]` |
| language | Yes | `eng` |
| license | Yes | `cc-by-4.0` |
| meeting (conference) | Yes | Title, acronym, place, URL |
| publication_date | Yes | `2024` |
| upload_type | Yes | `poster` |
| version | No | `"1"` in poster JSON, but not reaching Zenodo — needs investigation |
| related_identifiers | No | 3 DOIs + 1 arXiv ID in `identifiers[]` not sent |
| grants | No | N/A for this poster (no funding references) |
