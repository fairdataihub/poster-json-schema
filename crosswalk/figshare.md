# poster_schema.json to Figshare Crosswalk (draft)

> **Draft.** Posters.science cannot publish to Figshare yet. This is the proposed mapping for building that path. Nothing here is implemented. Rows marked **Agreed** were settled in the team meeting on 2026-10-23, **Proposed** are design choices to confirm, **Gap** means Figshare has no place for the field, and **Verify** needs a test deposit.

Mapping between [poster_schema.json v0.2](../poster_schema.json) (DataCite 4.7) and the item (article) metadata of the [Figshare API v2](https://docs.figshare.com/). Sister document to the [Zenodo crosswalk](zenodo.md).

Checked on 2026-10-27 against the Figshare API v2 specification (`https://docs.figshare.com/swagger.json`), the figshare.com posters [34701474](https://doi.org/10.6084/m9.figshare.34701474.v1) and [33990901](https://doi.org/10.6084/m9.figshare.33990901.v3), the UL Research Institutes portal poster [27063577](https://doi.org/10.60752/102376.27063577.v1), and the DataCite records Figshare registered for them.

## Data flow (proposed)

```
poster PDF
  -> poster2json (extraction + normalization)
  -> posters-science API (stores to Poster + PosterMetadata tables)
  -> figshare.ts (to be built: reads DB, builds the Figshare item payload)
  -> Figshare API v2, on the user's own Figshare account
  -> DataCite (Figshare registers the DOI and its own metadata)
```

### Where the poster lands

The API works on the user's own account, so the poster lands wherever that account lives:

- **figshare.com account:** the item is published on figshare.com with a `10.6084/m9.figshare.*` DOI, and DataCite lists the publisher as "figshare".
- **Institutional account:** the item is published on the institution's Figshare portal. It gets the institution's DOI prefix or a Handle, and the institution's publisher name, licenses, categories and custom fields (the UL Research Institutes portal uses `10.60752`). Some portals send items to a curation review before they are public.

Read the account first (`GET /account`, its `institution_id`, `GET /account/licenses`, `GET /account/categories`) and build the payload from what that account offers.

### Publish sequence (proposed)

1. **Create a private item:** `POST /account/articles` with the metadata, `defined_type: "poster"`.
2. **Reserve the DOI:** `POST /account/articles/{id}/reserve_doi`, so the deposited poster.json can carry it.
3. **Upload the files:** the poster file and `poster.json`. Each file is `POST /account/articles/{id}/files` (name, size, md5), then its parts are uploaded to the returned upload URL, then `POST /account/articles/{id}/files/{file_id}` completes it.
4. **Publish:** `POST /account/articles/{id}/publish`.
5. **Read back** the public item and write the DOI, publisher and version to `PosterMetadata`.

A new version is the same item, not a new one: update it (`PUT /account/articles/{id}`), replace the files, and publish again. Figshare numbers the version itself, and the versioned DOI ends in `.v2`. The DOI without a version always points to the latest version (DataCite: `IsIdenticalTo`).

Figshare does not publish an item without a title, authors, item type, categories, keywords, a description and a license (**Verify** the exact list on a test account).

## Field Mapping

### Required by Figshare

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `titles[0].title` | `Poster.title` | `title` | Plain string | Proposed |
| `creators[]` | `PosterMetadata.creators` (JSONB) | `authors[]` | See [Authors](#authors) | Proposed |
| (hardcoded) | | `defined_type` | Always `"poster"` (item type 5) | Proposed |
| `subjects[].subject` | `PosterMetadata.subjects` | `keywords[]` | Free text, empty strings dropped. At least one is required. | Proposed |
| `descriptions[]` | `Poster.description` and the submission abstract | `description` | See [Description](#description) | Proposed |
| `rightsList[0].rightsIdentifier` | `PosterMetadata.license` | `license` (integer id) | The license is chosen at the deposit step (step 3). When Figshare is the target, the dropdown lists only the licenses the user's Figshare account offers (`GET /account/licenses`), and the choice is sent by its id. figshare.com offers CC BY 4.0, CC0, MIT, GPL, GPL 2.0+, GPL 3.0+ and Apache 2.0, so CC BY 4.0 or CC0 for a poster; institutions may offer more (the UL Research Institutes portal has CC BY-SA 4.0). | Agreed |
| (none) | | `categories_by_source_id[]` | At least one category is required: an ANZSRC Fields of Research 2020 code on figshare.com, or a code from the institution's own taxonomy. The stored `researchField` (an OpenAlex domain) is too broad to map. | Open question |

### Authors

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `creators[].givenName` | `PosterMetadata.creators` | `authors[].first_name` | Direct | Proposed |
| `creators[].familyName` | `PosterMetadata.creators` | `authors[].last_name` | Direct | Proposed |
| `creators[].name` | `PosterMetadata.creators` | `authors[].name` | Used when there are no name parts (organizational creators) | Proposed |
| `creators[].nameIdentifiers[]` where scheme=ORCID | `PosterMetadata.creators` | `authors[].orcid_id` | Bare id, format and checksum checked as for Zenodo. DataCite gets it as an ORCID name identifier (seen on 34701474). | Proposed |
| `creators[].nameType` | `PosterMetadata.creators` | | No field. Figshare treats every author as a person. | Gap |
| `creators[].affiliation[]` | `PosterMetadata.creators` | | No affiliation field on figshare.com, and DataCite shows `affiliation: []` (34701474, 33990901). Institutional portals add the institution with its ROR id in DataCite (27063577). Kept in poster.json. | Gap |

An author `id` (an existing Figshare account) overrides every other author field, so it is not sent.

### Description

Figshare has one description field, which accepts HTML (not Markdown). DataCite receives it as plain text of type Abstract, with the HTML stripped, so paragraph breaks are lost ("Switzerland.We used ..." on 33990901).

Agreed on 2026-10-23: metadata Figshare has no field for goes into the description as a clean, readable block, so it is still shown on the item and found by Figshare's search. The conference is the first such block.

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `descriptions[0].description` | `Poster.description` | `description` | First paragraph | Proposed |
| `descriptions[]` where descriptionType=Abstract | `PosterMetadata.posterContent.submissionAbstract` | `description` | Under an "Abstract" heading | Proposed |
| (conference) | `PosterMetadata.conference*` | `description` | Under a "Conference" heading. See [Conference](#conference). | Agreed |
| (other fields Figshare cannot hold) | | `description` | Which ones join the conference (affiliations, funder ids, language) | Open question |
| `content.sections[]` | `PosterMetadata.posterContent` (JSONB) | `description` | Could be appended, as proposed for Zenodo | Open question |

### Keywords and Subjects

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `subjects[].subject` | `PosterMetadata.subjects` | `keywords[]` | Free text. Figshare does not pass keywords to DataCite (34701474 has 7 keywords on Figshare, none in DataCite). | Proposed |
| `researchField` | `PosterMetadata.domain` | `categories_by_source_id[]` | See the required fields. Figshare sends the category to DataCite as a subject with scheme "ANZSRC Fields of Research" and its code. | Open question |

### Language

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `language` | `PosterMetadata.language` | | No field. DataCite `language` is null on Figshare posters. | Gap |

### Dates

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| (publish date) | | | Figshare records the posted date itself; DataCite gets Created, Updated and Issued | Automatic |
| `dates[]` where dateType=Presented | `PosterMetadata.dates` | `description` | In the Conference block, with the conference dates. Figshare has no presentation date. | Proposed |
| `dates[]` (other types) | `PosterMetadata.dates` | | The `timeline` dates (first online, publisher publication, publisher acceptance) describe journal publication, not posters. Not sent. | N/A |

### Related Identifiers

Figshare's `related_materials` use DataCite's identifier and relation type names, so most values pass through unchanged.

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `relatedIdentifiers[].relatedIdentifier` | `PosterMetadata.relatedIdentifiers` (JSONB) | `related_materials[].identifier` | Direct | Proposed |
| `relatedIdentifiers[].relatedIdentifierType` | same | `related_materials[].identifier_type` | Same names. Figshare has no RAiD, SWHID or Other: send a URL-shaped value as `URL`, drop the rest. | Proposed |
| `relatedIdentifiers[].relationType` | same | `related_materials[].relation` | Same names. Figshare lacks Collects, IsCollectedBy, HasTranslation, IsTranslationOf and Other: drop those entries and log them. | Proposed |
| `relatedIdentifiers[].resourceTypeGeneral` | same | | No field | Gap |

The platform-added links:

- `IsDescribedBy` the poster's page on posters.science (URL): send.
- `IsNewVersionOf` the previous version's DOI: not needed, since Figshare links the versions of an item itself. Keep it in poster.json.

Whether Figshare passes related materials on to DataCite `relatedIdentifiers` is not known yet (the posters checked had none). **Verify**.

### Identifiers

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| (Figshare-minted DOI) | `PosterMetadata.doi` | `reserve_doi` | Reserved before publishing. After publishing, store the versioned DOI (`...v1`) in `PosterMetadata.doi` and as the first `identifiers` entry. **Verify** whether `reserve_doi` returns the versioned form. | Proposed |
| `identifiers[]` (a DOI from another source, arXiv, PMID, ...) | `PosterMetadata.identifiers` (JSONB) | `related_materials[]` with relation `IsIdenticalTo` | Figshare has no field for other identifiers of the same item | Proposed |

Some institutional portals mint Handles instead of DOIs. Store those with identifierType `Handle`.

### Conference

figshare.com has no conference fields. Institutions can define custom fields, but those differ per portal. Agreed on 2026-10-23: the conference goes in the description under its own heading, so Figshare's search finds posters by conference.

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `conference.conferenceName`, `conferenceAcronym`, `conferenceLocation`, `conferenceStartDate`, `conferenceEndDate`, `conferenceUri` | `PosterMetadata.conference*` | `description`, "Conference" block | A heading, then one line each: "Bioinformatics Open Source Conference (BOSC)", "Washington, DC, USA", "2026-07-14 - 2026-07-16", and the conference URL as a link | Agreed |
| `conference.conferenceUri` | `PosterMetadata.conferenceUri` | `related_materials[]` (identifier_type `URL`, title = conference name) | Optional, in addition to the block. DataCite has no relation type for "presented at". Choose one (`IsPartOf` or `References`). | Open question |
| `conference.conferenceIdentifier`, `conferenceSeries` | | | No field | N/A |

### Funding

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `fundingReferences[]` (funder, award number, award title) | `PosterMetadata.fundingReferences` (JSONB) | `funding_list[]` | If `POST /account/funding/search` finds exactly one grant with the same award number from the same funder, link it by `id`. Otherwise add a free-text entry: `{"title": "Funder name, award number: award title"}`. | Proposed |
| `fundingReferences[].funderIdentifier` | same | | No field (Figshare funding entries carry no ROR or Crossref Funder ID) | Gap |
| `fundingReferences[].awardUri` | same | | No field (a linked grant has its own URL) | Gap |

Figshare did not pass free-text funding to DataCite: `fundingReferences` is empty on 34701474 and 33990901. Whether linked grants reach DataCite is not known. **Verify**.

### Version

| poster_schema.json | DB source | Figshare field | Transform | Status |
|---|---|---|---|---|
| `version` | `PosterMetadata.version` | | Figshare numbers versions itself (1, 2, 3, one per publish) and has no version field. Write Figshare's number back after publishing. A version chosen in the publish form cannot be sent. | Gap |

### Publisher, formats and sizes

| poster_schema.json | Figshare | Status |
|---|---|---|
| `publisher` | Set by Figshare in DataCite: "figshare" on figshare.com, the institution's name on a portal | Automatic |
| `formats[]`, `sizes[]` | Read from the uploaded files (DataCite gets sizes like "3317612 Bytes") | Automatic |

### Content (poster-specific, no Figshare equivalent)

Same as for Zenodo: `content.sections[]`, `tableCaptions[]`, `imageCaptions[]` and `researchField` are kept in the deposited `poster.json` only.

## The deposited poster.json (proposed)

Figshare drops affiliations, funder ids, language, conference details, and (in DataCite) keywords, so the deposited `poster.json` is the only complete copy of the metadata. Upload it with every deposit. Compared with the database it adds:

- `publisher`: on figshare.com `{"name": "figshare", "publisherIdentifier": "https://doi.org/10.17616/R3PK5R", "publisherIdentifierScheme": "DOI", "schemeURI": "https://doi.org"}` (figshare's re3data DOI, r3d100010066). On an institutional portal, the institution's publisher name as Figshare registers it with DataCite, and its re3data DOI if it has one.
- `doi`, `prefix`, `suffix` and `identifiers[0]`: the reserved Figshare DOI (avoiding the gap noted for Zenodo).
- `dates`: a Submitted date, "Submitted to Figshare through Posters.science".
- `types`: `{"resourceType": "Conference Poster", "resourceTypeGeneral": "Poster"}`.

## What indexing Figshare has taught us

Since July 2026 we have indexed Figshare posters into the registry (the "Figshare Auto-Indexing" sheet of the Field Coverage workbook). The same limits apply to what Posters.science can write:

- Figshare authors carry a name and an ORCID only. Affiliations appear only in DataCite, and only for institutional portals.
- Institutional portals register their own publisher name with DataCite (for example "UL Research Institutes"), and some mint Handles instead of DOIs.
- Every publish is a new version with its own DOI (`.v1`, `.v2`), and the DOI without a version points to the latest.
- Funding is mostly free text, often a whole acknowledgement sentence, with no funder ids.
- DataCite spells the figshare.com publisher "figshare", in lowercase.

## Open questions

1. **Categories:** let users pick ANZSRC categories when publishing, or map them from a finer research field than the OpenAlex domain?
2. **Description block:** besides the conference, which fields Figshare cannot hold go into the description (affiliations, funder ids, language), and in what layout?
3. **Conference URL:** also send it as a related material, and with which relation type?
4. **Connection:** Figshare OAuth application or personal token (Zenodo now supports both).
5. **Institutional accounts:** support portals (their licenses, categories, custom fields and curation review) from the start, or figshare.com only?
6. **Test deposit** to check: the form `reserve_doi` returns, the fields required to publish, and whether related materials and linked grants reach DataCite.

Settled on 2026-10-23: the license dropdown shows only the Figshare account's licenses, and the conference goes in the description.
