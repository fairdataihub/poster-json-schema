# poster_schema.json to Zenodo Crosswalk

Mapping between [poster_schema.json v0.2](../poster_schema.json) (DataCite 4.7) and the metadata of a Zenodo record, written through the [InvenioRDM REST API](https://inveniordm.docs.cern.ch/reference/rest_api_drafts_records/).

Use this document as the reference when building or updating the Zenodo payload in `posters-science/server/utils/zenodo.ts` (`buildFullRdmPayload()`) and `server/utils/zenodoVocabulary.ts`.

The sister document for Figshare is the [draft Figshare crosswalk](figshare.md).

Last checked against posters-science `main` (872f8d0, 2026-10-25) and the production records [21829859](https://zenodo.org/records/21829859) and [22135150](https://zenodo.org/records/22135150) on 2026-10-27.

## Data flow

poster_schema.json fields are extracted by poster2json and stored in the `PosterMetadata` table (Prisma). The Zenodo publish path in `zenodo.ts` reads from this table, not from a poster_schema.json file directly. Some fields (like `title` and `description`) live on the `Poster` model itself rather than `PosterMetadata`.

```
poster PDF
  -> poster2json (extraction + normalization)
  -> posters-science API (stores to Poster + PosterMetadata tables)
  -> zenodo.ts (reads DB, builds the InvenioRDM payload)
  -> Zenodo InvenioRDM API
```

### Publish sequence

Since 2026-07-29 the code uses only the InvenioRDM API. The legacy deposit API (`/deposit/depositions`) and the old two-pass upload are gone.

1. **Create a draft:** `POST /records` with public record and file access, `resource_type` poster, publisher "Zenodo" and the creators. A new version of a poster that is already on Zenodo starts from `POST /records/{id}/versions` instead, so Zenodo links the versions.
2. **Write the metadata:** one `PUT /records/{id}/draft` with the full payload from `buildFullRdmPayload()`. The PUT replaces the whole draft, so a DOI the draft already carries is sent back in `pids`.
3. **Reserve the DOI:** `POST /records/{id}/draft/pids/doi` (`ensureReservedDoi()`), after the PUT so the PUT cannot drop it.
4. **Upload the files:** `poster.json` (built by `buildPosterJson()`) and the poster file the user uploaded.
5. **Publish:** `POST /records/{id}/draft/actions/publish`. The DOI, the publisher "Zenodo" and the version are then written back to `PosterMetadata`, and the DOI is added as the first entry of `identifiers`.

If Zenodo rejects the metadata PUT with a 400 "invalid value" (an affiliation or funder ROR id missing from its vocabulary), the code retries once without affiliation ROR ids and funder ids.

## Field Mapping

### Required by Zenodo

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `titles[0].title` | `Poster.title` | `metadata.title` | Plain string | Implemented |
| `creators[]` | `PosterMetadata.creators` (JSONB) | `metadata.creators[]` | See [Creators](#creators) | Implemented |
| (hardcoded) | | `metadata.resource_type.id` | Always `"poster"` | Implemented |
| (hardcoded) | | `metadata.publisher` | Always `"Zenodo"` | Implemented |
| (publish date) | | `metadata.publication_date` | The year the poster is published (e.g. `"2026"`). The stored `publicationYear` is not used. | Implemented |
| `rightsList[0].rightsIdentifier` | `PosterMetadata.license` | `metadata.rights[0].id` | SPDX id lowercased (e.g. `cc-by-4.0`). Must be a Zenodo license id. | Implemented |

### Creators

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `creators[].nameType` | `PosterMetadata.creators` | `person_or_org.type` | `"organizational"` when nameType is Organizational, else `"personal"` | Implemented |
| `creators[].familyName` | `PosterMetadata.creators` | `person_or_org.family_name` | Personal creators. If both name parts are empty, `name` is split at the first comma ("Family, Given"); a name without a comma becomes the family name. | Implemented |
| `creators[].givenName` | `PosterMetadata.creators` | `person_or_org.given_name` | See above | Implemented |
| `creators[].name` | `PosterMetadata.creators` | `person_or_org.name` | Organizational creators only (falls back to familyName). For people, Zenodo builds the name from the parts. | Implemented |
| `creators[].nameIdentifiers[]` where scheme=ORCID | `PosterMetadata.creators` | `person_or_org.identifiers[{scheme: "orcid"}]` | `extractOrcid()` takes the bare id. An id that fails the ORCID format or checksum check is dropped. Ids are also looked up in the ORCID registry; a miss only warns the user. Other name identifier schemes are not sent. | Implemented |
| `creators[].affiliation[].name` | `PosterMetadata.creators` | `affiliations[].name` | All affiliations, in order | Implemented |
| `creators[].affiliation[].affiliationIdentifier` (ROR) | `PosterMetadata.creators` | `affiliations[].id` | Bare ROR id (`extractRorId()`), when the scheme is ROR or the value is a ror.org URL. Other schemes are not sent. | Implemented |

### Description / Abstract

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `descriptions[0].description` | `Poster.description` | `metadata.description` | Plain string | Implemented |
| `descriptions[]` where descriptionType=Abstract | `PosterMetadata.posterContent.submissionAbstract` | `metadata.additional_descriptions[{type: {id: "abstract"}}]` | Sent when a submission abstract exists | Implemented |
| `descriptions[].descriptionType` (other types) | (not stored) | | Not persisted | Not mapped |
| `content.sections[]` | `PosterMetadata.posterContent` (JSONB) | | Full poster text sections. Not appended to `description`. | Not mapped (see action items) |

The Zenodo `description` is usually a single sentence. The poster text (Introduction, Methods, Results, ...) is only in the deposited `poster.json`.

### Keywords and Subjects

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `subjects[].subject` | `PosterMetadata.subjects` (string array) | `metadata.subjects[{subject}]` | Free-text subjects (Zenodo's keywords), empty strings dropped | Implemented |
| `subjects[]` with `subjectScheme` and `valueUri` | (not stored) | `metadata.subjects[{id}]` | Controlled-vocabulary subjects | Not mapped (low priority) |
| `researchField` | `PosterMetadata.domain` | | OpenAlex domain. Not sent. | Not mapped |

### Language

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `language` | `PosterMetadata.language` | `metadata.languages[{id}]` | A two-letter code is converted to ISO 639-3 (`en` -> `eng`); anything else is sent as stored | Implemented |

### Dates

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| (publish date) | | `metadata.dates[{type: {id: "submitted"}}]` | The day the poster is shared to Zenodo, with the description "Submitted to Zenodo through Posters.science". Added unless the record already has a Submitted date. | Implemented |
| `dates[]` where dateType=Presented | `PosterMetadata.dates` | `metadata.dates[{type: {id: "other"}}]` | Description "Poster presentation date" (Zenodo has no Presented type) | Implemented |
| `dates[]` (other DataCite types) | `PosterMetadata.dates` | `metadata.dates[{type: {id}}]` | dateType lowercased (`Created` -> `created`) | Implemented |

The platform does not record an Issued date. The deposited `poster.json` carries the same Submitted date.

### Related Identifiers

Translated by `toZenodoRelatedIdentifiers()` in `zenodoVocabulary.ts`, which keeps poster.json in DataCite terms and speaks Zenodo's vocabulary only in the payload.

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `relatedIdentifiers[].relatedIdentifier` | `PosterMetadata.relatedIdentifiers` (JSONB) | `related_identifiers[].identifier` | Direct. A DOI must start with `10.NNNN/` and a URL with `http(s)://`, or the entry is dropped. | Implemented |
| `relatedIdentifiers[].relatedIdentifierType` | same | `related_identifiers[].scheme` | Lowercased. `bibcode` -> `ads`, `swhid` -> `swh`. RAiD has no Zenodo scheme: sent as `url` when it is a URL, else dropped. | Implemented |
| `relatedIdentifiers[].relationType` | same | `related_identifiers[].relation_type.id` | Fully lowercased (`IsCitedBy` -> `iscitedby`). DataCite 4.7 types that Zenodo's vocabulary lacks (Collects, IsCollectedBy, HasTranslation, IsTranslationOf, Other) are sent unchanged and logged; Zenodo rejects them. | Implemented |
| `relatedIdentifiers[].resourceTypeGeneral` | same | `related_identifiers[].resource_type.id` | Mapped term by term (e.g. JournalArticle -> `publication-article`, Preprint -> `publication-preprint`, Dataset -> `dataset`, Poster -> `poster`); terms without a Zenodo equivalent -> `other` | Implemented |

The platform adds two related identifiers itself:

- `IsDescribedBy` the poster's page, `https://posters.science/discover/{id}` (URL), added after extraction.
- `IsNewVersionOf` the previous version's DOI (resourceTypeGeneral Poster), for a new version of a published poster. Only the immediate predecessor is kept.

### Identifiers

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| (Zenodo-minted DOI) | `PosterMetadata.doi` | `pids.doi` | Reserved before publishing. After publishing it is stored in `PosterMetadata.doi` and added as the first `identifiers` entry. | Implemented |
| `identifiers[]` (a DOI from another source, arXiv, PMID, ...) | `PosterMetadata.identifiers` (JSONB) | `metadata.identifiers[{identifier, scheme}]` (alternate identifiers) | Not sent | Not mapped (see action items) |

### Conference

InvenioRDM keeps conference details in the `meeting:meeting` custom field. The old deposit API names (`conference_title`, `conference_place`, ...) are no longer used.

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `conference.conferenceName` | `PosterMetadata.conferenceName` | `custom_fields["meeting:meeting"].title` | Direct | Implemented |
| `conference.conferenceAcronym` | `PosterMetadata.conferenceAcronym` | `custom_fields["meeting:meeting"].acronym` | Direct | Implemented |
| `conference.conferenceLocation` | `PosterMetadata.conferenceLocation` | `custom_fields["meeting:meeting"].place` | Direct (e.g. `"Washington, DC, USA"`) | Implemented |
| `conference.conferenceUri` | `PosterMetadata.conferenceUri` | `custom_fields["meeting:meeting"].url` | Direct | Implemented |
| `conference.conferenceStartDate` + `conferenceEndDate` | `PosterMetadata.conferenceStartDate` + `conferenceEndDate` | `custom_fields["meeting:meeting"].dates` | `"2026-07-14 - 2026-07-16"`; one date alone is sent as is; neither falls back to `conferenceYear` | Implemented |
| `conference.conferenceIdentifier` | `PosterMetadata.conferenceIdentifier` | | No Zenodo equivalent | N/A |
| `conference.conferenceSeries` | `PosterMetadata.conferenceSeries` | | No Zenodo equivalent | N/A |

### Funding

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `fundingReferences[].funderName` | `PosterMetadata.fundingReferences` (JSONB) | `funding[].funder.name` | Entries without a funder name are skipped | Implemented |
| `fundingReferences[].funderIdentifier` (ROR) | same | `funding[].funder.id` | Bare ROR id (`extractRorId()`). A Crossref Funder ID or other scheme is not sent. The share form looks funders up in ROR. | Implemented |
| `fundingReferences[].awardNumber` | same | `funding[].award.number` | Sent as entered. Zenodo's awards database is no longer queried. | Implemented |
| `fundingReferences[].awardTitle` | same | `funding[].award.title.en` | | Implemented |
| `fundingReferences[].awardUri` | same | `funding[].award.identifiers[{scheme: "url"}]` | An award is sent only with a number or a title. An award URI without either blocks publishing until the metadata is fixed. | Implemented |

### Version

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `version` | `PosterMetadata.version` | `metadata.version` | The version chosen in the publish form (a leading "v" is removed: "v2" -> "2"), else the stored version, else the version number of the poster (`"1"` for the first). Set before the metadata PUT. | Implemented |

The May 2026 bug, where the default version was set after the metadata was sent, is fixed.

### Formats and sizes

| poster_schema.json | DB source | Zenodo field | Transform | Status |
|---|---|---|---|---|
| `formats[]` | `PosterMetadata.format` | | Zenodo reads formats from the uploaded files | N/A |
| `sizes[]` | `PosterMetadata.size` | | No Zenodo field | N/A |

### Content (poster-specific, no Zenodo metadata equivalent)

These fields are kept in the deposited `poster.json` only.

| poster_schema.json | DB source | Notes |
|---|---|---|
| `content.sections[]`, `content.unstructuredContent` | `PosterMetadata.posterContent` (JSONB) | Full poster text. Could enrich the Zenodo `description`. |
| `tableCaptions[]` | `PosterMetadata.tableCaptions` (JSONB) | |
| `imageCaptions[]` | `PosterMetadata.imageCaptions` (JSONB) | |
| `researchField` | `PosterMetadata.domain` | OpenAlex domain |

## The deposited poster.json

`buildPosterJson()` writes the record as poster_schema.json v0.2 and the platform uploads it next to the poster file. Compared with the database it adds:

- `publisher`: `{"name": "Zenodo", "publisherIdentifier": "https://doi.org/10.17616/R3QP53", "publisherIdentifierScheme": "DOI", "schemeURI": "https://doi.org"}` (Zenodo's re3data DOI).
- `identifiers[0]`: the Zenodo DOI.
- `dates`: the Submitted date ("Submitted to Zenodo through Posters.science").
- `descriptions`: `Poster.description` as type Other and the submission abstract as type Abstract.
- `types`: `{"resourceType": "Conference Poster", "resourceTypeGeneral": "Poster"}`.
- `publicationYear`: the year of publishing.
- `formats`: the stored format, or `["PDF"]`.

## Implementation Notes

### Vocabulary translation

poster.json stays in DataCite 4.7 terms. Everything Zenodo spells differently (resource types, identifier schemes, relation types) is translated in `zenodoVocabulary.ts`. Check new terms against `{zenodoApiEndpoint}/vocabularies/` when the schema or Zenodo changes.

### ORCID handling

Zenodo accepts the bare ORCID. `extractOrcid()` strips the URI prefix, and the payload builder checks the format and checksum:

```
nameIdentifier: "https://orcid.org/0000-0001-2345-6789"
  -> identifiers: [{"scheme": "orcid", "identifier": "0000-0001-2345-6789"}]
```

### ROR ids

Affiliation and funder ids are sent as bare ROR ids (`02dfjrn59`). Zenodo resolves them in its own vocabulary and adds the GRID id on the record. When an id is not in that vocabulary the PUT fails with "invalid value", and the code resends the payload with names only.

### Relation type case

poster_schema.json uses DataCite PascalCase. InvenioRDM vocabulary ids are all lowercase:

```
IsCitedBy     -> iscitedby
Cites         -> cites
IsPartOf      -> ispartof
IsDescribedBy -> isdescribedby
```

## Action items for posters-science

Changes still needed, by priority.

### 1. Send `identifiers[]` to Zenodo (high)

Pre-existing DOIs and other identifiers of the poster (arXiv, PMID) are stored in `PosterMetadata.identifiers` but never sent. InvenioRDM has a field for them: `metadata.identifiers[{identifier, scheme}]` (alternate identifiers). Send every entry except the Zenodo DOI itself, with the scheme lowercased as for related identifiers.

### 2. Enrich the Zenodo `description` with the poster sections (high)

`metadata.description` gets only `Poster.description`, usually one sentence. The poster text is stored in `PosterMetadata.posterContent`. Lead with the existing description, then append each section with its title (Zenodo accepts an HTML subset including `<h3>` and `<p>`).

### 3. Fill `doi`, `prefix` and `suffix` in the deposited poster.json (low)

`buildPosterJson()` takes them from `PosterMetadata.doi`, which is only set after publishing, so the deposited poster.json has the Zenodo DOI in `identifiers` but no top-level `doi`, `prefix` or `suffix` (seen on 21829859). Take them from the reserved DOI (`zenodoDoi`) when it is given.

### 4. Send `researchField` as a subject (low)

`PosterMetadata.domain` (OpenAlex domain, e.g. `"Life Sciences"`) could be added to `metadata.subjects` so it is searchable on Zenodo.

### 5. Persist `descriptionType` (future)

The schema supports typed descriptions (Abstract, Methods, TechnicalInfo, Other). Only the submission abstract keeps its type today.

## Verified production records

[Zenodo 21829859](https://zenodo.org/records/21829859) (BOSC 2026, published 2026-08-06) and [Zenodo 22135150](https://zenodo.org/records/22135150) (AGU 2011, published 2026-08-27), read through the InvenioRDM API (`Accept: application/vnd.inveniordm.v1+json`):

| Field | Landed on Zenodo? | Notes |
|---|---|---|
| title, description | Yes | Description is the short `Poster.description` |
| additional_descriptions | Yes (21829859) | Submission abstract, type abstract |
| creators | Yes | Given and family names, ORCIDs, all affiliations with ROR ids |
| subjects | Yes | From `subjects[]` |
| languages | Yes | `eng` |
| rights | Yes | `cc-by-4.0` |
| publisher | Yes | `Zenodo` |
| publication_date | Yes | `2026`, the year of publishing |
| dates | Yes | Submitted (publish day) and Other "Poster presentation date" |
| meeting | Yes | Title, acronym, place, dates, URL |
| related_identifiers | Yes | `isdescribedby` the posters.science page, with resource type |
| funding | Yes | Funder with ROR id; award number, title and URL (21829859) |
| version | Yes | `1` |
| files | Yes | Poster PDF and `poster.json` |
| identifiers (alternate) | No | Not sent (action item 1) |
