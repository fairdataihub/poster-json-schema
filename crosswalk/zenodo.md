# poster_schema.json to Zenodo Crosswalk

Mapping between [poster_schema.json v0.2](../poster_schema.json) (DataCite 4.7) and the [Zenodo REST API deposit metadata](https://developers.zenodo.org/#deposit-metadata).

Use this document as the reference when building or updating the Zenodo metadata payload in `posters-science/server/utils/zenodo.ts`.

Last verified against sandbox record [496481](https://sandbox.zenodo.org/records/496481) on 2026-05-06.

## Field Mapping

### Required by Zenodo

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `titles[0].title` | `title` | First title, plain string | Mapped |
| `creators[].name` | `creators[].name` | Direct (`"Family, Given"` format) | Mapped |
| `creators[].affiliation[0].name` | `creators[].affiliation` | First affiliation name only (Zenodo takes a single string, not array) | Mapped (first only) |
| `creators[].nameIdentifiers[]` where scheme=ORCID | `creators[].orcid` | Pass the full URI or bare ID. Zenodo normalizes to bare ID automatically. Verified: sandbox record stores `0000-0002-4306-4464`. | Mapped |
| `descriptions[0].description` | `description` | First description, plain string. HTML subset allowed by Zenodo. | Mapped |
| `rightsList[0].rightsIdentifier` | `license` | SPDX ID lowercased, e.g. `CC-BY-4.0` -> `cc-by-4.0`. Must match a Zenodo license ID from `/api/licenses`. | Mapped |
| (hardcoded) | `upload_type` | Always `"poster"` | Mapped |
| (hardcoded) | `publication_type` | Always `"poster"` | Mapped |
| `publicationYear` | `publication_date` | Sent as year string (e.g. `"2025"`). Zenodo accepts `YYYY` or `YYYY-MM-DD`. | Mapped |

### Keywords and Subjects

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `subjects[].subject` | `keywords` | Collect all subject strings into a flat array | Mapped |
| `subjects[]` where `subjectScheme` and `valueUri` are present | `subjects[]` | Map to `{term, identifier, scheme}`. Only include entries that have a controlled-vocabulary URI. | Not mapped (low priority) |

### Language

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `language` | `language` | Zenodo handles ISO 639-1 to 639-2/3 conversion automatically. Send the 2-letter code directly. Verified: sandbox record shows `"eng"` from input `"en"`. | Mapped |

### Related Identifiers

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `relatedIdentifiers[].relatedIdentifier` | `related_identifiers[].identifier` | Direct | Mapped |
| `relatedIdentifiers[].relatedIdentifierType` | `related_identifiers[].scheme` | Lowercase the type. Zenodo auto-detects scheme from the identifier string, so this is optional. | Mapped |
| `relatedIdentifiers[].relationType` | `related_identifiers[].relation` | Lowercase first character: `IsCitedBy` -> `isCitedBy` | Mapped |
| `relatedIdentifiers[].resourceTypeGeneral` | `related_identifiers[].resource_type` | Lowercase, e.g. `Dataset` -> `dataset`, `ConferencePaper` -> `publication-conferencepaper` | Not mapped (low priority) |

### Conference

Zenodo stores conference fields internally but returns them under a `meeting` object (InvenioRDM migration renamed `conference_*` to `meeting.*`). Submit using the `conference_*` deposit API fields.

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `conference.conferenceName` | `conference_title` | Direct. Returns as `meeting.title`. | Mapped |
| `conference.conferenceAcronym` | `conference_acronym` | Direct. Returns as `meeting.acronym`. | Mapped |
| `conference.conferenceLocation` | `conference_place` | Direct. Zenodo expects `"city, country"` format. Returns as `meeting.place`. | Mapped |
| `conference.conferenceUri` | `conference_url` | Direct. Returns as `meeting.url`. | Mapped |
| `conference.conferenceStartDate` + `conference.conferenceEndDate` | `conference_dates` | Combine into string, e.g. `"2025-10-15 - 2025-10-17"`. Returns as `meeting.dates`. | Mapped |
| `conference.conferenceIdentifier` | (none) | No direct Zenodo equivalent. | N/A |
| `conference.conferenceSeries` | (none) | No direct Zenodo equivalent. | N/A |

### Funding

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `fundingReferences[].awardNumber` | `grants[].id` | Validated against Zenodo's `/awards?q={awardNumber}` API. If a match is found, uses Zenodo's canonical award ID. Grants with no award number or no Zenodo match are skipped. Falls back gracefully if Zenodo rejects any grant. | In progress (PR #20) |
| `fundingReferences[].funderName` (unrecognized funder) | (dropped) | Funders not in Zenodo's awards database are silently skipped. The data is still preserved in the poster.json file. | In progress (PR #20) |

### Version

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `version` | `version` | Direct | Mapped |

### Identifiers

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `identifiers[]` where `identifierType = "DOI"` (pre-existing) | `doi` | Use only if the poster already has a registered DOI from another source | Not mapped |
| (Zenodo-minted) | `prereserve_doi` | Set to `true` when creating a new deposit. Zenodo returns the reserved DOI. | Mapped |
| `identifiers[]` (non-DOI) | `related_identifiers[]` | Map as `relation: "isAlternateIdentifier"` with appropriate scheme | Not mapped |

### Content (poster-specific, no Zenodo equivalent)

These fields are poster-specific extensions with no direct Zenodo metadata mapping. They are preserved in the deposited `poster.json` file.

| poster_schema.json | Zenodo field | Notes |
|---|---|---|
| `content.sections[]` | (none) | Poster text sections. Could optionally be appended to `description` as HTML. |
| `tableCaptions[]` | (none) | Preserved in poster.json only. |
| `imageCaptions[]` | (none) | Preserved in poster.json only. |
| `researchField` | (none) | OpenAlex domain. No Zenodo equivalent. Could include in `keywords`. |

## Implementation Notes

### Creator ORCID handling

Zenodo accepts both the full URI and the bare ORCID. It normalizes to the bare ID on storage. No stripping needed on our side, but stripping is harmless if done.

```
nameIdentifier: "https://orcid.org/0000-0001-2345-6789"
  -> orcid: "https://orcid.org/0000-0001-2345-6789"  (Zenodo stores as "0000-0001-2345-6789")
```

### Language code handling

Zenodo converts ISO 639-1 (2-letter) to ISO 639-2/3 (3-letter) automatically. No conversion needed on our side. Verified on sandbox: input `"en"` stored as `"eng"`.

### Relation type case conversion

poster_schema.json uses PascalCase (DataCite convention). Zenodo uses camelCase. Lowercase the first character:

```
IsCitedBy    -> isCitedBy
Cites        -> cites
IsPartOf     -> isPartOf
References   -> references
```

### Grant validation

Rather than formatting grant IDs manually, the implementation validates each award number against Zenodo's `/awards` API and uses Zenodo's canonical award ID if a match is found. This avoids format mismatches regardless of the funder identifier type stored in poster.json. If Zenodo rejects the grants payload, the code retries without grants so the rest of the metadata still publishes.

### Conference field naming

The deposit API uses `conference_*` field names (flat), but the records API returns them under a nested `meeting` object. This is a Zenodo/InvenioRDM naming inconsistency, not a bug in our mapping:

| Deposit API (submit) | Records API (read back) |
|---|---|
| `conference_title` | `meeting.title` |
| `conference_acronym` | `meeting.acronym` |
| `conference_dates` | `meeting.dates` |
| `conference_place` | `meeting.place` |
| `conference_url` | `meeting.url` |

## Current mapping status

As of 2026-05-06, most fields are mapped. Remaining work:

- **Grants/funding**: In progress (PR #20), validates against Zenodo awards API
- **Identifiers**: Non-DOI identifiers (arXiv, etc.) not yet sent as `related_identifiers` with `isAlternateIdentifier` relation
- **Subjects with controlled vocabulary URIs**: Low priority, most poster subjects are free-text keywords
- **`related_identifiers[].resource_type`**: Low priority
