# Changelog

All notable changes to the Posters.science JSON Schema are documented here.

## [Unreleased]

### Changed
- `descriptionType` and `dateType` moved from inline `oneOf` blocks to `$ref` definitions, matching the existing pattern for `relationType`, `identifierType`, and `resourceType`. No validation change, just consistency.
- `descriptions` array description updated to clarify intended usage: `"Abstract"` for user-provided formal abstracts, `"Other"` for auto-generated summaries. A poster may have both.
- `researchField` examples updated to the four OpenAlex top-level domains: `Health Sciences`, `Life Sciences`, `Physical Sciences`, `Social Sciences`. Description tightened to instruct extractors to omit the field when the domain cannot be determined and never to emit `"Other"`, `"Unknown"`, or placeholder text. No structural change -- `type` remains `string`, no enum added (existing data with arbitrary discipline strings stays valid).

## [v0.2] - 2026-03-31

Updated to align with [DataCite Metadata Schema 4.7](https://datacite-metadata-schema.readthedocs.io/en/4.7/).

### Added
- `Poster` and `Presentation` values to the `resourceTypeGeneral` controlled list (properties 10.a, 12.f, 20.a)
- `RAiD` and `SWHID` values to the `relatedIdentifierType` controlled list (properties 12.a, 20.1.a)
- `Other` value to the `relationType` controlled list
- `relationTypeInformation` sub-property on `relatedIdentifiers` items (DataCite property 12.g)

### Changed
- Default `resourceTypeGeneral` changed from `"Other"` to `"Poster"`
- Base schema reference updated from DataCite 4.6 to DataCite 4.7
- `$id` updated from `v0.1` to `v0.2`

### Directory Structure
- Introduced `schemas/v0.1/` and `schemas/v0.2/` versioned directories
- Root `poster_schema.json` always points to the latest version

## [v0.1] - 2025-08-18

Initial release. Based on [DataCite Metadata Schema 4.6](https://datacite-metadata-schema.readthedocs.io/en/4.6/).

### Core DataCite Properties
- identifiers, creators, titles, publisher, publicationYear
- subjects, dates, language, types, relatedIdentifiers
- sizes, formats, version, rightsList, descriptions, fundingReferences

### Poster-Specific Extensions
- `conference` — structured conference metadata
- `content` — extracted poster text organized into sections
- `imageCaptions` / `tableCaptions` — visual element descriptions
- `researchField` — primary research discipline

[v0.2]: https://posters.science/schema/v0.2/poster_schema.json
[v0.1]: https://posters.science/schema/v0.1/poster_schema.json
