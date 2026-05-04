# poster_schema.json to Zenodo Crosswalk

Mapping between [poster_schema.json v0.2](../poster_schema.json) (DataCite 4.7) and the [Zenodo REST API deposit metadata](https://developers.zenodo.org/#deposit-metadata).

Use this document as the reference when building or updating the Zenodo metadata payload in `posters-science/server/utils/zenodo.ts`.

## Field Mapping

### Required by Zenodo

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `titles[0].title` | `title` | First title, plain string | Mapped |
| `creators[].name` | `creators[].name` | Direct (`"Family, Given"` format) | Mapped |
| `creators[].affiliation[0].name` | `creators[].affiliation` | First affiliation name only (Zenodo takes a single string, not array) | Mapped (first only) |
| `creators[].nameIdentifiers[]` where scheme=ORCID | `creators[].orcid` | Extract bare ORCID from nameIdentifier URI, e.g. `https://orcid.org/0000-0001-2345-6789` -> `0000-0001-2345-6789` | **Not mapped** |
| `descriptions[0].description` | `description` | First description, plain string. HTML subset allowed by Zenodo. | Mapped |
| `rightsList[0].rightsIdentifier` | `license` | SPDX ID lowercased, e.g. `CC-BY-4.0` -> `cc-by-4.0`. Must match a Zenodo license ID from `/api/licenses`. | Mapped |
| (hardcoded) | `upload_type` | Always `"poster"` | Mapped |
| (hardcoded) | `publication_type` | Always `"poster"` | Mapped |
| (derived) | `publication_date` | Use `dates[]` where `dateType = "Presented"`, fall back to conference year, fall back to `publicationYear`. Format: `YYYY-MM-DD`. | **Not mapped** |

### Keywords and Subjects

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `subjects[].subject` | `keywords` | Collect all subject strings into a flat array | **Not mapped** |
| `subjects[]` where `subjectScheme` and `valueUri` are present | `subjects[]` | Map to `{term, identifier, scheme}`. Only include entries that have a controlled-vocabulary URI. | **Not mapped** |

### Language

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `language` | `language` | poster_schema uses ISO 639-1 (2-letter, e.g. `en`). Zenodo uses ISO 639-2/3 (3-letter, e.g. `eng`). Convert using a lookup table. | **Not mapped** |

### Related Identifiers

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `relatedIdentifiers[].relatedIdentifier` | `related_identifiers[].identifier` | Direct | **Not mapped** |
| `relatedIdentifiers[].relatedIdentifierType` | `related_identifiers[].scheme` | Lowercase the type. Zenodo auto-detects scheme from the identifier string, so this is optional. | **Not mapped** |
| `relatedIdentifiers[].relationType` | `related_identifiers[].relation` | Convert to camelCase: `IsCitedBy` -> `isCitedBy`, `IsSupplementTo` -> `isSupplementTo`, etc. | **Not mapped** |
| `relatedIdentifiers[].resourceTypeGeneral` | `related_identifiers[].resource_type` | Lowercase, e.g. `Dataset` -> `dataset`, `ConferencePaper` -> `publication-conferencepaper` | **Not mapped** |

### Conference

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `conference.conferenceName` | `conference_title` | Direct | **Not mapped** |
| `conference.conferenceAcronym` | `conference_acronym` | Direct | **Not mapped** |
| `conference.conferenceLocation` | `conference_place` | Direct. Zenodo expects `"city, country"` format. | **Not mapped** |
| `conference.conferenceUri` | `conference_url` | Direct | **Not mapped** |
| `conference.conferenceStartDate` + `conference.conferenceEndDate` | `conference_dates` | Combine into human-readable string, e.g. `"15-18 October 2025"` | **Not mapped** |
| `conference.conferenceIdentifier` | (none) | No direct Zenodo equivalent. Could include in `notes`. | N/A |
| `conference.conferenceSeries` | (none) | No direct Zenodo equivalent. Could include in `notes`. | N/A |

### Funding

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `fundingReferences[].funderIdentifier` + `fundingReferences[].awardNumber` | `grants[].id` | Format as `{funderDOI}::{awardNumber}`. Zenodo only supports grants from its list of 27 recognized funders (via Crossref Funder DOI). If the funder is not recognized, the grant cannot be mapped. | **Not mapped** |
| `fundingReferences[].funderName` (unrecognized funder) | `notes` or `description` | Append as text, e.g. "Funded by {funderName}, grant {awardNumber}". | **Not mapped** |

### Dates

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `dates[]` where `dateType` in (`Collected`, `Valid`, `Withdrawn`) | `dates[]` | Map `dateType` directly. Zenodo only supports these three date types. Split date ranges into `start`/`end`. | **Not mapped** |
| `dates[]` where `dateType = "Presented"` | `publication_date` | Use as the publication_date (see Required section). | **Not mapped** |

### Publisher and Version

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `publisher.name` | `imprint_publisher` | Direct | **Not mapped** |
| `version` | `version` | Direct | **Not mapped** |

### Identifiers

| poster_schema.json | Zenodo field | Transform | Status |
|---|---|---|---|
| `identifiers[]` where `identifierType = "DOI"` (pre-existing) | `doi` | Use only if the poster already has a registered DOI from another source | **Not mapped** |
| (Zenodo-minted) | `prereserve_doi` | Set to `true` when creating a new deposit. Zenodo returns the reserved DOI. | Mapped |
| `identifiers[]` (non-DOI) | `related_identifiers[]` | Map as `relation: "isAlternateIdentifier"` with appropriate scheme | **Not mapped** |

### Content (poster-specific, no Zenodo equivalent)

These fields are poster-specific extensions with no direct Zenodo metadata mapping. They are preserved in the deposited `poster.json` file.

| poster_schema.json | Zenodo field | Notes |
|---|---|---|
| `content.sections[]` | (none) | Poster text sections. Could optionally be appended to `description` as HTML. |
| `tableCaptions[]` | (none) | Preserved in poster.json only. |
| `imageCaptions[]` | (none) | Preserved in poster.json only. |
| `researchField` | (none) | OpenAlex domain. No Zenodo equivalent. Could include in `keywords`. |

## Implementation Notes

### Creator ORCID extraction

Zenodo expects a bare ORCID string (e.g. `0000-0001-2345-6789`), not the full URI. Extract from `nameIdentifiers[]` where `nameIdentifierScheme = "ORCID"`:

```
nameIdentifier: "https://orcid.org/0000-0001-2345-6789"
  -> orcid: "0000-0001-2345-6789"
```

### Language code conversion

poster_schema.json uses ISO 639-1 (2-letter). Zenodo uses ISO 639-2/3 (3-letter). Common mappings:

| poster_schema | Zenodo |
|---|---|
| `en` | `eng` |
| `es` | `spa` |
| `fr` | `fra` |
| `de` | `deu` |
| `zh` | `zho` |
| `ja` | `jpn` |
| `pt` | `por` |
| `ko` | `kor` |
| `ar` | `ara` |

Use a library like `iso-639-1` or a static lookup table for the full set.

### Relation type case conversion

poster_schema.json uses PascalCase (DataCite convention). Zenodo uses camelCase. Convert the first character to lowercase:

```
IsCitedBy    -> isCitedBy
Cites        -> cites
IsPartOf     -> isPartOf
References   -> references
```

### Grant ID format

Zenodo requires grants in the format `{funderDOI}::{awardNumber}`. poster2json's funder enrichment already produces Crossref Funder DOIs when the funder is recognized by ROR:

```
funderIdentifier: "10.13039/100000001"  (NSF)
awardNumber: "2019511"
  -> grants[].id: "10.13039/100000001::2019511"
```

If `funderIdentifierType` is not `"Crossref Funder ID"`, the grant cannot be mapped to Zenodo's structured format. Fall back to including it in `notes`.

### Zenodo's recognized funders

Zenodo only accepts grants from funders with Crossref Funder DOIs in its registry (27 funders including NIH, NSF, ERC, Wellcome Trust, Australian Research Council, etc.). Grants from unrecognized funders should be included as free text in `notes` or `description`.

## Current mapping status

As of 2026-05-04, `posters-science/server/utils/zenodo.ts` only maps: title, creators (name + first affiliation), description, license, upload_type, publication_type, and prereserve_doi. All other fields above marked **Not mapped** need to be implemented.
